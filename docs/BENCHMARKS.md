# Benchmarks

## Analyzer

Target: analysis of a typical 200-byte statement in under 20 µs
(criterion bench, cached under 1 µs).

Run with:

```bash
cargo bench --bench analyze -p quell-analyzer
```

Measured on this development machine (Apple Silicon, `aarch64-apple-darwin`,
rustc 1.98.1, release profile), 2026-09-24:

| Benchmark | Statement | Time | Target |
|---|---|---|---|
| `analyze_uncached_typical_statement` | 183-byte guarded `UPDATE` | **14.20 µs** | < 20 µs |
| `analyze_cached_typical_statement` | same, via `AnalyzerCache` | **67.7 ns** | < 1 µs |
| `analyze_uncached_unqualified_delete` | `DELETE FROM t` (13 bytes) | **1.77 µs** | n/a |

Both spec targets are met with headroom: the uncached path runs about 30%
under budget, the cached path about 15x under budget. The cached path is
dominated by the `moka` lookup and hash, not parsing, which makes sense
since it skips the AST walk entirely on a hit.

Full statement used for the "typical" benchmark:

```sql
UPDATE accounts SET balance = balance - 100, updated_at = now(), last_modified_by = 'system-job-42' WHERE id = 42 AND status = 'active' AND region = 'us-east-1' AND deleted_at IS NULL
```

Re-run and update this table whenever the analyzer's hot path changes
meaningfully (new AST walk logic, cache key scheme, and so on), not on
every commit.

## Proxy throughput, latency, and memory

Targets: added p99 latency under 1 ms for pass-through reads at 5k QPS
on a laptop-class machine; proxy memory under 100 MB with 1k idle
connections; a comparison of `pgbench -S` and plain `pgbench`
(TPC-B-like) run directly against the database and through the proxy.

Measured on this development machine (Apple Silicon, `aarch64-apple-darwin`,
release profile, 2026-09-25), proxying to a local Postgres 16 container.
Docker Desktop for Mac's userspace port-forwarding is itself a source of
network latency and jitter, noted in the caveat below. `[audit]` and
`[estimator]` are both disabled, isolating the analyzer and policy path
itself, per the "pass-through reads" target (the estimator's own
latency is a separate, already-documented cost that only applies when
estimation is enabled). Data is scale-10 pgbench data (a 1M-row
`pgbench_accounts`).

### Read-only pass-through (`pgbench -S`), rate-limited to the target 5k QPS

```bash
pgbench -S -c 20 -j 4 -T 15 -R 5000 -l --log-prefix=<direct|proxy> quell_test
```

| | Direct | Through proxy |
|---|---|---|
| Achieved tps | 5013 | 4997 |
| Client-observed avg latency | 495 µs | 379 µs |
| Client-observed p99 latency (from `-l` per-transaction logs) | 1421 µs | 925 µs |

The proxied path's client-observed latency here is noise-dominated. It's
not a real signal that the proxy is faster than a direct connection.
Both direct and proxied traffic cross the same Docker Desktop
port-forwarding layer on this machine, and that layer's own jitter is
larger than Quell's actual processing cost (see below). A clean,
environment-independent measurement of "added latency" instead comes from
Quell's own `quell_added_latency_seconds` Prometheus histogram, wall-clock
time spent in the analyzer and policy path specifically, timed in-process
(see `record_metrics_and_audit` in both `quell-proto-pg`/
`quell-proto-mysql`), read via `/metrics` after this run:

- 74,729 statements observed. Average 4.5 µs. 100% fell in the histogram's
  smallest bucket (< 5 ms).

Target met with several orders of magnitude of headroom, consistent with
the analyzer's own cached-path number (67.7 ns) plus a cheap policy evaluation
against an empty rule set. The client-observed wall-clock numbers above
are dominated by this laptop's Docker networking stack, not by Quell.

### TPC-B-like (`pgbench`, default mixed script), unlimited rate

```bash
pgbench -c 10 -j 4 -T 10 quell_test
```

| | Direct | Through proxy |
|---|---|---|
| tps | 4446 | 2674 |
| Client-observed avg latency | 2.249 ms | 3.739 ms |

Unlike the read-only case, this workload shows a real, consistent gap
(roughly 1.5 ms added per five-statement transaction). The in-process
`quell_added_latency_seconds` histogram over this run's 186,968 write-path
statements shows why it isn't Quell's own processing: average around
14.2 µs per statement. The roughly 300 µs-per-statement gap implied by
the wall-clock numbers is consistent with each proxied statement crossing
the client-proxy-upstream path (one extra network hop each way compared to
a direct connection) through this same Docker networking layer, not with
CPU-bound work inside Quell. A deployment where the proxy and database are
colocated (a k8s sidecar, as in
`deploy/k8s-sidecar-example.yaml`, or same host) removes this extra-hop
cost almost entirely. This laptop setup, with the proxy and database both
behind Docker Desktop's userspace networking, is close to a worst case for
hop-doubling overhead specifically, not for Quell's own compute cost.

### Memory: 1,000 idle connections

Measured with `ps -o rss=` on the `quell` process, opening 1,000 real
authenticated Postgres connections through the proxy (staggered in batches
of 25 to avoid saturating Docker Desktop's connection-burst handling)
and holding them idle:

| | RSS |
|---|---|
| Baseline (0 connections) | 7.3 MB |
| With 1,000 idle connections (994 confirmed active via `/metrics`) | 17.3 MB |

Target met: 17.3 MB against the 100 MB budget, about 83% headroom, at
roughly 10 KB of resident memory per idle connection.

### Reproducing

No permanent benchmark script is checked in. The commands above are
copy-paste reproducible against any `docker-compose.test.yml` Postgres
instance plus a `quell.toml` with `[audit]`/`[estimator]` omitted and an
empty `policy.yaml` (`version: 1`, no rules). Re-run and update this
section when the proxy's hot path changes meaningfully (new per-statement
work on the enforcer loop), not on every commit.
