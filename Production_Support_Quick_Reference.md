# Production Support Quick Reference Card
## For OpenShift Kubernetes - Keep in Your Terminal

---

## IMMEDIATE INCIDENT RESPONSE (First 5 Minutes)

### Step 1: Confirm Application Down
```bash
# Test externally
curl -I https://my-app.example.com

# If fails: Check cluster connectivity
oc cluster-info
oc login -u <user> https://api.openshift.example.com:6443
oc project production
```

### Step 2: Check Deployment Status
```bash
# Quick check
oc get deployment my-app -n production
# Look for: READY, UP-TO-DATE, AVAILABLE columns

# Detailed status
oc describe deployment my-app -n production | grep -A 10 "Conditions:"

# Rollout status
oc rollout status deployment/my-app -n production
```

### Step 3: Check Pod Status
```bash
# List pods
oc get pods -n production -l app=my-app

# Problem indicators:
# CrashLoopBackOff     → Check logs
# ImagePullBackOff     → Check image registry  
# Pending              → Check node capacity
# Terminating          → Wait or force delete
# Running but 0/1 Ready → Check readiness probe
```

### Step 4: Check Events
```bash
# Recent cluster events
oc get events -n production --sort-by='.lastTimestamp' | tail -30

# Pod-specific events
oc describe pod <pod-name> -n production | tail -20
```

### Step 5: Decision Tree
```
Is Deployment READY 0/X?
├─ YES (0 replicas)
│  └─ Are pods Running?
│     ├─ NO → Pod issue (Step 4: Check Logs)
│     └─ YES but Ready=0 → Readiness probe failing
│        └─ kubectl logs <pod> --tail=50
│           Check: Is app running? Can it reach dependencies?
│
└─ NO (some replicas)
   └─ Partial outage (some traffic fails)
      └─ Check which pods are bad, same diagnostics as above
```

---

## COMMON FIXES (Quick Wins)

### Pod Restarting Due to Readiness Failure
```bash
# Immediate: Check readiness endpoint
oc port-forward pod/<pod-name> 8080:8080 -n production &
curl http://localhost:8080/health/ready
# If not 200, either:
# 1. Increase initialDelaySeconds
# 2. Fix app dependency connectivity
```

### Pod Pending (Won't Schedule)
```bash
# Check node capacity
oc top nodes
oc describe node <node-name> | grep "Allocated resources" -A 5

# Possible fixes:
# Option 1: Reduce pod resource requests
oc set resources deployment/my-app --requests=memory=256Mi -n production

# Option 2: Wait for node scaling (if using autoscaling)

# Option 3: Check node affinity/selectors
oc get pod <pod-name> -o yaml | grep -i "nodeSelector\|affinity"
```

### CrashLoopBackOff (App Crashing)
```bash
# Check most recent logs
oc logs <pod-name> -n production --tail=100

# Also check previous crash logs
oc logs <pod-name> --previous -n production --tail=100

# Common causes:
# - Database connection refused (check if DB is running)
# - Missing environment variable (check ConfigMap/Secret)
# - Port conflict (check if port is already in use)
# - Out of memory (check resource limits)

# If code change caused it, quick rollback:
oc rollout undo deployment/my-app -n production
oc rollout status deployment/my-app -n production
```

### ImagePullBackOff
```bash
# Check which image it's trying to pull
oc describe pod <pod-name> -n production | grep "Image:"

# Verify image exists
docker pull myregistry/my-app:v1.0

# If private registry, check credentials
oc get secrets -n production | grep -i image
oc describe secret <image-pull-secret>

# Fix: Either correct image name or create credentials
oc create secret docker-registry myregistry-creds \
  --docker-server=myregistry.io \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n production
```

---

## DIAGNOSTIC COMMANDS (Copy-Paste Ready)

### Full Pod Diagnostics
```bash
POD=my-app-5f4d8b9c-abc12
NS=production

# Overview
echo "=== POD STATUS ==="
oc describe pod $POD -n $NS | head -30

# Conditions
echo "=== CONDITIONS ==="
oc describe pod $POD -n $NS | grep -A 10 "Conditions:"

# Recent events
echo "=== RECENT EVENTS ==="
oc describe pod $POD -n $NS | tail -30

# Container details
echo "=== CONTAINER INFO ==="
oc get pod $POD -o yaml -n $NS | grep -A 20 "containers:"

# Logs
echo "=== LOGS (last 50 lines) ==="
oc logs $POD -n $NS --tail=50
```

### Health Check Verification
```bash
POD=my-app-5f4d8b9c-abc12
NS=production
PORT=8080

# Port-forward and test
oc port-forward pod/$POD $PORT:$PORT -n $NS &
PF_PID=$!
sleep 2

# Test health endpoints
echo "=== /health/startup ==="
curl -s http://localhost:$PORT/health/startup | jq . || echo "Failed or not JSON"

echo "=== /health/ready ==="
curl -s http://localhost:$PORT/health/ready | jq . || echo "Failed or not JSON"

echo "=== /health/live ==="
curl -s http://localhost:$PORT/health/live | jq . || echo "Failed or not JSON"

# Cleanup
kill $PF_PID
```

