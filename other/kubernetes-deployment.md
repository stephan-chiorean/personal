---
id: kubernetes-deployment
alias: Kubernetes Deployment
type: kit
is_base: false
version: 1
tags:
  - foundation
  - infrastructure
  - kubernetes
description: Complete Kubernetes deployment patterns with manifests, Helm charts, deployment strategies, service mesh, and production-ready configurations
---

# Kubernetes Deployment Kit

A comprehensive kit for deploying applications to Kubernetes with production-ready patterns, including deployments, services, ingress, configmaps, secrets, and Helm charts.

## End State

After applying this kit, the application will have:

**Kubernetes Resources:**
- Deployment manifests with proper resource limits and health checks
- Service definitions for internal and external access
- Ingress configuration with SSL/TLS termination
- ConfigMaps for application configuration
- Secrets for sensitive data (encrypted at rest)
- Horizontal Pod Autoscaler (HPA) for automatic scaling
- Pod Disruption Budget (PDB) for high availability
- Network policies for security isolation

**Deployment Strategy:**
- Rolling update strategy with zero-downtime deployments
- Blue-green or canary deployment support
- Health checks (liveness and readiness probes)
- Resource quotas and limits
- Namespace isolation

**Helm Charts:**
- Reusable Helm chart structure
- Values files for different environments
- Chart dependencies management
- Helm hooks for pre/post deployment tasks

**Monitoring & Observability:**
- Prometheus metrics exposure
- Log aggregation setup
- Distributed tracing integration
- Health check endpoints

## Implementation Principles

- **Declarative configuration**: Use YAML manifests, avoid imperative commands
- **Resource limits**: Always set CPU and memory requests/limits
- **Health checks**: Implement liveness and readiness probes
- **High availability**: Deploy multiple replicas across nodes
- **Security**: Use RBAC, network policies, and secrets management
- **Immutable infrastructure**: Use container images, avoid in-place updates
- **Configuration management**: Use ConfigMaps and Secrets, not hardcoded values
- **Rolling updates**: Use rolling update strategy for zero-downtime
- **Resource quotas**: Set namespace quotas to prevent resource exhaustion
- **Monitoring**: Expose metrics and logs for observability

## Pattern 1: Basic Deployment Manifest

Create a deployment with proper configuration:

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
  labels:
    app: {{APP_NAME}}
    version: {{VERSION}}
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: {{APP_NAME}}
  template:
    metadata:
      labels:
        app: {{APP_NAME}}
        version: {{VERSION}}
    spec:
      containers:
      - name: {{APP_NAME}}
        image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{VERSION}}
        imagePullPolicy: Always
        ports:
        - containerPort: {{PORT}}
          name: http
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{APP_NAME}}-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: {{APP_NAME}}-config
              key: redis-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: {{APP_NAME}}-config
      restartPolicy: Always
```

## Pattern 2: Service Definition

Expose the deployment with a service:

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
  labels:
    app: {{APP_NAME}}
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  selector:
    app: {{APP_NAME}}
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

**LoadBalancer Service (for cloud providers):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{APP_NAME}}-lb
  namespace: {{NAMESPACE}}
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
  selector:
    app: {{APP_NAME}}
```

## Pattern 3: Ingress Configuration

Configure ingress with SSL/TLS:

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  tls:
  - hosts:
    - {{DOMAIN_NAME}}
    secretName: {{APP_NAME}}-tls
  rules:
  - host: {{DOMAIN_NAME}}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{APP_NAME}}
            port:
              number: 80
```

## Pattern 4: ConfigMap and Secrets

Manage configuration and secrets:

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{APP_NAME}}-config
  namespace: {{NAMESPACE}}
data:
  redis-url: "redis://redis-service:6379"
  log-level: "info"
  api-timeout: "30s"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool-size: 10
      max-connections: 100
```

```yaml
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{APP_NAME}}-secrets
  namespace: {{NAMESPACE}}
type: Opaque
stringData:
  database-url: "postgresql://user:password@db:5432/dbname"
  api-key: "{{API_KEY}}"
  jwt-secret: "{{JWT_SECRET}}"
```

**Using Sealed Secrets (for GitOps):**
```yaml
# k8s/sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: {{APP_NAME}}-secrets
  namespace: {{NAMESPACE}}
spec:
  encryptedData:
    database-url: {{ENCRYPTED_VALUE}}
    api-key: {{ENCRYPTED_VALUE}}
```

## Pattern 5: Horizontal Pod Autoscaler

Auto-scale based on metrics:

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{APP_NAME}}
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max
```

## Pattern 6: Pod Disruption Budget

Ensure high availability during disruptions:

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: {{APP_NAME}}
```

## Pattern 7: Network Policy

Implement network isolation:

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  podSelector:
    matchLabels:
      app: {{APP_NAME}}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: {{PORT}}
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

## Pattern 8: Helm Chart Structure

Create a reusable Helm chart:

```
chart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   └── _helpers.tpl
└── charts/
```

**Chart.yaml:**
```yaml
apiVersion: v2
name: {{APP_NAME}}
description: Helm chart for {{APP_NAME}}
type: application
version: 1.0.0
appVersion: "{{VERSION}}"
dependencies:
- name: postgresql
  version: "12.0.0"
  repository: "https://charts.bitnami.com/bitnami"
  condition: postgresql.enabled
```

