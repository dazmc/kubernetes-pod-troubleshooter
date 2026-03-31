# kubernetes-pod-troubleshooter

Debug pod failures, analyze logs, and diagnose issues in local Kubernetes clusters. A pragmatic troubleshooting agent for Minikube, Docker Desktop, and other local K8s environments.

## Features

- **Pod Diagnostics** — List, describe, and inspect pod status and events
- **Log Analysis** — Fetch and parse container logs, identify error patterns
- **Resource Monitoring** — Check CPU/memory usage, diagnose resource constraints and probe failures
- **Error Pattern Recognition** — Knows common K8s failures (CrashLoopBackOff, OOMKilled, ImagePullBackOff, Pending, etc.)
- **Troubleshooting Workflow** — Step-by-step diagnostic checklist

## Run

```bash
gitagent run -r https://github.com/<username>/kubernetes-pod-troubleshooter -p "Troubleshoot pod 'nginx' in namespace 'default'"
```

### Example Prompts

```bash
# Debug a failing pod
gitagent run -r ... -p "Why is pod 'api-server' in namespace 'production' crashing?"

# Diagnose resource issues
gitagent run -r ... -p "Pod 'worker' is NotReady. Check logs and resource limits."

# Analyze multiple pods
gitagent run -r ... -p "List all pods in 'kube-system' and report any that are failing."

# Investigate pending pod
gitagent run -r ... -p "Why is pod 'data-loader' still Pending?"
```

## What It Can Do

### Pod Diagnostics
- List pods in a namespace with current status
- Describe full pod details (spec, events, container status)
- Check Kubernetes events for failure timeline
- Inspect init container status

### Log Analysis
- Fetch current and previous container logs
- Parse common error patterns (OOMKilled, panic, Exception, etc.)
- Extract stack traces for debugging
- Handle multi-container pods

### Resource Monitoring
- Check current CPU/memory usage
- Inspect resource requests and limits
- Diagnose liveness and readiness probe failures
- Identify node pressure conditions (memory, disk, PID)
- Compare usage vs limits for OOMKilled/throttling diagnosis

### Error Pattern Recognition
Knows diagnostics for:
- **CrashLoopBackOff** — app crashing on startup
- **ImagePullBackOff** — container image not accessible
- **Pending** — pod cannot be scheduled
- **OOMKilled** — memory limit exceeded
- **FailedScheduling** — insufficient resources or constraints
- **Probe Failures** — liveness/readiness probe misconfigured

## How It Works

1. **Initial Triage** — Check pod phase, container status, restart count
2. **Log Inspection** — Fetch logs and previous logs to find errors
3. **Resource Analysis** — Check CPU/memory usage vs limits
4. **Event Review** — See what happened to the pod (scheduling, restarts, errors)
5. **Root Cause** — Identify the problem
6. **Fix Recommendation** — Suggest how to resolve it

## Structure

```
kubernetes-pod-troubleshooter/
├── agent.yaml                           # Agent config (model, adapters, skills)
├── SOUL.md                              # Agent personality (pragmatic debugger)
├── RULES.md                             # Safety boundaries (read-only)
├── README.md                            # This file
├── skills/
│   ├── pod-diagnostics/
│   │   └── SKILL.md                     # Pod inspection & events
│   ├── log-analysis/
│   │   └── SKILL.md                     # Log fetching & error parsing
│   └── resource-monitoring/
│       └── SKILL.md                     # Resource usage & probe checks
└── knowledge/
    ├── k8s-error-patterns.md            # Common failures & diagnostics
    └── debugging-checklist.md           # Step-by-step workflow
```

## Requirements

- `kubectl` installed and configured locally
- Access to a Kubernetes cluster (Minikube, Docker Desktop, etc.)
- Target pods in accessible namespace

## Safety

- **Read-only by default** — Only inspects pod status, logs, events
- **No destructive actions** — Never deletes, restarts, or modifies pods without explicit permission
- **No cluster changes** — Diagnostics only, no RBAC or networking modifications
- **Pod-focused** — Focuses on individual pod debugging, not cluster-wide management

## Supported Adapters

- **Primary**: OpenCode (claude-sonnet-4-5-20250929)
- **Fallback**: OpenAI (gpt-4o-mini)

## Local Testing

```bash
cd ~/devenvs/kubernetes-pod-troubleshooter

# Validate agent config
gitagent validate -d .

# Run locally with sample prompt
gitagent run -d . -p "List all pods in 'default' namespace"
```

## Next Steps

- Ready to push to GitHub? Run: `gh repo create kubernetes-pod-troubleshooter --public --source=. --push`
- Want to register on the gitagent registry? After pushing: `gitagent registry -r <repo-url> -c developer-tools -a opencode,openai`

---

Built with [gitagent](https://github.com/open-gitagent/gitagent) — a git-native, framework-agnostic open standard for AI agents.
