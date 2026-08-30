# new-api Helm Chart

Helm chart for deploying **new-api** on Kubernetes. It packages the application Deployment, Service, optional PersistentVolumeClaim, optional Ingress, and optional Gateway API `HTTPRoute` so you can run a quick single-node install or a production deployment backed by external database and Redis services.

> [!IMPORTANT]
> For multi-replica or production deployments, configure a shared `SESSION_SECRET`, external `SQL_DSN`, and external `REDIS_CONN_STRING`. Running multiple replicas on the default embedded SQLite setup is not safe.

## Features

- Deploys `new-api` as a standard Kubernetes `Deployment`
- Supports persistent `/data` storage with a PVC or an existing claim
- Supports either traditional `Ingress` or Gateway API `HTTPRoute`
- Lets you inject sensitive settings through chart-managed secrets or an existing secret
- Includes health probes for `/api/status`
- Supports HPA-based autoscaling

## Prerequisites

- Kubernetes 1.24+
- Helm 3.12+
- A storage class for persistent volumes if `persistence.enabled=true`
- Optional: Gateway API CRDs and a compatible controller if `gateway.enabled=true`

## Getting Started

### Add the chart from the repository

```bash
git clone https://github.com/QuantumNous/new-api.git
cd new-api
```

### Install with the default SQLite setup

This is the fastest way to try the chart. It creates a single replica with persistent `/data` storage and exposes the service inside the cluster.

```bash
helm install new-api ./charts \
  --namespace new-api \
  --create-namespace
```

Then access it locally:

```bash
kubectl -n new-api port-forward svc/new-api 3000:3000
```

Open <http://127.0.0.1:3000>.

> [!WARNING]
> The first-time default admin credentials are `root` / `123456`. Change them immediately after signing in.

## Production Example

The chart supports external PostgreSQL or MySQL for the primary database and Redis for shared cache/session-related state.

```bash
openssl rand -hex 32
```

Create a values file:

```yaml
# values-production.yaml
replicaCount: 2

secrets:
  sessionSecret: "<generated-32-byte-hex>"
  sqlDsn: "postgres://newapi:password@postgresql.default.svc:5432/new-api?sslmode=disable"
  redisConnString: "redis://:password@redis.default.svc:6379"

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: new-api.example.com
      paths:
        - path: /
          pathType: Prefix
```

Then install or upgrade:

```bash
helm upgrade --install new-api ./charts \
  --namespace new-api \
  --create-namespace \
  -f values-production.yaml
```

## Using an Existing Secret

If you already manage secrets outside Helm, create a Kubernetes secret containing these keys:

- `SESSION_SECRET`
- `SQL_DSN`
- `REDIS_CONN_STRING`

Example:

```bash
kubectl -n new-api create secret generic new-api-env \
  --from-literal=SESSION_SECRET='<generated-32-byte-hex>' \
  --from-literal=SQL_DSN='postgres://newapi:password@postgresql.default.svc:5432/new-api?sslmode=disable' \
  --from-literal=REDIS_CONN_STRING='redis://:password@redis.default.svc:6379'
```

Then install the chart with:

```bash
helm install new-api ./charts \
  --namespace new-api \
  --create-namespace \
  --set existingSecret=new-api-env
```

When `existingSecret` is set, the chart does not create its own Secret.

## Networking

Choose one of these exposure methods depending on your cluster:

| Method | Enable with | Notes |
|---|---|---|
| ClusterIP + port-forward | default | Good for local access and internal-only deployments |
| Ingress | `ingress.enabled=true` | Standard Kubernetes ingress controllers |
| Gateway API | `gateway.enabled=true` | Requires Gateway API CRDs and a compatible controller |
| NodePort / LoadBalancer | `service.type` | Useful when exposing directly through the Service |

### Ingress example

```bash
helm upgrade --install new-api ./charts \
  --namespace new-api \
  --create-namespace \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.hosts[0].host=new-api.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=Prefix
```

### Gateway API example

```bash
helm upgrade --install new-api ./charts \
  --namespace new-api \
  --create-namespace \
  --set gateway.enabled=true \
  --set gateway.parentRefs[0].name=shared-gateway \
  --set gateway.parentRefs[0].namespace=infra \
  --set gateway.hostnames[0]=new-api.example.com
```

## Persistence

By default, the chart mounts `/data` from a PVC:

- `persistence.enabled=true`
- `persistence.size=1Gi`
- `persistence.accessModes[0]=ReadWriteOnce`

To reuse an existing claim:

```bash
helm upgrade --install new-api ./charts \
  --namespace new-api \
  --create-namespace \
  --set persistence.existingClaim=new-api-data
```

To run without persistence, disable it and the chart will use `emptyDir`:

```bash
helm upgrade --install new-api ./charts \
  --namespace new-api \
  --create-namespace \
  --set persistence.enabled=false
```

## Key Values

The full configuration lives in [`values.yaml`](./values.yaml). These are the most important settings:

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of replicas when HPA is disabled |
| `image.repository` | `calciumion/new-api` | Container image repository |
| `image.tag` | `""` | Image tag; defaults to chart `appVersion` when empty |
| `service.type` | `ClusterIP` | Kubernetes Service type |
| `service.port` | `3000` | Service port |
| `config.tz` | `Asia/Shanghai` | Container timezone |
| `config.errorLogEnabled` | `true` | Sets `ERROR_LOG_ENABLED` |
| `config.batchUpdateEnabled` | `true` | Sets `BATCH_UPDATE_ENABLED` |
| `config.nodeName` | `""` | Optional `NODE_NAME` env var |
| `config.streamingTimeout` | `""` | Optional `STREAMING_TIMEOUT` env var |
| `config.syncFrequency` | `""` | Optional `SYNC_FREQUENCY` env var |
| `config.extraEnvVars` | `[]` | Additional container environment variables |
| `secrets.sessionSecret` | `""` | Session secret; required for shared-state or multi-replica setups |
| `secrets.sqlDsn` | `""` | External database DSN; empty uses embedded SQLite |
| `secrets.redisConnString` | `""` | Redis connection string |
| `existingSecret` | `""` | Existing secret containing `SESSION_SECRET`, `SQL_DSN`, `REDIS_CONN_STRING` |
| `persistence.enabled` | `true` | Creates or mounts persistent storage at `/data` |
| `ingress.enabled` | `false` | Creates an Ingress resource |
| `gateway.enabled` | `false` | Creates a Gateway API `HTTPRoute` |
| `autoscaling.enabled` | `false` | Enables HorizontalPodAutoscaler |

## Upgrade and Uninstall

Upgrade an existing release:

```bash
helm upgrade new-api ./charts -n new-api
```

Uninstall:

```bash
helm uninstall new-api -n new-api
```

## Resources

- Project repository: <https://github.com/QuantumNous/new-api>
- Main documentation: <https://docs.newapi.pro/en/docs>
- Chart defaults: [`values.yaml`](./values.yaml)

## License

This chart is distributed with the **new-api** project under the terms of the repository [LICENSE](../LICENSE).