### Database Connectivity Check
```bash
POD=my-app-5f4d8b9c-abc12
NS=production

# Exec into pod and check database
oc exec -it $POD -n $NS -- /bin/bash -c '
  echo "=== Environment ==="
  env | grep -i database

  echo "=== DNS Resolution ==="
  nslookup database.production.svc.cluster.local

  echo "=== Network Connectivity ==="
  nc -zv database.production.svc.cluster.local 5432 || echo "Connection failed"

  echo "=== Running Processes ==="
  ps aux
'
```

### Resource Usage Check
```bash
NS=production

# Current usage
echo "=== Node Resources ==="
oc top nodes

echo "=== Pod Resources (memory-sorted) ==="
oc top pods -n $NS --sort-by=memory

echo "=== Pod Resources (cpu-sorted) ==="
oc top pods -n $NS --sort-by=cpu

# Detailed for specific deployment
echo "=== Deployment Resource Requests ==="
oc get deployment my-app -o yaml -n $NS | grep -A 10 "resources:"
```

### Network Diagnostics
```bash
NS=production

# Service discovery
echo "=== Service ==="
oc get svc my-app -o yaml -n $NS | grep -A 5 "selector:" -A 5 "ports:"

# Service endpoints
echo "=== Endpoints ==="
oc get endpoints my-app -n $NS

# Check pod labels match service selector
echo "=== Pod Labels ==="
oc get pods -n $NS -l app=my-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}'

# Network policies
echo "=== Network Policies ==="
oc get networkpolicies -n $NS
oc describe networkpolicy my-app-netpol -n $NS
```

---

## COMMON COMMAND PATTERNS

### Scaling Operations
```bash
# Scale deployment
oc scale deployment my-app --replicas=5 -n production

# Auto-scaling
oc get hpa -n production
oc describe hpa my-app-hpa -n production

# Manual scaling when HPA stuck
oc patch hpa my-app-hpa -p '{"status":{"desiredReplicas":5}}' -n production
```

### Image Updates
```bash
# Update image in deployment
oc set image deployment/my-app my-app=myregistry/my-app:v2.0 -n production

# Watch rollout
oc rollout status deployment/my-app -n production

# Rollback if needed
oc rollout undo deployment/my-app -n production
oc rollout history deployment/my-app -n production
```

### Environment Variable Updates
```bash
# Update env var
oc set env deployment/my-app LOG_LEVEL=debug -n production

# Remove env var
oc set env deployment/my-app LOG_LEVEL- -n production

# View current env
oc exec <pod-name> -n production -- env | sort
```

### Resource Updates
```bash
# Update CPU/Memory
oc set resources deployment/my-app \
  --requests=cpu=500m,memory=512Mi \
  --limits=cpu=1000m,memory=1Gi \
  -n production

# View current resources
oc get deployment my-app -o yaml -n production | grep -A 10 "resources:"
```

### Restart Deployment
```bash
# Restart all pods (rolling restart)
oc rollout restart deployment/my-app -n production

# Force new pod creation
oc delete pod -l app=my-app -n production --grace-period=0 --force
```

### View Configuration
```bash
# ConfigMap
oc get configmap my-app-config -o yaml -n production
oc edit configmap my-app-config -n production

# Secret (base64 encoded)
oc get secret my-app-secret -o yaml -n production
oc get secret my-app-secret -o jsonpath='{.data.DATABASE_PASSWORD}' -n production | base64 -d

# View all in namespace
oc get all -n production
```

---

## ESCALATION PATHS

### When to Escalate

| Symptom | Check First | Escalate To |
|---------|------------|-------------|
| All pods crashing | Logs + recent changes | Dev team (code issue) |
| Pending pods | Node capacity, labels | Platform team (infrastructure) |
| Network errors | DNS, NetworkPolicy | Platform team (networking) |
| Database errors | DB service status | DB team + Platform team |
| Disk/Memory full | PVC usage, node capacity | Platform team (storage) |
| Certificate errors | Certificate expiry | Security team |
| Performance degradation | Resource usage, HPA | Dev team (optimization) + Platform team |

### Escalation Template
```
INCIDENT REPORT

Application: my-app
Namespace: production
Status: [Degraded/Down]
Symptom: [What users see]
Duration: [How long]

DIAGNOSTICS PERFORMED:
- Deployment status: [output]
- Pod status: [output]
- Recent errors: [output]
- Attempted fixes: [what you tried]

CURRENT STATE:
[Describe current status, what's running, what's not]

NEXT STEPS:
[What you think should happen]

RELEVANT LOGS/YAML:
[Attach logs, error messages, manifest]
```

---

## MONITORING & ALERTING (Preventive)

### Health Checks to Set Up

```bash
# CPU usage alert
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: production
spec:
  groups:
  - name: my-app
    interval: 30s
    rules:
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total{pod=~"my-app-.*"}[5m]) > 0.8
      for: 5m
      annotations:
        summary: "Pod {{ \$labels.pod }} high CPU usage"
    
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes{pod=~"my-app-.*"} / (2 * 1024 * 1024 * 1024) > 0.8
      for: 5m
      annotations:
        summary: "Pod {{ \$labels.pod }} high memory usage"
    
    - alert: PodRestartingFrequently
      expr: rate(kube_pod_container_status_restarts_total{pod=~"my-app-.*"}[15m]) > 0.1
      for: 5m
      annotations:
        summary: "Pod {{ \$labels.pod }} restarting frequently"
EOF
```

