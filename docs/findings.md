# Verified implementation gaps and proposed follow-ups

These findings reflect checked-in source/manifests, not an inspection of live cluster state. Documentation changes do not alter runtime behavior.

| Finding | Evidence | Proposed follow-up |
| --- | --- | --- |
| DEV PostgreSQL aliases cannot select shared pods | `platform-infra/k8s/base/database-dev-aliases/postgres*.yaml` use selectors in `scaffoldops-dev`; PostgreSQL runs in `scaffoldops` | Use cross-namespace DNS/ExternalName or explicit DB URL for API and Keycloak; selectors do not span namespaces |
| API DB secret not provisioned in DEV by shared base | DB Kustomization puts `postgres-credentials` in `scaffoldops`; API references it in app namespace | Provision app password secret in DEV independently |
| deployment-worker API URL incomplete | `worker.yaml` and defaults use `http://generator-api:8080`; API Service is port 80 and alias is port 8081; API servlet path is required | Set `GENERATOR_API_BASE_URL=http://generator-api-service/api/generator/v1` |
| Direct deployment-worker image differs from CI | Manifest `scaffoldops/deployment-worker:latest`; CI `victodomvar/scaffoldops-deployment-worker` | Align direct manifest; workflow already overrides image |
| Docker cleanup env mapping under wrong prefix | generator-worker `application.yml` nests cleanup under `info`, adapter reads `app.cleanup.docker-hub.*` | Move mapping to `app.cleanup`; until then bind application properties directly |
| Generated pods omit pull secrets | `KubernetesResourceFactory` sets containers only | Support configured pull secret or provision target ServiceAccount authentication for private images |
| PRE not fully provisioned | Infra has no `k8s/overlays/pre`; API PRE issuer references `keycloak`/realm `scaffoldops`; worker Kustomization fixed to DEV | Define consistent PRE infra, issuer and namespace-specific app overlays |
| Only two topics explicitly bootstrapped | Kafka Job creates generation and cleanup; broker auto-creation supplies others | Bootstrap all lifecycle/recovery/DLT topics explicitly if auto-creation is disabled |
| Keycloak realm/client provisioning manual | No realm import in infra manifests | Add reproducible realm/client bootstrap with secrets managed externally |
| Generated Kubernetes image naming differs from builder | Generator template uses `<repositoryPrefix>/<serviceName>:<UUID>`; image builder publishes `<repositoryPrefix>:<serviceName>-<UUID>` | Align generated manifests with actual imageRef; deployment-worker uses imageRef directly |
| Template/features exceed generator behavior | `ManifestWritingProjectGenerationAdapter` writes fixed minimal project and metadata | Implement template/capability handling or narrow product contract |
| Cleanup lacks durable external result/retry | Generator cleanup catches/logs individual external failures | Persist outcomes and add operator reconciliation/retry |
| PostgreSQL/Kafka publication not atomic | Lifecycle/delete transactions publish without an outbox | Implement transactional outbox and reconciliation |
| Both worker branches publish latest | Develop/main use shared latest image tag | Deploy immutable SHA tags per environment |

Public local credentials, node Docker socket access, single-instance infrastructure and Keycloak `start-dev` are MVP constraints. Production hardening is follow-up work, not current functionality.
