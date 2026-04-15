# OpenShift/Kubernetes Troubleshooting Decision Tree
## Application Availability Issues - Step-by-Step Guide

---

## THE GOLDEN RULE

```
Before you panic:
1. Take a screenshot of the error
2. Note the exact time issue started
3. Run: oc get all -n production
4. Check: Has anything changed recently? (deployments, configs, infra)
```

---

## MASTER DECISION TREE

```
APPLICATION UNAVAILABLE
│
├─ Is the entire cluster down?
│  ├─ YES → Can't reach API server
│  │   └─ CHECK: oc cluster-info
│  │       └─ Network issue? Contact Infrastructure team
│  │
│  └─ NO → API server responding
│      └─ GO TO: Step 1
│
└─ Step 1: IS DEPLOYMENT RUNNING?
   └─ Run: oc get deployment my-app -n production
      ├─ READY: 3/3 ✓ → GO TO: Step 2
      ├─ READY: 0/3 ✗ → GO TO: PROBLEM A
      └─ READY: 1/3 ⚠ → GO TO: PROBLEM B
```

---

## STEP 1: DEPLOYMENT STATUS

### ✓ READY: 3/3 (All pods should be running)

**Verify:**
```bash
oc get deployment my-app -n production
# Expected: READY 3/3, UP-TO-DATE 3, AVAILABLE 3
```

If deployment looks good, go to **Step 2: Pod Health**

---

### ✗ PROBLEM A: READY: 0/3 (No pods running)

**Diagnosis:**
```bash
oc describe deployment my-app -n production | grep -A 10 "Conditions:"
oc get pods -n production -l app=my-app
```

**Solution Path:**

```
Check Pod Status:
├─ All pods CrashLoopBackOff
│  └─ GO TO: PROBLEM C (Container Crashing)
│
├─ All pods ImagePullBackOff
│  └─ GO TO: PROBLEM D (Image Pull Failed)
│
├─ All pods Pending
│  └─ GO TO: PROBLEM E (Scheduling Failed)
│
├─ All pods NotReady (Running but 0/1 Ready)
│  └─ GO TO: PROBLEM F (Readiness Probe Failing)
│
└─ Some other status
   └─ Run: oc describe pod <pod-name>
      └─ Look for Event messages (bottom)
```

---

### ⚠ PROBLEM B: READY: 1/3 (Partial Outage)

**This means:**
- Some users can reach app
- Some requests fail
- Partial functionality available

**Diagnosis:**
```bash
oc get pods -n production -l app=my-app -o wide
# Identify which pods are Ready=1/1 and which are 0/1
```

**Solution Path:**

```
Identify Good vs Bad Pods:
├─ Some pods Running, 1/1 Ready
├─ Some pods Running, 0/1 Ready → These are the problem
└─ Investigate the bad pods using PROBLEM paths below
```

---

## STEP 2: POD HEALTH (If deployment looks good)

### Is the Service endpoint reachable?

```bash
# Check service is finding pods
oc get endpoints my-app -n production
# Expected: Should show 3 IP:port combinations

# If endpoints empty:
# Either pods not ready OR pods being deleted/created
oc get pods -n production -l app=my-app -o wide
```

### Is traffic routing correctly?

```bash
# Port-forward to service
oc port-forward svc/my-app 8080:80 -n production &
curl http://localhost:8080
# Should get app response

# If fails:
# GO TO: PROBLEM F (Readiness Probe)
```

---

## DETAILED PROBLEM PATHS

---

## PROBLEM C: Container Crashing (CrashLoopBackOff)

**Symptom:**
- Pod Status: CrashLoopBackOff
- Restarts: Keep increasing
- Pod Age: Long (but keeps restarting)

**Root Cause Detection:**
```bash
# 1. Check current logs (may not show much if just crashed)
oc logs <pod-name> -n production --tail=50

# 2. Check previous logs (most important)
oc logs <pod-name> -n production --previous --tail=200
# This shows what happened before crash

# 3. Check why it's crashing
oc describe pod <pod-name> -n production
# Look for "Last State" section for exit code

# Exit codes:
# 1   = Generic error (check logs)
# 127 = Command not found (check entrypoint)
# 137 = SIGKILL (out of memory)
# 143 = SIGTERM (graceful shutdown timeout)
```

**Common Causes & Fixes:**

| Cause | Log Message | Fix |
|-------|------------|-----|
| Database down | "connection refused" | Check database service is running |
| Missing env var | "var undefined" | Check ConfigMap/Secret mounted |
| Port conflict | "Address already in use" | Different port or remove duplicate pod |
| Out of memory | No logs, just dies | Increase limits: `oc set resources ... --limits=memory=2Gi` |
| App bug | Stack trace in logs | Fix code, rebuild, deploy new version |
| Wrong image | "file not found" | Check image exists in registry |

