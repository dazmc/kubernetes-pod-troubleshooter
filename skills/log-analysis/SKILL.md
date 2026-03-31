---
name: log-analysis
description: Fetch pod logs, parse error patterns, and diagnose application failures
license: MIT
allowed-tools: Bash Read Grep
metadata:
  version: "1.0.0"
  author: kubernetes-pod-troubleshooter
  category: kubernetes
---

# Log Analysis Skill

## Purpose
Extract and interpret pod logs to identify application errors, crashes, and failures.

## Core Capabilities

### 1. Fetch Current Logs
```bash
kubectl logs <pod-name> -n <namespace>
```
Shows logs from the running container (last 100 lines by default).

**Options:**
- `-f` (follow): Stream logs in real-time
- `--tail=50`: Last 50 lines
- `--timestamps=true`: Include timestamps
- `-c <container>`: Specific container (for multi-container pods)

**When to use**: Pod is running but behaving incorrectly, or initial failure assessment.

### 2. Fetch Previous Logs
```bash
kubectl logs <pod-name> -n <namespace> --previous
```
Shows logs from the last terminated container (if it crashed and restarted).

**When to use**: Pod is CrashLoopBackOff — previous logs often contain the error.

### 3. Stream Multi-Container Logs
```bash
kubectl logs <pod-name> -n <namespace> -c <container> --all-containers=true
```
Shows logs from all containers in the pod simultaneously.

**When to use**: Sidecar containers, init containers, or multi-container debugging.

### 4. Parse Common Error Patterns
Look for these keywords in logs:

| Error Pattern | Meaning | Action |
|---------------|---------|--------|
| **OOMKilled** | Out of memory | Increase memory limit |
| **Exit code 137** | SIGKILL (usually OOM) | Check memory usage |
| **Connection refused** | App not listening / not started | Check app startup logs |
| **ImagePullBackOff** | Cannot pull container image | Check image name, registry creds, network |
| **ConfigMap/Secret not found** | Missing mounted config | Verify volume mounts |
| **PermissionDenied** | File access issues | Check volume permissions, security context |
| **Segmentation fault** | Container crash (code bug) | Check app logs for stack trace |
| **panic** (Go/Rust) | Application panic | See stack trace in logs for root cause |
| **Exception** (Python/Java) | Unhandled exception | Full stack trace usually in logs |

### 5. Extract Stack Traces
```bash
kubectl logs <pod-name> -n <namespace> | grep -A 20 "Exception\|panic\|Error"
```
Shows context around errors for root cause analysis.

**When to use**: Application is crashing; stack trace reveals why.

### 6. Check for Retry/Timeout Patterns
```bash
kubectl logs <pod-name> -n <namespace> | grep -i "timeout\|retry\|backoff"
```
Indicates connectivity issues or dependency problems.

**When to use**: Pod is failing intermittently or can't reach other services.

## Log Interpretation Workflow

1. **Fetch current logs** → see if app is running
2. **Check for errors** → grep for Exception, Error, panic
3. **If crashed, fetch previous** → see what went wrong before restart
4. **Look for patterns** → OOMKilled, ImagePullBackOff, connection errors
5. **Extract stack traces** → understand root cause
6. **Correlate with events** → combine with pod events for full picture

## Common Log Findings

### App Won't Start
- **Log**: Connection refused / Address already in use
- **Cause**: Port already bound, app can't listen
- **Fix**: Check port config, kill process on that port, or change port

### Out of Memory
- **Log**: OOMKilled or Exit code 137
- **Cause**: Memory leak or insufficient limit
- **Fix**: Increase memory limit, profile app for leaks

### Config Not Found
- **Log**: ConfigMap 'X' not found / Secret 'Y' not found
- **Cause**: Volume mount missing or config not created
- **Fix**: Create missing ConfigMap/Secret, verify volume mounts

### Can't Pull Image
- **Log**: (in events, not pod logs): ImagePullBackOff
- **Cause**: Wrong image name, private registry without credentials, or network issue
- **Fix**: Verify image name, add imagePullSecrets, check registry credentials

### Dependency Timeout
- **Log**: Connection timeout to database / Can't reach service
- **Cause**: Service not running, network policy blocking, or DNS issue
- **Fix**: Verify service exists, check network connectivity, verify DNS

## Safety
- All commands are read-only (logs only)
- No log modification or deletion
- Grep filtering is safe for searching
