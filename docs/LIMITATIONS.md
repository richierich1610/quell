# Limitations

Known, deliberate boundaries of what Quell protects against, and of its
blast-radius estimation specifically. These aren't bugs. Each one is
either an inherent property of the approach or a deliberate scope
decision.

## What Quell does not protect against

- **Stored procedures/triggers executing DML internally.** Quell only ever
  sees the SQL text a client actually sends over the wire. A client calling
  `CALL risky_procedure()`, or a trigger that internally runs its own
  `DELETE`/`UPDATE`, is invisible to the analyzer. That inner statement
  never crosses the wire as its own message, so no rule can match it, no
  estimate gets computed for it, and no audit event gets emitted for it.
  Auditing and guarding what a stored procedure or trigger does internally
  is the database's own permission and logging model's job, not something
  a wire-protocol proxy can see into.
- **Direct connections that bypass the proxy entirely.** Quell enforces
  nothing for a client that connects straight to the real database instead
  of through Quell's listener. See docs/THREAT_MODEL.md ("not a network
  security boundary") for the full trust-boundary discussion. Restricting
  network access to the database, so Quell's listener is the only path in,
  is the deployer's responsibility.
- **SCRAM channel binding.** PostgreSQL's `SCRAM-SHA-256` authentication
  supports channel binding (`tls-server-end-point`), which cryptographically
  ties a client's SASL exchange to the specific TLS connection it's
  happening over. It's designed to detect a MITM relay. A TLS-terminating
  proxy is, by construction, exactly the kind of relay channel binding
  exists to catch: the client's TLS session ends at Quell, and Quell's own
  connection to the real database is a second, separate TLS (or plaintext)
  session Quell establishes. Quell relays authentication messages
  transparently instead of implementing SCRAM itself, so a
  client requesting channel binding (`SCRAM-SHA-256-PLUS`) will fail to
  authenticate through the proxy, or, depending on client configuration,
  may silently fall back to non-channel-bound `SCRAM-SHA-256` and lose that
  specific MITM protection between client and proxy. This is an inherent
  property of TLS-terminating proxies generally, nothing specific to
  Quell's implementation.
- **Oracle.** Quell supports PostgreSQL and MySQL/MariaDB only (SQL
  Server/TDS is optional future scope, not started). There's no Oracle
  support and none currently planned.
- **Infrastructure-level destruction that never touches a database
  connection.** A leaked cloud-provider API token used to delete a database
  volume or instance directly never crosses Quell's listener as SQL, so
  there's nothing here to
  analyze, estimate, or block. That's a secrets-management and
  least-privilege-IAM problem, not a query-guarding one. Don't position
  freeze mode or any other Quell control as covering it.

## Freeze mode's cross-process propagation latency

Toggling freeze through `POST /freeze` against a given process's own admin
API takes effect on that process's very next statement. It updates an
in-memory flag synchronously, no filesystem or watcher involved. `quell
freeze on|off` (the CLI) tries that same admin-API call first for exactly
this reason, and reports whether it succeeded.

