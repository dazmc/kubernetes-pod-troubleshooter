---
name: pod-diagnostics
description: List pods, describe pod details, check status, and inspect Kubernetes events
license: MIT
allowed-tools: Bash Read
metadata:
  version: "1.0.0"
  author: kubernetes-pod-troubleshooter
  category: kubernetes
---

# Pod Diagnostics Skill

## Purpose
Inspect pod state, gather event history, and collect diagnostic information needed for troubleshooting.

## Core Capabilities

### 1. List Pods in Namespace
```bash
kubectl get pods -n <namespace> -o wide
```
Shows pod name, status, restarts, age, IP, node. Use this to:
- Find failing pods (NotReady, CrashLoopBackOff, Pending)
- Check restart count (high count = crash loop)
- Identify unscheduled pods

**When to use**: Initial triage — what pods exist and which are failing?

### 2. Describe Pod
```bash
kubectl describe pod <pod-name> -n <namespace>
```
Returns:
- Pod metadata (labels, annotations)
- Container specs (image, ports, mounts)
- Resource requests/limits
- Events (most recent first)
- Init container status (if present)

**When to use**: After identifying a failing pod, to see full context.

### 3. Check Pod Events
```bash
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
```
Shows chronological events:
- FailedScheduling → node constraints, resource unavailability
- Failed → container exit codes
- FailedMount → volume mount failures

**When to use**: Understand what happened to the pod (why it's stuck/failed).

### 4. Inspect Container Status
From `kubectl describe`, look for:
- **Ready/NotReady** → liveness/readiness probe status
- **ContainerStatus.State** → Running/Waiting/Terminated
- **Last State.Reason** → OOMKilled, Error, Completed

**When to use**: Determine if pod is healthy or degraded.

### 5. Check Init Containers
```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.initContainerStatuses}'
```
Shows init container status before main containers start.

**When to use**: Pod is Pending — init container may be stuck.

## Workflow

1. **List all pods** in namespace → identify failing ones
2. **Describe the failing pod** → collect full context
3. **Check events** → understand failure timeline
4. **Inspect container status** → see if running/crashed
5. **Pass findings to log-analysis skill** → dig into why

## Common Pod Phases

| Phase | Meaning | Action |
|-------|---------|--------|
| **Pending** | Pod not yet scheduled or containers not running | Check events (FailedScheduling), init containers |
| **Running** | At least one container running | Check logs, liveness probes |
| **Succeeded** | Pod completed successfully (Jobs/Tasks) | Check exit code, logs |
| **Failed** | One or more containers exited with error | Check logs, exit codes |
| **Unknown** | State cannot be determined | Check kubectl connectivity |
| **CrashLoopBackOff** | Container crashes repeatedly | Check logs, resource limits, app errors |

## Safety
- All commands are read-only (get, describe, events)
- No pod modification or deletion
- Safe to run repeatedly
