# ScaffoldOps Architecture

## 1. Scope

This document describes the architecture that is currently implemented across the local ScaffoldOps repositories, not the original aspirational MVP.

Implemented repositories in scope:

- `generator-api`
- `generator-worker`
- `platform-infra`

Operational support repositories referenced for delivery:

- `generator-api-actions-runner`
- `generator-worker-actions-runner`
- `platform-infra-actions-runner`

## 2. Architectural Position

ScaffoldOps is currently an event-driven request intake and worker-processing platform with shared Kubernetes infrastructure.

At a high level:

- `generator-api` is the synchronous request entrypoint
- PostgreSQL is the system of record for accepted requests
- Kafka is the async handoff between request intake and worker execution
- `generator-worker` is the asynchronous execution shell
- Keycloak provides JWT issuance for protected API access
- Kubernetes manifests and environment overlays are managed in `platform-infra`


## 3. Implemented Components

### 3.1 `generator-api`

Responsibilities:

- expose REST endpoints for generation requests
- validate request payloads
- persist request records in PostgreSQL
- publish a `generation-requested` Kafka event after persistence
- expose request lookup endpoints
- enforce JWT bearer authentication on business endpoints

Non-responsibilities in the current code:

- no project generation execution
- no deployment execution
- no inbound lifecycle update API from the worker yet

Architectural style:

- hexagonal / ports-and-adapters
- dependency direction:
  - `presentation -> application -> domain`
  - `infrastructure -> application -> domain`

Key runtime contract:

- API base path: `/api/generator/v1`
- request creation endpoint: `POST /generation-requests`
- request lookup endpoint: `GET /generation-requests/{id}`
- request list endpoint: `GET /generation-requests`
- Kafka topic default: `generation-requested`

### 3.2 `generator-worker`

Responsibilities:

- consume `generation-requested` Kafka events
- map inbound events into internal generation jobs
- orchestrate placeholder generation flow
- emit placeholder lifecycle updates via an outbound port

Non-responsibilities in the current code:

- no generated project artifacts yet
- no persistence
- no outbound HTTP lifecycle delivery yet
- no deployment execution

Architectural style:

- ports-and-adapters
- dependency direction:
  - `infrastructure -> application -> domain`

Inbound runtime contract:

- Kafka listener topic: `app.kafka.topics.generation-requested`
- default topic: `generation-requested`
- default consumer group: `generator-worker`

Operational surface:

- no public business API
- actuator only

### 3.3 Shared PostgreSQL

Managed by `platform-infra` in namespace `scaffoldops`.

Current role:

- persistence backend for `generator-api`
- shared database host for multiple logical service databases

The bootstrap init script creates at least these logical databases:

- `keycloakdb`
- `generatorapidb`
- `generatorworkerdb`
- `deploymentworkerdb`

The presence of `deploymentworkerdb` indicates future support planning, not an implemented deployment worker service.

### 3.4 Keycloak

Managed by `platform-infra` in namespace `security`.

Current role:

- identity provider for the `scaffoldops` realm
- JWT issuer consumed by `generator-api`

Keycloak is a supporting platform dependency, not part of the application domain logic.

### 3.5 Kafka and Kafka UI

Managed by `platform-infra` in namespace `scaffoldops-dev`.

Current role:

- asynchronous handoff from `generator-api` to `generator-worker`
- operational inspection via Kafka UI
- infra-owned topic bootstrap for `generation-requested`

Current topology:

- single-node Kafka KRaft broker
- single topic bootstrap job
- one topic known to be managed here: `generation-requested`

## 4. Request Processing Flow

### 4.1 Synchronous intake

1. A client authenticates against Keycloak and obtains a bearer token.
2. The client calls `generator-api`.
3. `generator-api` validates the request payload.
4. `generator-api` stores the request in PostgreSQL.
5. The request is persisted with status `RECEIVED`.
6. `generator-api` publishes a Kafka event to `generation-requested`.

### 4.2 Asynchronous worker path

1. `generator-worker` consumes the Kafka event.
2. The event is mapped into an internal `GenerationJob`.
3. The worker emits a `GENERATING` lifecycle update through its lifecycle port.
4. The worker invokes its generation port.
5. On success, the worker emits `GENERATED`.
6. On failure, the worker emits `FAILED` and rethrows.

### 4.3 Current architectural break in the loop

The worker's lifecycle and generation adapters are still placeholders:

- lifecycle updates are logged, not delivered to a real endpoint
- generation work is logged, not executed into files or repositories

As a result, the system is currently strongest at intake, persistence, and asynchronous dispatch, but incomplete at real generation completion and downstream status convergence.

## 5. Data and Event Model

### 5.1 Generation request model

The current API contract includes fields such as:

- `id`
- `name`
- `template`
- `deploymentTarget`
- `database`
- `restApi`
- `security`
- `messaging`
- `status`
- `createdAt`
- `updatedAt`

### 5.2 Status model

The status values present in `generator-api` are:

- `RECEIVED`
- `GENERATING`
- `GENERATED`
- `DEPLOYING`
- `DEPLOYED`
- `FAILED`

Current implementation note:

- `RECEIVED` is set by `generator-api`
- `GENERATING`, `GENERATED`, and `FAILED` are used inside `generator-worker`
- `DEPLOYING` and `DEPLOYED` are contract-level future states, not part of a current deployed workflow

