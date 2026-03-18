# Kubernetes - Scenario-Based Questions
> Real Production Scenarios | Senior SRE Level

---

## SCENARIO 1: Production Pod Keeps Restarting

**Situation**: Your production microservice pod keeps restarting. Alert fired at 2 AM. Restart count is 47. Users are experiencing intermittent errors.

**Your Response**:

```bash
# Step 1: Assess the damage
kubectl get pods -n production | grep myservice
# STATUS: CrashLoopBackOff, RESTARTS: 47

# Step 2: Get recent logs (previous container)
kubectl logs myservice-pod-xyz -n production --previous --tail=50
# Found: "FATAL: Cannot connect to database: connection timeout"

# Step 3: Check events
kubectl describe pod myservice-pod-xyz -n production
# Found: Last State: OOMKilled, Exit Code 137

# Step 4: Check if it's a DB or resource issue
kubectl exec -it myservice-pod-xyz -n production -- nc -zv postgres-service 5432
# Or check DB service endpoints:
kubectl get endpoints postgres-service -n production

# Step 5: Check resource usage
kubectl top pod myservice-pod-xyz -n production
```

**Diagnosis**: DB connection pool exhaustion causing OOM.

**Fix**:
1. Immediately: Increase memory limit temporarily
2. Root cause: App wasn't closing DB connections — fix in code
3. Add connection pool limits: max_connections=10
4. Add readiness probe to detect DB connectivity

**Post-incident**:
- Set up alert for restart count > 3
- Add DB connection metrics to dashboard
- Write runbook for this failure mode

---

## SCENARIO 2: Cluster Node Not Ready

**Situation**: You get an alert "Node not ready". 3 pods are not rescheduled and users are affected.

**Your Response**:

```bash
# Step 1: Identify the node
kubectl get nodes
# NAME              STATUS     ROLES    AGE
# worker-node-3     NotReady   <none>   45d

# Step 2: Describe the node
kubectl describe node worker-node-3
# Events: kubelet stopped posting node status, network plugin not ready

# Step 3: Check pods on that node
kubectl get pods --all-namespaces --field-selector=spec.nodeName=worker-node-3

# Step 4: SSH to the node
ssh worker-node-3
systemctl status kubelet
journalctl -u kubelet -n 50

# Common findings:
# - kubelet crashed (OOM on node)
# - containerd not running
# - Disk pressure (df -h)
# - Network issue

# Step 5: If node can't be recovered quickly, drain and delete
kubectl drain worker-node-3 --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node worker-node-3

# Step 6: Replace node via ASG (or Cluster Autoscaler will add new one)
# AWS: Terminate the EC2 instance, ASG will launch replacement
```

**Prevention**: 
- Set up disk space monitoring on nodes
- Use Cluster Autoscaler with node health checks
- Set node NotReady tolerance: kube-controller-manager --pod-eviction-timeout=30s

---

## SCENARIO 3: Memory Spike Causes Cascading Failure

**Situation**: One service has a memory leak. It's consuming all memory on several nodes, causing other pods to be evicted. The service autoscales but makes it worse.

**Your Response**:

**Immediate (Triage)**:
```bash
# 1. Identify the culprit
kubectl top pods -n production --sort-by=memory | head -20

# 2. Check if HPA is making it worse
kubectl get hpa -n production
# myservice: Current replicas 47, target 3

# 3. Pause HPA (scale down manually)
kubectl scale deployment myservice -n production --replicas=3

# 4. Add memory limit to contain the leak
kubectl patch deployment myservice -n production --patch '
spec:
  template:
    spec:
      containers:
      - name: myservice
        resources:
          limits:
            memory: "512Mi"
'

# 5. Check evicted pods
kubectl get pods --all-namespaces | grep Evicted
```

**Medium-term fix**:
- Set ResourceQuota on namespace to prevent runaway consumption
- Configure proper memory limits
- Add heap dump collection on OOMKill via init container

**Root cause fix**:
- Profile app with memory profiler
- Fix the leak in code

---

## SCENARIO 4: GitOps Deployment Not Syncing (ArgoCD Scenario)

**Situation**: Dev team pushed a new deployment to Git 10 minutes ago. ArgoCD shows it's still out of sync but not progressing.

