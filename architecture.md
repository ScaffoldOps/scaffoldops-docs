# ScaffoldOps Architecture

## 1. Purpose

ScaffoldOps is a platform for automated generation and deployment of microservices on Kubernetes. Its purpose is to standardize the creation of microservices from a declarative specification and automate the path from request to deployment.

## 2. Architectural Goals

- Standardize microservice creation
- Reduce repetitive setup work
- Support automated deployment to Kubernetes
- Track lifecycle status of generation and deployment
- Keep the platform extensible for future capabilities

## 3. MVP Scope

The MVP includes:
- generator-api
- generator-worker
- deployment-worker
- Keycloak for authentication
- PostgreSQL for request persistence
- Kubernetes as deployment target

The MVP does not initially include:
- API gateway
- advanced multi-tenancy
- full observability stack
- complex event choreography between generated services

## 4. High-Level Components

### generator-api
Receives generation requests, validates them, stores them, and exposes request status.

### generator-worker
Consumes validated requests and generates the microservice project from a template.

### deployment-worker
Deploys the generated service into Kubernetes and updates lifecycle status.

### Keycloak
Provides authentication and token issuance for platform users.

### PostgreSQL
Stores generation request data and lifecycle state.

### Kafka (future / optional)
Can be used for lifecycle events such as request received, generation started, generation completed, deployment succeeded, or deployment failed.

## 5. High-Level Flow

1. User authenticates with Keycloak
2. User submits a generation request to generator-api
3. generator-api validates and stores the request
4. generator-worker generates the microservice
5. deployment-worker deploys it to Kubernetes
6. generator-api exposes the resulting status to the user

## 6. Namespaces

- `security`: shared security/platform components such as Keycloak
- `scaffoldops`: ScaffoldOps application services
- `energyco`: reserved for a future separate project

## 7. Core Data Model

### GenerationRequest
Fields:
- id
- name
- template
- deploymentTarget
- databaseEnabled
- restApiEnabled
- securityEnabled
- messagingEnabled
- status
- requestedBy
- specJson
- createdAt
- updatedAt

### Status Lifecycle
- RECEIVED
- VALIDATED
- GENERATING
- GENERATED
- DEPLOYING
- DEPLOYED
- FAILED

## 8. Technology Stack

- Java 17
- Spring Boot 3
- Maven
- Hexagonal Architecture
- PostgreSQL
- OpenAPI
- Docker
- Kubernetes
- GitHub Actions
- Keycloak

## 9. Architectural Style

All core ScaffoldOps services follow hexagonal architecture with these layers:
- domain
- application
- infrastructure
- presentation

## 10. Deployment Model

ScaffoldOps services are deployed to Minikube for local development.
- Keycloak runs in namespace `security`
- ScaffoldOps services run in namespace `scaffoldops`

Docker and Minikube are configured to start automatically in the WSL2 development environment.

## 11. Future Evolution

Possible future improvements:
- Kafka-based lifecycle orchestration
- GitOps deployment with Argo CD
- API gateway
- template catalog
- organization/team support
- quotas and ownership rules
- richer deployment strategies
