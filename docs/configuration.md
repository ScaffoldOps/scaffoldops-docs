# Environment and secrets

Component READMEs list runtime variables; full defaults are in `src/main/resources/application*.yml` and Kubernetes env blocks. Secrets are namespace-scoped and must exist before the pod starts. Do not commit exported Secret manifests or real credential values.

## Docker Hub: three separate credential uses

| Location/name | Format | Purpose |
| --- | --- | --- |
| Actions `DOCKER_USERNAME`, `DOCKER_PASSWORD` | Repository/organization Actions secrets | CI Docker login and image push |
| Kubernetes `docker-hub-pull-secret` | `kubernetes.io/dockerconfigjson`, key `.dockerconfigjson` | Kubelet authentication via `imagePullSecrets` |
| Kubernetes `docker-hub-credentials` | Opaque, keys `DOCKER_USERNAME`, `DOCKER_PASSWORD` | generator-worker Docker CLI login and optional Hub tag API cleanup |

`docker-hub-credentials` must not be used as `imagePullSecrets`. Actions credentials do not create Kubernetes secrets. Worker namespace pull credentials do not automatically authenticate generated pods in `generated-dev`.

Use Docker Hub access tokens: write permission for push and appropriate delete permission for enabled tag cleanup. Set `DOCKER_USERNAME` and `DOCKER_PASSWORD` in repository Actions settings or organization secrets with access to each app repo. Environment-only secrets are insufficient for push jobs that do not select an environment.

With credentials exported locally, create or replace the pull secret:

```bash
kubectl -n scaffoldops-dev create secret docker-registry docker-hub-pull-secret   --docker-server=https://index.docker.io/v1/   --docker-username="$DOCKER_USERNAME" --docker-password="$DOCKER_PASSWORD"   --dry-run=client -o yaml | kubectl apply -f -
kubectl -n scaffoldops-dev get secret docker-hub-pull-secret -o jsonpath='{.type}'
kubectl -n scaffoldops-dev create secret generic docker-hub-credentials   --from-literal=DOCKER_USERNAME="$DOCKER_USERNAME" --from-literal=DOCKER_PASSWORD="$DOCKER_PASSWORD"   --dry-run=client -o yaml | kubectl apply -f -
```

Repeat pull-secret provisioning in any namespace whose pods reference it. Deployment-worker itself references the pull secret; generated Deployment manifests currently do not add one. For private generated images, configure the target ServiceAccount's pull secrets as an operator step or implement support in the worker.

## MinIO

Infrastructure runs MinIO on ports 9000 (API) and 9001 (console), with `minio-credentials` keys `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`. Local defaults are `minioadmin`/`minioadmin`. Kustomize renders `generator-worker-minio` compatibility keys `access-key`/`secret-key` from the canonical Secret; reapply after changes to keep them synchronized.

`minio-create-bucket` creates `scaffoldops-artifacts` idempotently. Worker DEV selects `GENERATOR_ARTIFACT_STORAGE_TYPE=minio`, `GENERATOR_MINIO_ENDPOINT=http://minio:9000`, `GENERATOR_MINIO_BUCKET=scaffoldops-artifacts` and Secret-backed `GENERATOR_MINIO_ACCESS_KEY`/`GENERATOR_MINIO_SECRET_KEY`. `GENERATOR_ARTIFACT_MINIO_ENDPOINT` and `GENERATOR_ARTIFACT_MINIO_BUCKET` are not bound by current code.

## PostgreSQL

Shared `postgres:16` runs in `scaffoldops` with a PVC. Bootstrap creates database/user pairs `keycloakdb`/`keycloakuser`, `generatorapidb`/`generatorapiuser`, `generatorworkerdb`/`generatorworkeruser`, `deploymentworkerdb`/`deploymentworkeruser`. Worker databases are provisioned but unused by current workers.

`postgres-credentials` under the database base supplies `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` and per-service passwords. API needs `POSTGRES_GENERATOR_API_PASSWORD` in its own namespace; secrets are not copied automatically by namespace aliases. Keycloak has separate namespace-local database secrets. Initializer scripts only run on an empty PostgreSQL data directory; see [migrations](operations.md#database-migrations) for existing state.

## Keycloak

Infrastructure provides `keycloak-dev` and `keycloak-pre` in `security`, using `start-dev`; DEV overlay disables PRE. Admin/database credentials are separate secrets under `k8s/base/security`. DEV local admin defaults are `admin`/`admin`.

Create the `scaffoldops-dev` realm and clients/users in Keycloak; no realm import or automatic client provisioning is present in these manifests. Enable client authentication/service accounts for `scaffoldops-generator-worker` and `deployment-worker`, then provision matching OAuth secrets:

```bash
kubectl -n scaffoldops-dev create secret generic generator-api-worker-client   --from-literal=client-secret="$GENERATOR_WORKER_CLIENT_SECRET"   --dry-run=client -o yaml | kubectl apply -f -
kubectl -n scaffoldops-dev create secret generic deployment-worker-oauth   --from-literal=token-url=http://keycloak-dev.security.svc.cluster.local:8080/realms/scaffoldops-dev/protocol/openid-connect/token   --from-literal=client-secret="$DEPLOYMENT_WORKER_CLIENT_SECRET"   --dry-run=client -o yaml | kubectl apply -f -
```

DEV API validates signatures using the realm JWKS directly, so a token obtained through localhost forwarding can be accepted. Local profile uses issuer discovery and issuer validation; obtain tokens from the same local issuer URL. All business/internal API routes currently require authentication; the code does not enforce separate worker-specific roles.

## Kafka and recovery controls

Kafka is single-node KRaft (`confluentinc/cp-kafka:7.7.7`), plaintext `kafka:9092`, with auto-topic creation enabled. UI is `provectuslabs/kafka-ui:v0.7.2`. Bootstrap creates only generation and cleanup topics with one partition and replication factor one. Topic overrides follow the names in component YAML, e.g. `GENERATION_REQUESTED_TOPIC`, `IMAGE_BUILD_RETRY_REQUESTED_TOPIC`, `DEPLOYMENT_REQUESTED_TOPIC`, `UNDEPLOYMENT_REQUESTED_TOPIC`, `ARTIFACT_CLEANUP_REQUESTED_TOPIC`.

API scheduler controls: `GENERATION_RECOVERY_ENABLED=true`, `GENERATION_RECOVERY_FIXED_DELAY=5m`, `GENERATION_RECOVERY_MAX_RETRIES=5`, `GENERATION_RECOVERY_BATCH_SIZE=10`, `GENERATION_RECOVERY_RESERVATION_TIMEOUT=30m`. Worker image retry controls: `GENERATOR_IMAGE_MAX_ATTEMPTS=3`, `GENERATOR_IMAGE_RETRY_BACKOFF_MS=1000`. Set reservation timeout above queue delay plus worst-case image processing duration.

Docker Hub cleanup reads `app.cleanup.docker-hub.enabled` and `.base-url`. Due to the current YAML nesting issue, use Spring environment properties `APP_CLEANUP_DOCKERHUB_ENABLED=true` and `APP_CLEANUP_DOCKERHUB_BASEURL` if enabling it before the config is corrected; the advertised `GENERATOR_DOCKER_HUB_CLEANUP_ENABLED` override is currently under the wrong prefix. See [findings](findings.md).