**Your Response**:

```bash
# Check ArgoCD app status
argocd app get myapp --namespace argocd

# Check ArgoCD app events
kubectl describe application myapp -n argocd

# Common causes:
# 1. Sync failed - check what ArgoCD tried to apply
argocd app sync myapp --dry-run

# 2. Hook failed - check sync hooks
kubectl get jobs -n production | grep argocd

# 3. Resource health issue
argocd app get myapp --show-operation

# 4. Image pull error in new deployment
kubectl describe pod <new-pod> -n production
# ImagePullBackOff: image not found

# 5. Sync policy set to manual (not auto-sync)
argocd app set myapp --sync-policy automated
```

**Fix based on root cause**:
- Image not in registry: Fix CI pipeline, push image
- Invalid YAML: Fix and re-push
- Resource quota exceeded: Clean up old resources or increase quota
- Hook failed: Fix the post-deploy hook job

---

## SCENARIO 5: Slow Rolling Deployment Taking 45 Minutes

**Situation**: A deployment that usually takes 5 minutes is stuck at 3/10 new pods ready. Old pods haven't terminated yet.

**Diagnosis**:

```bash
# Check rollout status
kubectl rollout status deployment/myapp -n production

# Check why new pods aren't ready
kubectl describe pod <new-pod-name> -n production
# Readiness probe failing: GET /health 500

# Check new pod logs
kubectl logs <new-pod> -n production
# "Failed to connect to Redis: NOAUTH Authentication required"

# The new version requires Redis auth but secret wasn't updated
```

**Resolution**:
```bash
# Option 1: Rollback immediately
kubectl rollout undo deployment/myapp -n production

# Option 2: Fix the secret first, then re-deploy
kubectl create secret generic redis-secret --from-literal=password=mypassword -n production --dry-run=client -o yaml | kubectl apply -f -

# Option 3: If stuck and can't rollback due to maxUnavailable
# Force scale down old RS manually, then fix
```

**Prevention**:
- Smoke test after deployment in staging with real config
- Validate secrets exist before deployment in CI pipeline

---

## SCENARIO 6: Persistent Volume Claim Stuck in Pending

**Situation**: New pod won't start because PVC is stuck in Pending state.

```bash
# Check PVC status
kubectl describe pvc myapp-pvc -n production
# Events: waiting for a volume to be created, volume "pv-xyz" not found

# Check StorageClass
kubectl get storageclass

# Check if CSI driver is running
kubectl get pods -n kube-system | grep csi

# Check node capacity for EBS volumes
# (AWS: each instance type has max EBS volumes)

# Check for zone mismatch
kubectl get node worker-1 -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
# vs PVC's volumeBindingMode: WaitForFirstConsumer

# Fix: Use WaitForFirstConsumer in StorageClass
# So volume is created in same zone as pod
```

---

## SCENARIO 7: Inter-Pod Communication Broken After Network Policy Applied

**Situation**: You applied a NetworkPolicy to improve security. Now frontend pods can't reach backend pods.

```bash
# Test connectivity
kubectl exec -it frontend-pod -n production -- curl backend-service:8080

# Check existing network policies
kubectl get networkpolicy -n production
kubectl describe networkpolicy backend-policy -n production

# The policy blocks all ingress by default but didn't allow frontend
# Missing: ingress rule for frontend pods
```

**Fix**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # ← This was missing
    ports:
    - protocol: TCP
      port: 8080
```

---

## SCENARIO 8: Cluster Autoscaler Not Scaling Up

**Situation**: Pods are stuck in Pending. Node count is at minimum. Cluster Autoscaler should have added nodes but hasn't.

**Diagnosis**:
```bash
# Check CA logs
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100 | grep -E "scale_up|error|failed"

# Common issues:
# 1. Max node group size reached
# → Increase max_size in ASG/node pool config

# 2. Pods have unsatisfiable requests (resource > largest node type)
kubectl describe pod <pending-pod>
# requests: cpu: "16", memory: "64Gi" — no node has this

# 3. Node group has taint but pod has no toleration
# 4. Pod affinity requirements can't be met
# 5. Service account for CA lacks IAM permissions (AWS)

# 6. Spot instance unavailable in AZ
# → Add multiple instance types to node group
```