### 5.3 Kafka event

The worker consumes a JSON event compatible with `GenerationRequestedEvent`.

That event carries request metadata such as:

- `requestId`
- `name`
- `template`
- feature booleans
- `deploymentTarget`
- `status`
- `createdAt`

The worker currently ignores the inbound `status` field for control flow.

## 6. Security Model

### 6.1 API protection

`generator-api` is configured as a Spring Security OAuth2 resource server using JWT validation.

Business endpoints require a bearer token:

- `POST /generation-requests`
- `GET /generation-requests`
- `GET /generation-requests/{id}`

Public endpoints:

- actuator health endpoints
- Swagger UI
- OpenAPI JSON

### 6.2 Trust boundary

The current trust boundary is centered on the API:

- external clients are expected to enter through `generator-api`
- worker processing is internal
- Kafka and PostgreSQL are internal platform dependencies

## 7. Deployment Architecture

### 7.1 Repository ownership

Deployment ownership is split across repositories:

- `platform-infra`
  - owns shared infrastructure manifests
- `generator-api`
  - owns API deployment and service manifests
- `generator-worker`
  - owns worker deployment and service manifests

This keeps application deployment configuration colocated with the service code while centralizing shared platform dependencies.

### 7.2 Namespaces

Current namespace model visible in source control:

- `security`
- `scaffoldops`
- `scaffoldops-dev`
- `scaffoldops-pre` as an application deployment target

Observed nuance:

- `platform-infra` bootstraps `security`, `scaffoldops`, and `scaffoldops-dev`
- application workflows deploy to `scaffoldops-dev` and `scaffoldops-pre`
- `scaffoldops-pre` is therefore not fully described by the current infra repository state

### 7.3 Overlays

`platform-infra` overlays currently behave as follows:

- `local-dev`
  - renders namespaces, PostgreSQL, Keycloak, Kafka, Kafka UI, and topic bootstrap
- `dev`
  - renders namespaces plus Kafka-related resources only

This means local development and shared dev are not identical. Local dev includes more infra from this repo than the dev overlay does.

## 8. CI/CD Architecture

### 8.1 Application pipelines

Both application repositories follow the same broad delivery pattern:

- PR checks on pull requests to `develop` and `main`
- branch pipelines on `develop`
  - quality
  - tests
  - Docker build
  - Docker push
  - deploy to `scaffoldops-dev`
- branch pipelines on `main`
  - quality
  - tests
  - Docker build
  - Docker push
  - deploy to `scaffoldops-pre`

Deployment mechanics:

- Docker images are tagged with `${project.version}-${shortSha}`
- branch tags are also published (`develop` or `main`)
- deployment workflows mutate image tag and Spring profile at deploy time

### 8.2 Infrastructure pipeline

`platform-infra` currently:

- renders the `local-dev` overlay during validation
- performs client-side manifest validation
- supports manual deployment of `local-dev` through workflow dispatch

### 8.3 Runner model

The `*-actions-runner` repositories are local self-hosted GitHub Actions runner installations used to execute the above workflows.

Architecturally, these are part of the delivery platform rather than the runtime platform.

## 9. Technology Stack

Current stack across the implemented repos:

- Java 17
- Spring Boot 3
- Maven
- Spring Web
- Spring Validation
- Spring Data JPA
- Spring Security OAuth2 Resource Server
- Spring Kafka
- PostgreSQL
- Kafka
- Keycloak
- Docker
- Kubernetes
- Kustomize
- GitHub Actions on self-hosted runners

## 10. Architectural Gaps and Current Risks

### 10.1 Placeholder worker adapters

The most important functional gap is that `generator-worker` still uses placeholder outbound adapters. The async path exists structurally, but not functionally end to end.

### 10.2 Status divergence risk

The API owns persisted request status, but the worker does not yet push real lifecycle updates back into the API. That creates a risk that persisted state and actual worker progress diverge.

### 10.3 Environment definition mismatch

Application pipelines target `scaffoldops-pre`, while `platform-infra` documentation and bootstrap manifests are more explicit about `security`, `scaffoldops`, and `scaffoldops-dev`. PRE environment ownership is therefore incomplete in the current docs and infra source.

### 10.4 Deployment stage is contract-only

The status model and bootstrap database names still reflect a future deployment stage, but there is no implemented deployment worker or deployment orchestration path in the active repositories.

## 11. Near-Term Evolution Path

The codebase is currently positioned for these next steps:

1. replace the worker's no-op generation adapter with a real scaffold generation engine
2. implement a real lifecycle update adapter from worker back to API
3. decide whether deployment remains a separate worker or is folded into a different orchestration model
4. align environment ownership so PRE is explicitly managed and documented
5. add retry, dead-letter, and idempotency controls around Kafka processing

## 12. Bottom Line

ScaffoldOps is currently a partially implemented internal platform with a solid synchronous intake layer, a real asynchronous Kafka handoff, shared Kubernetes infrastructure, and working delivery pipelines.

Its strongest implemented areas are:

- API contract and persistence
- Kafka-based decoupling
- environment deployment automation

Its weakest implemented areas are:

- real generation execution
- status convergence after async processing
- deployment-stage implementation