**Fix Process:**
```bash
# 1. Read the logs carefully
oc logs <pod-name> --previous -n production | head -100

# 2. Identify the issue (database? config? memory?)

# 3. Fix:
# - If config: update ConfigMap, restart pod
oc rollout restart deployment/my-app -n production

# - If image: rollback previous version
oc rollout undo deployment/my-app -n production

# - If code: fix, rebuild, push new image, deploy
oc set image deployment/my-app my-app=myregistry/my-app:v2 -n production

# 4. Watch rollout
oc rollout status deployment/my-app -n production
```

---

## PROBLEM D: Image Pull Failed (ImagePullBackOff)

**Symptom:**
- Pod Status: ImagePullBackOff
- Pod never starts
- Events mention "image pull" or "unauthorized"

**Diagnosis:**
```bash
# 1. Check which image it's trying to pull
oc describe pod <pod-name> -n production | grep "Image:"
# Output: Image: myregistry/my-app:v1.0

# 2. Verify image exists
docker login myregistry
docker pull myregistry/my-app:v1.0
# Success? → Credentials issue (Step 3)
# Failed? → Image doesn't exist (upload it)

# 3. Check pod's image pull secrets
oc get pod <pod-name> -o yaml -n production | grep -A 2 "imagePullSecrets"
```

**Fix Options:**

**Option 1: Image doesn't exist in registry**
```bash
# Build and push image
docker build -t myregistry/my-app:v2 .
docker push myregistry/my-app:v2

# Update deployment to use new tag
oc set image deployment/my-app my-app=myregistry/my-app:v2 -n production
```

**Option 2: Credentials missing**
```bash
# Create image pull secret
oc create secret docker-registry myregistry-creds \
  --docker-server=myregistry.io \
  --docker-username=<username> \
  --docker-password=<password> \
  -n production

# Update deployment to use secret
# Edit manifest and add:
# spec:
#   template:
#     spec:
#       imagePullSecrets:
#       - name: myregistry-creds

oc apply -f updated-deployment.yaml
```

**Option 3: Private registry, different namespace credentials**
```bash
# Check if credentials exist elsewhere
oc get secrets -A | grep pull

# Copy secret to production namespace
oc get secret <secret-name> -n <source-ns> -o yaml | \
  oc create -f - -n production
```

---

## PROBLEM E: Pod Won't Schedule (Pending)

**Symptom:**
- Pod Status: Pending
- Pod age increases but never starts
- Events mention "insufficient", "node selector", "affinity"

**Diagnosis:**
```bash
# 1. Check pod details and events
oc describe pod <pod-name> -n production

# Look for Events section:
# "0/3 nodes are available: 3 Insufficient memory"
# "0/3 nodes are available: node(s) did not match Pod's node selector"

# 2. Check node capacity
oc top nodes
# Look for Available columns

# 3. Check what pod is requesting
oc get pod <pod-name> -o yaml -n production | grep -A 10 "resources:"

# 4. Check node selectors/affinity
oc get pod <pod-name> -o yaml -n production | grep -A 5 "nodeSelector\|affinity"
```

**Fix Options:**

**Option 1: Not enough resources on nodes**
```bash
# Check current capacity
oc describe node <node-name> | grep -A 5 "Allocated resources"

# Option A: Add more nodes (contact infrastructure)

# Option B: Reduce pod resource requests
oc set resources deployment/my-app \
  --requests=cpu=250m,memory=256Mi \
  -n production
```

**Option 2: Node selector doesn't match**
```bash
# Check what labels are available
oc get nodes --show-labels

# Option A: Update pod's nodeSelector to match available labels
oc patch deployment my-app -p '
{
  "spec": {
    "template": {
      "spec": {
        "nodeSelector": {
          "workload-type": "general"
        }
      }
    }
  }
}
' -n production

# Option B: Remove nodeSelector if not needed
oc patch deployment my-app --type json -p '
[
  {
    "op": "remove",
    "path": "/spec/template/spec/nodeSelector"
  }
]
' -n production
```

**Option 3: Pod affinity can't be satisfied**
```bash
# Check current affinity rules
oc get pod <pod-name> -o yaml -n production | grep -A 20 "affinity:"

# Temporarily remove affinity to test
oc patch deployment my-app --type json -p '
[
  {
    "op": "remove",
    "path": "/spec/template/spec/affinity"
  }
]
' -n production

# If pods schedule now, reconfigure affinity less strictly
```