**values.yaml:**
```yaml
replicaCount: 3

image:
  repository: {{IMAGE_REGISTRY}}/{{APP_NAME}}
  pullPolicy: Always
  tag: "{{VERSION}}"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: {{DOMAIN_NAME}}
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: {{APP_NAME}}-tls
      hosts:
        - {{DOMAIN_NAME}}

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    postgresPassword: {{POSTGRES_PASSWORD}}
```

## Pattern 9: Blue-Green Deployment

Implement blue-green deployment strategy:

```yaml
# k8s/blue-green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{APP_NAME}}-blue
  namespace: {{NAMESPACE}}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: {{APP_NAME}}
      version: blue
  template:
    metadata:
      labels:
        app: {{APP_NAME}}
        version: blue
    spec:
      containers:
      - name: {{APP_NAME}}
        image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{BLUE_VERSION}}
        # ... rest of config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{APP_NAME}}-green
  namespace: {{NAMESPACE}}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: {{APP_NAME}}
      version: green
  template:
    metadata:
      labels:
        app: {{APP_NAME}}
        version: green
    spec:
      containers:
      - name: {{APP_NAME}}
        image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{GREEN_VERSION}}
        # ... rest of config
---
apiVersion: v1
kind: Service
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  selector:
    app: {{APP_NAME}}
    version: blue  # Switch to 'green' to deploy new version
  ports:
  - port: 80
    targetPort: http
```

## Pattern 10: Canary Deployment

Implement canary deployment with Istio:

```yaml
# k8s/canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{APP_NAME}}-canary
  namespace: {{NAMESPACE}}
spec:
  replicas: 1  # Small number for canary
  selector:
    matchLabels:
      app: {{APP_NAME}}
      version: canary
  template:
    metadata:
      labels:
        app: {{APP_NAME}}
        version: canary
    spec:
      containers:
      - name: {{APP_NAME}}
        image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{CANARY_VERSION}}
        # ... rest of config
---
# VirtualService for traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  hosts:
  - {{APP_NAME}}
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: {{APP_NAME}}
        subset: canary
      weight: 100
  - route:
    - destination:
        host: {{APP_NAME}}
        subset: stable
      weight: 90
    - destination:
        host: {{APP_NAME}}
        subset: canary
      weight: 10
```

## Pattern 11: Job and CronJob

Run one-time and scheduled tasks:

```yaml
# k8s/job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{APP_NAME}}-migration
  namespace: {{NAMESPACE}}
spec:
  template:
    spec:
      containers:
      - name: migration
        image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{VERSION}}
        command: ["npm", "run", "migrate"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{APP_NAME}}-secrets
              key: database-url
      restartPolicy: Never
  backoffLimit: 3
---
# k8s/cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{APP_NAME}}-cleanup
  namespace: {{NAMESPACE}}
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{VERSION}}
            command: ["npm", "run", "cleanup"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

## Pattern 12: StatefulSet for Stateful Applications

Deploy stateful applications:

```yaml
# k8s/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  serviceName: {{APP_NAME}}
  replicas: 3
  selector:
    matchLabels:
      app: {{APP_NAME}}
  template:
    metadata:
      labels:
        app: {{APP_NAME}}
    spec:
      containers:
      - name: {{APP_NAME}}
        image: {{IMAGE_REGISTRY}}/{{APP_NAME}}:{{VERSION}}
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
```

## Best Practices

1. **Resource limits**: Always set CPU and memory requests and limits
2. **Health checks**: Implement both liveness and readiness probes
3. **High availability**: Deploy multiple replicas across nodes
4. **Rolling updates**: Use rolling update strategy for zero-downtime
5. **Secrets management**: Use sealed secrets or external secret managers
6. **Configuration**: Use ConfigMaps for non-sensitive config, Secrets for sensitive data
7. **Namespaces**: Use namespaces for environment isolation
8. **RBAC**: Implement role-based access control
9. **Network policies**: Use network policies for security isolation
10. **Monitoring**: Expose Prometheus metrics and aggregate logs

## Common Pitfalls

- **Missing resource limits**: Can cause node resource exhaustion
- **No health checks**: Pods may serve traffic before ready
- **Hardcoded values**: Use ConfigMaps and Secrets
- **Single replica**: No high availability
- **Missing PDB**: Pods can be disrupted during updates
- **No resource quotas**: Namespace can consume all cluster resources
- **Imperative commands**: Use declarative YAML manifests
- **Missing monitoring**: Difficult to troubleshoot issues

## Verification Criteria

After generation, verify:
- ✓ Deployment has proper resource limits set
- ✓ Liveness and readiness probes are configured
- ✓ Multiple replicas are deployed
- ✓ Service correctly routes traffic to pods
- ✓ Ingress is configured with SSL/TLS
- ✓ ConfigMaps and Secrets are properly referenced
- ✓ HPA is configured for auto-scaling
- ✓ Pod Disruption Budget ensures availability
- ✓ Network policies restrict traffic appropriately
- ✓ Helm chart can be deployed to different environments
- ✓ Rolling updates work without downtime
- ✓ Monitoring and logging are configured
