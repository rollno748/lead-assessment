# JMeter Load Test Using Github Runner

This Load test targets the public mock API server [https://jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com).
Which uses Github Actions for CI/CD with automated pass/fail and publishing results on github pages.

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

## API used in test

  - `POST /posts` — AddPost
  - `GET /posts` — GetPosts
  - `GET /posts/{id}` — GetPostById (uses the id captured from GetPosts)
  - `GET /posts/{id}/comments` — GetCommentsByPostId
  - `PUT /posts/{id}` — UpdatePost
  - `PATCH /posts/{id}` — PatchPost
  - `DELETE /posts/{id}` — DeletePost

  `Note: For WLM, refer scripts/config/<env>.properties`


## Prerequisites

- Apache JMeter 5.6.3
- Java 11+
- Git hub repository enabled with github pages write access

## How to run

Pushing any change to `master` (or manually triggering via `workflow_dispatch`) runs [.github/workflows/jmeter-ci.yml](.github/workflows/jmeter-ci.yml), which:

1. Installs JDK 11 and JMeter 5.6.3.
2. Runs the test plan against `scripts/config/perf.properties`.
3. Generates the HTML dashboard and uploads it as a build artifact (`jmeter-results-<run-id>`).
4. Publishes the same dashboard to GitHub Pages.
5. Parses `statistics.json` and fails the build if `p95 > 800ms` or any errors were recorded.

## Output / Results

- **HTML dashboard**: will be published directly to github pages using CI. which can be accessed using [https://rollno748.github.io/lead-assessment/](https://rollno748.github.io/lead-assessment/)
- **CI artifact**: downloadable from the Actions run summary, named `jmeter-results-<run-id>`.

## Results

- **[RESULTS.md](RESULTS.md)**: Contains the information of the last run stats and findings. This is manually updated file, not a CI integrated one.

`Note: Refer the README.md file on dev branch not the master. `

## Roadmap

- **[ROADMAP.md](ROADMAP.md)** — forward-looking plan: near-term suite improvements against the shared mock API, and a separate section on hardening this for production on GCP/AWS.

`Note: Refer the README.md file on dev branch not the master. `

`Pushing the RESULTS.md and ROADMAP.md will trigger the CI to run the test again, which might override the results in the github pages`
