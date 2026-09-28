# Policy file reference

Quell's behavior is driven by a YAML policy file (`policy.yaml`), hot
reloaded without dropping connections. This document is the authoritative
reference for the file format, including precedence rules not fully
pinned down elsewhere.

A full example lives at [`examples/policy.yaml`](../examples/policy.yaml).
Validate any policy file with:

```bash
quell policy check path/to/policy.yaml
```

Try it against a synthetic statement without touching a database:

```bash
quell policy test path/to/policy.yaml \
  --sql "DELETE FROM orders" --user alice --db prod_orders
```

## Top-level structure

```yaml
version: 1          # required, must be 1
defaults: { ... }    # optional, see below
rules: [ ... ]        # optional, see below
scopes: [ ... ]        # optional, see below
overrides: { ... }      # optional, see below
```

## `defaults`

```yaml
defaults:
  mode: enforce               # enforce | monitor (default: enforce)
  on_parse_error: block       # block | warn | allow (default: block)
  estimation:
    enabled: true
    method: explain           # explain | count | off
    count_timeout_ms: 500
    on_estimate_failure: block  # block | warn | allow
```

- **`mode: monitor`** never actually blocks or holds. A decision that would
  have been `block`/`hold` is downgraded to `warn`, but `matched_rule_ids`
  and `message` still record what would have happened, for audit.
- **`on_parse_error`** governs statements the analyzer couldn't fully
  parse. It only applies when the statement looks like a write, per the
  analyzer's lightweight fallback lexer. A statement that looks like
  `SELECT`/`SHOW`/`EXPLAIN` is always allowed even if unparseable,
  regardless of this setting.
- **`estimation`** configures the blast-radius estimator. The policy
  engine understands `rows_gt`/`or_table_pct_gt` rule conditions (see
  below); a rule that depends on an estimate simply doesn't match unless
  `[estimator]` is configured in `quell.toml`.

## `rules`

```yaml
rules:
  - id: no-unqualified-writes        # required, unique
    match: { statements: [update, delete] }
    when: { where: missing_or_tautology }
    action: block                    # allow | warn | hold | block
    message: "..."                   # optional, shown to the client/CLI
```

### `match`: when does this rule apply at all?

```yaml
match:
  statements: [update, delete]   # select|insert|update|delete|merge|truncate|drop|alter
  databases: ["prod_*"]           # globs against the session's database
  schemas: ["public"]              # globs against the statement's target table(s)' schema
  tables: ["accounts", "pay_*"]     # globs against the statement's target table(s)' name
```

Every field in `match` is AND'd together. A rule only applies when the
statement's kind, the session's database, and the statement's target
table(s) all satisfy their respective filters. An empty or omitted list
means "no restriction" for that dimension and matches everything. That's
the general convention for every glob list in this schema except
`overrides.allowed_users` (see below).

`schemas`/`tables` match against the statement's target tables (the ones
an `UPDATE`/`DELETE` actually writes to), not just the session's default
schema. If a statement has no identifiable target table and the rule
specifies a non-empty `schemas`/`tables` filter, the rule doesn't match.
That's a fail-closed choice: an unprovable match is not a match.

### `when`: which trigger condition(s), if any?

```yaml
when:
  where: missing_or_tautology                       # WHERE is absent or always-true
  rows_gt: 1000                                       # blast radius exceeds N rows
  or_table_pct_gt: 10                                   # OR exceeds N% of the table
  time_window: { cron: "0 18 * * FRI", duration: "62h", tz: "Asia/Kolkata" }
```

Unlike `match`, every condition present in `when` is OR'd together. The
rule's `when` matches if any specified condition holds. This generalizes
the `or_table_pct_gt` field's own name (which pairs with `rows_gt` in
blast-radius rules) to the whole block: if a rule specifies
both `where` and `rows_gt`, either one being true is enough. An empty or
absent `when` always matches, since there's no condition to check: a rule
that only sets `match` (a blanket `block` on `truncate` for a given table,
say) applies unconditionally whenever `match` is satisfied.

- **`where: missing_or_tautology`**: true when the statement either has no
  `WHERE` clause at all, or has one that constant-folds to always-true
  (`1=1`, `TRUE`, `col = col`, `x OR 1=1`, and so on). See
  the analyzer's tautology detector.
