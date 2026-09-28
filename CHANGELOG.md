# Changelog

All notable changes to Quell are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Fixed

- A real, reachable panic in the MySQL handshake relay
  (`strip_client_ssl_capability`): a misconfigured `[upstream]` pointing at
  something that isn't actually MySQL could send a packet too short to
  index safely, panicking before the function's own "relay unchanged
  rather than panic" fallback ever ran.
- Panics in a connection-handling task were invisible until process
  shutdown: nothing polled the per-connection `JoinSet` outside of the
  shutdown drain, so a recurring bug looked identical to an ordinary
  disconnect in every log and metric. Now logged immediately with the
  real panic message and counted in a new `quell_connection_panics_total`
  metric.
- `[upstream].addr` was typed as a literal `SocketAddr`, rejecting the
  hostname every real Kubernetes/managed-database deployment actually
  uses. Now a plain string, resolved fresh on every new client connection.

### Added

- `SECURITY.md`, `CONTRIBUTING.md`, a bug-report issue template, and
  `.github/dependabot.yml` for automated dependency updates.
- A public landing page: <https://richierich1610.github.io/quell/>.
- An explicit "Supported databases" table in the README.

### Removed

- `quell policy suggest` (AI-assisted policy drafting via the Anthropic
  API), built and then cut during a production-readiness pass: it was the
  only feature sending policy content to a third party and needing a
  non-database credential, and the only one never verified against a real
  external network call.

## [0.1.0] - 2026-09-26

First tagged release. A database guardrail proxy that estimates the real
blast radius of a statement before deciding, rather than pattern-matching
SQL text.

### Added

- PostgreSQL and MySQL wire-protocol proxies (simple and extended/prepared
  statement protocols), tested against PostgreSQL 16/17, MySQL 8.4, and
  MariaDB 11.
- Blast-radius estimation via `EXPLAIN` or an exact `COUNT` on a dedicated
  side connection.
- A hot-reloading YAML policy engine: `allow`/`warn`/`hold`/`block`,
  scopes, `quell:allow` overrides, scheduled time windows.
- Freeze mode: a hard write lockdown outside the policy engine entirely.
- A tamper-evident, hash-chained audit log (`quell audit verify`).
- Hold & approve, with an optional Slack/Teams webhook notification.
- MCP mode (`quell mcp`): the same guardrails for AI agents over stdio.
- `[cluster]`: fleet-wide freeze/approvals/sessions coordinated through
  the database itself.
- Client TLS termination on both protocols.
- `${VAR_NAME}` credential substitution, so nothing in `quell.toml` has to
  be plaintext.

[Unreleased]: https://github.com/richierich1610/quell/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/richierich1610/quell/releases/tag/v0.1.0
