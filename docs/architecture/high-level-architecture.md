# ScaffoldOps High-Level Architecture

## Overview

ScaffoldOps currently has an implemented intake-and-generation MVP, and a defined target state for generation-to-deployment orchestration.

- The current implemented platform includes `generator-api`, `generator-worker`, PostgreSQL, Kafka, Kafka UI, Keycloak, and shared Kubernetes manifests in `platform-infra`.
- The current Kubernetes worker deployment persists generated artifacts with `generator-worker-artifacts-pvc` mounted at `/var/lib/generator-worker`.
- The refined target state adds a future `deployment-worker`, a real Artifact Store, and a durable artifact-reference handoff between generation and deployment.
- `generator-api` remains the lifecycle system of record in both the current and target architecture.

## Current Implemented Architecture

The implemented system today is a hybrid REST + event-driven flow:

- `generator-api` accepts authenticated generation requests.
- `generator-api` persists requests in PostgreSQL with initial status `RECEIVED`.
- `generator-api` publishes `generation-requested` to Kafka.
- `generator-worker` consumes that event and patches generation lifecycle state back to `generator-api`.
- `generator-worker` writes a minimal Spring Boot project and `generation-manifest.json`.
- In Kubernetes, generated output is stored under `/var/lib/generator-worker/manifests` on `generator-worker-artifacts-pvc`.
- When a request is deleted, `generator-api` publishes `artifact-cleanup-requested` and `generator-worker` deletes its matching artifact directory.
- Cleanup is eventually consistent, not transactional with the PostgreSQL delete; if `generator-worker` is down, cleanup waits until Kafka is consumed.

### Current System Context

```mermaid
flowchart LR
    user[Platform User or Client]
    keycloak[Keycloak]
    api[generator-api]
    db[(PostgreSQL)]
    kafka[(Kafka)]
    worker[generator-worker]
    artifacts[(generator-worker-artifacts-pvc)]
    ui[Kafka UI]
    infra[platform-infra]

    user -->|Authenticate| keycloak
    user -->|Bearer token + generation request| api
    api -->|Persist request + status RECEIVED| db
    api -->|Publish generation-requested| kafka
    worker -->|Consume generation-requested| kafka
    worker -->|Generate Spring Boot project| artifacts
    worker -->|PATCH lifecycle: GENERATING / GENERATED / FAILED| api
    api -->|Persist lifecycle callback| db
    api -->|Publish artifact-cleanup-requested on delete| kafka
    worker -->|Consume cleanup + delete matching directory| artifacts
    ui -->|Inspect topics and messages| kafka

    infra -. provides shared infra definitions .-> db
    infra -. provides shared infra definitions .-> kafka
    infra -. provides shared infra definitions .-> keycloak
    infra -. provides shared infra definitions .-> ui
```

## Refined Target-State Architecture

The target state extends the current platform without changing core ownership:

- `generator-api` continues to own request persistence and lifecycle status.
- `generator-worker` keeps the generation engine internally and does not split it into a separate microservice.
- A real Artifact Store replaces the current pod-local/PVC-backed `file://` artifact reference.
- After successful generation, `generator-worker` publishes a `deployment-requested` event as the deployment handoff.
- `deployment-worker` consumes that event and deploys the generated artifact or manifest bundle to Kubernetes.
- Clients continue to read lifecycle state only from `generator-api`.

Current ownership boundaries:

- `generator-api` owns request lifecycle state and the deletion API.
- `generator-worker` owns generated artifacts and PVC cleanup.
- `generator-api` must not access the worker PVC directly.

### Target-State Sequence Diagram

The diagram below is the target-state architecture flow. It describes the intended end-to-end generation-to-deployment lifecycle and is not a claim that the deployment stage is already implemented in the current platform.

`docs/uml/scaffoldops-flow.puml` remains the source of truth for this flow. This Mermaid rendering exists so the diagram is visible in markdown-based documentation where native PlantUML rendering is not configured.

[PlantUML source: `docs/uml/scaffoldops-flow.puml`](/home/victor/workspace/ScaffoldOps/scaffoldops-docs/docs/uml/scaffoldops-flow.puml)

```mermaid
sequenceDiagram
    actor Client
    participant Keycloak
    participant API as generator-api
    participant DB as PostgreSQL
    participant GenTopic as Kafka topic: generation-requested
    participant GenWorker as generator-worker
    participant ArtifactStore as Artifact Store
    participant DepTopic as Kafka topic: deployment-requested
    participant DepWorker as deployment-worker
    participant K8s as Kubernetes

    Client->>Keycloak: Authenticate
    Keycloak-->>Client: JWT

    Client->>API: Create generation request
    API->>DB: Save request (RECEIVED)
    API->>GenTopic: Publish generation-requested
    API-->>Client: Request accepted

    GenTopic->>GenWorker: Consume generation-requested
    GenWorker->>API: Update status (GENERATING)
    API->>DB: Persist GENERATING

    GenWorker->>ArtifactStore: Generate artifact/manifests
    ArtifactStore-->>GenWorker: artifactRef

    GenWorker->>API: Update status (GENERATED)
    API->>DB: Persist GENERATED

    GenWorker->>DepTopic: Publish deployment-requested
    Note over GenWorker,DepTopic: requestId, artifactRef, target

    DepTopic->>DepWorker: Consume deployment-requested
    DepWorker->>API: Update status (DEPLOYING)
    API->>DB: Persist DEPLOYING

    DepWorker->>K8s: Deploy artifact/manifests
    alt deployment success
        K8s-->>DepWorker: Ready
        DepWorker->>API: Update status (DEPLOYED)
        API->>DB: Persist DEPLOYED
    else deployment failure
        K8s-->>DepWorker: Error
        DepWorker->>API: Update status (FAILED)
        API->>DB: Persist FAILED
    end

    Client->>API: Get request status
    API->>DB: Read current status
    API-->>Client: Current lifecycle state
```

