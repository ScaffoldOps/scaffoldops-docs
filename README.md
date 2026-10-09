# ScaffoldOps documentation

ScaffoldOps is an MVP platform for asynchronous Spring Boot project generation and explicit Kubernetes deployment. This repository is the main documentation hub. Component READMEs cover their own build, runtime configuration and delivery.

## Guides

- [Platform overview and repository map](architecture.md)
- [Architecture diagram](docs/architecture/high-level-architecture.md)
- [Kafka, generation, deployment and cleanup lifecycles](docs/lifecycles.md)
- [Local development, Minikube, WSL and port-forwarding](docs/local-development.md)
- [Environment variables, Docker Hub and Kubernetes secrets](docs/configuration.md)
- [GitHub Actions and self-hosted runners](docs/delivery.md)
- [API examples](docs/api-usage.md)
- [Operations and troubleshooting](docs/operations.md)
- [Verified configuration gaps and follow-up work](docs/findings.md)

## Documentation maintenance

Review source, application YAML, Kubernetes manifests and workflows before updating behavior descriptions. Mermaid diagrams describe implemented responsibilities; environment gaps are listed explicitly. Commands assume Bash and execution from the named repository root. Replace placeholders locally and keep credentials outside Git. There is no docs build or deployment workflow in this repository; validate relative links, code fences and source references before publishing changes.
