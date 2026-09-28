# Features

Everything Quell actually does, in one place. Each section says what the
feature is, why it exists, and where to go for the full detail.

## Blast-radius estimation

Most guardrail tools block on the shape of a query: a missing `WHERE`
clause, a keyword match. Quell does that too, but its real differentiator
is estimating the actual impact of a statement before deciding. A
perfectly valid `DELETE ... WHERE region = 'us-east'` can still wipe out
40% of a table, and a shape-based check would wave it through.

Quell computes a real row-count estimate on a dedicated side connection,
never the client's own session, using either the query planner's
statistics (`EXPLAIN`) or, on Postgres, an exact count run inside a
rolled-back transaction. A policy rule can then say "hold anything that
would touch more than 10,000 rows or 5% of the table" and mean it
literally, not approximately.

- Enable it: `[estimator]` in `quell.toml`, then `rows_gt`/`or_table_pct_gt`
  in a rule's `when` block.
- Full reference: [`docs/POLICY.md`](POLICY.md)
- What it doesn't cover: [`docs/LIMITATIONS.md`](LIMITATIONS.md)'s "Blast-
  radius estimation limitations" section, in particular that MySQL only
  supports the planner-estimate method, and that estimates are skipped for
  bound parameters in the Postgres extended protocol.

## The policy engine

A YAML file (`policy.yaml`) that Quell hot-reloads without dropping
connections. Rules match on statement kind, database, schema, and table
(all glob-based), and on conditions like a missing `WHERE` clause, a
tautological one (`WHERE 1=1`), a blast-radius threshold, or a scheduled
time window (cron syntax). Each rule resolves to `allow`, `warn`, `hold`,
or `block`.

Scopes let you exempt specific users, IP ranges, databases, or application
names from specific rules, with a documented precedence order when more
than one scope matches. Overrides let a statement carry a
`/* quell:allow reason="..." ticket="..." */` comment that bypasses a
block, but only for an allow-listed set of users and only when required
fields are present, and every attempt (successful or not) is recorded for
audit.

A `monitor` mode exists for rolling out a new policy safely: every
decision that would have blocked or held instead gets logged as a warning,
so you can see what a policy *would* do before it actually does it.

- Full reference: [`docs/POLICY.md`](POLICY.md)
- Validate a policy file offline: `quell policy check policy.yaml`
- Try a rule against a synthetic statement, no database needed: `quell
  policy test policy.yaml --sql "..." --user alice --db prod`

## Hold and approve

When a rule's action is `hold`, the statement doesn't just fail. The
client connection is kept waiting, a pending approval is registered,
and an optional Slack or Teams webhook
fires. An operator resolves it with `POST /approvals/{id}/approve` or
`/deny` through the admin API, or it times out and defaults to denying,
never to silently allowing.

- Configure the webhook: `[approvals]` in `quell.toml`
- See what's pending: `GET /approvals`
- Works across a fleet of instances too, see "Running a fleet" below.

## Freeze mode

The direct answer to a real incident: an AI coding agent at Replit ran
destructive commands during an active, human-declared code freeze, because
"don't touch prod" was a sentence, not an enforced rule (see the README
for the full story). Freeze mode is a hard write lockdown that sits
outside the policy engine entirely. No rule, scope, or in-SQL override can
get around it, and it blocks every write instantly while leaving reads
untouched, so you or an agent can still look around while it's on.

```bash
quell freeze on
quell freeze status
quell freeze off
```

Reaches every listener and any separately running `quell mcp` process
through a shared marker file (or the shared database, if `[cluster]` is
configured). The CLI tries the running server's admin API first for
instant effect, and tells you plainly if it had to fall back to the
slower file-watcher path instead.

- The honest latency caveats: [`docs/LIMITATIONS.md`](LIMITATIONS.md)

## Tamper-evident audit log

Every decision Quell makes gets logged: what ran, who ran it, which rule
matched, what happened. With `[audit.file]` configured, each record in the
file gets hash-chained by default: every entry's hash covers the one
before it, so editing, deleting, or reordering a past line becomes
detectable instead of just hopeful.

```bash
quell audit verify /var/log/quell/quell-audit.2026-09-26
```

