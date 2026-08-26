# Development guidelines

These guidelines should be followed for code to be used in production. PoCs can decide not to follow all of the guidelines, however, if they are intended to be used in production, extra work needs to be done to meet production quality.
The following topics are covered:

- [1. Version Control of Sources](#1-version-control-of-sources)
  - [git-flow](#git-flow)
- [2. Coding style](#2-coding-style)
  - [Fail-Fast](#fail-fast)
  - [Environments](#environments)
- [Observability](#observability)
  - [Logging](#logging)
  - [Tracing](#tracing)
  - [Metrics](#metrics)
- [3. Testing](#3-testing)
  - [1. Unit Tests](#1-unit-tests)
  - [2. Integration Tests](#2-integration-tests)
  - [3. E2E Tests](#3-e2e-tests)
  - [4. System Integration Tests](#4-system-integration-tests)
  - [5. Load and Performance Tests](#5-load-and-performance-tests)
  - [6. Manual Tests](#6-manual-tests)

## 1. Version Control of Sources

`git` is used as the version control system of source code. Git repositories are hosted on [github.com](github.com). All sorts of code, scripts (bash, SQL, ...) and documentation should be tracked.

**IMPORTANT:** read [Git Flow](GIT_FLOW.md) to learn about the `git` and `PR` workflow.

### git-flow

[Git Flow](GIT_FLOW.md)

## 2. Coding style

Some guidelines that are generic to all languages are noted in this chapter. Language specific guidelines can be found on the following mardown files:

- [Coding Guidelines - Python](PYTHON.md)
- [Coding Guidelines - JavaScript](JAVASCRIPT.md)
- [Coding Guidelines - Ansible](ANSIBLE.md)
- [Coding Guidelines - Bash](BASH.md)
- [Coding Guidelines - Docker](DOCKER.md)
- [Coding Guidelines - Golang](GO.md)
- [Coding Guidelines - Terraform](TERRAFORM.md)

> [!IMPORTANT]
> Keep in mind that most of the time is spent **reading** code (not writing), yours or from someone else. Hence try to make life easier for the one **reading** and trying to understand your code.

### Fail-Fast

> [!TIP]
> **The goal is not to make any mistakes. The goal is to find and fix them quickly!**

- When there is a missing environment variable or start-up parameter, instead of still starting up the system normally or using a fall-back strategy (fall-back to default environments/parameters), the system should fail and stop so that we can be notified and fix the problem right away.

- When a client sends a request with invalid parameters, instead of silently correcting the parameters and continuing to handle the request, the server should let the request fail so that the client can be notified and fix the problem as soon as possible. Sometimes it can make sense to be permissive on what you accept and restrictive on what you send (i.e. sanitize wrong input if possible).

- Exceptions should never be silently swallowed. Exceptions should only be caught when the catcher knows how to handle it; otherwise, let the exception be thrown outside. And let the app crash if no part of the app knows how to handle it (an exception caused by unexpected bugs). And then fix it.

### Environments

Three environments are available:

- *dev*: the development environment is mainly used to test and develop things that either need access to hardware other than the notebook provides or for interface development, where the component under development must be accessible by other systems. Whenever development needs other resources, it should be *dev* resources.
- *int*: the integration or staging environment is used to verify that production-ready changes to a component work as expected together with other resources (in terms of functionality, performance, ...)
- *prod*: everything that other people and systems rely on for doing their work. This can be other internal systems, external services, internal or external users of apps and services.

Development should happen as much as possible locally on the notebook.

## Observability

For observability, each service should use OpenTelemetry logging, tracing and metrics. All telemetry data should be sent or collected by the OTEL collector.

### Logging

Each service must write logs using different log levels. The logs should be written to `stdout` by default in a human readable format. Then, whenever possible, we should also use an OpenTelemetry log exporter in parallel that sends the log directly in the correct OTEL format to the OTEL log collector. Logs are finally centralized in the Elasticsearch/Kibana observability infrastructure.

### Tracing

Each service should use OpenTelemetry tracing to trace the execution of its requests. Traces are sent to the OTEL collector and are finally visualized in the Kibana observability infrastructure. For local development, traces should be sent to a local `otel/opentelemetry-collector-contrib` docker container and visualized in a local Jaeger docker container.

### Metrics

Each service should use OpenTelemetry metrics to collect and report metrics about its execution (mostly done by opentelemetry instrumentation libraries). Metrics are sent to the OTEL collector and are finally visualized in the Kibana observability infrastructure. For local development, metrics should be sent to a local Prometheus docker container which also provides a simple web UI for visualizing metrics.

## 3. Testing

Testing is an integral part of development and mandatory for production-targeted code. This applies to apps as well as scripts. We distinguish the following categories of tests:

| Level | Term | Scope | Mandatory | Lifecycle | Automated | Description | Typical tools |
| ----- | ---- | ----- | --------- | --------- | --------- | ----------- | ------------- |
| 1 | Unit Tests | Individual functions, classes, objects, ui components, ... | :white_check_mark: | PR | :white_check_mark: | Simple, small and fast tests that should be used extensively | Vitest, pytest, go test, ... |
| 2 | Integration Tests | Test application in a sandbox | :white_check_mark: | PR | :white_check_mark: | Testing an application in a sandbox environment (not an actual application deployment, with all external dependencies mocked). E.g. testing a backend service endpoint using a test client, or a frontend application using a local test deployment. | Playwright e2e tests, Cypress e2e tests, Django test client, FastAPI test client |
| 3 | E2E Tests | Application deployment test | :white_check_mark: | Deployment/Daily | :white_check_mark: | Testing the full application deployment without any mocking | pytest, pytest-playwright |
| 4 | System Integration Tests | Whole system, across multiple services/applications | :no_entry_sign: | Deployment/Daily | :white_check_mark: | Testing a whole system deployment; verifies consistent behavior when multiple applications interact (e.g. an action in application A produces the expected result in application B) | pytest, pytest-playwright |
| 5 | Load and Performance Tests | System/application deployment | :no_entry_sign: | On demand/manual | :no_entry_sign: | Throughput and robustness tests on application- or system-level | k6 |
| 6 | Manual Tests | System/application deployment | :no_entry_sign: | On demand/manual | :no_entry_sign: | Manual testing; should only be used when automated tests are too complex or would require significant effort to automate | Manual scripts or manual interaction (e.g. GUI interaction) |

Language specific details about testing can be found here:

- [python](PYTHON.md#9-unit-testing-frameworks)
- [bash](BASH.md#4-unit-tests--shellspec)
- [javascript](JAVASCRIPT.md#testing)

### 1. Unit Tests

Unit Tests can be performed with a dedicated test runner before the Docker image is built. They do not require external resources, though such resources can be mocked when useful. Target code coverage for Unit Tests should be above 70%.

### 2. Integration Tests

Integration Tests are application acceptance tests. They verify that an isolated application works as expected with all external dependencies mocked. They can run against the final application bundle (e.g. a Docker image or frontend bundle) deployed in a sandbox environment such as CI, or they can use test clients such as the FastAPI test client. They run automatically as part of the PR pipeline before the application bundle is pushed (e.g. before a Docker image is pushed to Amazon ECR). All external resources must be mocked.

### 3. E2E Tests

E2E Tests are performed against a Docker image deployed to a staging environment with access to external resources. They should verify that the newly built version works correctly with existing data and other dependent services.

> [!IMPORTANT]
> Mutating E2E Tests MUST NOT run in the `PROD` environment; they may run only in the `DEV` and `INT` environments.

### 4. System Integration Tests

System Integration Tests are similar to E2E Tests but involve multiple applications or services in the same test. For example, a test might use application A to trigger a behavior and then verify it in application B. In contrast, E2E Tests act on a single application. Although that application may rely on others behind the scenes, the test itself exercises only one.

> [!IMPORTANT]
> As with E2E Tests, System Integration Tests are performed against deployed applications in all staging environments. Mutating tests MUST NOT run in the `PROD` environment; they may run only in the `DEV` and `INT` environments.

### 5. Load and Performance Tests

Load and Performance Tests are performed against a Docker image deployed to a staging environment with access to external resources. They should verify that the newly built version performs well with existing data and other services under normal and high loads.

> [!IMPORTANT]
> They MUST NEVER run in the `PROD` environment and should generally run only in `INT`, with scaling comparable to `PROD`.

### 6. Manual Tests

Manual Tests should be performed directly in `PROD` after major deployments to verify that the deployment was successful. They are not always required and should be automated whenever possible.
