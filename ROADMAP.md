# Roadmap

Clear Roadmap for this load-test suite:

1. **[CI Improvement](#ci-improvement)** — still targeting the public `jsonplaceholder.typicode.com` mock API.
2. **[GCP/AWS Hardening](#gcpaws-harndening)** — what changes if this pointed at a real, owned service instead.

## CI Improvement

### Constraints

- `jsonplaceholder.typicode.com` is a free, publicly shared mock API, not infrastructure we own or control. Other people are hitting it at the same time we use. 

- No scaling threads/throughput up to "find the breaking point". If we do, We'd just be degrading service for other users.
- No stress/spike/soak tests at meaningfully higher load than the current ~50 req/s. 
- A soak test that runs for hours at even modest load is inconsiderate against shared infra with no SLA to us.
- No assuming stable baselines run-over-run. Someone else load test will impact our test results.


### Scripting

- **Adding reusability concepts**: Adding reusablity using non test elements, include controller will help reduce the size of the jmx file. 
- **CI Environment variable**: Providing a external env file to override the CI yaml values will maintain consistent steps in execution rather than editing ci.yaml frequently
- **Threshold env**: Adding a external threshold to override the threshold to determine pass/fail rate would havily improvise the ci flow.
- **CSV-data**: The currrent setup uses random data generation, which should be moved to CSV dataset config. Which imples AddPost/UpdatePost/PatchPost vary their payloads. As this is amock service, it is okay to have a random generation. but implementing would be nice to show a standard approach.
- **Performance trend runs**: It would be nice to have a setup for periodic traciking of data on response time, errors and providing a trendline on it. Teh current run just overrides the previous results, where we lose data of older runs.
- **Config validation in CI**: Add a pre-check step that fails fast if `perf.properties` throughput/threads are having invalid values. 

### Setting right threshold

- **Scheduled off-peak smoke run.** Add a low-frequency (e.g. weekly) scheduled run at the *current* modest load, rather than only on push to `master`, to catch drift without increasing how often or how hard we hit the shared host.
- **Alerting on IM.** Notify (Slack/GitHub issue) on a failed threshold gate instead of relying on someone noticing a red CI run — still zero additional load on the target, purely a CI-side improvement.


## GCP/AWS Hardening

The current setup proves the approach (script → CI gate → published report) against a public mock API with no auth, no data volume, and no real backend to protect. Taking this to a production-grade NFT solution for a service actually running on GCP or AWS would mean:

### 1. Environment and data realism
- Point `scripts/config/*.properties` at real staging/pre-prod endpoints per environment (GCP: Cloud Run/GKE service URLs behind Cloud Load Balancing; AWS: ALB/API Gateway endpoints), never prod, unless running sanctioned game-day/chaos tests.
- Replace hardcoded request bodies with CSV-driven data sets (`scripts/data/`) using JMeter's CSV Data Set Config, so each virtual user exercises distinct records instead of colliding on the same IDs — critical once the backend has real uniqueness constraints or rate limits.
- Add auth (OAuth2/service-account tokens on GCP via `gcloud auth print-identity-token` via an HTTP Authorization Manager or a pre-test setUp thread group that fetches a token once and shares it across threads.

### 2. Distributed, higher-scale load generation
- Single-runner GitHub Actions is fine for tens of users; beyond a few hundred concurrent threads, move to distributed JMeter (`-R` remote hosts) or a managed load-injection service:
  - GCP: run JMeter workers on GKE/Cloud Run jobs, or use Google Cloud's load testing partners; scale injector pods horizontally per test.
  - AWS: using blazemeter or AWS distributed solution on ECS/EKS for cheaper horizontal scale-out than self-managed JMeter clusters.

### 3. CI/CD and environments
- Split the single workflow into a matrix across environments (dev/qa/perf/prod-canary) with environment-specific thresholds and approval gates before hitting anything beyond staging.
- Run performance gates on a schedule (nightly/weekly regression) in addition to push-triggered smoke runs, so performance regressions are caught even without a code change

### 4. Observability and correlation
- Enable monitoring or observability with backend telemetry: GCP Cloud Monitoring/Cloud Trace or AWS CloudWatch/X-Ray, so any regression can be traced to a specific service, DB query, or cold start rather than inferred from the client side alone.
- Having a self hosted TSDB would allow us to feed the api metrics live to track the runs, compare trends.
- Add alerting (Cloud Monitoring alerting policies / CloudWatch Alarms) tied to the same thresholds enforced in CI, so production drift is caught outside of test runs too.

### 5. Broader NFT strategy
- Extend beyond a single load-test: Including soak, stress test as separate JMX files or property profiles.
- Monitoring the app is a key factor in running laod test. It is vague to run a test on a blackbox. It provides meaningful insights, if provided with monitoring 

