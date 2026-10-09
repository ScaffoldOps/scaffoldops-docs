# Local development, Minikube and WSL

Clone all five [repositories](../architecture.md) as sibling directories. Use Java 17 and Maven for applications; install Docker, Minikube, kubectl and optionally GitHub CLI, MinIO `mc`, PostgreSQL `psql` and DBeaver. API and generator-worker include Maven wrappers; deployment-worker uses Maven directly.

## Infrastructure startup

From `platform-infra`:

```bash
minikube start --driver=docker
kubectl config use-context minikube
kubectl kustomize k8s/overlays/dev > /tmp/scaffoldops-dev.yaml
kubectl apply --dry-run=client -f /tmp/scaffoldops-dev.yaml
kubectl apply -k k8s/overlays/dev
kubectl -n scaffoldops get pods,pvc
kubectl -n scaffoldops-dev get pods,pvc,jobs
kubectl -n security get pods
kubectl -n scaffoldops-dev wait --for=condition=complete job/minio-create-bucket --timeout=600s
kubectl -n scaffoldops-dev wait --for=condition=complete job/kafka-topic-bootstrap --timeout=600s
```

Bootstrap Jobs are TTL-cleaned after 300 seconds; if already absent, inspect the bucket/topics. For a failed Job, inspect logs, resolve the cause, delete only that Job and reapply. Both overlays include all shared infrastructure; DEV scales PRE Keycloak down. Resolve [current configuration gaps](findings.md), provision [secrets](configuration.md), configure Keycloak, then deploy the component manifests following their READMEs. Supplied deployment RBAC creates `generated-dev`; additional target namespaces need bindings.

## Port-forwarding

Run each command in a separate terminal, leave it running, and restart it after pod replacement:

```bash
kubectl -n scaffoldops-dev port-forward svc/generator-api-service 8081:80
kubectl -n scaffoldops-dev port-forward svc/minio 9000:9000 9001:9001
kubectl -n security port-forward svc/keycloak-dev 8080:8080
kubectl -n scaffoldops port-forward svc/postgres 5432:5432
kubectl -n scaffoldops-dev port-forward svc/kafka-ui 8083:8080
```

| Service | Local access |
| --- | --- |
| generator-api | `http://localhost:8081/api/generator/v1` |
| MinIO API / UI | `http://localhost:9000` / `http://localhost:9001` |
| Keycloak | `http://localhost:8080` |
| PostgreSQL / DBeaver | `localhost:5432`, database `generatorapidb`, user `generatorapiuser` |
| Kafka UI | `http://localhost:8083` |

DBeaver uses the database user's password, not Keycloak admin credentials. Change the left-hand local port when occupied. Bind defaults to loopback; use localhost in the same WSL environment first.

Port-forwarding uses the Kubernetes API connection to expose internal Services without depending on node IP routing, making it suitable for the local Minikube/WSL MVP. It is temporary access for development, not a persistent production endpoint. NodePort requires reaching the node IP and port; WSL/Docker networking can complicate that. Ingress provides HTTP routing but needs a controller, reachable address and host/DNS setup. API has a separate Ingress manifest; shared infra does not enable it automatically. An Ingress is not a general substitute for PostgreSQL/Kafka TCP connectivity.

## Component execution on the host

Use each component README for commands and variables. Start API with `SPRING_PROFILES_ACTIVE=local`, `DB_PASSWORD`, `SERVER_PORT=8081`, Keycloak local issuer and a reachable Kafka broker. Generator-worker local profile uses API `http://localhost:8081/api/generator/v1`, static bearer token by default, and filesystem artifacts unless MinIO is selected. Set `SERVER_PORT=8082` to avoid Keycloak's port.

The cluster broker advertises `kafka:9092`. Forwarding port 9092 alone does not make that metadata address reachable from a host process. Use an appropriately configured external advertised listener/broker or run the applications inside the cluster; do not assume localhost bootstrap fixes Kafka metadata. No external listener setup is implemented in the infra manifests.

## WSL notes

Keep tools and kubeconfig in Linux paths and ensure the runner user can read kubeconfig/certificates and access Docker. Check `command -v docker`, `docker version`, `minikube status` and `kubectl config current-context`. WindowsApps Docker shims may produce EACCES; workers' CI explicitly prioritizes `/usr/bin/docker`. Avoid switching Docker contexts or exporting Minikube Docker environment midway through an image build/publish job. The generator-worker's node socket mount requires `/var/run/docker.sock` inside the Minikube node, not merely a working host CLI.
