# Results

## Run metadata

| Field | Value |
|---|---|
| Date/time | 2026-08-19 08:31–09:01 UTC |
| Env Properties file | `scripts/config/perf.properties` |
| User Load | 25 threads |
| Ramp-up | 30 s |
| Duration | 30 mins |
| Target throughput | 50 req/s |
| JMeter version | 5.6.3 |
| report link | https://rollno748.github.io/lead-assessment/ |


## Response time table

| API | Samples | Error % | Avg (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Max (ms) | Throughput (req/s) |
|---|---|---|---|---|---|---|---|---|
| GET /posts | 43893 | 0.00% | 11 | 13 | 14 | 18 | 790 | 24.39 |
| GET /posts/{id} | 21941 | 0.00% | 11 | 14 | 15 | 23 | 104 | 12.20 |
| GET /posts/{id}/comments | 8775 | 0.00% | 12 | 15 | 17 | 30 | 231 | 4.88 |
| POST /posts | 4389 | 0.00% | 16 | 17 | 18 | 24 | 295 | 2.44 |
| PUT /posts/{id} | 4386 | 0.00% | 20 | 29 | 31 | 37 | 248 | 2.44 |
| PATCH /posts/{id} | 4384 | 0.00% | 16 | 17 | 18 | 28 | 137 | 2.44 |
| DELETE /posts/{id} | 2194 | 0.00% | 16 | 18 | 19 | 25 | 55 | 1.22 |
| **Total** | **89962** | **0.00%** | **12** | **16** | **18** | **28** | **790** | **49.99** |

## Pass/fail against thresholds

| Metrics | Threshold | Actual | Result |
|---|---|---|---|
| p90th percentile response time | < 1 sec | 16 ms (Total) | ✅ Pass |
| Error count | 0 | 0 (of 89,962) | ✅ Pass |

## Bottleneck / Risk

- The run was performed with 50 tps with no error (clean execution)
- The response times were less than ~20 ms (although this is not a consistent behavior with the app). which mich shows a different result upon the resource avialability of teh mock server(which is publicly hosted).
- We cannot run a stress test on this server to identify the capacity(publicly hosted shared server)
- Good thing is, all the request has received an accepted connection, observed no httpresponseException
- Observed no bottleneck when running with 25 users @50 tps 
- All the reponse times were served well within 20 ms
- As this is mock server, there will be no actual business logic implied to process it differently for each request. 
- The POST call is not inserting the data acually to the application. 
- It returns a static data, no matter teh load is, it is impossible to hit a bottleneck except the connection limit and rate limit if implied.
- As there is no database involved in the app, it is hard to look on connection pooling issues, it acts exactly like static server or the provider's shared, CDN-backed infrastructure.


## Scaling into a broader NFT strategy

- A single 30-minute, fixed-load run against a public mock API is a smoke test, not a full NFT strategy. 
- Scaling this up would require multiple tests involving, Load, Stress and Soak.
- Focussing on WLM to achieve the right distribution on the load will give a precise info on the resources needed.
- Having access to monitoring might give a greater view on what is impacting teh resources and provides greater view on what went well/wromg.
- If the app is deployed as self hosted service wither in VM/aws cluster, instead of a black-box public API, it is possible to analyse on more findings rather than the tool generated report.
- Defining the threshold for the acceptance criteria might improve a CI heavily towards full automation point.


## Time tracking

| Task | Est. | Actual |
|---|---|---|
| Script the test |20 min | ~30 min |
| Github configs |30 min | ~60 min |
| CI yaml | 1 hours | ~2 hours |
| Dry runs | 1 hours | ~ 3 hours |
| Actual run | 30 mins | ~ 50 mins |
| Summarise findings | 30 min | ~40 min |
| **Total** | **230 min** | **~480 min** |

`It is achievable within 90 mins when the CI config and execution template is already built within the github CI.`

## Things to consider

- **No throughput ceiling established**: task didn't specify a target beyond 50 concurrent users. It should specify the req/sec to achieve, so this run only confirms the system is healthy at 50 req/sec load. A stepped-load or stress test would be needed to find an actual limit(which can be performed on a owned hosted server).

- **CI Threshold**: The current threshold set on teh CI is a standard set used across teh project. this might change according to the organixzation which needs to be updated.

- **Load generation**: This test is conducted on single Github runner (free version), would need distributed JMeter or a cloud load-testing service to push meaningfully higher load to mimic production behavior.

- **Monitoring**: Lack of monitoring, it is not possible against a third-party public API. It would be easier to do, If it is a self-hosted owned service.

## Roadmap

Roadmap for the load test on `https://jsonplaceholder.typicode.com` is written on, [ROADMAP.md](ROADMAP.md).