- **`rows_gt`** / **`or_table_pct_gt`**: require a blast-radius estimate.
  Without `[estimator]` configured, these simply don't match
  (`quell policy test` accepts `--estimate-rows` and
  `--estimate-table-pct` so you can exercise them by hand without a
  database).
- **`time_window`**: `cron` uses the standard 5-field crontab syntax
  (`min hour day-of-month month day-of-week`), or the underlying `cron`
  library's native 6/7-field syntax with a leading
  seconds field. Both are accepted; a 5-field expression is widened
  automatically. `duration` is a simple `<number><unit>` string
  (`d`/`h`/`m`/`s`, e.g. `"1d12h30m"`). `tz` is an IANA timezone name. The
  window is `[most recent trigger T, T + duration)`.

### `action`

`allow | warn | hold | block`. When multiple rules match, after scope
exemptions are applied, the most severe decision wins:
**`block` > `hold` > `warn` > `allow`**. `matched_rule_ids` records every
matched-and-not-exempted rule, not just the one that decided the outcome.
`message` comes from the first rule, in declaration order, that produced
the winning decision.

A `hold` decision keeps the client connection waiting, registers a
pending approval, and optionally posts to Slack or Teams; see
[`docs/FEATURES.md`](FEATURES.md)'s "Hold and approve" section.

## `scopes`: narrowing or exempting rules per principal

```yaml
scopes:
  - principals: { users: [etl_service], cidrs: [10.0.5.0/24] }
    exempt: [blast-radius]
```

A scope's `principals` block (`users`, `cidrs`, `databases`,
`application_names`, all glob/CIDR lists, empty meaning "matches
anything") is checked against the current session. If it matches, the
rule IDs in `exempt` are removed from consideration for that session, as
if those rules didn't exist for this particular statement.

### Precedence when multiple scopes match

Quell's design goal is "the most specific matching scope wins," without
a universal definition of specificity. The implemented, tested contract:

1. Every scope whose `principals` matches the current session is a
   candidate.
2. Specificity is the total count of entries across `users` + `cidrs` +
   `databases` + `application_names` in that scope's `principals`. More
   listed constraints counts as more specific. This is an imperfect proxy
   (a scope naming 50 users has a higher count than one naming a single
   user and CIDR pair even though the latter is arguably narrower in
   effect), but it's simple, deterministic, and documented here.
3. Ties are broken by declaration order. The earlier-declared scope wins.
4. Only the single winning scope's `exempt` list applies. Exemptions from
   other matching-but-less-specific scopes are ignored entirely, not
   merged in. This is deliberately the more conservative reading (fewer
   exemptions granted by default), since exemptions weaken a block.

## `overrides`: the `quell:allow` escape hatch

```yaml
overrides:
  allowed_users: [dba_lead, dba_oncall]
  require_fields: [reason, ticket]
```

A statement carrying a `/* quell:allow reason="..." ticket="..." */`
comment (parsed from actual comment tokens only, never from string
literals) can bypass a `block`/`hold`
decision, but only when:

1. The session's `user` matches `overrides.allowed_users`, and
2. every field listed in `overrides.require_fields` is present and
   non-empty in the hint.

`allowed_users` is the one glob list in this whole schema where empty or
absent means "matches no one," not "matches everyone." Overrides are
honored only for roles permitted by policy, and an empty allowlist
permits none. Every other glob list in this file (`match.*`,
`scopes[].principals.*`) uses the opposite convention, where empty means
unrestricted, because those describe when a rule applies, and fail-open is
the reasonable default there. `allowed_users` describes who may bypass a
block, where fail-closed is the safe default.

An override that's present but doesn't satisfy both conditions is
rejected. The original decision stands, unchanged. Either way, the attempt
is recorded on the `Verdict` (`override_attempted`, `override_applied`) for
audit, whether or not it succeeded. A rejected override is still an
auditable event: it does not silently disappear just because it failed.

Overrides only ever apply to a `block`/`hold` decision. Attempting one on
an already-`allow`/`warn` statement is recorded as attempted but never
marked applied, since there's nothing to override.

## Hot reload

The current policy is held behind a lock-free swap, so readers never
block while a reload is in progress. A reload re-parses and re-validates
the file from disk; on any error, the previously-loaded policy stays in
effect. A bad edit, or a half-written file caught mid-save, never leaves
Quell without a usable policy. A background file watcher reloads
automatically on every change to `policy.yaml`, no restart needed.