Additional sinks (stdout, a webhook) can run alongside the file sink. Raw
SQL text is redacted by default; only the literal-stripped fingerprint
(`DELETE FROM orders WHERE id = ?`) is always recorded, since a `WHERE`
clause can carry PII.

- Configure sinks: `[audit]` in `quell.toml`
- Full reference: [`docs/THREAT_MODEL.md`](THREAT_MODEL.md)'s "Audit data
  sensitivity" section

## Credentials that don't live in the config file

`quell.toml` never needs a plaintext secret. Any `${VAR_NAME}` in the
file, including mid-string inside a connection URL, gets replaced with
that environment variable's value before the file is parsed. A referenced
variable that isn't set is a startup error naming exactly which one, not
a silently broken connection string.

```toml
[estimator]
connection = "postgres://appuser:${ESTIMATOR_DB_PASSWORD}@host:5432/db"
```

- The one honest caveat: a secret containing `"` or `\` can break the
  surrounding string, since substitution happens before the config file
  is parsed.

## MCP mode: the same guardrails for AI agents

`quell mcp` runs an MCP server over stdio, exposing `run_query`,
`explain_query`, and `list_tables` to any MCP-speaking agent host. It goes
through the exact same analyzer, policy, and estimator pipeline a `psql`
connection does. An agent doesn't get a separate, weaker set of rules, and
block messages are phrased so an agent can actually act on them, e.g. "add
a WHERE clause on a key column; estimated 48,213 rows would be affected,
limit is 1,000".

- Configure it: `[mcp]` in `quell.toml`
- Runs alongside, not instead of, the wire-protocol proxy: both can read
  the same `quell.toml`.

## Running a fleet of instances

A single process is enough for most deployments. If you're running
several replicas behind a load balancer, or one-per-pod sidecars, add
`[cluster]` and freeze status, pending approvals, and open sessions all
become fleet-wide, instead of stuck on whichever instance happened to
create them. A hold created through one instance's proxy connection can be
approved through a different instance's admin API entirely, with the
original connection actually completing once approved.

```toml
[cluster]
connection = "postgres://appuser:${CLUSTER_DB_PASSWORD}@host:5432/db"
protocol = "postgres"
```

This reuses the database itself as the coordination backend: a handful of
small tables Quell creates and owns, not a new piece of infrastructure to
run. Postgres gets near-instant push-based freeze propagation through
`LISTEN`/`NOTIFY`; MySQL polls every second or so. A genuinely sharded
database (Citus, Vitess) needs its own router; point `[upstream].addr` at
that router the same way you'd point it at a single database. `[cluster]`
coordinates Quell's own state across instances, it doesn't shard queries.

- Latency specifics: [`docs/LIMITATIONS.md`](LIMITATIONS.md)

## Client-facing TLS

Both the Postgres and MySQL listeners can terminate client TLS with the
same `[tls]` config section. A client that doesn't request TLS still
connects in plaintext (mixed mode, matching how a real database server
behaves) unless you pair it with a policy or network control that
requires it. The proxy's own connection to the real upstream database is
always plaintext, or TLS if the upstream itself requires it independently
of Quell.

- Configure it: `[tls]` in `quell.toml` (`cert`, `key`)

## Admin API and metrics

A small HTTP API, bound to `127.0.0.1` by default, that every mode above
runs alongside: `/healthz`, `/readyz`, `/metrics` (Prometheus text
format), `/policy` and `/policy/reload`, `/events`, `/sessions`,
`/approvals`, and `/freeze`. An optional bearer token gates every route
except the two health checks, which orchestrators need to probe without
credentials.

- What each route exposes and why it's loopback-only by default:
  [`docs/THREAT_MODEL.md`](THREAT_MODEL.md)'s "Admin API attack surface"

## Offline policy tooling

Two commands that need no database or running proxy at all:

```bash
quell policy check policy.yaml
quell policy test policy.yaml --sql "DELETE FROM orders" --user alice --db prod
```

Useful for CI: validate a policy change before it ever reaches a real
deployment, or check that a specific statement would (or wouldn't) get
blocked under a proposed rule.
