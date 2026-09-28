# Quell

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**[richierich1610.github.io/quell](https://richierich1610.github.io/quell/)**

Quell sits between whoever's talking to your database (a person, an app, or
an AI agent) and the database itself. It reads every statement before the
database ever sees it and stops the ones that would do real damage. Not just
an `UPDATE`/`DELETE` with no `WHERE` clause. A *perfectly valid* one that
would still touch 40% of a table gets stopped too. Quell estimates the actual
blast radius, how many rows a statement will really touch, before deciding,
instead of just pattern-matching the SQL text.

It exists because of incidents like [the one at Replit in July 2025](https://incidentdatabase.ai/cite/1152/).
An AI coding agent ran destructive commands against a live production
database during an active code freeze, after being told in plain English not
to touch anything. It then told the operator the deletion couldn't be rolled
back. That wasn't true. "Don't touch prod" is a sentence, and sentences
aren't enforcement. Freeze mode locks down every write instantly no matter
what an agent decides, and the audit log is hash-chained so nobody has to
take an agent's word for what happened. That's what enforcement looks like
instead.

> **Status:** production-shaped, not production-proven. PostgreSQL and MySQL
> proxies, the blast-radius estimator, audit sinks and the admin API, hold &
> approve, and an MCP server for AI agents are all implemented and tested.
> SQL Server/TDS support is optional future scope, not started, not a gap.

### 📖 [Getting started guide](docs/GETTING_STARTED.md)

The five-minute demo below gets you a blocked `DELETE` fast. The
[full getting started guide](docs/GETTING_STARTED.md) is the actual
onboarding path: prerequisites, writing your own policy, blast-radius
estimation, freeze mode, the audit log, hold & approve, MCP mode, running a
fleet, TLS, and deploying it for real. Start there if you're setting this up
for anything beyond a demo.

### ⚙️ [Configuration reference](docs/CONFIGURATION.md)

Every `quell.toml` section and property: type, required or optional,
default, and what it actually does. Unknown fields are a load error, not
a silent no-op, so if `quell` starts without complaint, every property in
your file is one it actually reads.

## Supported databases

Quell speaks the real wire protocol, not a vendor SDK, so it works with
anything that speaks real Postgres or real MySQL, not just one hosting
provider.

| Engine | Versions tested in CI | Auth methods | Blast-radius estimation |
|---|---|---|---|
| PostgreSQL | 16, 17 | any `pg_hba.conf`-permitted method (Quell doesn't intercept auth) | `EXPLAIN` (planner estimate) or an exact `COUNT` in a rolled-back transaction |
| MySQL | 8.4 | `mysql_native_password`, `caching_sha2_password` | `EXPLAIN` (planner estimate only) |
| MariaDB | 11 | same MySQL wire protocol as above | `EXPLAIN` (planner estimate only) |

Managed Postgres/MySQL (Amazon RDS/Aurora, Google Cloud SQL, Azure
Database) and self-hosted instances all work the same way, since Quell
proxies the actual protocol traffic. `[cluster]` mode (coordinating a fleet
of Quell instances) has been tested with Postgres and MySQL as the
coordination backend, independent of which database Quell is guarding.

**Not supported**: SQL Server / TDS (optional future scope, genuinely
optional and not started, not a gap), Oracle, and other wire protocols.
Exact per-engine caveats (join-probe approximation, bound parameters,
table-size sourcing): [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md).

## Runs with no internet access at all

Quell makes no outbound connection you didn't configure: the database
it's proxying, and only if you explicitly set them, a Slack/Teams
approval webhook or an audit webhook sink. No telemetry, no update
check, no license/activation check, no phone-home of any kind. Verified
by actually running the compiled binary, start to finish, inside a
Docker network with a confirmed zero route to the internet. The complete
list of every connection Quell ever makes is in
[`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md)'s "Network dependencies"
section.

## Getting access

Binaries and the container image aren't public downloads: everything on
this page (the docs, the source of the pitch above) is public, but
running the software itself is by invitation. That's not enforced by
Quell: nothing at runtime checks a license or calls home (see above).
It's controlled entirely by who has access to the private repo the
binaries are published to.

Email richhpalgora1610@gmail.com or [open an issue](https://github.com/richierich1610/quell/issues)
describing what you're evaluating Quell for, and you'll get an invite.
Once you have access, downloads work exactly like any other GitHub
release or container pull, just authenticated.

## Try it in five minutes

```bash
# 1. Get the quell binary for your platform (see "Getting access" above)
chmod +x quell

# 2. Bring up a test Postgres
docker run -d --name quell-quickstart-db -p 55432:5432 \
  -e POSTGRES_USER=quell -e POSTGRES_PASSWORD=quell -e POSTGRES_DB=quell_test \
  postgres:16-alpine

# 3. Run Quell pointed at it
./quell --config examples/quell.toml

# 4. In another terminal, connect through the proxy instead of directly to Postgres
psql "host=localhost port=6432 dbname=quell_test user=quell"
# (password: quell)

# 5. Try something dangerous — it gets blocked, not executed
quell_test=> CREATE TABLE some_table (id serial primary key, val text);
quell_test=> INSERT INTO some_table (val) VALUES ('a'), ('b');
quell_test=> DELETE FROM some_table;
ERROR:  UPDATE/DELETE without a selective WHERE clause is blocked.
DETAIL:  matched rule(s): no-unqualified-writes

# A qualified DELETE goes through exactly like it would without Quell:
quell_test=> DELETE FROM some_table WHERE id = 1;
DELETE 1
```

That policy lives in [`examples/quickstart-policy.yaml`](examples/quickstart-policy.yaml).
It's small and deterministic on purpose, so the demo behaves the same no
matter when you run it. For what a real policy looks like, with blast-radius
thresholds, scopes, overrides, and a scheduled freeze window, see
[`examples/policy.yaml`](examples/policy.yaml) and
[`docs/POLICY.md`](docs/POLICY.md).

While the proxy is running, its admin API sits on `http://localhost:9091`.
Try `curl localhost:9091/events` right after the blocked `DELETE` above and
you'll see exactly what got logged.

A container image is published alongside every release too, tagged both
`:latest` and per-version (e.g. `:v0.2.0`) — see "Getting access" below
for both.

## How a statement actually gets from client to database

```
clients ──► [ listener (TLS optional) ] ──► [ protocol frontend: pg | mysql | mcp ]
                                                   │ extracts SQL + session ctx
                                                   ▼
                                          [ analyzer (SQL → AST) ]
                                                   ▼
                                          [ policy engine ] ◄── policy.yaml (hot reload)
                                                   │ needs an estimate?
                                                   ▼
                                          [ estimator ] ──► side connection → DB (EXPLAIN)
                                                   ▼
                              decision: Allow | Warn | Block | Hold (approval)
                                                   ▼
                          forward to upstream  OR  hand the client a real error
                                                   ▼
                            [ audit: stdout/file/webhook sinks ]
     [ admin HTTP API: /healthz /readyz /metrics /policy /events /sessions /approvals ]
```

A `Hold` decision doesn't just fail the statement. It registers a pending
approval (optionally pinging a Slack or Teams webhook), leaves the client
waiting, and resolves through `POST /approvals/{id}/approve|deny` or a
timeout that defaults to denying it. Nobody has to babysit a queue for this
to be safe by default.

## Three ways to run it

- **No subcommand** starts the wire-protocol proxy itself. Set
  `[listener]`/`[upstream]` in `quell.toml` and it speaks Postgres or MySQL
  depending on `listener.protocol`.
- **`quell mcp`** runs an MCP server over stdio for AI agents (configure
  `[mcp]` instead). It exposes `run_query`, `explain_query`, and
  `list_tables` through the same analyzer, policy, and estimator pipeline
  the wire proxy uses. An agent doesn't get a separate, weaker set of
  guardrails.
- **`quell policy check`** / **`quell policy test`** validate and dry-run a
  policy file offline. No database, no proxy, just "would this statement
  have been blocked."

One `quell.toml` can configure any combination of these. Each subcommand
just checks that its own section is actually there before it starts.

## Running it day to day

**Keeping secrets out of the config file.** Nothing in `quell.toml` has to
be plaintext. Write `${VAR_NAME}` anywhere in the file, including
mid-string, and it gets replaced with that environment variable before the
file is parsed:

```toml
[estimator]
connection = "postgres://appuser:${ESTIMATOR_DB_PASSWORD}@host:5432/db"

[admin]
token = "${ADMIN_BEARER_TOKEN}"
```

If a variable you reference isn't set, Quell refuses to start and tells you
which one. It won't quietly boot with half a connection string. One honest
caveat: this substitution runs before TOML parsing, so a secret containing a
literal `"` or `\` can break the surrounding string.

**When something's actually wrong: freeze mode.** This is the direct answer
to the Replit incident above. It's a lockdown that blocks every write
immediately, no matter what any policy rule, scope, or in-SQL
`quell:allow` override would otherwise permit. Reads keep working, so you
(or an agent) can still look around while it's on.

```bash
quell freeze on      # every write blocked immediately; reads still work
quell freeze status
quell freeze off
```

It reaches every listener and any separately-running `quell mcp` process
reading the same config, through a shared marker file. `on`/`off` also try
the running server's admin API first, since that applies instantly, and
tell you plainly if that didn't work and it's falling back to the slower
file-watcher path instead. See [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md)
for that latency spelled out honestly.

**Knowing the audit log wasn't tampered with.** Turn on `[audit.file]` and
every record gets hash-chained by default. Editing, deleting, or reordering
a past entry becomes something you can actually catch instead of something
you just have to hope didn't happen:

```bash
quell audit verify /var/log/quell/quell-audit.2026-09-26
```

**Running more than one instance.** A single process is all most
deployments need. If you're running a fleet, several replicas behind a load
balancer, or one-per-pod sidecars, add `[cluster]`. Freeze status, pending
approvals, and open sessions all become fleet-wide instead of stuck on
whichever instance happened to create them:

```toml
[cluster]
connection = "postgres://appuser:${CLUSTER_DB_PASSWORD}@host:5432/db"
protocol = "postgres"
```

This reuses the database itself as the coordination backend: a handful of
small `quell_*` tables it creates and owns, not a new dependency to run.
Postgres gets near-instant push-based freeze propagation. MySQL polls every
second or so. A genuinely *sharded* database (Citus, Vitess) needs its own
router. Point `[upstream].addr` at that router the same way you'd point it
at a single database. `[cluster]` coordinates Quell's own state across
instances; it doesn't shard queries for you. See
[`docs/LIMITATIONS.md`](docs/LIMITATIONS.md) for the latency specifics.

## Read more

- [`docs/GETTING_STARTED.md`](docs/GETTING_STARTED.md): a step-by-step path
  from nothing installed to a production-shaped deployment
- [`docs/FEATURES.md`](docs/FEATURES.md): everything Quell does, in one
  place, with why each feature exists
- [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md): what Quell doesn't protect
  against, stated plainly
- [`docs/CONFIGURATION.md`](docs/CONFIGURATION.md): every `quell.toml`
  property, in full
- [`docs/POLICY.md`](docs/POLICY.md): the policy file format, in full
- [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md): what's in scope, what
  isn't, and where the trust boundaries actually sit
- [`docs/BENCHMARKS.md`](docs/BENCHMARKS.md): real, reproducible
  performance numbers
- [`SECURITY.md`](SECURITY.md): how to report a bypass or vulnerability
  privately
- [`CHANGELOG.md`](CHANGELOG.md): what changed in each release

## Deploying it

- **Binary**: available for Linux x86_64, macOS (Intel and Apple Silicon),
  and Windows x86_64. See "Getting access" below.
- **Docker**: a distroless, non-root image, published alongside every
  release. Bind-mount or ConfigMap-mount `quell.toml` and `policy.yaml`
  into `/etc/quell`.
- **Kubernetes**: [`deploy/k8s-sidecar-example.yaml`](deploy/k8s-sidecar-example.yaml)
  shows Quell as a sidecar in front of an app's own database connection.
- **systemd**: [`deploy/quell.service`](deploy/quell.service) is a
  hardened unit file (least-privilege, `NoNewPrivileges`, a read-only
  `/etc/quell`).

## Where this stands

Source isn't public. This repository carries documentation and downloadable
releases only, kept accurate against the real, tested behavior of the
software it describes, not a marketing description of it. No third-party
security audit has been done yet, and no production users beyond the
testing that built it. That's worth knowing before you point this at
anything that matters. Found a way around a block, freeze mode, or the
admin API's auth? See [`SECURITY.md`](SECURITY.md) for how to report it
privately. Other issues and reports of the shape "here's an incident this
wouldn't have caught" are genuinely welcome on this repo's issue tracker.
