# Kubernetes Error Patterns & Diagnostics

## Pod Phase Failures

### CrashLoopBackOff
**What it means**: Container starts, crashes, Kubernetes restarts it, and the cycle repeats.

**Root causes**:
- Application panics on startup (misconfiguration, missing dependency)
- Missing environment variables or secrets
- Resource limits too low (OOMKilled on startup)
- Liveness probe too aggressive (kills healthy app)

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace>  # Check restart count
kubectl logs <pod-name> -n <namespace> --previous  # See why it crashed
```

**Fix**:
- Check logs for app error messages
- Verify environment variables, ConfigMaps, Secrets exist
- Increase resource limits if OOMKilled
- Relax liveness probe (increase initialDelaySeconds)

---

### ImagePullBackOff
**What it means**: Kubernetes cannot pull the container image.

**Root causes**:
- Wrong image name or tag
- Private registry but no imagePullSecret
- Registry credentials expired
- Network issue (cannot reach registry)
- Image doesn't exist on registry

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace>  # Check Events section
```
Look for: "Failed to pull image 'X': ..." with specific error.

**Fix**:
- Verify image name and tag: `docker pull <image>` locally
- For private registries, create imagePullSecret and add to pod spec
- Check registry credentials (expired tokens, wrong credentials)
- Verify network connectivity to registry

---

### Pending
**What it means**: Pod is waiting to be scheduled on a node or for resources to become available.

**Root causes**:
- No node with sufficient resources (CPU, memory)
- Node selectors or affinity constraints can't be satisfied
- PVC (persistent volume) not available
- Init container stuck (waiting for volume, downloading image)
- Taints on nodes preventing pod placement

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace>  # Check Events for FailedScheduling
kubectl get nodes  # Check node resources
kubectl top nodes  # See current node usage
```

**Fix**:
- Scale up cluster (add nodes in cloud) or free resources
- Check nodeSelector/affinity constraints
- Verify PVC exists and is Bound
- Check init container (may be pulling large image)
- Check for taints that conflict with tolerations

---

### FailedScheduling
**What it means**: Scheduler cannot place pod on any node.

**Root causes**:
- Insufficient CPU/memory on all nodes
- Pod requests exceed available node capacity
- nodeSelector labels don't match any nodes
- Affinity/anti-affinity rules can't be satisfied

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace>  # Events show why
kubectl get nodes --show-labels  # Check node labels
kubectl top nodes  # See resource availability
```

**Fix**:
- Reduce resource requests
- Add nodes to cluster
- Fix nodeSelector labels to match actual nodes
- Relax affinity rules

---

## Container State Failures

### OOMKilled (Out of Memory)
**What it means**: Container exceeded memory limit and was killed.

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace>  # Reason = OOMKilled
kubectl top pod <pod-name> -n <namespace>  # Current memory usage
```

**Fix**:
- Increase memory limit in pod spec
- Check for memory leaks in application (profile with pprof, etc.)
- Reduce batch sizes or data processing per request

---

### Exit Code 137
**What it means**: SIGKILL signal sent to container (usually OOMKilled or evicted).

**Fix**: Same as OOMKilled above.

---

### Exit Code 1, 127, 255
**What it means**: Application exited with error.

**Diagnosis**:
```bash
kubectl logs <pod-name> -n <namespace> --previous
```
Look for stack trace or error message before exit.

**Fix**: Depends on error in logs (missing dependency, config error, app bug).

---

## Probe Failures

### Liveness Probe Failing
**What it means**: Pod is considered unhealthy and Kubernetes restarts it.

**Signs**: High restart count, pod in CrashLoopBackOff.

**Root causes**:
- App needs more time to start (initialDelaySeconds too low)
- Probe endpoint not responding (HTTP, TCP, Exec)
- Probe checking too frequently (periodSeconds too low)
- App is healthy but probe has bug

**Fix**:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # Give app time to start
  periodSeconds: 10
  failureThreshold: 3
```

---

### Readiness Probe Failing
**What it means**: Pod is Running but NotReady; no traffic sent to it.

**Signs**: Pod Running but NotReady status.

**Fix**: Same as liveness probe above.

---

## Resource Constraint Failures

### Memory Limit Exceeded
**Signs**: OOMKilled in describe, high restart count.

**Fix**: Increase memory limit, profile for leaks.

### CPU Throttling
**Signs**: App slow despite running, CPU at limit.

**Fix**: Increase CPU limit, optimize code, or horizontal scale.

### Disk Pressure
**Signs**: Pod evicted or stuck Pending.

**Cause**: Node disk full.

**Fix**: Clean up node disk, add storage, or kill consuming pod.

---

## Network & Config Failures

### ConfigMap/Secret Not Found
**Signs**: Pod logs show "ConfigMap 'X' not found" or similar.

**Cause**: Referenced ConfigMap/Secret doesn't exist.

**Fix**:
```bash
kubectl get configmap -n <namespace>  # Verify it exists
kubectl get secret -n <namespace>
```
Create missing ConfigMap/Secret.

### Connection Refused
**Signs**: Pod logs show connection errors to database/service.

**Cause**: Service not running, or network policy blocking.

**Fix**:
```bash
kubectl get svc -n <namespace>  # Verify service exists
kubectl logs <service-pod> -n <namespace>  # Check if service running
```

---

## Quick Reference: Common Exit Codes

| Exit Code | Meaning | Action |
|-----------|---------|--------|
| 0 | Success (Job completed normally) | Check if pod should stay running |
| 1 | General error (app crashed) | Check logs for error |
| 137 | SIGKILL (OOMKilled or evicted) | Increase memory or investigate eviction |
| 139 | SIGSEGV (Segmentation fault) | Likely code bug; check logs |
| 255 | Exit status out of range (usually error) | Check logs |
