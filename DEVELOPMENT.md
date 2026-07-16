# Development guidelines

These guidelines should be followed for code to be used in production. PoCs can decide not to follow all of the guidelines, however, if they are intended to be used in production, extra work needs to be done to meet production quality.
The following topics are covered:

- [1. Version Control of Sources](#1-version-control-of-sources)
  - [git-flow](#git-flow)
- [2. Coding style](#2-coding-style)
  - [Fail-Fast](#fail-fast)
  - [Environments](#environments)
- [Logs](#logs)
- [3. Testing](#3-testing)
  - [Unit Tests](#unit-tests)
  - [Integration Tests](#integration-tests)
  - [Manual Tests](#manual-tests)

## 1. Version Control of Sources

`git` is used as version control system of source code. Git repositories are hosted on [github.com](github.com). All sorts of code, scripts (bash, SQL, ...) and documentation should be tracked.

**IMPORTANT:** read [Git Flow](GIT_FLOW.md) to learn about the `git` and `PR` workflow.

### git-flow

[Git Flow](GIT_FLOW.md)

## 2. Coding style

Some guidelines that are generic to all languages are noted here. Language specific things can be found here:

[Coding Guidelines](README.md#coding-guidelines)

> [!NOTE]
> Keep in mind that most of the time is spent **reading** code (not writing), yours or from someone else. Hence try to make life easier for the one **reading** and trying to understand your code.

### Fail-Fast

> [!TIP]
> **The goal is not to make any mistakes. The goal is to find and fix them quickly!**


- When there is a missing environment variable or start-up parameter, instead of still starting up the system normally or using a fall-back strategy (fall-back to default environments/parameters), the system should fail and stop so that we can be notified and fix the problem right away.

- When a client sends a request with invalid parameters, instead of silently correcting the parameters and continuing to handle the request, the server should let the request fail so that the client can be notified and fix the problem as soon as possible. Sometimes it can make sense to be permissive on what you accept and restrictive on what you send (i.e. sanitize wrong input if possible).

- Exceptions should never be silently swallowed. Exceptions should only be caught when the catcher knows how to handle it; otherwise, let the exception be thrown outside. And let the app crash if no part of the app knows how to handle it (an exception caused by unexpected bugs). And then fix it.

### Environments

Three environments are available

- *dev*: the development environment is mainly used to test and develop things that either need access to hardware other than the notebook provides or for interface development, where the component under development must be accessible by other systems. Whenever development needs other resources, it should be *dev* resources.
- *int*: the integration or staging environment is used to verify that production-ready changes to a component work as expected together with other resources (in terms of functionality, performance, ...)
- *prod*: everything that other people and systems rely on for doing their work. This can be other internal systems, external services, internal or external users of apps and services.

Development should happen as much as possible locally on the notebook.

## Observability

For observability, each service should use OpenTelemetry logging, tracing and metrics. All telemetry data should be sent or collected by the OTEL collector.

### Logging

Each service must write logs using different log levels. The logs should be by default written in `stdout` in a human readable format. Then whenever possible we should use in parallel OpenTelemetry log exporter that sends directly log in correct OTEL format to the OTEL log collector. Logs are finally centralized in the Elasticsearch/Kibana observability infrastructure.

### Tracing

Each service should use OpenTelemetry tracing to trace the execution of its requests. Traces are sent to the OTEL collector and are finally visualized in the Kibana observability infrastructure. For local development, traces should be sent to a local `otel/opentelemetry-collector-contrib` docker container and visualized in a local Jeager docker container.

### Metrics

Each service should use OpenTelemetry metrics to collect and report metrics about its execution (mostly done by opentelementry instrumentation libraries). Metrics are sent to the OTEL collector and are finally visualized in the Kibana observability infrastructure. For local development, metrics should be sent to a local Prometheus docker container which also provides a simple web UI for visualizing metrics.

## 3. Testing

Testing is an integral part of development and mandatory for production targeted code. This applies to apps as well as scripts. We distinguish three categories of tests:

- unit tests
- integration/end-to-end tests
- manual tests

The first two are automated tests, the third one obviously manual. Language specific details about testing can be found here:

- [python](PYTHON.md#9-unit-testing-frameworks)
- [bash](BASH.md#4-unit-tests--shellspec)
- [javascript](JAVASCRIPT.md#testing)

### Unit Tests

Unit Tests can be performed before a docker image is built using a dedicated test runner for Unit tests. Unit Tests furthermore don't require external resources. If useful, external resources can be mocked in Unit Tests. Target code coverage for Unit Tests should be above 70%.

### Integration/End-to-End Tests

Integration/End-to-End tests are performed with the built docker image and have access to external resources. Integration test should make sure that the newly built version works well together with the existing data and other services. Integration tests should use staging (or integration) environment and not production resources.

### Manual Tests

Manual tests should be performed after major deploys directly on prod to verify that deployment was successful. Those manual tests are not always required and we should always try to automate them whenever possible.
