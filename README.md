# ScaffoldOps Docs

Cross-repository architecture documentation for the current local ScaffoldOps workspace and its refined target state.

This repository is the documentation index for the platform code that currently exists in:

- `generator-api`
- `generator-worker`
- `platform-infra`
- `generator-api-actions-runner`
- `generator-worker-actions-runner`
- `platform-infra-actions-runner`

## What ScaffoldOps Is

ScaffoldOps is a platform for accepting microservice generation requests, persisting them, publishing them to Kafka, and processing them asynchronously through worker stages.

Today, the implemented platform is centered on:

- a REST entrypoint service: `generator-api`
- an asynchronous generation-stage worker: `generator-worker`
- shared Kubernetes infrastructure: PostgreSQL, Keycloak, Kafka, Kafka UI, and namespaces
- GitHub Actions pipelines running on self-hosted runners

The target-state architecture now also defines:

- a future `deployment-worker`
- a future `deployment-requested` Kafka topic
- a durable artifact handoff between generation and deployment

The codebase is beyond the original planned MVP, but it is not yet a complete end-to-end scaffold-and-deploy platform. In particular, there is currently no implemented `deployment-worker`, no real project generation engine output, and no real lifecycle callback integration from workers back into the API.

## Architecture References

- Current and target-state architecture narrative: `architecture.md`
- High-level diagrams and responsibility model: `docs/architecture/high-level-architecture.md`
- Refined target-state sequence UML: `docs/uml/scaffoldops-flow.puml`

## Current Platform Topology

### Application repositories

- `generator-api`
  - Spring Boot 3 / Java 17 REST API
  - accepts generation requests
  - validates and stores them in PostgreSQL
  - publishes `generation-requested` Kafka events
  - secures generation endpoints with JWT bearer auth
- `generator-worker`
  - Spring Boot 3 / Java 17 background worker
  - consumes `generation-requested` events from Kafka
  - performs placeholder generation handling
  - emits placeholder lifecycle updates to logging adapters
  - exposes only actuator endpoints

### Target-state application component

- `deployment-worker`
  - not implemented yet
  - will consume `deployment-requested`
  - will deploy generated output to Kubernetes
  - will report deployment-stage lifecycle transitions back through `generator-api`

### Infrastructure repository

- `platform-infra`
  - Kustomize-based Kubernetes source of truth for shared environments
  - manages namespaces, shared PostgreSQL, Keycloak, Kafka, Kafka UI, and Kafka topic bootstrap

### Operational repositories

- `generator-api-actions-runner`
- `generator-worker-actions-runner`
- `platform-infra-actions-runner`

These are local self-hosted GitHub Actions runner installations used by the pipelines in the application and infrastructure repositories. They are operational support assets, not product code.

## End-to-End Flow

The current implemented flow is:

1. A client obtains a JWT from Keycloak for the `scaffoldops` realm.
2. The client calls `POST /api/generator/v1/generation-requests` on `generator-api`.
3. `generator-api` validates the request and stores it in PostgreSQL with status `RECEIVED`.
4. `generator-api` publishes a Kafka message to the `generation-requested` topic.
5. `generator-worker` consumes the Kafka message.
6. `generator-worker` emits placeholder lifecycle transitions.
7. The worker currently logs those lifecycle updates instead of sending them back to a real downstream API.

The refined target-state flow extends that model:

1. `generator-worker` keeps the generation engine internally.
2. On successful generation it writes output to a durable artifact location.
3. `generator-worker` publishes `deployment-requested`.
4. `deployment-worker` consumes that event and deploys to Kubernetes.
5. `generator-api` remains the lifecycle system of record for both generation and deployment stages.

The API status model already includes future-facing values such as `DEPLOYING` and `DEPLOYED`, but the deployed code does not yet implement the deployment stage.

## Repository Map

### `generator-api`

Main responsibilities:

- persistence of generation requests
- JWT-protected REST interface
- OpenAPI-backed contract
- Kafka publication after request creation

Key details:

