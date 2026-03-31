# Kubernetes Pod Troubleshooting Checklist

Follow this step-by-step workflow to systematically diagnose pod failures.

---

## Phase 1: Initial Triage (30 seconds)

### 1. Check Pod Status
```bash
kubectl get pod <pod-name> -n <namespace> -o wide
```

**What to look for:**
- **Phase**: Running, Pending, CrashLoopBackOff, Failed, Unknown
- **Ready**: X/Y containers ready (e.g., 1/1 or 0/1)
- **Restart Count**: High count = crash loop
- **Age**: When was pod created
- **Node**: Which node is it on

### 2. Describe Pod (Quick View)
```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Quick scan for:**
- **Status.Phase**: Current phase
- **Container Status**: Is it Running/Waiting/Terminated
- **Recent Events**: Last 3-4 events (chronological)
- **Resource Requests/Limits**: Visible in container specs

**Output**: You now know if pod is running, crashed, pending, or stuck.

---

## Phase 2: Targeted Diagnostics (2-3 minutes)

Based on Phase 1 findings, follow one of these paths:

### Path A: Pod is Running but Not Working (NotReady or Error Behavior)

**1. Check Liveness & Readiness Probes**
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Liveness\|Readiness"
```

- Is readiness probe failing? → Pod is broken, not traffic-ready
- Is liveness probe failing? → Pod will restart soon

**2. Fetch Current Logs**
```bash
kubectl logs <pod-name> -n <namespace> --tail=50
```

**Look for:**
- App startup errors
- Connection failures (to database, API, etc.)
- Panic/exception messages
- Warnings or repeated errors

**3. Check Resource Usage vs Limits**
```bash
kubectl top pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

- Is memory usage near limit? → May get OOMKilled
- Is CPU at limit? → App may be throttled

**Diagnosis**: App is running but unhealthy (bad config, dependency down, or resource constrained).

---

### Path B: Pod is CrashLoopBackOff or Terminated

**1. Fetch Previous Logs (Why It Crashed)**
```bash
kubectl logs <pod-name> -n <namespace> --previous
```

**Look for:**
- Exception/panic at end of logs
- Out of memory errors
- Connection failures
- Missing config/environment variables

**2. Check Liveness Probe**
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Liveness"
```

- Is probe too aggressive? (short initialDelaySeconds, low failureThreshold)
- If app starts OK but probe fails immediately, probe is misconfigured

**3. Check Events for Clues**
```bash
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
```

- Killed by liveness probe?
- OOMKilled?
- Failed to pull image?

**Diagnosis**: App crashes on startup or shortly after → check previous logs for the root error.

---

### Path C: Pod is Pending (Not Scheduled)

**1. Check Scheduling Events**
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 2 "FailedScheduling"
```

**Look for:**
- "Insufficient cpu" / "Insufficient memory"
- "Node(s) had nodeSelector label mismatch"
- "Node(s) had untolerated taint"

**2. Check Node Resources**
```bash
kubectl get nodes -o wide
kubectl top nodes
```

- Are nodes full?
- Is there a node with matching labels?

**3. Check PVC Status (if using volumes)**
```bash
kubectl get pvc -n <namespace>
```

- Is PVC Bound or Pending?

**Diagnosis**: Cannot schedule due to resource constraints, label mismatch, or volume unavailability.

---

### Path D: Pod is Pending (ImagePull)

**1. Check Image Pull Events**
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 2 "ImagePull"
```

**Look for:**
- "ImagePullBackOff"
- Error message about image not found, or auth failure

**2. Verify Image Name & Credentials**
- Test locally: `docker pull <image-name>`
- For private registries: Verify imagePullSecret exists in pod spec

**Diagnosis**: Image doesn't exist, is misspelled, or credentials are missing/expired.

---

## Phase 3: Deep Dive (5-10 minutes)

If Phase 2 hasn't revealed root cause, dig deeper:

### 3A: Exec Into Pod (If Running)
```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```

Once inside:
- Check if app process is running: `ps aux`
- Verify config files exist: `ls -la /etc/config/`
- Check network: `curl http://localhost:8080/health`
- Check disk: `df -h`
- Check processes: `top` or `htop`

### 3B: Stream Logs in Real-Time
```bash
kubectl logs <pod-name> -n <namespace> -f
```

Restart the pod in another terminal, then watch logs as it starts.

### 3C: Check Init Container (If Present)
```bash
kubectl logs <pod-name> -n <namespace> -c init-container-name --previous
```

Init container runs before main containers. If it fails, pod stuck Pending.

### 3D: Multi-Container Pod
```bash
kubectl logs <pod-name> -n <namespace> -c <specific-container>
```

Check each container's logs separately if pod has multiple containers.

---

## Common Quick Fixes

| Symptom | Quick Fix |
|---------|-----------|
| **CrashLoopBackOff** | Check previous logs: `kubectl logs --previous` |
| **ImagePullBackOff** | Verify image name, test: `docker pull <image>` |
| **Pending** | Check events for scheduling error: `kubectl describe pod` |
| **NotReady** | Check readiness probe, app logs |
| **High Memory** | Increase limit: `limits.memory: 512Mi` |
| **Slow App** | Check CPU throttling: `kubectl top` |
| **No Connectivity** | Check service exists, pods ready: `kubectl get svc` |

---

## Investigation Command Reference

```bash
# Full diagnostic snapshot
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous --tail=100
kubectl top pod <pod> -n <ns>
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>

# For multi-container pods
kubectl logs <pod> -n <ns> -c <container>
kubectl exec -it <pod> -n <ns> -- /bin/sh

# Node-level checks
kubectl get nodes
kubectl top nodes
kubectl describe node <node-name>

# Check volumes
kubectl get pvc -n <ns>
kubectl get configmap -n <ns>
kubectl get secret -n <ns>
```

---

## When to Escalate

If after Phase 3 you still don't have a clear answer:
1. Collect full diagnostic output: `kubectl describe pod`, `kubectl logs --previous`, `kubectl events`
2. Check if issue is reproducible
3. Review pod spec for any obvious misconfigurations
4. Ask: Is this a known cluster issue, or app-specific?
5. Consider reaching out to app team or cluster admin with full logs
