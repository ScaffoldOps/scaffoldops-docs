# Platform overview

ScaffoldOps accepts a microservice generation request with a selected template, creates a minimal Spring Boot project asynchronously, stores its ZIP in MinIO and pushes its image to Docker Hub. Users then explicitly deploy or undeploy it in Kubernetes. Permanent delete requests asynchronous cleanup.

## Repository map

| Repository | Responsibility |
| --- | --- |
| [generator-api](https://github.com/ScaffoldOps/generator-api) | REST, JWT validation, PostgreSQL lifecycle state, Kafka publication, image recovery scheduler |
| [generator-worker](https://github.com/ScaffoldOps/generator-worker) | Project generation, MinIO artifact upload, Docker image build/push, asset cleanup |
| [deployment-worker](https://github.com/ScaffoldOps/deployment-worker) | Kubernetes Deployments/Services, rollout, undeploy and resource cleanup |
| [platform-infra](https://github.com/ScaffoldOps/platform-infra) | Shared Kustomize infrastructure and bootstrap Jobs |
| [scaffoldops-docs](https://github.com/ScaffoldOps/scaffoldops-docs) | Platform documentation |

Local `*-actions-runner` directories are runner installations, not product repositories.

## Current MVP scope

The three applications use Java 17 and Spring Boot. API persistence is PostgreSQL; workers report status through authenticated HTTP callbacks rather than owning the API database. Infrastructure also creates worker databases, but neither worker currently uses them.

The generator records the selected template and feature flags, but the current generation adapter writes one minimal Hello World Spring Boot project with web dependency, `/hello`, Dockerfile, Kubernetes manifests and `generation-manifest.json`. It does not implement different template engines or requested database/security/messaging modules. These fields describe requested intent, not a guarantee of generated capability.

DEV worker manifests select MinIO, while host defaults retain a filesystem backend. Kubernetes output is backed by `generator-worker-artifacts-pvc`; the PVC is a workspace, while MinIO stores the published ZIP. Deployment-worker uses the image reference directly and does not download or apply the ZIP's Kubernetes manifests.

The MVP uses a single Kafka broker, PostgreSQL instance and MinIO instance. It does not provide HA, a transactional outbox, durable external-cleanup tracking or a fully provisioned PRE environment. A checked-in API Ingress exists separately; shared infrastructure does not install an ingress controller or provide external routing automatically.

See [architecture](docs/architecture/high-level-architecture.md), [lifecycles](docs/lifecycles.md) and [implementation gaps](docs/findings.md).