A separate process reading the same `[policy]` config, most notably `quell
mcp`, which has no admin API of its own today, only learns of a freeze by
noticing the shared marker file change through a filesystem watcher. Found
via live testing: in this development environment, that watcher took
several seconds to notice a marker file a different process had written
(macOS FSEvents specifically; inotify on Linux is typically much faster but
hasn't been independently measured here). A freeze declared while relying
solely on the marker-file path, no admin API reachable, or targeting a
process other than the one the CLI's config points at, should be assumed to
take a few seconds to apply to those other processes, not to be instant.

With `[cluster]` configured, the same latency question applies to every instance in
the fleet, not just `quell mcp`. A freeze reaches the instance whose admin
API actually received the `POST /freeze` instantly, and reaches every
other instance through the shared database: push-based (well under a
second, verified live) on Postgres, polled every second or so on MySQL.
Approvals always poll on both backends, by design. Resolving a hold
created on a different instance takes on the order of a
second to be noticed, not the input-latency timescales freeze mode
targets.

## `[cluster]` does not shard queries across a database cluster

A genuinely sharded database (Citus, Vitess, manually-sharded Postgres or
MySQL) needs a shard-key-aware router to work at all. Quell is not that
router, and deliberately doesn't try to be one. Point `[upstream].addr`
at the cluster's own router, Vitess's `vtgate` or
Citus's coordinator, the same way you'd point it at a single database.
`[cluster]` is solely about coordinating Quell's own state across multiple
Quell instances. It has no idea what's sharded, or how, on the other side
of `[upstream]`.

## `/readyz` does not check that the upstream database is reachable

It reports ready the moment the proxy's own listener starts accepting
connections, and stays that way for the rest of the process's life. It
never attempts to reach `[upstream].addr`, `[estimator].connection`, or
anything else downstream.

This is a deliberate tradeoff, not an oversight: a readiness check that
fails whenever a downstream dependency is briefly unreachable can make an
outage worse, not better, by having an orchestrator pull healthy proxy
capacity out of rotation right when the database is already struggling to
recover. The cost is the flip side of that same tradeoff: if the upstream
is unreachable from the moment the process starts (a typo'd
`[upstream].addr`, a database that isn't actually up yet in a sidecar
deployment where both containers start together), `/readyz` still reports
ready, and a Kubernetes `readinessProbe` wired to it (as in
`deploy/k8s-sidecar-example.yaml`) will route real traffic to a proxy that
fails every single connection. Every failed attempt is still logged
(`connection ended with an I/O error`) and the process itself doesn't
crash or crash-loop, but nothing surfaces the specific "misconfigured at
launch" case as "not ready" the way a first deploy might reasonably expect.

A burst of `connection ended with an I/O error` warnings right after a
rollout, with `/readyz` reporting ready the whole time, is the signature of
this specific gap: check `[upstream].addr` and whether the real database
was actually reachable yet at that moment, not a crash or a bug in Quell
itself.

## The audit file sink never deletes old files itself

`[audit.file]` rotates to a new file daily (`quell-audit.2026-09-26`, and so
on) but nothing in Quell ever removes an old one. Left alone indefinitely,
the audit directory grows forever and will eventually fill the disk.

This is deliberate, not a missing feature: an audit log whose entire
purpose is "nobody has to take an agent's word for what happened" is the
wrong place for Quell to unilaterally decide when a record is old enough
to discard, especially under a compliance regime with its own mandated
retention period. Pick a retention policy and enforce it yourself, with
`logrotate` and a `maxage`, a cron job, or shipping older files to
long-term storage and deleting them from local disk once archived, exactly
as you would for any other audit log this policy applies to. `verify_file`
only ever needs one day's file at a time, so rotating/archiving a past
file doesn't affect the ability to verify any other day's.

## Blast-radius estimation limitations

Read this before treating an `Estimate`'s `rows`/`table_pct` as more
precise than it actually is.

### Estimates are estimates

- **`explain` method** reports the query planner's statistical row-count
  guess (`Plan Rows` from `EXPLAIN (FORMAT JSON)` on Postgres, the
  equivalent field on MySQL), not an exact count. It can be off by a wide
  margin on skewed data, stale statistics, or unusual predicates the
  planner can't estimate selectivity for well.
- **`count` method** (Postgres only) is exact. It runs the real predicate
  as `SELECT count(*)` inside a rolled-back transaction, but costs a real
  table scan (bounded by `count_timeout_ms`) and falls back to `explain` on
  timeout, silently trading exactness for latency at that point.
- MySQL only supports `explain`. There's no documented MySQL `count`
  method equivalent to Postgres's exact count, and this estimator rejects
  that method for MySQL rather than guessing at an undocumented one.

### The side connection cannot see the client's uncommitted rows

Every estimate is computed on the estimator's own dedicated side
connection, not the client's session. This is intentional, specifically
to avoid interfering with the client's own transaction state. One
consequence: if the client's own transaction has
already inserted, updated, or deleted rows it hasn't committed yet, the
estimator's probe runs in a different snapshot and won't see those
changes. A blast-radius estimate for a statement later in the same
uncommitted transaction reflects already-committed data, not the client's
in-flight writes.

### Statistics may be stale

- **Postgres**: `table_pct` is computed from `pg_class.reltuples`, which
  is only refreshed by `ANALYZE` or autovacuum, never by the estimator
  itself. Forcing an `ANALYZE` before every check would be a real,
  potentially expensive write-adjacent side effect, the opposite of what a
  read-only guardrail should do. A table that's brand new, rarely
  analyzed, or has autovacuum disabled or lagging can report `table_pct`
  as `None` or understated relative to its true size. Found via live
  testing: a freshly created and populated 100-row table read back
  `reltuples = 0` until explicitly `ANALYZE`d.
- **MySQL**: `table_pct` is computed from
  `information_schema.TABLES.TABLE_ROWS`, which for InnoDB is itself an
  estimate maintained by the storage engine, not a live count. The same
  staleness caveat applies, one layer earlier.
- **`explain`-method row estimates** on both engines depend on the query
  planner's own statistics, which age the same way as any other planner
  statistics and can drift after bulk loads, deletes, or schema changes
  until the next `ANALYZE` or equivalent runs.

### MySQL `JOIN ... ON` probes over-approximate

Postgres's `DELETE ... USING` / `UPDATE ... FROM` fold their join condition
into the statement's own `WHERE` clause, so the probe built from the
analyzer's parsed `where_expr` reproduces the real semantics.
MySQL's `UPDATE t1 JOIN t2 ON <cond> SET ...` keeps the join condition in a
separate `ON` clause that isn't captured by the analyzer's join tracking,
so a MySQL join probe becomes an unfiltered cross join
between the tables. That's a conservative over-approximation. It only ever
inflates the estimate, never deflates it, and only affects statements that
both join another table and need a blast-radius estimate. Ordinary
single-table `UPDATE`/`DELETE` is unaffected.

### Extended-protocol (bound-parameter) estimates generally fail

Bind-message parameter values aren't extracted from the Postgres extended
query protocol (`Parse`/`Bind`/`Execute`), a deliberate scope decision.
A probe built from a statement like `DELETE FROM orders
WHERE id = $1` has no real value to substitute for `$1`, so `EXPLAIN`ing it
against an unbound placeholder generally fails. Rather than skip
estimation for `Execute`-kind statements outright, Quell still attempts it
and lets the failure flow through the same `on_estimate_failure` path used
for every other estimation error (timeout, connection loss, unparseable
output). An operator running `on_estimate_failure: block` gets fail-closed
behavior here too, not a silent bypass, but shouldn't expect a real
row-count estimate on parameterized statements executed through the
binary/extended protocol, which is what most real drivers use.

### Only top-level statements are estimated

For a batch containing a CTE like `WITH x AS (DELETE FROM orders ...)
SELECT ...`, Quell's rule evaluation (unqualified-write detection,
tautology detection) does recurse into the nested `DELETE`. Blast-radius
estimation does not independently estimate it, though. Estimating every
nested statement would add an unbounded number of extra side-connection
round trips per top-level statement, proportional to how deeply a client
nests CTEs.