### Target-State System Context

```mermaid
flowchart LR
    user[Platform User or Client]
    keycloak[Keycloak]
    api[generator-api]
    db[(PostgreSQL)]
    kafka[(Kafka)]
    genworker[generator-worker]
    depworker[deployment-worker]
    artifacts[(Artifact Store or Repo)]
    k8s[Kubernetes]
    ui[Kafka UI]
    infra[platform-infra]

    user -->|Authenticate| keycloak
    user -->|Bearer token + generation request| api
    api -->|Persist request + status RECEIVED| db
    api -->|Publish generation-requested| kafka

    genworker -->|Consume generation-requested| kafka
    genworker -->|Run internal generation engine| artifacts
    genworker -->|Lifecycle update: GENERATING / GENERATED / FAILED| api
    genworker -->|Publish deployment-requested| kafka

    depworker -->|Consume deployment-requested| kafka
    depworker -->|Deploy artifact or manifests| k8s
    depworker -->|Lifecycle update: DEPLOYING / DEPLOYED / FAILED| api

    api -->|Persist lifecycle state| db
    ui -->|Inspect topics and messages| kafka

    infra -. provides shared infra definitions .-> db
    infra -. provides shared infra definitions .-> kafka
    infra -. provides shared infra definitions .-> keycloak
    infra -. provides shared infra definitions .-> ui
```

## Responsibility Boundaries

### `generator-api`

- Accepts generation requests through REST.
- Validates incoming payloads.
- Persists request records in PostgreSQL.
- Sets the initial lifecycle state to `RECEIVED`.
- Publishes `generation-requested`.
- Exposes request retrieval endpoints.
- Owns persisted lifecycle state for the full request lifecycle.

### `generator-worker`

- Consumes `generation-requested`.
- Orchestrates generation asynchronously.
- Updates lifecycle through `generator-api` for generation-stage progress.
- Runs the generation engine internally.
- Produces a generated project reference on success.
- Uses `/var/lib/generator-worker/manifests` in Kubernetes, backed by `generator-worker-artifacts-pvc`.
- Cleans up generated artifact directories after `artifact-cleanup-requested`.
- Publishes `deployment-requested` after generation succeeds.

### `deployment-worker`

- Future-state component, not implemented yet.
- Consumes `deployment-requested`.
- Orchestrates deployment asynchronously.
- Deploys the generated output to Kubernetes.
- Updates lifecycle through `generator-api` for deployment-stage progress.

### PostgreSQL

- Stores request records and lifecycle status owned by `generator-api`.

### Kafka

- Carries asynchronous handoff events between lifecycle stages.
- Current topic: `generation-requested`.
- Current cleanup topic: `artifact-cleanup-requested`.
- Target-state additional topic: `deployment-requested`.

### Artifact Store or Repo

- Future target-state component; the current MVP uses `generator-worker-artifacts-pvc`, not a real Artifact Store.
- Holds the durable output from generation once the deployment stage exists.
- Provides the reference passed from `generator-worker` to `deployment-worker`.
- Must be treated as a required architectural boundary in the target state rather than an implicit in-memory handoff.

## Target-State Flow

1. A client authenticates with Keycloak and submits a generation request to `generator-api`.
2. `generator-api` persists the request in PostgreSQL with status `RECEIVED`.
3. `generator-api` publishes `generation-requested`.
4. `generator-worker` consumes the event and updates `generator-api` to `GENERATING`.
5. `generator-worker` runs its internal generation engine and writes output to a durable artifact location.
6. On successful generation, `generator-worker` publishes `deployment-requested` with `requestId`, artifact reference, and deployment target.
7. `generator-worker` updates `generator-api` to `GENERATED`.
8. `deployment-worker` consumes `deployment-requested` and updates `generator-api` to `DEPLOYING`.
9. `deployment-worker` deploys the generated output to Kubernetes.
10. `deployment-worker` updates `generator-api` to `DEPLOYED` on success or `FAILED` on deployment-stage failure.
11. The client reads the latest lifecycle state from `generator-api`.

## Key Contracts To Preserve

- `generator-api` is the system of record for lifecycle status.
- `generator-worker` owns generation execution, not persistent lifecycle ownership.
- `deployment-worker` owns deployment execution, not persistent lifecycle ownership.
- `generation-requested` is the intake-to-generation handoff.
- `deployment-requested` is the generation-to-deployment handoff.
- The generation-to-deployment handoff must carry a durable artifact reference, not raw generated content.

## Practical Incremental Notes

- The current code already supports `RECEIVED`, `GENERATING`, `GENERATED`, `DEPLOYING`, `DEPLOYED`, and `FAILED` in the API contract.
- The current deployed workflow implements intake plus generation to `GENERATED`.
- The current `artifactRef` is a pod-local `file://` URI under `/var/lib/generator-worker/manifests`.
- The PVC preserves generated files across `generator-worker` pod recreation, but it is not a replacement for a real Artifact Store.
- Artifact cleanup is asynchronous and idempotent over `artifact-cleanup-requested`.
- The target state should also add idempotency handling, retry strategy, and durable event publication around the generation-to-deployment handoff.

For the target-state sequence source, see [PlantUML source: `docs/uml/scaffoldops-flow.puml`](/home/victor/workspace/ScaffoldOps/scaffoldops-docs/docs/uml/scaffoldops-flow.puml).
