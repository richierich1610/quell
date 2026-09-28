# Threat model

Quell sits between database clients and a real Postgres/MySQL server. This
document states what it defends against, what it deliberately doesn't, and
where the trust boundaries are, so a deployer can judge whether it fits
their actual threat instead of just reading the feature list.

## What Quell is

A **guardrail against mistakes and over-broad automation**, not a network
security boundary. Its job is to catch a human or an AI agent about to run
`DELETE FROM orders` with no `WHERE` clause, or an `UPDATE` that would touch
40% of a table, before the database ever sees it, and to produce an audit
trail of every decision. It sits in the trust boundary between
already-authenticated clients and the database. It does not sit between the
internet and the database.

## Assets being protected

- **Data integrity**: rows in the upstream database, from being
  bulk-modified or deleted by a mistaken or overly broad statement.
- **Availability**: the upstream database, from a runaway statement. Blast-
  radius blocking is a side effect of protecting this, though, not Quell's
  primary target. See "Not a substitute for database-level protections"
  below.
- **Auditability**: a record of every write decision (allowed, blocked,
  warned, overridden), for after-the-fact investigation.

## Adversaries and misuse cases actually addressed

- **A well-meaning human or script** running an unqualified
  `DELETE`/`UPDATE`, a tautological `WHERE 1=1`, or a statement that would
  touch far more rows than intended. This is the primary case. Most
  incidents this tool is built for are mistakes, not attacks.
- **An AI agent** given database access and a loosely-scoped task,
  generating a technically valid but catastrophic statement. Quell's block
  messages are written so an agent can self-correct rather than assuming
  a human is in the loop to react to an alert.
- **An agent that ignores or misreads an explicit human instruction not to
  act.** This is the actual shape of the July 2025 Replit incident
  (incidentdatabase.ai/cite/1152): an agent under a stated "code freeze" ran
  destructive commands against a live database anyway, then told the
  operator the deletion couldn't be rolled back. It could. Freeze mode
  is Quell's direct answer to this: a DBA-declared lockdown enforced by
  the proxy itself, independent
  of policy rules, scope exemptions, or any override in the SQL text. It's
  not a sentence the agent has to correctly obey. The hash-chained audit
  log (same section) is the other half of the answer: an independently
  verifiable record of what actually ran, so an agent's own account of an
  incident isn't the only version of events available afterward.
- **A client attempting to bypass detection via SQL surface tricks.** The
  analyzer is specifically hardened against burying a write inside a CTE
  (`WITH x AS (DELETE ... ) SELECT * FROM x`), SQL-level `PREPARE`/`EXECUTE`
  indirection, comments or string literals containing text that looks like
  SQL keywords, and multi-statement batches where only one statement is
  dangerous. Specific bypasses like these were found and fixed during
  development, not just reasoned about.

## Explicitly out of scope (trust boundaries)

- **Network access to the listener.** Quell does not authenticate or
  authorize connections. It relays whatever auth the upstream database
  itself performs, passing authentication messages through transparently.
  Anyone who could already open a TCP connection to the
  real database and pass its own auth can do the same through Quell.
  Restricting network access to Quell's listener (firewall rules, security
  groups, a private network) is the deployer's responsibility, the same as
  it would be for the database itself.
- **A malicious, credentialed client determined to cause damage.** The
  analyzer and policy engine assume adversarial SQL shapes (the
  obfuscation attempts above), not an adversarial client identity. A user
  with legitimate `DELETE` privileges who deliberately crafts a qualified,
  low-row-count statement to still cause harm, many small deletes instead
  of one large one, say, isn't something row-count or table-percentage
  thresholds can catch. That's an access-control problem, not a
  blast-radius problem. Quell is a guardrail, not a replacement for
  least-privilege database roles.
- **The estimator's side connection.** The blast-radius estimator connects
  to the database with its own credentials, configured in `quell.toml`'s
  `[estimator]` section, deliberately never the client's own connection.
  Those credentials need at least read access to the tables being probed
  and to `pg_class`/`information_schema` for table sizing. They're a
  second credential to secure: file permissions on `quell.toml`, or
  environment-variable injection via the process supervisor. Quell doesn't
  do its own secret management.
