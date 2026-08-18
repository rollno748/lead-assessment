# JMeter Load Test Using Github Runner

Load test suite targeting the public mock API [https://jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com), wired into GitHub Actions CI/CD with automated pass/fail and publishing results on github pages.

## About the directory

Apache JMeter 5.6.3 is used to execute the testplan. which uses a config file properties to switch between different environments(QA, DEV and Performacne).

The CI workflow test gets triggered only on every push/pull request made to `master`. Which does the following 

 - Runs the test 
 - Generates an HTML dashboard
 - Uploads it as a build artifact
 - Publishes it to GitHub Pages
 - Provide a sneak peak of pass/fails thresholds.

## Folder structure

```
.
├── .github/
│   └── workflows/
│       └── jmeter-ci.yml      # CI pipeline for jmeter github runner
├── scripts/
│   ├── jsonplaceholder-test.jmx # JMeter test plan
│   ├── config/           # Contains properties for diff env
│   │   ├── dev.properties 
│   │   ├── qa.properties     
│   │   └── perf.properties
│   └── data  #directory to hold csv files
├── results/            # JMeter output — .jtl results and HTML 
├── FINDINGS.md          
├── RESULTS.md           
└── README.md          # How to guide 
```

## Test plan design

- **Thread Group**: `${THREADS}` users, `${RAMPUP}` second ramp-up, `${DURATION}` second scheduler duration, `${LOOP}` loops (see property files).
- **Endpoints under test** (via `ThroughputController`, weighted by `*_pctl` properties so the mix is tunable per environment):
  - `POST /posts` — AddPost
  - `GET /posts` — GetPosts
  - `GET /posts/{id}` — GetPostById (uses the id captured from GetPosts)
  - `GET /posts/{id}/comments` — GetCommentsByPostId
  - `PUT /posts/{id}` — UpdatePost
  - `PATCH /posts/{id}` — PatchPost
  - `DELETE /posts/{id}` — DeletePost (asserts HTTP 200)
- **Assertions**: JSONPath assertions confirming `$.id` is present on each response; explicit response-code assertion on DELETE.
- **Throughput shaping**: `ConstantThroughputTimer` paces overall requests/min via `${OVERALL_TPS}` (`throughput` property × 60).
- **Listeners**: Aggregate Report (always on) and View Results Tree (disabled by default, enable locally for debugging).

## Prerequisites

- [Apache JMeter 5.6.3](https://jmeter.apache.org/download_jmeter.cgi) (matches the version installed in CI)
- Java 11+ (Temurin distribution used in CI)
- Git hub repository enabled with github pages write access

## How to run

Pushing any change to `master` (or manually triggering via `workflow_dispatch`) runs [.github/workflows/jmeter-ci.yml](.github/workflows/jmeter-ci.yml), which:

1. Installs JDK 11 and JMeter 5.6.3.
2. Runs the test plan against `scripts/config/perf.properties`.
3. Generates the HTML dashboard and uploads it as a build artifact (`jmeter-results-<run-id>`).
4. Publishes the same dashboard to GitHub Pages.
5. Parses `statistics.json` and fails the build if `p95 > 800ms` or any errors were recorded.

## Output / Results

- **Raw results**: `results/results_<run-id>.jtl` (CSV/XML sample log).
- **HTML dashboard**: `results/dashboard/<run-id>/index.html` (+ `statistics.json` for machine-readable metrics).
- **CI artifact**: downloadable from the Actions run summary, named `jmeter-results-<run-id>`.
- **GitHub Pages**: latest dashboard published automatically (see the Pages URL in the workflow run's environment link).
- **Findings**: see [RESULTS.md](RESULTS.md) for the write-up of the submitted run (metrics, bottleneck/risk, NFT strategy notes), using [FINDINGS.md](FINDINGS.md) as the reusable template for future runs.

## Roadmap: hardening this for production on GCP/AWS

The current setup proves the approach (script → CI gate → published report) against a public mock API with no auth, no data volume, and no real backend to protect. Taking this to a production-grade NFT solution for a service actually running on GCP or AWS would mean:

### 1. Environment and data realism
- Point `scripts/config/*.properties` at real staging/pre-prod endpoints per environment (GCP: Cloud Run/GKE service URLs behind Cloud Load Balancing; AWS: ALB/API Gateway endpoints), never prod, unless running sanctioned game-day/chaos tests.
- Replace hardcoded request bodies with CSV-driven data sets (`scripts/data/`) using JMeter's CSV Data Set Config, so each virtual user exercises distinct records instead of colliding on the same IDs — critical once the backend has real uniqueness constraints or rate limits.
- Add auth (OAuth2/service-account tokens on GCP via `gcloud auth print-identity-token`, or SigV4/Cognito tokens on AWS) via an HTTP Authorization Manager or a pre-test setUp thread group that fetches a token once and shares it across threads.

### 2. Distributed, higher-scale load generation
- Single-runner GitHub Actions is fine for tens of users; beyond a few hundred concurrent threads, move to distributed JMeter (`-R` remote hosts) or a managed load-injection service:
  - GCP: run JMeter workers on GKE/Cloud Run jobs, or use Google Cloud's load testing partners; scale injector pods horizontally per test.
  - AWS: use [AWS Distributed Load Testing solution](https://aws.amazon.com/solutions/implementations/distributed-load-testing-on-aws/) (Fargate-based JMeter workers) or Locust/k6 on ECS/EKS for cheaper horizontal scale-out than self-managed JMeter clusters.
- Consider swapping JMeter for a cloud-native load tool (k6, Gatling) if the team wants load scripts as code with native cloud-runner integrations (k6 Cloud, Grafana k6 on GCP/AWS).

### 3. CI/CD and environments
- Split the single workflow into a matrix across environments (dev/qa/perf/prod-canary) with environment-specific thresholds and approval gates before hitting anything beyond staging.
- Store secrets (API keys, service account JSON) in GitHub Environments/OIDC federation to GCP Workload Identity or AWS IAM roles — never long-lived static credentials in workflow YAML.
- Run performance gates on a schedule (nightly/weekly regression) in addition to push-triggered smoke runs, so performance regressions are caught even without a code change (e.g., after infra/config drift).

### 4. Observability and correlation
- Correlate JMeter results with backend telemetry: GCP Cloud Monitoring/Cloud Trace or AWS CloudWatch/X-Ray, so a p95 regression can be traced to a specific service, DB query, or cold start rather than inferred from the client side alone.
- Feed `statistics.json` (or Prometheus-formatted JMeter output via the Backend Listener) into a time-series store (Cloud Monitoring custom metrics, Amazon Managed Prometheus/Grafana) to track trends across runs instead of comparing single HTML reports by hand.
- Add alerting (Cloud Monitoring alerting policies / CloudWatch Alarms) tied to the same thresholds enforced in CI, so production drift is caught outside of test runs too.

### 5. Broader NFT strategy
- Extend beyond a single load-shape test: add soak (multi-hour steady load, checking for memory/connection leaks), spike (sudden burst to N× baseline), and stress (find the breaking point) test plans as separate JMX files or property profiles.
- Layer in autoscaling validation — confirm GKE HPA / ECS Service Auto Scaling actually reacts within SLA during ramp-up, not just that the app survives.
- Treat performance budgets as code: version thresholds alongside the service, review them in PRs like any other contract, and block merges (not just report) when budgets regress.

## What's left / known gaps

See [RESULTS.md](RESULTS.md) for the current, honest list of what wasn't completed given the time budget.