- base path: `/api/generator/v1`
- public docs: Swagger UI and OpenAPI JSON
- public actuator health endpoints
- Kafka topic default: `generation-requested`
- JWT issuer default: `http://localhost:8091/realms/scaffoldops`

Important references in that repository:

- `README.md`
- `docs/API.md`
- `src/main/resources/openapi/generator-api.yaml`
- `k8s/deployment/`
- `.github/workflows/`

### `generator-worker`

Main responsibilities:

- Kafka consumption
- asynchronous job orchestration shell
- placeholder lifecycle handling
- placeholder project generation adapter

Key details:

- no public business API
- actuator endpoints only
- Kafka consumer group default: `generator-worker`
- topic default: `generation-requested`
- local lifecycle base URL default: `http://localhost:8081`

Important references in that repository:

- `README.md`
- `docs/ARCHITECTURE.md`
- `src/main/resources/application*.yml`
- `k8s/deployment/`
- `.github/workflows/`

### `platform-infra`

Main responsibilities:

- namespace bootstrap
- shared PostgreSQL in `scaffoldops`
- Keycloak in `security`
- Kafka and Kafka UI in `scaffoldops-dev`
- Kafka topic bootstrap job for `generation-requested`

Important references in that repository:

- `README.md`
- `k8s/base/`
- `k8s/overlays/local-dev`
- `k8s/overlays/dev`
- `.github/workflows/platform-infra.yml`

## Environments

The current repositories model two main deployment tracks:

- `develop` branch
  - application repos build, test, publish Docker images, and deploy to `scaffoldops-dev`
  - Spring profile: `dev`
- `main` branch
  - application repos build, test, publish Docker images, and deploy to `scaffoldops-pre`
  - Spring profile: `pre`

Shared infrastructure behaves slightly differently by overlay:

- `k8s/overlays/local-dev`
  - deploys namespaces, PostgreSQL, Keycloak, Kafka, Kafka UI, and topic bootstrap
- `k8s/overlays/dev`
  - deploys namespaces plus Kafka, Kafka UI, and topic bootstrap
  - does not deploy PostgreSQL or Keycloak from this repo

## Namespaces

Namespaces currently visible in the source repos:

- `security`
  - Keycloak
- `scaffoldops`
  - shared PostgreSQL
- `scaffoldops-dev`
  - Kafka, Kafka UI, dev application deployments
- `scaffoldops-pre`
  - targeted by application PRE deployments, but not bootstrapped by the current `platform-infra` base

That last point matters: the application pipelines deploy to `scaffoldops-pre`, while the current `platform-infra` README and manifests are centered on `security`, `scaffoldops`, and `scaffoldops-dev`. This should be treated as a current environment gap or an external prerequisite, not as something already fully managed here.

## Delivery Model

### Application CI/CD

Both `generator-api` and `generator-worker` currently use similar GitHub Actions pipelines:

- PR checks for pull requests to `develop` and `main`
- `develop` pipeline
  - Maven quality step
  - Maven test step
  - Docker image build
  - Docker image push
  - Kubernetes deploy to `scaffoldops-dev`
- `main` pipeline
  - Maven quality step
  - Maven test step
  - Docker image build
  - Docker image push
  - Kubernetes deploy to `scaffoldops-pre`

### Infra CI/CD

`platform-infra` currently:

- validates rendered manifests on pull requests and `main`
- supports workflow-dispatch deployment of the `local-dev` overlay

## Current State Summary

Implemented:

- request submission API
- PostgreSQL persistence for requests
- JWT resource server configuration
- Kafka publication from API
- Kafka consumption in worker
- worker lifecycle state shell
- Kubernetes manifests for API, worker, PostgreSQL, Keycloak, Kafka, and Kafka UI
- self-hosted GitHub Actions based delivery pipelines

Not implemented yet:

- real scaffold generation output
- lifecycle callback/update path back into `generator-api`
- deployment worker
- actual deployment stage execution
- retry, dead-letter, or idempotency workflow in the worker
- ingress, API gateway, or external managed infra

## Related Document

For the implementation-aligned system architecture, see [architecture.md](architecture.md).
