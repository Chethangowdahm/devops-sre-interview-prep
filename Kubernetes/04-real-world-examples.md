# Kubernetes - Real-World Examples
> Production Patterns from Senior SRE Experience

---

## 1. Production-Ready Microservice Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: user-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: user-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: user-service
        image: gcr.io/myproject/user-service:2.4.1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: user-service
```

---

## 2. HPA with Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

---

## 3. KEDA - Scale to Zero (Event-Driven)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pubsub-worker-scaler
spec:
  scaleTargetRef:
    name: pubsub-worker
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  - type: gcp-pubsub
    metadata:
      subscriptionName: my-subscription
      mode: SubscriptionSize
      value: "10"
```

---

## 4. Namespace Resource Quota (Multi-Tenant)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-platform
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-platform
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

---

## 5. Init Containers - DB Migration Pattern

```yaml
initContainers:
- name: wait-for-db
  image: postgres:13
  command: ['sh', '-c',
    'until pg_isready -h postgres-service -p 5432 -U app; do
      echo "Waiting for database..."; sleep 2;
    done;']
- name: run-migrations
  image: gcr.io/myproject/app:latest
  command: ["./migrate", "up"]
  env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: app-db-secret
        key: url
```

---

## 6. Canary Deployment Pattern

```yaml
# Stable: 9 pods = 90% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
---
# Canary: 1 pod = 10% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
---
# Service selects ALL pods (stable + canary)
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp  # No track label
```

---

## 7. GKE Node Pool Upgrade (Production Pattern)

```bash
# 1. Create new node pool
gcloud container node-pools create pool-v2 \
  --cluster=production --num-nodes=3

# 2. Cordon old nodes
for node in $(kubectl get nodes -l gke-nodepool=pool-v1 -o name); do
  kubectl cordon $node
done

# 3. Drain old nodes (respects PDBs)
for node in $(kubectl get nodes -l gke-nodepool=pool-v1 -o name); do
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data --timeout=5m
done

# 4. Delete old pool
gcloud container node-pools delete pool-v1 --cluster=production
```

---

## 8. PodDisruptionBudget for HA

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-service-pdb
  namespace: production
spec:
  minAvailable: 2  # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: critical-service
```

---

## 9. External Secrets Operator (GCP Secret Manager)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: projects/myproject/secrets/db-password
      version: latest
```

---

## 10. Network Policy - Zero-Trust

```yaml
# Default deny all ingress/egress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow specific service communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-service
    ports:
    - port: 5432
```
