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
  - [Unit Tests](#unit-tests)
  - [Application Tests](#application-tests)
  - [System/Application E2E Tests](#systemapplication-e2e-tests)
  - [System/Application Load and Performance Tests](#systemapplication-load-and-performance-tests)
  - [Manual Tests](#manual-tests)

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
| ----- | ---- | ----- | --------- | --------- | ----------- | ------------- |
| 1 | Unit Tests | Individual functions, classes, objects | :white_check_mark: | PR | :white_check_mark: |Simple and small tests, should be used extensively | Vitest, pytest, go test, ... |
| 2 | Component Tests | A single UI component, a software component in isolation | :no_entry_sign: | PR | :white_check_mark: |Optional test category mostly used in frontend application. | Playwrith Component Testing, Cypress Component Testing |
| 3 | Application Tests | An entire application in isolation | :white_check_mark: | PR | :white_check_mark: |Testing a full application in isolation, meaning any external application involvment should be mocked. | Playwrith e2e Testing, Cypress e2e Testing, django test client, fastapi test client |
| 4 | Application E2E Tests | Testing Application deployment | :white_check_mark: | Deployment/Daily | :white_check_mark: | Testing the full application deployment without mocking | Pytest, Pytest playwright |
| 5 | System Integration E2E Tests | Testing a whole system deployment | :no_entry_sign: | Deployment/Daily | :white_check_mark: | Testing a whole system deployment. System are consistent on several application interaction. | Pytest, Pytest playwright |
| 6 | Performance Application Tests | Application deployment | :no_entry_sign: | On demand/manual | :no_entry_sign: | Testing the performance of an application under normal load, how quick are API calls under normal load | k6 |
| 7 | Load Application Tests | Application deployment | :no_entry_sign: | On demand/manual | :no_entry_sign: | Testing the application performance and behavior under high load |
| 8 | Load System Tests | System deployment | :no_entry_sign: | On demand/manual | :no_entry_sign: | Testing the whole system performance and behavior under high load. A system is the interaction of several applications.  |
| 9 | Manual Tests | System and/or application deployment | :no_entry_sign: | On demand/manual | :no_entry_sign: | Those should only be used when automated test are too complex or would require a big effort to automate | Manual scripts |

Language specific details about testing can be found here:

- [python](PYTHON.md#9-unit-testing-frameworks)
- [bash](BASH.md#4-unit-tests--shellspec)
- [javascript](JAVASCRIPT.md#testing)

### Unit Tests

Unit Tests can be performed before a docker image is built using a dedicated test runner for Unit tests. Unit Tests furthermore don't require external resources. If useful, external resources can be mocked in Unit Tests. Target code coverage for Unit Tests should be above 70%.

### Application Tests

Application tests are performed with the built docker image and all external resources must be mocked. They should make sure that the newly built version works well with all its internal dependencies, and that it can start properly.

### System/Application E2E Tests

System/Application E2E tests are performed with the docker image deployed on a staging and have access to external resources. E2E tests should make sure that the newly built version works well together with the existing data and other services. Mutating E2E tests should not run on the `PROD` envronement but only on the `DEV` and `INT` environment.

### System/Application Load and Performance Tests

System/Application load and performance tests are performed with the docker image deployed on a staging and have access to external resources. Those tests should make sure that the newly built version performs well together with the existing data and other services under normal and high load. They should never run on `PROD` staging, and mostly only on `INT` with the same scaling as on `PROD`.

### Manual Tests

Manual tests should be performed after major deploys directly on prod to verify that deployment was successful. Those manual tests are not always required and we should always try to automate them whenever possible.
