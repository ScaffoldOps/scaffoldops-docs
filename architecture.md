# ScaffoldOps Architecture

## 1. Scope

This document describes both:

- the architecture that is currently implemented across the local ScaffoldOps repositories
- the refined target-state architecture represented by the current UML in `docs/uml/scaffoldops-flow.puml`

Implemented repositories in scope today:

- `generator-api`
- `generator-worker`
- `platform-infra`

Operational support repositories referenced for delivery:

- `generator-api-actions-runner`
- `generator-worker-actions-runner`
- `platform-infra-actions-runner`

Future-state component in scope for target design:

- `deployment-worker`

## 2. Architectural Position

ScaffoldOps is an event-driven request intake and worker-processing platform with shared Kubernetes infrastructure.

At a high level:

- `generator-api` is the synchronous request entrypoint
- PostgreSQL is the lifecycle system-of-record backend used by `generator-api`
- Kafka is the asynchronous handoff bus between processing stages
- `generator-worker` owns generation execution
- generated artifacts are stored under `/var/lib/generator-worker/manifests`; in Kubernetes this path is backed by `generator-worker-artifacts-pvc`
- generated artifact cleanup is asynchronous over Kafka topic `artifact-cleanup-requested`
- `deployment-worker` is the future deployment-stage worker
- Keycloak provides JWT issuance for protected API access
- Kubernetes manifests and overlays are managed in `platform-infra`

## 3. Current Implemented Components

### 3.1 `generator-api`

Current responsibilities:

- expose REST endpoints for generation requests
- validate request payloads
- persist request records in PostgreSQL
- set initial status to `RECEIVED`
- publish a `generation-requested` Kafka event after persistence
- publish an `artifact-cleanup-requested` Kafka event after deleting a request
- expose request lookup endpoints
- accept internal generation lifecycle callbacks from `generator-worker`
- enforce JWT bearer authentication on business endpoints

Not implemented yet in current code:

- no project generation execution
- no deployment execution

### 3.2 `generator-worker`

Current responsibilities:

- consume `generation-requested` Kafka events
- map inbound events into internal generation jobs
- patch `generator-api` to `GENERATING`, `GENERATED`, or `FAILED`
- generate a deterministic Spring Boot Hello World project
- write `pom.xml`, `Dockerfile`, Kubernetes manifests, Java source files, and `generation-manifest.json`
- expose generated artifacts through a `file://` `artifactRef`
- mount `generator-worker-artifacts-pvc` at `/var/lib/generator-worker` in the Kubernetes DEV deployment
- consume `artifact-cleanup-requested` and delete matching artifact directories under the configured manifest output directory

Not implemented yet in current code:

- no real Artifact Store such as MinIO/S3 or artifact repository yet
- no deployment execution

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

`deploymentworkerdb` exists as infrastructure preparation, not as evidence of a deployed `deployment-worker`.

### 3.4 Keycloak

Managed by `platform-infra` in namespace `security`.

Current role:

- identity provider for the `scaffoldops` realm
- JWT issuer consumed by `generator-api`

### 3.5 Kafka and Kafka UI

Managed by `platform-infra` in namespace `scaffoldops-dev`.

Current role:

- asynchronous handoff from `generator-api` to `generator-worker`
- operational inspection via Kafka UI
- infra-owned topic bootstrap for `generation-requested`
- infra-owned topic bootstrap for `artifact-cleanup-requested`

Current known infra-managed topic:

- `generation-requested`
- `artifact-cleanup-requested`

Application code may also publish or consume additional topics as the workflow
evolves; shared infra currently bootstraps the generation intake and artifact
cleanup topics.

## 4. Current Implemented Flow

### 4.1 Synchronous intake

1. A client authenticates against Keycloak and obtains a bearer token.
2. The client calls `generator-api`.
3. `generator-api` validates the request payload.
4. `generator-api` stores the request in PostgreSQL.
5. The request is persisted with status `RECEIVED`.
6. `generator-api` publishes a Kafka event to `generation-requested`.

### 4.2 Asynchronous generation stage

1. `generator-worker` consumes the Kafka event.
2. The event is mapped into an internal generation job.
3. The worker patches `generator-api` to `GENERATING`.
4. The worker generates a minimal Spring Boot project.
5. The worker writes the project under `/var/lib/generator-worker/manifests/<serviceName>-<requestId>/` in Kubernetes.
6. The Kubernetes deployment persists `/var/lib/generator-worker` on `generator-worker-artifacts-pvc`.
7. On success, the worker patches `generator-api` to `GENERATED` with `artifactRef`.
8. On failure, the worker patches `generator-api` to `FAILED` and rethrows.

### 4.3 Asynchronous artifact cleanup

1. A client deletes a generation request through `generator-api`.
2. `generator-api` reads the existing request data, deletes the PostgreSQL row, and publishes `artifact-cleanup-requested`.
3. `generator-worker` consumes the cleanup event.
4. `generator-worker` derives the artifact directory from the stored request id and service name.
5. `generator-worker` deletes only below the configured manifest output directory.
6. If the directory is already absent, cleanup is logged and treated as success.

In flow form:

```text
DELETE /generation-requests/{id}
  -> generator-api
  -> Kafka topic artifact-cleanup-requested
  -> generator-worker
  -> PVC directory deletion
```

Cleanup is eventually consistent, not transactional with the PostgreSQL delete.
If `generator-worker` is down, cleanup waits until Kafka is consumed. Full
reconciliation of stuck cleanup or generation states remains future work.

### 4.4 Current artifact persistence boundary

The current MVP artifact reference is pod-local:

