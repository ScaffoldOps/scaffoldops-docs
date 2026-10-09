# Operations and troubleshooting

## Cluster and rollout inspection

```bash
minikube status
kubectl config current-context
kubectl -n scaffoldops-dev get pods,svc,pvc,jobs
kubectl -n generated-dev get deploy,svc,pods
kubectl -n scaffoldops-dev rollout status deploy/generator-api-deployment --timeout=300s
kubectl -n scaffoldops-dev rollout status deploy/generator-worker --timeout=300s
kubectl -n scaffoldops-dev rollout status deploy/deployment-worker --timeout=300s
kubectl -n scaffoldops-dev get events --sort-by=.metadata.creationTimestamp
kubectl -n scaffoldops-dev logs deploy/generator-api-deployment --tail=150
kubectl -n scaffoldops-dev logs deploy/generator-worker --tail=150
kubectl -n scaffoldops-dev logs deploy/deployment-worker --tail=150
kubectl -n scaffoldops-dev describe pod POD_NAME
```

Use `logs --previous` for a restarted container. `CreateContainerConfigError` often means a missing namespace-local secret/key. `ImagePullBackOff` requires checking registry, tag, pull-secret type and credentials; [recreate the registry secret](configuration.md#docker-hub-three-separate-credential-uses). Generated private images need target-namespace pull authentication. TCP readiness alone does not prove HTTP business behavior.

## Database migrations

From `generator-api`, with PostgreSQL forwarded to 5432 and database credentials exported:

```bash
PGPASSWORD="$DB_PASSWORD" psql -h localhost -p 5432 -U generatorapiuser -d generatorapidb   -v ON_ERROR_STOP=1 -f docs/image-recovery.sql
```

That additive script covers recovery columns on existing installations. The broader current schema initializer is `src/main/resources/schema-postgresql.sql`, used automatically by local/dev/pre profiles (`spring.sql.init.mode=always`) before Hibernate validation. It also removes legacy `status`; inspect/backup legacy data before applying the entire schema manually:

```bash
PGPASSWORD="$DB_PASSWORD" psql -h localhost -p 5432 -U generatorapiuser -d generatorapidb   -v ON_ERROR_STOP=1 -f src/main/resources/schema-postgresql.sql
```

There is no Flyway/Liquibase migration command. Infra PostgreSQL init only runs for an empty data directory; existing database/user changes require explicit SQL. Do not delete a PVC as a migration technique.

## Inspect artifacts and images

With MinIO forwarded and `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` exported:

```bash
mc alias set scaffoldops-local http://localhost:9000 "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD"
mc ls --recursive scaffoldops-local/scaffoldops-artifacts
mc stat "scaffoldops-local/scaffoldops-artifacts/$REQUEST_ID/project.zip"
mc cp "scaffoldops-local/scaffoldops-artifacts/$REQUEST_ID/project.zip" /tmp/project.zip
unzip -l /tmp/project.zip
kubectl -n scaffoldops-dev exec deploy/generator-worker -- ls -la /var/lib/generator-worker/manifests
docker buildx imagetools inspect "$IMAGE_REF"
```

Take `IMAGE_REF` from `generation.imageRef`, not from a guessed tag. Docker Hub repository defaults to `victodomvar/scaffoldops-generated`. A workspace or intended manifest reference is insufficient evidence that the image was pushed.

## Kafka inspection

```bash
kubectl -n scaffoldops-dev exec deploy/kafka -- kafka-topics --bootstrap-server kafka:9092 --list
kubectl -n scaffoldops-dev exec deploy/kafka -- kafka-consumer-groups --bootstrap-server kafka:9092 --all-groups --describe
kubectl -n scaffoldops-dev logs job/kafka-topic-bootstrap
```

Use forwarded Kafka UI on port 8083 to inspect topics, groups, messages and DLT. An absent TTL-cleaned bootstrap Job is not proof of failure. Host Kafka clients must reach the broker's advertised `kafka:9092`, not just a forwarded bootstrap port.

## Failure diagnosis

| Symptom | Check |
| --- | --- |
| API fails DB startup | Correct password, schema and namespace DNS; DEV alias Services currently lack endpoints |
| API/worker 401 | Realm/client secret, token expiry and issuer/JWKS URL; callback base URL must include API path |
| Callback 404 | Wrong API base path, deleted request or superseded lifecycle |
| MinIO upload fails | Bucket Job, endpoint port 9000, credentials and matching worker namespace Secret |
| IMAGE_BUILD failure | Worker Docker CLI, node socket, generated Dockerfile and dependency network access |
| IMAGE_PUSH failure | Registry repository, credentials and write permission; scheduler retries eligible failures |
| Deploy RBAC denial | Target RoleBinding grants worker ServiceAccount Deployments/Services access |
| Deploy rollout failure | Generated image exists, private pull credentials, container port and pod events |
| Hub login lacks username/password | Exact Actions secret names and organization repository access |
| Hub repeated-login lockout | Stop retries, verify token, wait for service cooldown |
| WSL EACCES / WindowsApps docker | Linux `/usr/bin/docker`, daemon permissions and runner PATH |
| Port-forward refuses connection | Active terminal, free local port, correct service/namespace and running endpoint |

Delete cleanup has no durable external success report. Check both workers, MinIO object, PVC workspace, image tag and generated resources. Generator-worker failures/skips require manual reconciliation from retained event payloads; Docker tag cleanup is disabled by default and has a [configuration gap](findings.md). Stop active generation before deleting to reduce asset recreation races.
