# Paved Road Service Helm Chart

A flexible, production-ready Helm chart for deploying containerized applications to Kubernetes. Supports Java, Node.js, Golang, Python, and React applications with HTTP, HTTPS, and gRPC protocols.

## Quick Start

### Install the chart
```bash
helm install my-app . --values values.yaml
```

### Install with custom values
```bash
# For Java Spring Boot
helm install my-java-app . --values examples/java-springboot-values.yaml

# For Node.js Express
helm install my-node-app . --values examples/nodejs-express-values.yaml

# For Golang gRPC
helm install my-grpc-service . --values examples/golang-grpc-values.yaml

# For Python FastAPI
helm install my-python-app . --values examples/python-fastapi-values.yaml

# For React SPA
helm install my-react-app . --values examples/react-nginx-values.yaml
```

## Features

- ✅ **Multi-port support** - HTTP, HTTPS, gRPC, and custom protocols
- ✅ **Flexible health probes** - Liveness, readiness, and startup probes
- ✅ **Environment variables** - From inline values, ConfigMaps, or Secrets
- ✅ **Init containers** - For database migrations and pre-deployment tasks
- ✅ **Autoscaling** - Horizontal Pod Autoscaler with CPU/Memory metrics
- ✅ **Ingress options** - Traditional Ingress or Gateway API HTTPRoute
- ✅ **Service Account** - Automatic creation with RBAC support
- ✅ **Application examples** - Ready-to-use configurations for common stacks

## Configuration Examples

### Multiple Ports
```yaml
service:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
    - name: grpc
      port: 9090
      targetPort: 9090
      protocol: TCP
    - name: metrics
      port: 9091
      targetPort: 9091
      protocol: TCP
```

### Environment Variables
```yaml
env:
  - name: NODE_ENV
    value: "production"
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: url

envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secrets
```

### Init Containers (Database Migrations)
```yaml
initContainers:
  - name: db-migration
    image: myapp/migrations:latest
    command: ['python', '-m', 'alembic', 'upgrade', 'head']
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: postgres-credentials
            key: connection-string
```

### Health Probes by Framework

**Java Spring Boot:**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: actuator
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: actuator
startupProbe:
  httpGet:
    path: /actuator/health
    port: actuator
  failureThreshold: 30
```

**Node.js/Express:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: http
readinessProbe:
  httpGet:
    path: /ready
    port: http
```

**Golang:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: http
readinessProbe:
  httpGet:
    path: /readyz
    port: http
```

### Autoscaling
```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 50  # Remove max 50% of current pods
        periodSeconds: 60
      - type: Pods
        value: 2  # Or max 2 pods per minute
        periodSeconds: 60
      selectPolicy: Min  # Use policy that removes fewer pods
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Add up to 100% (double)
        periodSeconds: 60
      - type: Pods
        value: 4  # Or max 4 pods per minute
        periodSeconds: 60
      selectPolicy: Max  # Use policy that adds more pods
```

**Scaling Behavior Patterns:**
- **Conservative** (Java): Long stabilization windows, slow scale-down (expensive JVM warmup)
- **Aggressive** (Node.js): Immediate scale-up, fast response to traffic spikes
- **Balanced** (Go, Python): Moderate stabilization, controlled scaling
- **Reactive** (React/SPA): Quick scale-up for static content serving

### Ingress (Traditional)
```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: myapp.cloudwalkersinc.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.cloudwalkersinc.com
```

### HTTPRoute (Gateway API)
```yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: gateway
      namespace: istio-system
  hostnames:
    - myapp.cloudwalkersinc.com
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /
```

## Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Container image repository | `nginx` |
| `image.tag` | Container image tag | `appVersion` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port (legacy) | `80` |
| `service.ports` | Multi-port configuration | `[]` |
| `env` | Environment variables | `[]` |
| `envFrom` | Load env from ConfigMap/Secret | `[]` |
| `initContainers` | Init containers | `[]` |
| `livenessProbe` | Liveness probe configuration | HTTP GET `/` |
| `readinessProbe` | Readiness probe configuration | HTTP GET `/` |
| `startupProbe` | Startup probe configuration | `{}` |
| `resources` | CPU/Memory resource requests/limits | `{}` |
| `autoscaling.enabled` | Enable HPA | `false` |
| `ingress.enabled` | Enable Ingress | `false` |
| `httpRoute.enabled` | Enable HTTPRoute | `false` |

## Development

### Test the chart
```bash
# Dry-run to validate templates
helm template paved-road-service . --values values.yaml

# Lint for errors
helm lint .

# Debug a specific template
helm template paved-road-service . --values examples/java-springboot-values.yaml --debug
```

### Upgrade a release
```bash
helm upgrade my-app . --values values.yaml
```

### Uninstall
```bash
helm uninstall my-app
```

## GitOps Deployment

This chart includes complete ArgoCD and Kargo configurations for multi-environment deployments:

### Traditional GitOps (Static Configuration)
Deploy across 11 flavors (dev, qa1-3, preview, prod, etc.) using ArgoCD ApplicationSet:

```bash
# Apply ApplicationSet to generate all 11 Applications
kubectl apply -f gitops/argocd/applicationset.yaml

# Deploy Kargo for progressive delivery
kubectl apply -f gitops/kargo/project.yaml
kubectl apply -f gitops/kargo/warehouse.yaml
kubectl apply -f gitops/kargo/stages/
```

See [gitops/README.md](gitops/README.md) for complete deployment instructions.

### Backstage Integration (Dynamic Configuration)
Automatically deploy applications using metadata from Backstage service catalog:

```bash
# Deploy plugin ConfigMap
kubectl apply -f gitops/argocd/plugin-configmap.yaml

# Create Backstage API credentials
kubectl create secret generic backstage-api-credentials \
  --namespace argocd \
  --from-literal=token="YOUR_TOKEN" \
  --from-literal=api-url="http://service-lifecycle-api.backstage.svc/graphql"

# Deploy Backstage-driven ApplicationSet
kubectl apply -f gitops/argocd/applicationset-backstage.yaml
```

**Features:**
- Dynamic namespace creation from Backstage productName
- Automatic metadata label injection (team, owner, cost center)
- FinOps cost tracking with Kubernetes labels
- Multi-tenancy support

See [gitops/BACKSTAGE.md](gitops/BACKSTAGE.md) for complete integration guide.

## License

This chart is provided as-is for use within the organization.
