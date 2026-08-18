# Results

Findings from the load test run against `https://jsonplaceholder.typicode.com`, using [FINDINGS.md](FINDINGS.md) as the template.

## Run metadata

| Field | Value |
|---|---|
| Date/time | 2026-08-18 22:07–22:12 BST |
| Environment (config file used) | `scripts/config/perf.properties` |
| Threads (concurrent users) | 25, ramped over 30s |
| Ramp-up period | 30 s |
| Duration / loops | 300 s scheduler duration (loop=-1) |
| Target throughput | 25 req/s (`ConstantThroughputTimer`, no target specified by the task) |
| JMeter version | 5.6.3 |
| Triggered by | Local CLI run (non-GUI): `results/results_20260818-220658.jtl`, dashboard at `results/dashboard/20260818-220658/` |

## Key metrics

Source: `results/dashboard/20260818-220658/statistics.json`. p90/p95/p99 below are JMeter's `pct1/pct2/pct3ResTime`.

| Endpoint | Samples | Error % | Avg (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Max (ms) | Throughput (req/s) |
|---|---|---|---|---|---|---|---|---|
| GET /posts | 3627 | 0.00% | 87 | 192 | 341 | 670 | 21,131 | 12.09 |
| GET /posts/{id} | 1806 | 0.00% | 56 | 104 | 127 | 383 | 2,540 | 6.04 |
| GET /posts/{id}/comments | 722 | 0.14% | 106 | 201 | 284 | 626 | 21,126 | 2.41 |
| POST /posts | 363 | 0.00% | 129 | 174 | 200 | 413 | 726 | 1.21 |
| PUT /posts/{id} | 359 | 0.00% | 216 | 300 | 344 | 537 | 972 | 1.20 |
| PATCH /posts/{id} | 358 | 0.00% | 129 | 169 | 197 | 436 | 928 | 1.20 |
| DELETE /posts/{id} | 181 | 0.00% | 139 | 192 | 282 | 623 | 707 | 0.61 |
| **Total** | **7416** | **0.01%** | **93** | **197** | **291** | **592** | **21,131** | **24.72** |

## Pass/fail against thresholds

| Check | Threshold | Actual | Result |
|---|---|---|---|
| p95 response time | < 800 ms | 291 ms (Total) | ✅ Pass |
| Error count | 0 | 1 (of 7,416 — 0.013%) | ❌ Fail (as currently gated in CI) |

## Bottleneck / risk

No throughput-driven bottleneck was observed at 25 concurrent users / ~24.7 req/s — every endpoint's p95 stayed well under the 800ms gate (Total p95 = 291ms), and mean latencies for GETs were under 110ms. This is expected: `jsonplaceholder.typicode.com` is a static/mock API with no real database or business logic behind it, so this test cannot surface genuine infrastructure bottlenecks (DB connection pools, GC pauses, downstream service calls) — anything observed here is either a client-side artifact of the load generator or the provider's shared, CDN-backed infrastructure.

The one concrete risk this run *did* surface: a single `GetCommentsByPostId` request failed with `NoHttpResponseException` (`jsonplaceholder.typicode.com:443 failed to respond`), and the Total/GetPosts/GetCommentsByPostId **max latency spiked to ~21.1 seconds** against averages of ~90-100ms — a >200x tail-latency outlier. Both point to the same underlying risk: transient connection drops / severe tail-latency spikes on the provider's shared infrastructure, invisible at p95/p99 but real at the max. Under the CI workflow's current gate (`errorCount > 0` fails the build), this single transient failure would have **failed the pipeline** even though the p95 SLA was comfortably met — worth revisiting the gate to tolerate a small error-rate threshold (e.g., <0.1%) rather than zero, since a hard zero-error gate against a third-party public API is prone to flaking on transient network blips outside your control.

Because no throughput target was specified by the task, this result should be read as "no bottleneck found at this load," not "this is the system's ceiling" — headroom above 25 req/s is unverified.

## Scaling into a broader NFT strategy

A single 5-minute, fixed-load run against a public mock API is a smoke test, not a full NFT strategy. Scaling this up would mean layering in soak tests (multi-hour steady load to catch leaks/degradation), spike and stress tests (to actually find a breaking point, since none was found here), and distributed load generation once concurrency needs exceed what a single JMeter instance can drive — combined with correlating client-side metrics against real backend observability (APM/traces) once this is pointed at an owned service instead of a black-box public API. Performance gates like the p95 check here should also move further left (run on every PR against a staging environment, not just post-merge) and use tolerant, not zero, error thresholds to avoid flaking on transient third-party network noise.

## Time tracking

| Task | Est. | Actual |
|---|---|---|
| Script the test | 35 min | ~40 min |
| Wire into CI/CD | 30 min | ~35 min |
| Summarise findings | 15 min | ~25 min |
| **Total** | **90 min** | **~100 min** |

> Per the ground rules: running over/under 90 min is fine. Actual time ran slightly over budget, mainly due to iterating on the CI threshold-gating step and re-running the test headlessly to capture real statistics (the original run had been done via the JMeter GUI, which doesn't produce `results/` artifacts).

## What's left / not done

- **No throughput ceiling established** — task didn't specify a target, so this run only confirms the system is healthy at 25 req/s, not where it breaks. A stepped-load or stress test would be needed to find an actual limit.
- **CI error-rate gate is binary (zero-tolerance)** — given the transient `NoHttpResponseException` seen in this run, the gate in [.github/workflows/jmeter-ci.yml](.github/workflows/jmeter-ci.yml) should likely tolerate a small error budget instead of failing on any single error.
- **No response-time assertion inside the JMX itself** — p95 gating currently happens only in the CI step via `jq` against `statistics.json`, not as a JMeter Duration Assertion in the test plan.
- **Single-machine, non-distributed load generation** — fine at 25 threads, would need distributed JMeter or a cloud load-testing service to push meaningfully higher.
- **No correlation with server-side observability** — not possible against a third-party public API; noted here as a gap that would need to close before this is production-grade against an owned service.