```text
file:///var/lib/generator-worker/manifests/<serviceName>-<requestId>/
```

The PVC means artifacts survive `generator-worker` pod recreation, but this is
not a real Artifact Store. There is no object storage, download API, artifact
repository, or cross-cluster artifact contract yet.

Generated artifacts currently include `pom.xml`, `Dockerfile`, Kubernetes
manifests, `HelloApplication.java`, `HelloController.java`, and
`generation-manifest.json`.

Ownership boundary:

- `generator-api` owns request lifecycle state and the deletion API.
- `generator-worker` owns generated artifacts and PVC cleanup.
- `generator-api` must not access the worker PVC directly.

## 5. Refined Target-State Architecture

The target state keeps the current separation of concerns and extends it into a full generation-to-deployment flow.

### 5.1 Responsibility model

#### `generator-api`

- remains the lifecycle system of record
- persists the canonical request state
- owns client-facing request retrieval
- publishes `generation-requested`
- accepts lifecycle updates from workers through an internal contract

#### `generator-worker`

- consumes `generation-requested`
- keeps the generation engine internally
- updates generation-stage lifecycle through `generator-api`
- produces a generated project reference
- publishes `deployment-requested` after successful generation

#### `deployment-worker`

- consumes `deployment-requested`
- deploys the generated output to Kubernetes
- updates deployment-stage lifecycle through `generator-api`

### 5.2 Target-state stage handoff

The generation-to-deployment boundary must eventually include a durable handoff artifact outside the worker pod filesystem.

That means the design should explicitly include:

- an artifact store, repository, or other durable output location such as MinIO/S3 or an artifact repository
- an artifact reference in `deployment-requested`
- request identity and target deployment metadata in both Kafka contracts

The artifact itself should not be passed through Kafka.

### 5.3 Target-state lifecycle ownership

Persisted lifecycle ownership stays with `generator-api`.

Workers do not become systems of record for partial status ranges. Instead:

- `generator-worker` owns generation execution
- `deployment-worker` owns deployment execution
- `generator-api` owns persisted status for the whole request lifecycle

## 6. Target-State Flow

1. Client authenticates with Keycloak.
2. Client calls `POST /generation-requests` on `generator-api`.
3. `generator-api` validates the request, saves it with status `RECEIVED`, and publishes `generation-requested`.
4. `generator-worker` consumes `generation-requested`.
5. `generator-worker` updates `generator-api` to `GENERATING`.
6. `generator-worker` runs the internal generation engine and writes output to a durable artifact location.
7. `generator-worker` publishes `deployment-requested` with `requestId`, artifact reference, and deployment target.
8. `generator-worker` updates `generator-api` to `GENERATED`.
9. `deployment-worker` consumes `deployment-requested`.
10. `deployment-worker` updates `generator-api` to `DEPLOYING`.
11. `deployment-worker` deploys the referenced output to Kubernetes.
12. `deployment-worker` updates `generator-api` to `DEPLOYED` on success or `FAILED` on deployment-stage failure.
13. Client reads the current lifecycle state from `generator-api`.

## 7. Data and Event Model

### 7.1 Request model

The current API contract already includes fields such as:

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

### 7.2 Status model

The current API enum contains:

- `RECEIVED`
- `GENERATING`
- `GENERATED`
- `DEPLOYING`
- `DEPLOYED`
- `FAILED`

Status ownership intent:

- `RECEIVED` is set by `generator-api`
- `GENERATING` and `GENERATED` are reported by `generator-worker` but persisted by `generator-api`
- `DEPLOYING` and `DEPLOYED` are reported by `deployment-worker` but persisted by `generator-api`
- `FAILED` may occur in either stage and therefore requires stage-qualified detail in the lifecycle update payload or audit trail

### 7.3 Kafka contracts

Current implemented event:

- `generation-requested`

Target-state additional event:

- `deployment-requested`

Both stage-handoff events should include, at minimum:

- `eventId`
- `requestId`
- `occurredAt`
- `deploymentTarget`
- `artifactRef` where applicable
- enough request metadata to process the stage without rehydrating from Kafka history

## 8. Architectural Constraints

The current UML and target state assume these constraints:

- `deployment-worker` does not exist yet
- `generator-worker` keeps the generation engine internally and is not decomposed into a separate generation microservice
- current Kubernetes artifact persistence is PVC-backed and uses `file://` references, not a real Artifact Store
- target-state generation-to-deployment handoff requires a real Artifact Store or equivalent durable artifact boundary
- `generator-worker` publishes a deployment event after successful generation when deployment handoff is enabled
- `deployment-worker` consumes that event and deploys to Kubernetes
- `generator-api` remains the lifecycle system of record

## 9. Risks To Address In Implementation

The target-state design should explicitly account for:

- duplicate Kafka deliveries
- idempotent processing by both workers
- failure after generation succeeds but before `deployment-requested` is durably published
- failure after deployment succeeds but before lifecycle status is persisted back in `generator-api`
- durable artifact-reference handoff between generation and deployment
- race handling if a request is deleted while an older generation event is still being processed
- migration from pod-local `file://` artifact references to an Artifact Store reference usable by future deployment workers

## 10. Documentation Guidance

When updating repository docs from this point forward:

- document current implementation separately from target-state design
- do not describe `deployment-worker` as implemented until the repository and deployment assets exist
- do not describe the current PVC as a real Artifact Store
- do not model the generation engine as a standalone service
- keep `generator-api` as the documented lifecycle source of truth
- keep `deployment-requested` and artifact handoff explicit in architecture diagrams once target-state material is discussed