---

## PROBLEM F: Pod Not Ready (Readiness Probe Failing)

**Symptom:**
- Pod Status: Running
- Pod Ready: 0/1 (red X)
- Service has no endpoints
- All requests get 503 errors

**Diagnosis:**
```bash
# 1. Check readiness probe definition
oc get pod <pod-name> -o yaml -n production | grep -A 15 "readinessProbe:"

# 2. Check probe status
oc describe pod <pod-name> -n production | grep "Ready" -A 5

# 3. Test probe endpoint manually
oc port-forward pod/<pod-name> 8080:8080 -n production &
curl -v http://localhost:8080/health/ready
# If not 200, probe is failing correctly

# 4. Check container logs for why it's not ready
oc logs <pod-name> -n production --tail=100
```

**Root Causes & Fixes:**

| Cause | Log Message | Fix |
|-------|------------|-----|
| App still initializing | Startup takes time | Increase `initialDelaySeconds` (30 → 60) |
| Database not reachable | "Cannot connect to DB" | Check DB service, connectivity |
| Config not loaded | "Config file not found" | Check ConfigMap volume mounts |
| Dependency timeout | "Timeout waiting for service" | Increase `timeoutSeconds`, fix dependency |

**Fix Process:**

**If app needs more startup time:**
```bash
oc patch deployment my-app --type json -p '
[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/initialDelaySeconds",
    "value": 60
  }
]
' -n production
```

**If database unreachable:**
```bash
# Check if database service exists and is healthy
oc get svc database -n production
oc get endpoints database -n production
oc get pods -n production -l app=database

# If database pods are down:
oc rollout restart deployment database -n production
oc rollout status deployment database -n production

# Once DB is up, restart app pods
oc rollout restart deployment my-app -n production
```

**If config missing:**
```bash
# Check ConfigMap exists and has data
oc get configmap my-app-config -n production
oc get configmap my-app-config -o yaml -n production

# Check it's mounted in deployment
oc get deployment my-app -o yaml -n production | grep -A 10 "volumeMounts:" | grep -A 5 "configMap"

# If not mounted, add it to manifest and apply
oc apply -f updated-deployment.yaml
```

---

## PROBLEM G: High Latency / Slow Responses

**Symptom:**
- Pods Running + Ready: 1/1
- Service responding
- But very slow (>2-5 seconds)

**Diagnosis:**
```bash
# 1. Check pod resource usage
oc top pods -n production -l app=my-app
# If CPU/Memory near limits → resource constraint

# 2. Check node resource usage
oc top nodes
# If node >80% → node saturation

# 3. Check pod logs for slowness indicators
oc logs <pod-name> -n production | grep -i "slow\|timeout\|wait"

# 4. Check database if applicable
oc exec <pod-name> -n production -- \
  timeout 5 curl -s http://database:5432 | head
# If hangs → network or DB problem
```

**Fix Options:**

**Option 1: Pod resource constraints**
```bash
# Increase resource limits
oc set resources deployment/my-app \
  --requests=cpu=1000m,memory=1Gi \
  --limits=cpu=2000m,memory=2Gi \
  -n production

# Watch performance
oc top pods -n production -l app=my-app --watch
```

**Option 2: Node saturation**
```bash
# Temporarily scale down other apps to free resources
# Or request additional nodes from infrastructure team

# Check HPA is working
oc get hpa -n production
oc describe hpa my-app-hpa -n production
```

**Option 3: Dependency (database) slow**
```bash
# Check database performance
oc exec <db-pod> -n production -- psql -c "EXPLAIN ANALYZE ..."
# Look for slow queries

# Restart database if necessary
oc rollout restart deployment database -n production
```

---

## PROBLEM H: Network/Connectivity Issues

**Symptom:**
- "Connection refused" errors
- "Host unreachable" errors
- Services can't talk to each other

**Diagnosis:**
```bash
# 1. Check service exists and has endpoints
oc get svc <service-name> -n production
oc get endpoints <service-name> -n production
# If endpoints empty → pods not ready (Problem F)

# 2. Check DNS resolution from pod
oc exec <pod-name> -n production -- \
  nslookup <service-name>.production.svc.cluster.local
# Should resolve to service IP

# 3. Check connectivity directly
oc exec <pod-name> -n production -- \
  nc -zv <service-name>.production.svc.cluster.local 5432

# 4. Check network policies
oc get networkpolicies -n production
oc describe networkpolicy <policy-name>
```

**Fix Options:**

