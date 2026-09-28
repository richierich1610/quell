# Getting started

A step-by-step path from nothing installed to a production-shaped
deployment. Each section builds on the one before it and works on its own,
so stop wherever your setup actually needs to stop.

## 1. Prerequisites

- Docker, for a test database to point Quell at in this guide.
- `psql` and/or the `mysql` CLI, for trying statements against the proxy
  by hand.

## 2. Get the binary

Binaries and the container image are gated behind a private repo, not a
public download — see the main [README](../README.md#getting-access) for
how to request access. Once you have it (Linux x86_64, macOS Intel/Apple
Silicon, or Windows x86_64), make it executable:

```bash
chmod +x quell
./quell --version
```

Everything below uses `./quell`; substitute
`docker run ghcr.io/richierich1610/quell ...` if you're running the
container image instead.

## 3. Run it against a test database

```bash
docker run -d --name quell-quickstart-db -p 55432:5432 \
  -e POSTGRES_USER=quell -e POSTGRES_PASSWORD=quell -e POSTGRES_DB=quell_test \
  postgres:16-alpine
./quell --config examples/quell.toml
```

In another terminal:

```bash
psql "host=localhost port=6432 dbname=quell_test user=quell"
# password: quell
```

Try a couple of statements:

```sql
CREATE TABLE orders (id serial primary key, status text);
INSERT INTO orders (status) VALUES ('open'), ('closed');
DELETE FROM orders;
```

That last one gets blocked: `examples/quickstart-policy.yaml` (already
wired up by `examples/quell.toml`) rejects any `UPDATE`/`DELETE` with no
`WHERE` clause. A qualified one goes through normally:

```sql
DELETE FROM orders WHERE id = 1;
```

Check the admin API while the proxy's running:

```bash
curl localhost:9091/events
```

You should see the blocked `DELETE` and the successful one, each with the
rule that matched (or didn't).

## 4. Write your own policy

Copy the example and start from something closer to what you actually
need:

```bash
cp examples/policy.yaml my-policy.yaml
```

A minimal rule blocking unqualified writes on a specific table:

```yaml
version: 1
rules:
  - id: no-unqualified-orders-writes
    match: { statements: [update, delete], tables: [orders] }
    when: { where: missing_or_tautology }
    action: block
    message: "orders needs a WHERE clause. Add one or use quell:allow with a ticket."
```

Validate it before pointing anything at it:

```bash
./quell policy check my-policy.yaml
```

Try a specific statement against it without touching a database:

```bash
./quell policy test my-policy.yaml \
  --sql "DELETE FROM orders" --user alice --db quell_test
```

Point `quell.toml`'s `[policy].path` at it and Quell picks up further
edits automatically, no restart needed. Full rule syntax, scopes,
overrides, and time windows: [`docs/POLICY.md`](POLICY.md).

## 5. Turn on blast-radius estimation

So a policy can say "block anything that would touch more than N rows,"
not just "block anything with no `WHERE` clause":

```toml
[estimator]
connection = "host=127.0.0.1 port=55432 user=quell password=quell dbname=quell_test"
```

```yaml
rules:
  - id: large-blast-radius
    match: { statements: [update, delete] }
    when: { rows_gt: 10000, or_table_pct_gt: 20 }
    action: hold
```

This connection is separate from the client's own session on purpose (see
[`docs/THREAT_MODEL.md`](THREAT_MODEL.md)): it needs read access to the
tables it probes, nothing more.

## 6. Turn on audit logging

```toml
[audit]
stdout = true

[audit.file]
directory = "/var/log/quell"
```

Hash-chaining is on by default, so tampering with a past record is
detectable:

```bash
./quell audit verify /var/log/quell/quell-audit.2026-09-26
```

## 7. Turn on hold and approve

Change a rule's `action` to `hold` and, optionally, get notified in Slack
or Teams when one fires:

```toml
[approvals]
webhook_url = "https://hooks.slack.com/services/..."
```

```bash
curl localhost:9091/approvals
curl -X POST "localhost:9091/approvals/<id>/approve?by=alice"
```

The held connection stays open until someone resolves it, or until
`defaults.hold_timeout_secs` in the policy file elapses, which always
denies rather than silently letting the statement through.

## 8. Turn on freeze mode

No config needed. Use it during an incident, or to test that it works:

```bash
./quell --config quell.toml freeze on
# every write is now blocked, on every listener reading this config
./quell --config quell.toml freeze status
./quell --config quell.toml freeze off
```

## 9. Turn on MCP mode, for AI agents

Add an `[mcp]` section (can live in the same `quell.toml` as
`[listener]`/`[upstream]`, or its own file) and run it as a separate
process:

```toml
[mcp]
connection = "host=127.0.0.1 port=55432 user=quell password=quell dbname=quell_test"
protocol = "postgres"
```

```bash
./quell mcp --config quell.toml
```

Point an MCP-speaking agent host at this process over stdio. It gets
`run_query`, `explain_query`, and `list_tables`, enforced by the exact
same policy the wire-protocol proxy uses.

## 10. Run more than one instance

If you're scaling out (several replicas behind a load balancer, or
one-per-pod sidecars), add `[cluster]` so freeze status, pending
approvals, and open sessions are shared across all of them instead of
stuck on whichever instance created them:

```toml
[cluster]
connection = "host=127.0.0.1 port=55432 user=quell password=quell dbname=quell_test"
protocol = "postgres"
```

Point every instance at the same `[cluster].connection` and they'll
coordinate automatically. See [`docs/FEATURES.md`](FEATURES.md) for what
this does and does not cover (in particular: it doesn't shard queries
across a sharded database cluster; see [`docs/LIMITATIONS.md`](LIMITATIONS.md)).

## 11. Enable TLS for clients

```toml
[tls]
cert = "/etc/quell/cert.pem"
key = "/etc/quell/key.pem"
```

Works for both `protocol = "postgres"` and `protocol = "mysql"` listeners.
Clients that don't request TLS still connect in plaintext unless a policy
or network control says otherwise.

## 12. Lock down the admin API

The admin API binds to `127.0.0.1` by default. If you widen it, protect it:

```toml
[admin]
bind = "0.0.0.0:9091"
token = "${ADMIN_BEARER_TOKEN}"
```

Every route except `/healthz`/`/readyz` then requires
`Authorization: Bearer <token>`.

## 13. Deploy it

- **Docker**: `ghcr.io/richierich1610/quell` is a distroless, non-root
  image, published alongside every release. Bind-mount or ConfigMap-mount
  `quell.toml` and `policy.yaml` into `/etc/quell`.
- **Kubernetes**: [`deploy/k8s-sidecar-example.yaml`](../deploy/k8s-sidecar-example.yaml)
  shows Quell as a sidecar in front of an app's own database connection.
- **systemd**: [`deploy/quell.service`](../deploy/quell.service) is a
  hardened unit file (least-privilege, `NoNewPrivileges`, a read-only
  `/etc/quell`).

## 14. Before you trust it with something real

- [`docs/THREAT_MODEL.md`](THREAT_MODEL.md): what Quell defends against,
  and what it explicitly doesn't (it's not a network security boundary,
  not an access-control system, not a replacement for least-privilege
  database roles).
- [`docs/LIMITATIONS.md`](LIMITATIONS.md): the honest edges of blast-radius
  estimation and freeze mode's propagation latency.
- No third-party security audit has been done yet. Said here plainly, not
  left for you to discover later.
