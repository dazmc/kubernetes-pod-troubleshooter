---
name: resource-monitoring
description: Check CPU/memory usage, diagnose resource constraints, and identify probe failures
license: MIT
allowed-tools: Bash Read
metadata:
  version: "1.0.0"
  author: kubernetes-pod-troubleshooter
  category: kubernetes
---

# Resource Monitoring Skill

## Purpose
Monitor pod resource consumption and identify resource-related failures (OOMKilled, CPU throttling, probe failures).

## Core Capabilities

### 1. Check Resource Usage (Current)
```bash
kubectl top pod <pod-name> -n <namespace>
```
Shows CPU and memory usage at this moment.

**Output:**
- CPU (millicores): 100m = 0.1 CPU
- Memory (Mi): 256Mi = 256 megabytes

**When to use**: Understand current resource consumption.

### 2. Check Resource Requests & Limits
```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'
```

Better with formatting:
```bash
kubectl get pod <pod-name> -n <namespace> -o json | jq '.spec.containers[].resources'
```

Shows:
- **requests**: Minimum resources guaranteed
- **limits**: Maximum resources allowed

| Field | Meaning |
|-------|---------|
| **requests.cpu** | Min CPU guaranteed |
| **requests.memory** | Min memory guaranteed |
| **limits.cpu** | Max CPU before throttling |
| **limits.memory** | Max memory before OOMKilled |

**When to use**: Diagnose resource constraint issues.

### 3. Check Liveness Probe Status
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Liveness"
```

Shows:
- Probe type (HTTP, TCP, Exec)
- Check frequency (initialDelaySeconds, periodSeconds)
- Success/failure thresholds
- Last probe result

**When to use**: Pod is repeatedly restarting — liveness probe may be too aggressive.

### 4. Check Readiness Probe Status
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Readiness"
```

Shows if pod is considered "Ready" for traffic.

**When to use**: Pod is Running but showing NotReady — readiness probe failing.

### 5. Node Pressure Check
```bash
kubectl get nodes -o wide
kubectl describe node <node-name> | grep -A 10 "Conditions"
```

Look for:
- **MemoryPressure** → node low on memory
- **DiskPressure** → node disk full
- **PIDPressure** → too many processes

**When to use**: Pod is Pending or evicted — node may be under pressure.

### 6. Compare Usage vs Limits
When fetching resource limits and current usage, compare:

| Scenario | Indicator | Fix |
|----------|-----------|-----|
| **Usage near limit** | CPU/memory usage 90%+ of limit | Increase limit, optimize app |
| **No limit set** | Memory limit missing | Set limit to prevent OOMKill on overload |
| **Request too low** | Pod requested 64Mi but needs 256Mi | Increase request to match typical usage |
| **Usage way below limit** | CPU/memory at 10% of limit | Can decrease limit to save resources |

## Diagnosing Resource Failures

### OOMKilled (Out of Memory)
**Signs:**
- `kubectl describe pod`: Reason = OOMKilled
- `kubectl logs --previous`: May show OOM error
- Memory usage spike to limit value

**Fix:**
1. Increase `limits.memory` in pod spec
2. Check app for memory leaks (profile if necessary)
3. Reduce data being processed per request

### CPU Throttling
**Signs:**
- Pod running slowly but CPU limit not exceeded
- High latency in app logs
- Usage constantly at CPU limit

**Fix:**
1. Increase `limits.cpu`
2. Optimize app code (profile for hot spots)
3. Consider vertical scaling (more CPU per pod) vs horizontal (more pods)

### Liveness Probe Killing Pod
**Signs:**
- Pod Pending or CrashLoopBackOff
- `kubectl logs --previous`: Shows app startup but then killed
- Restart count high

**Fix:**
1. Check `initialDelaySeconds` — app needs more time to start
2. Check `periodSeconds` — probe checking too frequently
3. Check probe endpoint — may be failing when app is healthy
4. Verify app is actually responding to probe (HTTP status, TCP port, exec command)

### Readiness Probe Blocking Traffic
**Signs:**
- Pod Running but NotReady status
- App working fine in logs

**Fix:**
1. Check readiness probe conditions (same as liveness above)
2. Verify probe endpoint is truly working
3. If app is healthy but probe failing, disable readiness probe (use liveness only)

## Workflow

1. **Check current usage** → `kubectl top` for CPU/memory
2. **Check requested limits** → compare usage vs limits
3. **Check probe status** → see if liveness/readiness is failing
4. **Check node health** → see if node has pressure conditions
5. **Diagnose mismatch** → if usage exceeds limit, pod will be killed
6. **Recommend fix** → increase limits, optimize app, or tune probes

## Safety
- All commands are read-only (describe, top, get)
- No pod or resource modification
- Probe tuning requires direct user confirmation