- **The `[cluster]` control-plane connection**, when configured, is a
  third credential in the same category as the estimator's. It needs
  write access to a handful of `quell_*` tables it
  creates and owns itself. Anyone who can write to those tables directly,
  not through Quell, can declare a freeze, forge or resolve approvals, or
  inject fake session rows into `GET /sessions`. That's the same trust
  level already implied by holding valid database credentials at all, not
  a new privilege boundary. Still worth naming: this connection's blast
  radius is every Quell instance sharing it, not just the one process
  holding it.
- **MySQL client TLS** is terminated the same way Postgres's is.
  Configure `[tls]` in `quell.toml` and it applies to whichever protocol
  the listener speaks. A client that
  doesn't request TLS still connects in plaintext (mixed mode, matching
  real MySQL server behavior). `[tls]` isn't a "require encryption" switch
  on its own, so pair it with a policy or network control if plaintext
  connections must be rejected outright. As with Postgres, this is
  client-facing termination only. The proxy's own connection to the real
  upstream database is always plaintext, or TLS if the upstream database
  itself requires it at the network layer, independent of this proxy. A
  compromised network path between Quell and the real database isn't
  covered by this.
- **Denial of service via connection/statement flooding.** Quell adds
  per-statement analysis, policy evaluation, and (for `UPDATE`/`DELETE`) a
  blast-radius probe on its own side connection. All of that is
  bounded-latency work, but none of it is rate-limited. A client that can
  already open enough connections to the real database to cause problems
  can do the same through the proxy. Quell adds latency and a second
  connection's worth of resource use per client connection, not new DoS
  surface beyond what direct database access already has.
- **Not a substitute for database-level protections.** Backups,
  point-in-time recovery, replication, and database-level permissions
  remain necessary. A blast-radius estimate is exactly that, an estimate
  (see docs/LIMITATIONS.md), computed against possibly stale statistics on
  a side connection that can't see the client's own uncommitted state.

## Admin API attack surface

The admin API binds to `127.0.0.1` by default specifically because it
exposes operationally sensitive data and actions. `GET /policy`
reveals the active policy, which tables and rules are being enforced,
useful reconnaissance for an attacker probing what's guarded. `GET /events`
and `GET /sessions` reveal recent SQL activity (fingerprints always, full
SQL text if `audit.log_full_sql: true`, see redaction below) and connected
users and databases. `POST /policy/reload` forces an immediate re-read of
the policy file. Binding the admin API to a non-loopback address requires
deliberately widening `[admin].bind` in `quell.toml`, and should be paired
with `[admin].token` (bearer-token auth, gating every route except
`/healthz`/`/readyz`, which orchestrators need to probe without
credentials) and network-level access control. It has no built-in rate
limiting or per-endpoint authorization beyond the single shared token.

## Audit data sensitivity

`fingerprint` (literals stripped, e.g. `DELETE FROM orders WHERE id = ?`)
is always recorded and considered safe to log or export freely. Raw `sql`
is only included when `audit.log_full_sql: true`, off by default, since
literal values in a `WHERE`/`SET` clause can be PII or otherwise sensitive.
This flag governs every configured sink (stdout, file, webhook) uniformly,
not per sink (see `quell-audit::AuditPipeline::emit`). A webhook sink
transmits whatever's in the event, including full SQL if enabled, to a
third-party URL over HTTPS. Treat that URL and its receiving system as
being inside the same trust boundary as the audit data itself.

## Failure-mode posture

- **`on_parse_error` / `on_estimate_failure` default to `block`.**
  An analyzer or estimator failure fails closed, not open. A
  client whose statement can't be parsed, or whose blast-radius estimate
  can't be computed (side connection down, timeout, unparseable `EXPLAIN`
  output), is blocked by default rather than silently let through.
- **Audit sinks fail open on overload, by design.** A full sink queue
  drops the new event and counts it, a bounded queue that never blocks
  the data path, rather than blocking the statement itself. An
  audit sink being slow or down must never become a way to stall or deny
  legitimate database traffic. This is a deliberate availability-over-
  completeness tradeoff for the audit trail specifically, distinct from
  the policy engine's own fail-closed default above.
- **A crashed Quell process is a closed door, not an open one.** No
  listener means no traffic passes, at least through the proxy path. See
  "not a network security boundary" above for what that does and doesn't
  mean for direct database access.
