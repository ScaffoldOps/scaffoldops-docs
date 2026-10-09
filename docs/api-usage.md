# API usage

Forward API to port 8081 and Keycloak to 8080. All generation and internal callback routes require a bearer JWT. Configure a confidential client with service accounts in `scaffoldops-dev`, then obtain a token using its actual ID/secret:

```bash
TOKEN=$(curl -fsS http://localhost:8080/realms/scaffoldops-dev/protocol/openid-connect/token   -d grant_type=client_credentials -d "client_id=$CLIENT_ID"   --data-urlencode "client_secret=$CLIENT_SECRET" | jq -r .access_token)
API=http://localhost:8081/api/generator/v1
curl -fsS -X POST "$API/generation-requests"   -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json'   -d '{"name":"billing-service","template":"spring-boot-hexagonal","database":false,"restApi":true,"security":false,"messaging":false,"deploymentTarget":"KUBERNETES"}'
```

Use a unique service name. Save the returned `id` as `REQUEST_ID`. Requested template/features currently produce the minimal project described in [MVP scope](../architecture.md).

```bash
curl -fsS -H "Authorization: Bearer $TOKEN" "$API/generation-requests/$REQUEST_ID"
curl -fsS -H "Authorization: Bearer $TOKEN" "$API/generation-requests?generationStatus=GENERATED"
curl -fsS -X POST "$API/generation-requests/$REQUEST_ID/deployment"   -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json'   -d '{"namespace":"generated-dev","replicas":1}'
```

Wait for `generation.status=GENERATED` with both refs before deploy. Observe `deployment.status` until `DEPLOYED` or `DEPLOYMENT_FAILED`.

```bash
kubectl -n generated-dev get svc -l "scaffoldops.io/request-id=$REQUEST_ID"
kubectl -n generated-dev port-forward svc/RESOURCE_NAME 8088:8080
# In another terminal:
curl -fsS http://localhost:8088/hello
curl -fsS -X DELETE -H "Authorization: Bearer $TOKEN" "$API/generation-requests/$REQUEST_ID/deployment"
```

Undeploy retains assets for redeploy. Permanent deletion is a separate destructive action:

```bash
curl -fsS -X DELETE -H "Authorization: Bearer $TOKEN" "$API/generation-requests/$REQUEST_ID"
```

HTTP 204 accepts cleanup, not proof of external completion. Inspect workers and assets. Invalid lifecycle transitions return 409, missing resources 404, missing/invalid token 401, incomplete GENERATED callbacks 422.

Swagger: `http://localhost:8081/api/generator/v1/swagger-ui.html`; JSON: `/api/generator/v1/v3/api-docs`. Health: `/api/generator/v1/actuator/health`. These documentation/health routes are public.

The authoritative contract is [generator-api OpenAPI](https://github.com/ScaffoldOps/generator-api/blob/main/src/main/resources/openapi/generator-api.yaml). Internal PATCH routes end with `/internal/generation-requests/{requestId}/generation-status` and `/deployment-status`; they are worker callbacks. There is no public image-recovery endpoint.