**Option 1: Service has no healthy pods**
```bash
# Check why pods aren't ready
oc get pods -n production -l app=<service-name>
# Follow Problem F (readiness probe) or Problem C (crashing)
```

**Option 2: NetworkPolicy blocking traffic**
```bash
# Check policy rules allow traffic
oc describe networkpolicy <policy-name> -n production

# If missing rule, add it:
oc edit networkpolicy <policy-name> -n production
# Add to egress section:
# - to:
#   - podSelector:
#       matchLabels:
#         app: <target-app>
#   ports:
#   - protocol: TCP
#     port: <port>
```

**Option 3: Wrong service name/namespace**
```bash
# If service in different namespace:
# Use: <service>.<namespace>.svc.cluster.local
# Not: <service>

# Update app config/connection string
oc set env deployment/my-app \
  DATABASE_HOST=database.other-namespace.svc.cluster.local \
  -n production
```

---

## PROBLEM I: Certificate/TLS Errors

**Symptom:**
- "certificate verify failed"
- "untrusted certificate"
- "cert has expired"

**Diagnosis:**
```bash
# 1. Check certificate expiry (from first certificate analysis)
oc get configmap openshiftca.crt -n production -o \
  jsonpath='{.data.service-ca\.crt}' | \
  openssl x509 -noout -dates

# 2. Check service account cert (if MTLS)
oc get secret -n production | grep -i cert
oc get secret <cert-secret> -o yaml -n production

# 3. Check TLS in route (if external)
oc describe route my-app -n production | grep -i tls
```

**Fix Options:**

**Option 1: Certificate expired**
```bash
# Service CA certificates auto-rotate
# But if manual cert expired:
# Create new certificate or request renewal

# Force service CA regeneration
oc delete configmap openshiftca.crt -n production
oc rollout restart deployment service-ca -n openshift-service-ca

# Wait for new cert
sleep 60
oc get configmap openshiftca.crt -n production
```

**Option 2: Route TLS config wrong**
```bash
# Check route TLS settings
oc get route my-app -o yaml -n production | grep -A 10 "tls:"

# Common fixes:
# - Change termination from "edge" to "passthrough" if app handles TLS
# - Update certificate reference
# - Enable insecureEdgeTerminationPolicy: Redirect

oc patch route my-app -p '
{
  "spec": {
    "tls": {
      "termination": "edge",
      "insecureEdgeTerminationPolicy": "Redirect"
    }
  }
}
' -n production
```

---

## QUICK DECISION CHART (Laminated Card)

```
SYMPTOM → ACTION

0/3 Pods Ready?
├─ CrashLoopBackOff        → Check logs: oc logs <pod> --previous
├─ ImagePullBackOff        → Verify image exists, check credentials
├─ Pending                 → Check node resources: oc top nodes
├─ Running but 0/1 Ready   → Check readiness probe: curl /health/ready
└─ Other status            → Describe: oc describe pod <pod>

Some 503 errors?
├─ Check if it's readiness probe issue (above)
├─ Check if it's network policy blocking (oc get networkpolicies)
└─ Check if database down (oc get pods -l app=database)

High latency/slow?
├─ Check CPU/Memory: oc top pods
├─ Check logs for errors
└─ Check database performance

Can't reach external URL?
├─ Check route: oc get route my-app
├─ Test via port-forward: oc port-forward svc/my-app 8080:80
└─ Check TLS cert: oc describe route my-app

All else fails?
├─ 1. Get deployment YAML: oc get deploy my-app -o yaml > deploy.yaml
├─ 2. Get pod YAML: oc get pod <pod> -o yaml > pod.yaml
├─ 3. Get events: oc get events --sort-by=.lastTimestamp > events.txt
├─ 4. Escalate with all 3 files
└─ 5. State what you already tried
```

---

## RECOVERY PLAYBOOK

When you find and fix an issue:

```
1. Fix Applied:
   oc set image deployment/my-app my-app=...
   # OR
   oc rollout restart deployment/my-app

2. Watch Rollout:
   oc rollout status deployment/my-app -n production

3. Verify Recovery:
   oc get pods -n production -l app=my-app -o wide
   oc port-forward svc/my-app 8080:80
   curl http://localhost:8080/health/live

4. Test Functionality:
   # Run your smoke tests
   ./smoke-tests.sh

5. Monitor for Regression:
   oc top pods -n production -l app=my-app --watch
   # Watch for 5-10 minutes

6. Document Issue:
   # Create ticket with:
   - Root cause
   - Symptoms observed
   - Fix applied
   - Prevention (if any)
```

---

*Keep this decision tree open while troubleshooting. Most issues follow one of these paths.*