### Daily Monitoring Script
```bash
#!/bin/bash
# save as: daily-health-check.sh

NS=production

echo "=== $(date) Production Health Check ==="
echo

# 1. Deployments
echo "✓ Deployment Status:"
oc get deployments -n $NS | awk '{if(NR==1) print; if($2 != $3) print "  WARNING: " $1 " not fully updated"}'
echo

# 2. Pods
echo "✓ Pod Status:"
oc get pods -n $NS | grep -E "CrashLoop|ImagePull|Pending" && echo "  Issues found!" || echo "  All healthy"
echo

# 3. Node capacity
echo "✓ Node Capacity:"
oc describe nodes | grep -E "Name:|Allocatable:|Allocated"
echo

# 4. PVC usage
echo "✓ Storage Usage:"
oc get pvc -n $NS | tail -n +2 | awk '{if($4 > "80%") print "  WARNING: " $1 " at " $4}'
echo

# 5. Recent errors
echo "✓ Recent Errors:"
oc get events -n $NS --sort-by='.lastTimestamp' | grep -i error | tail -5 || echo "  None found"
```

---

## CHEAT SHEET: Decode Common Statuses

```
POD STATUS MEANINGS:
├─ Pending              = Not scheduled, waiting for resources
├─ Running              = Container started, may not be ready
├─ Succeeded            = Job completed successfully
├─ Failed               = Job/container exited with error
├─ Unknown              = Can't determine status (control plane issue)
├─ CrashLoopBackOff     = App keeps crashing, restarting
├─ ImagePullBackOff     = Can't download image
├─ CreateContainerConfigError = Invalid config
├─ OOMKilled            = Out of memory
├─ Terminating          = Pod being deleted, waiting for graceful shutdown
└─ ContainerStatusWaiting = Waiting for condition (probe, startup)

CONDITION MEANINGS:
├─ Initialized          = init containers completed
├─ Ready                = Pod ready for traffic (readiness probe passed)
├─ ContainersReady      = All containers ready
├─ PodScheduled         = Pod assigned to node
└─ (if False)           = Problem blocking readiness

PROBE FAILURE MEANINGS:
├─ startupProbe fails   = App not starting (wait longer or fix startup)
├─ readinessProbe fails = App not ready for traffic (dependencies down?)
└─ livenessProbe fails  = App dead, needs restart

RESOURCE MEANINGS:
├─ requests            = Minimum guaranteed (pod only scheduled if available)
├─ limits              = Maximum allowed (pod killed if exceeded)
└─ no limits set       = Pod can use all available (dangerous!)
```

---

## TIPS & TRICKS

### Bash Aliases for Production
```bash
# Add to ~/.bashrc
alias k=kubectl
alias oc='oc --context=production'
alias gpods='oc get pods'
alias gdeploy='oc get deployments'
alias glogs='oc logs'
alias kex='oc exec -it'
alias ktop='oc top pods'

# Quick diagnostics function
pod-info() {
  oc describe pod "$1" -n "${2:-production}" | head -40
  echo "---LOGS---"
  oc logs "$1" -n "${2:-production}" --tail=30
}
```

### Avoid Common Mistakes
```bash
# ✗ WRONG: Modifying pod directly
oc exec pod/my-pod -c app -- touch /important-file  # Lost on restart!

# ✓ CORRECT: Update deployment manifest
oc set env deployment/my-app IMPORTANT_FILE=/path

# ✗ WRONG: Manual scale then forget
oc scale deployment/my-app --replicas=5

# ✓ CORRECT: Update manifest + HPA
oc set deployment/my-app replicas=5
# Or let HPA manage: oc get hpa

# ✗ WRONG: Delete pod to restart
oc delete pod my-app-123 -n production

# ✓ CORRECT: Rollout restart
oc rollout restart deployment/my-app -n production
```

---

## USEFUL LINKS & REFERENCES

```
Official Docs:
├─ Kubernetes Docs       https://kubernetes.io/docs/
├─ OpenShift Docs        https://docs.openshift.com/
├─ kubectl Cheatsheet    https://kubernetes.io/docs/reference/kubectl/cheatsheet/
└─ oc Cheatsheet         https://docs.openshift.com/container-platform/latest/welcome/index.html

Internal (Your Company):
├─ Architecture Docs     [Your Wiki]
├─ Runbooks              [Your Repo]
├─ Slack Channels        #openshift, #production-support
└─ On-Call Rotation      [Your PagerDuty/etc]

Learning:
├─ Kubernetes Fundamentals    edx.org/course/introduction-kubernetes
├─ OpenShift Developer Guide  developers.redhat.com/openshift
└─ Play with Kubernetes       play.kubernetes.io
```

---

*This card is best printed and posted by your desk.*
*Last Updated: 2026-04-15 | Version: 1.0*
