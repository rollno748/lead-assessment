# Findings

Template for capturing findings from a JMeter run against `https://jsonplaceholder.typicode.com`.
Fill this in per test run (or copy this file to `RESULTS.md` and complete it — see task requirements).

## Run metadata

| Field | Value |
|---|---|
| Date/time | |
| Environment (config file used) | `scripts/config/perf.properties` |
| Threads (concurrent users) | |
| Ramp-up period | |
| Duration / loops | |
| Target throughput | |
| JMeter version | 5.6.3 |
| Triggered by | CI run / local run (link to Actions run or artifact) |

## Key metrics

| Endpoint | Samples | Error % | Avg (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Max (ms) | Throughput (req/s) |
|---|---|---|---|---|---|---|---|---|
| GET /posts | | | | | | | | |
| GET /posts/{id} | | | | | | | | |
| GET /posts/{id}/comments | | | | | | | | |
| POST /posts | | | | | | | | |
| PUT /posts/{id} | | | | | | | | |
| PATCH /posts/{id} | | | | | | | | |
| DELETE /posts/{id} | | | | | | | | |
| **Total** | | | | | | | | |

Source: `results/dashboard/<TEST_RUN_ID>/statistics.json` (or the HTML dashboard `index.html`).

## Pass/fail against thresholds

| Check | Threshold | Actual | Result |
|---|---|---|---|
| p95 response time | < 800 ms | | ☐ Pass ☐ Fail |
| Error count | 0 | | ☐ Pass ☐ Fail |

## Bottleneck / risk

_One bottleneck or risk observed during this run (e.g., a specific endpoint's latency under load, rate-limiting behavior, error clustering at ramp-up peak, etc.)._

-

## Scaling into a broader NFT (non-functional testing) strategy

_2–3 sentences on how this load test would extend into a broader NFT strategy — e.g., soak/endurance tests, spike tests, larger user models, integration with APM/observability, shift-left performance gates in CI._

-

## Time tracking

| Task | Est. | Actual |
|---|---|---|
| Script the test | 35 min | |
| Wire into CI/CD | 30 min | |
| Summarise findings | 15 min | |
| **Total** | **90 min** | |

> Per the ground rules: running over/under 90 min is fine. Note the actual time honestly, and list anything left undone below rather than omitting it silently.

## What's left / not done

-
