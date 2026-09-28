# Security policy

Quell sits in front of a production database. If you find a way to make it
allow, hold, or audit a statement incorrectly, or a way to bypass freeze
mode, the policy engine, or the admin API's auth, please report it
privately rather than opening a public issue.

## Reporting a vulnerability

Use GitHub's private vulnerability reporting for this repository
(the "Report a vulnerability" button under the Security tab), or open a
draft security advisory directly:
https://github.com/richierich1610/quell/security/advisories/new

Please include:

- What you did, and what Quell did instead of what it should have.
- The config (`quell.toml`, `policy.yaml`) and Quell version or commit
  used, with any real credentials or connection strings redacted.
- Whether it's a bypass of a specific feature (policy matching, freeze
  mode, TLS, the admin API, audit hash-chaining) or something broader.

You'll get an acknowledgement, and a fix or a clear explanation of why it
isn't one, before anything is disclosed publicly.

## What's in scope

- The policy engine matching or deciding incorrectly for a statement it
  claims to cover.
- Freeze mode being bypassable by any rule, scope, or override.
- The blast-radius estimator's side connection being used to read or write
  data it shouldn't.
- Audit log entries being alterable without `quell audit verify` detecting
  it.
- The admin API's bearer-token check being bypassable, or protected routes
  being reachable without one when a token is configured.
- Credential handling: `${VAR_NAME}` substitution leaking a secret into
  logs, error messages, or the audit trail.

## What's already known and out of scope

Quell states its own limits plainly rather than leaving them for someone
else to find:

- [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md): what Quell defends
  against and what it explicitly doesn't (it's not a network security
  boundary, not an access-control system, not a replacement for
  least-privilege database roles).
- [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md): the honest edges of
  blast-radius estimation and freeze mode's propagation latency.

If your report matches something already documented there, it's still
worth confirming, but it won't be treated as a novel finding.

## No third-party audit yet

Quell hasn't had an independent security audit. That's stated here plainly,
not left for you to discover later. Reports that would change that
picture, in either direction, are especially welcome.
