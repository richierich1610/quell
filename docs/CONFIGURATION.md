# `quell.toml` configuration reference

`quell.toml` is the main config file, passed via `--config` to every `quell`
subcommand that needs one (`serve`, `mcp`, `freeze`). It's TOML. This
document lists every section and every property inside it: its type,
whether it's required, its default when omitted, and what it actually
does. `policy.yaml` (the separate file referenced by `[policy].path`) has
its own reference: [`docs/POLICY.md`](POLICY.md).

Every struct this file parses into rejects unknown fields. A typo'd
property name, or a misspelled section header, is a load error naming the
bad field, e.g. `unknown field "bnid", expected "bind" or "protocol"`, not
something silently accepted and quietly doing nothing. If `quell` starts
without complaint, every property in your file is one it actually reads.

Any string value anywhere in this file can contain `${VAR_NAME}`, replaced
with that environment variable's value before parsing. It can be the whole
value or part of one (`connection = "mysql://user:${DB_PASSWORD}@host/db"`).
A referenced variable that isn't set is a load error, not a blank
substitution. This is how a password, admin token, or webhook URL stays out
of the file entirely; see "Secrets" at the bottom.

## Which sections a given command needs

| Command | Needs |
|---|---|
| `quell serve` (or no subcommand) | `[listener]`, `[upstream]`, `[policy]` |
| `quell mcp` | `[mcp]`, `[policy]` |
| `quell freeze on\|off` | `[policy]` (and `[admin]` if the admin API isn't at its default bind) |
| `quell policy check\|test` | none, takes a policy file path directly |
| `quell audit verify` | none, takes an audit log file path directly |

`[policy]` is always required, regardless of subcommand: every mode
evaluates statements or tool calls against it. Every other section is
optional and enables a specific capability when present.

## `[listener]`

Required for `quell serve`. Not read by `quell mcp`, which has no network
listener of its own: that mode's traffic is stdio, not the wire protocol.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `bind` | `"host:port"` | yes | *(required)* | Address to accept client (frontend) connections on, e.g. `"0.0.0.0:6432"`. |
| `protocol` | `"postgres"` \| `"mysql"` | no | `"postgres"` | Which wire protocol this listener speaks. |

```toml
[listener]
bind = "0.0.0.0:6432"
protocol = "postgres"
```

## `[upstream]`

Required for `quell serve`.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `addr` | `"host:port"` | yes | *(required)* | The real database server to proxy to. A hostname is fine, not just a literal IP: it's resolved fresh on every new client connection, not once at startup, so it keeps working through a failover that changes the underlying IP. |

```toml
[upstream]
addr = "prod-postgres.internal:5432"
```

## `[policy]`

Always required, on every subcommand that reads config at all.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `path` | path | yes | *(required)* | Path to `policy.yaml`, hot-reloaded on change. See [`docs/POLICY.md`](POLICY.md) for that file's own format. |
| `freeze_marker` | path | no | a `.quell-freeze` file next to `path` | Freeze-mode marker file. Its mere existence, not its content, means "block all writes." Only set this explicitly if two separately-running processes (e.g. a `serve`-only config and an `mcp`-only config) point at different `policy.yaml` files but should still share one freeze switch. |

```toml
[policy]
path = "policy.yaml"
```

## `[tls]`

Optional. Client-facing TLS termination. Present or absent as a whole,
there's no way to half-configure it. Works for both `protocol = "postgres"`
and `protocol = "mysql"` listeners. The proxy's own connection to the real
upstream database is always plaintext regardless of this section (or TLS,
if the upstream database itself requires it at the network layer,
independent of Quell). A client that doesn't request TLS still connects in
plaintext: this section terminates TLS when offered, it doesn't reject
connections that don't offer it. Pair it with a policy or network control
if plaintext connections must be refused outright.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `cert` | path | yes (if `[tls]` present) | *(required)* | PEM certificate file. |
| `key` | path | yes (if `[tls]` present) | *(required)* | PEM private key file. |

```toml
[tls]
cert = "/etc/quell/cert.pem"
key = "/etc/quell/key.pem"
```

## `[estimator]`

Optional. The blast-radius estimator's own side connection, never the
connecting client's own connection, and never the same credential
`[upstream]` proxies to unless you point them at the same place on
purpose. Omitting this section disables blast-radius estimation entirely,
regardless of what `policy.yaml`'s `defaults.estimation` says: a rule that
depends on `rows_gt`/`or_table_pct_gt` simply never matches without one.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `connection` | string | yes (if `[estimator]` present) | *(required)* | A `tokio-postgres`-style connection string for Postgres (`host=... port=... user=... password=... dbname=...`), or a `mysql://user:pass@host:port/db` URL for MySQL. |

```toml
[estimator]
connection = "host=127.0.0.1 port=5432 user=appuser password=${ESTIMATOR_DB_PASSWORD} dbname=prod"
```

## `[audit]`

Optional; every property inside it defaults to off. Every decision is
always logged at the `tracing` level regardless of this section, this
section is for *additional*, structured JSON sinks, each one off unless
explicitly turned on. None configured is a valid, deliberate default, not
something missing.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `log_full_sql` | bool | no | `false` | Include the statement's raw SQL text in every audit event, across every sink configured below. Off by default because literal values in a `WHERE`/`SET` clause can be PII. `fingerprint` (the same statement with literals stripped) is always included either way, regardless of this setting. |
| `stdout` | bool | no | `false` | Emit one JSON line per decision to stdout. |
| `queue_capacity` | integer | no | `4096` | Bounded queue size for each configured sink below. Once full, a sink drops new events (and counts the drops) rather than making a statement wait on the sink's own I/O: a slow or down audit sink can never stall live traffic. |
| `file` | table | no | absent (disabled) | See `[audit.file]` below. |
| `webhook` | table | no | absent (disabled) | See `[audit.webhook]` below. |

### `[audit.file]`

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `directory` | path | yes (if `[audit.file]` present) | *(required)* | Directory audit log files are written into. Rotated daily. |
| `file_name_prefix` | string | no | `"quell-audit"` | Prefix for each rotated file's name. |
| `hash_chain` | bool | no | `true` | Hash-chain every record, so after-the-fact tampering with a past line is detectable via `quell audit verify`, instead of trusting the raw file content (or an agent's own account of what it did) at face value. The cost is one SHA-256 per audit event, on the file sink's own background task, never the statement's own hot path. |

### `[audit.webhook]`

An HTTP sink. Transmits whatever's in the event, including full SQL if
`log_full_sql` is on, to a third-party URL over HTTPS. Treat that URL and
whatever receives it as inside the same trust boundary as the audit data
itself.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `url` | string | yes (if `[audit.webhook]` present) | *(required)* | Destination URL. |
| `bearer_token` | string | no | none | Sent as `Authorization: Bearer <token>` on every request, if set. |
| `batch_size` | integer | no | `50` | Events are batched; a batch is sent once it reaches this many events or `batch_interval_ms` elapses, whichever comes first. |
| `batch_interval_ms` | integer | no | `2000` | See above. |
| `max_retries` | integer | no | `5` | Retries for a batch that fails to send, before it's dropped and counted as lost. |

```toml
[audit]
log_full_sql = false
stdout = true

[audit.file]
directory = "/var/log/quell"

[audit.webhook]
url = "https://example.com/audit-hook"
bearer_token = "${AUDIT_WEBHOOK_TOKEN}"
```

## `[admin]`

Optional, the admin API always runs, binding to localhost by default;
this section only needs to be present to change its bind address or add a
token. Because it exposes operationally sensitive data (recent SQL
activity, the active policy) and actions (forcing a policy reload,
freeze), binding it to anything other than loopback should always be
paired with `token` below and a network-level access control.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `bind` | `"host:port"` | no | `"127.0.0.1:9091"` | Address the admin API listens on. |
| `token` | string | no | none | When set, every route except `/healthz`/`/readyz` requires `Authorization: Bearer <token>`. Those two routes stay open without credentials, since orchestrators (Kubernetes, systemd) need to probe them. |

```toml
[admin]
bind = "0.0.0.0:9091"
token = "${ADMIN_API_TOKEN}"
```

## `[approvals]`

Optional. `action: hold` in `policy.yaml` works via the admin API's
`/approvals` endpoints even with this section entirely absent, it's only
needed to *also* post a notification when a hold is created.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `webhook_url` | string | no | none | A Slack (or Teams-Connector-compatible) incoming webhook URL, posted to whenever a `hold`-decision statement creates a pending approval. |

```toml
[approvals]
webhook_url = "${SLACK_APPROVALS_WEBHOOK_URL}"
```

## `[mcp]`

Required for `quell mcp`. The MCP server's own connection to the database
it runs tool calls (`run_query`, `explain_query`) against, separate from
`[estimator]`, which is a different credential for a different purpose
(that one only ever probes, `[mcp]`'s connection actually executes).

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `connection` | string | yes | *(required)* | Same format as `[estimator].connection`: a `tokio-postgres`-style string for Postgres, or a `mysql://...` URL for MySQL. |
| `protocol` | `"postgres"` \| `"mysql"` | no | `"postgres"` | Which database this connection is talking to. |
| `user` | string | no | `"mcp"` | The identity policy rules matching on `match.databases` or `scopes[].principals.users` evaluate MCP tool calls as. An MCP tool call has no per-call client identity the way a wire-protocol connection's own auth handshake provides one, so this is what stands in for it. |
| `database` | string | no | `"mcp"` | Same idea as `user`, for whatever a policy rule's `match.databases` checks against. |

```toml
[mcp]
connection = "mysql://appuser:${MCP_DB_PASSWORD}@host:3306/db"
protocol = "mysql"
user = "ai-agent"
database = "prod_orders"
```

## `[cluster]`

Optional. Shared control-plane state for a *fleet* of `quell` processes:
freeze status, pending approvals, and open sessions become visible and
actionable from any instance in the fleet, not just the one that created
them. Omitting this section keeps every one of those per-process, exactly
as if this option didn't exist: a single instance, or a fleet sitting
behind sticky routing, needs nothing here at all.

| Property | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `connection` | string | yes (if `[cluster]` present) | *(required)* | Same connection-string format as `[estimator].connection`, Postgres or MySQL, whichever this points at. Reusing the same database `[estimator]`/`[mcp]` already talk to is fine; a handful of small control tables are created automatically. A dedicated database isn't required, just supported if you'd rather isolate control-plane traffic from application traffic. |
| `protocol` | `"postgres"` \| `"mysql"` | no | `"postgres"` | Which database `connection` points at. |

```toml
[cluster]
connection = "host=cluster-db.internal port=5432 user=quell password=${CLUSTER_DB_PASSWORD} dbname=quell_control"
```

## Secrets

Nothing above needs to be plaintext in this file. Set the real value in
the process's environment (a systemd `EnvironmentFile=`, a Docker or
Kubernetes secret mounted as an env var, direnv, your orchestrator's
secret store, whatever fits) and reference it as `${VAR_NAME}` anywhere a
string value is expected: a password inside a connection string, an
`[admin].token`, a webhook URL. One caveat: substitution happens on the
raw file text before TOML parsing, so a substituted value containing a
literal `"` or `\` will break the surrounding string's own quoting, so
keep secrets referenced here free of those two characters.

## A complete example

```toml
[listener]
bind = "0.0.0.0:6432"
protocol = "postgres"

[upstream]
addr = "prod-postgres.internal:5432"

[policy]
path = "/etc/quell/policy.yaml"

[tls]
cert = "/etc/quell/cert.pem"
key = "/etc/quell/key.pem"

[estimator]
connection = "host=prod-postgres.internal port=5432 user=quell_estimator password=${ESTIMATOR_DB_PASSWORD} dbname=prod"

[audit]
stdout = true

[audit.file]
directory = "/var/log/quell"

[admin]
bind = "127.0.0.1:9091"
token = "${ADMIN_API_TOKEN}"

[approvals]
webhook_url = "${SLACK_APPROVALS_WEBHOOK_URL}"

[cluster]
connection = "host=prod-postgres.internal port=5432 user=quell_estimator password=${ESTIMATOR_DB_PASSWORD} dbname=prod"
```
