# OpenShift Kubernetes Production Support Guide
## For Application Support Engineers - Zero to Production Ready

---

## TABLE OF CONTENTS
1. [Learning Roadmap](#learning-roadmap)
2. [Kubernetes Manifest Integration in OpenShift](#manifest-integration)
3. [Production Support Fundamentals](#fundamentals)
4. [Application Availability Troubleshooting](#troubleshooting)
5. [Real-World Scenarios](#scenarios)
6. [Quick Reference Checklists](#checklists)

---

## LEARNING ROADMAP

### Phase 1: Kubernetes Fundamentals (Week 1-2)
**Time: ~20-30 hours | Priority: HIGH**

#### 1.1 Core Concepts You MUST Know

**Pods - The Basic Unit**
```
Pod = Smallest deployable unit (like a Docker container wrapper)
├─ Can contain 1+ containers (usually 1)
├─ Shared network namespace (same IP, different ports)
├─ Short-lived (temporary, can be killed/recreated anytime)
└─ Never create pods directly in production
```

**Key Understanding:**
- A pod is ephemeral - treat it as disposable
- If a pod crashes, it's replaced by the orchestrator
- Don't rely on pod persistence or fixed IPs

**ReplicaSets - High Availability**
```
ReplicaSet = "Keep N pods running at all times"
├─ Monitors pods continuously
├─ If pod dies → creates new one instantly
├─ If too many pods → terminates extras
└─ Ensures desired state = actual state
```

**Deployments - Rolling Updates**
```
Deployment = ReplicaSet + Update Strategy
├─ Manages rolling updates (gradual replacement)
├─ Enables rollback to previous versions
├─ Controls how fast pods are updated
└─ This is what you interact with in production
```

**Services - Internal Networking**
```
Service = Stable network endpoint for pods
├─ ClusterIP (default) = Internal only
├─ NodePort = Internal + external port on all nodes
├─ LoadBalancer = External load balancer
└─ Service discovery via DNS (my-service.namespace.svc.cluster.local)
```

**Namespaces - Logical Isolation**
```
Namespace = Virtual cluster inside cluster
├─ Isolates resources (pods, services, configmaps)
├─ Separate RBAC permissions per namespace
├─ Multi-tenancy within one cluster
└─ Default, kube-system, openshift-* are system namespaces
```

#### 1.2 Essential Commands You'll Use Daily

```bash
# Get cluster info
kubectl cluster-info
oc version

# List resources
kubectl get pods -n production
kubectl get deployments -n production
kubectl get services -n production
kubectl get configmaps,secrets -n production

# Get detailed info
kubectl describe pod <pod-name> -n <namespace>
kubectl describe deployment <deployment-name> -n <namespace>

# View pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container-name> -n <namespace>  # Multi-container
kubectl logs <pod-name> --tail=100 -f                       # Follow logs
kubectl logs <pod-name> --previous                          # Crashed pod logs

# Execute commands in pod
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash
kubectl exec <pod-name> -- env                              # View env vars

# Edit resources
kubectl edit pod <pod-name> -n <namespace>
kubectl edit deployment <deployment-name> -n <namespace>

# Delete resources
kubectl delete pod <pod-name> -n <namespace>
kubectl delete deployment <deployment-name> -n <namespace>

# Get resource YAML
kubectl get deployment <name> -o yaml -n <namespace>

# Port forwarding (access service locally)
kubectl port-forward svc/<service-name> 8080:8080 -n <namespace>
```

#### 1.3 Must-Know Kubernetes Objects

| Object | Purpose | Lifespan | Use Case |
|--------|---------|----------|----------|
| **Pod** | Container runtime | Temporary | Debug, one-off jobs |
| **Deployment** | Manage replicas + updates | Long-lived | Production apps |
| **StatefulSet** | Ordered deployment with storage | Long-lived | Databases, queues |
| **DaemonSet** | Run on every node | Long-lived | Logging, monitoring agents |
| **Job** | Run to completion | Temporary | Batch processing |
| **CronJob** | Scheduled jobs | Recurring | Daily backups, cleanup |
| **Service** | Internal networking | Long-lived | App discovery, routing |
| **Ingress** | External HTTP routing | Long-lived | Public API endpoints |
| **ConfigMap** | Non-sensitive config | Long-lived | App settings, certs |
| **Secret** | Sensitive config | Long-lived | Passwords, API keys |
| **PersistentVolume** | Storage allocation | Long-lived | Data persistence |
| **PersistentVolumeClaim** | Storage request | Long-lived | Apps needing storage |

### Phase 2: OpenShift-Specific Features (Week 2-3)
**Time: ~15-20 hours | Priority: HIGH**

#### 2.1 What Makes OpenShift Different

**OpenShift = Kubernetes + Developer/Ops Tools**

```
Kubernetes Core          |  OpenShift Additions
─────────────────────────┼──────────────────────────
Deployments              |  DeploymentConfigs (legacy)
Ingress                  |  Routes (simpler DNS)
ServiceAccount + RBAC    |  OAuth2 + Project-scoped roles
kubectl                  |  oc (OpenShift CLI)
Generic Webhooks         |  Build Triggers
Manual Image Registry    |  Integrated Image Registry
Bare-metal storage       |  Storage Classes + PVs
```

**Routes vs Ingress:**
```yaml
# Standard Kubernetes Ingress (verbose)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: my-service
            port: 8080

# OpenShift Route (simpler)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
spec:
  host: myapp.example.com
  to:
    kind: Service
    name: my-service
    weight: 100
  port:
    targetPort: 8080
```

#### 2.2 OpenShift Projects vs Namespaces

- **Projects** = OpenShift's term for namespaces + permissions
- Everything in a project has RBAC rules
- You interact via `oc project <project-name>`
- Projects can have quotas (resource limits)

```bash
# Work with projects
oc projects                              # List all projects
oc project <project-name>                # Switch project
oc new-project my-app                    # Create new project
oc describe project <project-name>       # Quota + resource limits
```

#### 2.3 OpenShift Service Accounts & RBAC

Every pod runs as a **service account** with specific permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-app-role
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
```

#### 2.4 OpenShift Build & Image Registry

```bash
# OpenShift can build images from source code
oc new-build --binary --name my-app                    # Create BuildConfig
oc start-build my-app --from-dir=.                     # Build from code
oc logs -f build/my-app-1                              # Watch build logs

# Push image to OpenShift registry
# Format: image-registry.openshift-image-registry.svc:5000/<project>/<image>:<tag>
docker push image-registry.openshift-image-registry.svc:5000/my-app/myapp:v1.0
```

### Phase 3: Manifest Deep Dive (Week 3-4)
**Time: ~20-25 hours | Priority: CRITICAL**

---

## MANIFEST INTEGRATION

### Understanding Kubernetes YAML Structure

Every manifest has this structure:
```yaml
apiVersion: v1                    # API version (kubernetes version compatibility)
kind: Deployment                  # Object type
metadata:                         # Object identity
  name: my-app                    # Must be unique in namespace
  namespace: production           # Which project/namespace
  labels:                         # Key-value tags (for selection)
    app: my-app
    version: v1
spec:                             # Object specification
  replicas: 3                     # Desired state
  selector:                       # Which pods this controls
    matchLabels:
      app: my-app
  template:                       # Pod template (how to create pods)
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: myregistry/myapp:v1.0
        # More pod-level config...
```

### A Complete Production-Grade Manifest

```yaml
---
# 1. NAMESPACE (Logical separation)
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
# 2. SERVICE ACCOUNT (Security identity for the app)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production

---
# 3. ROLE (Permissions - what the app can do)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

---
# 4. ROLEBINDING (Connect ServiceAccount to Role)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-app-role
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production

---
# 5. CONFIGMAP (Non-sensitive config)
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: production
data:
  app.properties: |
    app.name=My App
    app.debug=false
    database.pool.size=20
  
---
# 6. SECRET (Sensitive config - base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: production
type: Opaque
stringData:  # Use stringData in YAML (auto base64-encoded)
  DATABASE_PASSWORD: "super-secret-password"
  API_KEY: "sk-1234567890"

---
# 7. DEPLOYMENT (Main application)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: v1
spec:
  # How many pod replicas?
  replicas: 3
  
  # Which pods does this control?
  selector:
    matchLabels:
      app: my-app
  
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired count during update
      maxUnavailable: 0  # Min pods available during update
  
  # Pod template (what to deploy)
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        description: "My production application"
    
    spec:
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      
      # Service account for pod identity
      serviceAccountName: my-app-sa
      
      # Init containers (run before app containers)
      initContainers:
      - name: migrate-db
        image: myregistry/my-app:v1.0
        command: ["./migrate.sh"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: my-app-secrets
              key: DATABASE_PASSWORD
      
      # Application containers
      containers:
      - name: my-app
        image: myregistry/my-app:v1.0
        imagePullPolicy: IfNotPresent
        
        # Ports exposed by container
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 9090
          name: metrics
          protocol: TCP
        
        # Environment variables
        env:
        # Static values
        - name: APP_ENV
          value: "production"
        - name: LOG_LEVEL
          value: "info"
        
        # From ConfigMap
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: my-app-config
              key: app.properties
        
        # From Secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-app-secrets
              key: DATABASE_PASSWORD
        
        # From pod metadata
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        # Resource limits (CRITICAL for production)
        resources:
          requests:           # Minimum guaranteed resources
            cpu: "500m"       # 0.5 CPU core
            memory: "512Mi"
          limits:             # Maximum resources allowed
            cpu: "1000m"      # 1.0 CPU core
            memory: "1Gi"
        
        # Readiness probe (is app ready for traffic?)
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Liveness probe (is app alive? if not, restart)
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Startup probe (is app starting? wait before liveness)
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30  # 30 * 10 = 5 minutes max startup
        
        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /etc/app
          readOnly: true
        - name: cache
          mountPath: /tmp/cache
        
        # Lifecycle hooks
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Graceful shutdown
      
      # Pod-level volumes
      volumes:
      - name: config
        configMap:
          name: my-app-config
      - name: cache
        emptyDir: {}  # Temporary storage
      
      # Pod scheduling
      nodeSelector:
        workload-type: general  # Run on nodes with this label
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["my-app"]
              topologyKey: kubernetes.io/hostname  # Spread across nodes
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 30
      
      # Pull image from private registry
      imagePullSecrets:
      - name: registry-credentials

---
# 8. SERVICE (Internal DNS + Load Balancing)
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP

---
# 9. ROUTE (External HTTP endpoint - OpenShift specific)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
  namespace: production
spec:
  host: my-app.example.com
  to:
    kind: Service
    name: my-app
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect

---
# 10. NETWORKPOLICY (Firewall rules)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 5432  # Database
    - protocol: TCP
      port: 443   # External APIs

---
# 11. HORIZONTAL POD AUTOSCALER (Auto-scaling)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60

---
# 12. PODDISRUPTIONBUDGET (High availability during maintenance)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  minAvailable: 2  # Keep at least 2 pods running
  selector:
    matchLabels:
      app: my-app
```

### How These Components Work Together

```
User Request Flow:
  1. User hits: my-app.example.com (DNS resolves to OpenShift router)
  2. Router matches Route object → finds Service: my-app
  3. Service uses selector (app: my-app) → finds Pods with that label
  4. Service load-balances request to one of the 3 pods
  5. Pod runs container with image: myregistry/my-app:v1.0
  6. Container starts via Deployment
     - ReplicaSet ensures 3 pods running
     - HPA may scale up/down based on CPU/Memory
  7. Pod mounts ConfigMap (read config from disk)
  8. Pod mounts Secret (read passwords from environment)
  9. Pod uses ServiceAccount to authenticate (if calling other APIs)
  10. Pod lifetime managed by liveness/readiness probes
```

### Manifest Deployment Steps

```bash
# Step 1: Validate syntax
kubectl apply -f manifest.yaml --dry-run=client -o yaml

# Step 2: Deploy to cluster
kubectl apply -f manifest.yaml

# Step 3: Watch rollout
kubectl rollout status deployment/my-app -n production

# Step 4: Verify pods are running
kubectl get pods -n production -l app=my-app

# Step 5: Check pod details
kubectl describe pod <pod-name> -n production

# Step 6: View logs
kubectl logs <pod-name> -n production

# Step 7: Test connectivity
kubectl port-forward svc/my-app 8080:80 -n production
# Visit: http://localhost:8080
```

### Common Manifest Mistakes (and how to fix them)

| Mistake | Symptom | Fix |
|---------|---------|-----|
| No resource requests/limits | Pod evicted when node memory full | Add resources.requests + limits |
| No readiness probe | 503 errors on app startup | Add readinessProbe checking /health |
| No liveness probe | Dead pod not restarted | Add livenessProbe checking app health |
| Wrong image name | ImagePullBackOff error | Verify image exists in registry, use `docker pull` to test |
| Secret not mounted | Pod can't find password | Add volumeMount for Secret, verify Secret exists |
| ConfigMap wrong format | App can't parse config | Verify ConfigMap data format matches app expectations |
| No service account | Pod auth fails | Create ServiceAccount + RoleBinding with needed permissions |
| Replicas: 1 | Single pod down = outage | Set replicas ≥ 3 for production |

---

## FUNDAMENTALS

### The Three States of Kubernetes

```
Desired State  ←→  Actual State  ←→  Applied State
(Your YAML)        (Running pods)     (What K8s is doing)

Kubernetes continuously works to make Actual = Desired
If pod dies → K8s creates new one
If image updated → K8s rolls out new pods
If resource limits hit → K8s evicts/restarts
```

### Health Check Hierarchy

**Pod Lifecycle Events** (in order):

```
Pod Created
    ↓
Init Containers Run (startup scripts, migrations)
    ↓
StartupProbe Checks (is app initializing? if fails 30 times → kill pod)
    ↓
Container Starts (app process begins)
    ↓
ReadinessProbe Checks (is app ready for traffic? if fails → remove from service)
    ↓
Traffic Flows (requests routed to pod)
    ↓
LivenessProbe Checks (is app alive? if fails 3 times → restart pod)
    ↓
Termination Signal (kubectl delete, update, etc.)
    ↓
PreStop Hook (graceful shutdown: sleep, flush caches)
    ↓
SIGTERM sent to app (15 second timeout)
    ↓
Pod Removed
```

### Replica vs Instance vs Pod

```
Deployment Spec:
  replicas: 3       ← "I want 3 copies"
    ↓ (ReplicaSet ensures this)
    Pod 1 (ip: 10.0.0.1)
    Pod 2 (ip: 10.0.0.2)  ← Each is independent
    Pod 3 (ip: 10.0.0.3)

Service:
  LoadBalances across all 3 pods
  Presents single IP/DNS to consumers
  If pod 2 dies → ReplicaSet creates pod 4
  If pod 4 healthy → Service routes to it

So "1 Deployment" = "3 Pods" = "3 Instances" (terminology varies)
```

---

## TROUBLESHOOTING

### The 5-Step Production Troubleshooting Framework

When an application is down or degraded:

#### STEP 1: Is the Deployment healthy?

```bash
# Check deployment status
kubectl get deployment my-app -n production
# Look for:
#   READY: X/3 (desired)
#   UP-TO-DATE: 3 (all updated to latest image)
#   AVAILABLE: 3 (all ready for traffic)

# Get more details
kubectl describe deployment my-app -n production
# Check: LastUpdateTime, Conditions, Status

# Watch rollout
kubectl rollout status deployment/my-app -n production
# Shows: "deployment "my-app" successfully rolled out"
# Or: "Waiting for deployment "my-app" to roll over new pods..."
```

**Possible Issues:**
- READY: 1/3 → Some pods not running
- UP-TO-DATE: 1/3 → Rollout stuck
- AVAILABLE: 0/3 → All pods failing

**Next Step:** Check PODS (Step 2)

---

#### STEP 2: Are the Pods Running?

```bash
# List all pods
kubectl get pods -n production -l app=my-app
# Look for STATUS column:
#   Running = Good
#   Pending = Not scheduled (Step 3)
#   CrashLoopBackOff = App crashing (Step 4)
#   ImagePullBackOff = Can't get image
#   Terminating = Being stopped

# Get pod details
kubectl describe pod <pod-name> -n production
# Check Conditions:
#   - Initialized: True
#   - Ready: True
#   - ContainersReady: True
#   - PodScheduled: True
# Check Events (bottom of output):
#   "Pulled image", "Created container", "Started container"
#   Any error events?

# View recent events cluster-wide
kubectl get events -n production --sort-by='.lastTimestamp'
```

**Status Meanings:**

| Status | Meaning | Action |
|--------|---------|--------|
| Running | Pod started, healthchecks may still be pending | Check readiness probe |
| Pending | Pod not yet scheduled | Check node resources or node affinity |
| CrashLoopBackOff | App keeps crashing, K8s restarts it | Check logs |
| ImagePullBackOff | Can't download container image | Check image name, registry credentials |
| CreateContainerConfigError | Invalid config (env var, volume, etc.) | Check deployment manifest |
| OOMKilled | Pod ran out of memory | Increase limits in manifest |
| Evicted | Node ran out of resources | Check node capacity, HPA scaling |
| Terminating | Pod being deleted | Wait or force delete |

**Next Step:** Check LOGS (Step 4)

---

#### STEP 3: Are Nodes Healthy?

```bash
# List all nodes
kubectl get nodes
# Look for STATUS: Ready, NotReady, SchedulingDisabled

# Get node details
kubectl describe node <node-name>
# Check Conditions (all should be True):
#   - Ready
#   - MemoryPressure
#   - DiskPressure
#   - PIDPressure
# Check Capacity:
#   - CPU, Memory, Storage available?
# Check Allocatable:
#   - How much available for pods?

# Check if node is cordoned
kubectl describe node <node-name> | grep -i cordoned
# If yes, uncordon:
kubectl uncordon <node-name>
```

**Common Node Issues:**
- NotReady = Node infrastructure problem (AWS down, network issue)
- MemoryPressure = Node running out of RAM, pods getting evicted
- DiskPressure = Node disk full
- Unschedulable = Node cordoned or labeled differently

**Next Step:** If nodes OK, go back to Step 2 (pods)

---

#### STEP 4: Are Container Logs Showing Errors?

```bash
# View logs from running pod
kubectl logs <pod-name> -n production
# Common issues:
#   - "Connection refused" = app can't reach database/dependency
#   - "Out of memory" = increase limits
#   - "Port already in use" = port conflict
#   - Stack traces = application bug

# View logs from previous crashed pod
kubectl logs <pod-name> --previous -n production
# Shows what happened before crash

# Stream logs in real-time
kubectl logs <pod-name> -f -n production
# Ctrl+C to stop

# View logs from multiple pods
kubectl logs -l app=my-app --tail=50 -n production

# View logs with timestamps
kubectl logs <pod-name> --timestamps=true -n production
```

**Next Step:** If logs show app issues, fix code. If infra issues, continue to Step 5.

---

#### STEP 5: Check Application Health Endpoints

```bash
# Port-forward to pod to test endpoints directly
kubectl port-forward pod/<pod-name> 8080:8080 -n production
# In another terminal:
curl http://localhost:8080/health/live
curl http://localhost:8080/health/ready
curl http://localhost:8080/metrics

# Or exec into pod to debug
kubectl exec -it <pod-name> -n production -- /bin/bash
# Inside pod:
curl http://localhost:8080/
ps aux  # See running processes
env     # See environment variables
cat /etc/hostname  # Verify pod identity
ping <service-name>  # Test DNS resolution
```

---

### Common Failure Scenarios & Solutions

#### Scenario 1: "Pod is Running but Request Timing Out"

```bash
# Step 1: Is readiness probe failing?
kubectl get pod <pod-name> -o yaml | grep -A 5 "readinessProbe"
kubectl describe pod <pod-name> | grep -A 10 "Conditions"

# If Ready: False, check logs
kubectl logs <pod-name> --tail=100

# Step 2: Is service endpoint found?
kubectl get endpoints my-app -n production
# Should show IP:port for each pod

# Step 3: Is service pointing right port?
kubectl get svc my-app -o yaml | grep -A 5 "ports:"

# Step 4: Can you reach pod directly?
kubectl port-forward pod/<pod-name> 8080:8080
curl http://localhost:8080  # Direct pod access

# Step 5: Is network policy blocking?
kubectl get networkpolicies -n production
kubectl describe networkpolicy <policy-name>

# Fix: Either update readiness probe or fix app health endpoint
```

#### Scenario 2: "ImagePullBackOff Error"

```bash
# Check which image it's trying to pull
kubectl describe pod <pod-name> | grep "Image:"

# Verify image exists in registry
docker pull myregistry/my-app:v1.0

# If using private registry, check credentials
kubectl get secrets -n production
kubectl describe secret <image-pull-secret>

# Fix: Either:
# - Correct image name in deployment
# - Ensure image exists in registry
# - Ensure secret is created and mounted in spec.imagePullSecrets
```

#### Scenario 3: "CrashLoopBackOff - Pod Keeps Restarting"

```bash
# Check crash reason
kubectl describe pod <pod-name> | grep -A 5 "State:"

# View last container logs before crash
kubectl logs <pod-name> --previous --tail=200

# Check liveness probe settings
kubectl get pod <pod-name> -o yaml | grep -A 10 "livenessProbe:"

# Possible causes:
# 1. App process exiting (check logs)
# 2. Out of memory (increase limits)
# 3. Liveness probe failing (check probe endpoint)
# 4. Missing dependency/config (check env vars, volumes)

# Fix:
# - Fix application code
# - Increase resource limits
# - Update health check endpoint
# - Add missing ConfigMap/Secret
```

#### Scenario 4: "Pod Pending - Won't Schedule"

```bash
# Check pod details
kubectl describe pod <pod-name>

# Look for events like:
# "0/3 nodes are available: insufficient CPU"
# "0/3 nodes are available: node(s) did not match Pod's node selector"

# Check node capacity
kubectl top nodes
kubectl describe node <node-name>

# Check pod resource requests
kubectl get pod <pod-name> -o yaml | grep -A 5 "resources:"

# Possible causes:
# 1. Not enough CPU/Memory on nodes
# 2. Node selector doesn't match any nodes
# 3. Pod affinity/anti-affinity can't be satisfied

# Fix:
# - Add more nodes to cluster
# - Reduce resource requests
# - Remove restrictive node selectors
# - Add nodes with required labels
```

#### Scenario 5: "503 Service Unavailable - Readiness Probe Failing"

```bash
# Check readiness status
kubectl get pod <pod-name> -o yaml | grep "ready" -i

# Check readiness probe definition
kubectl get pod <pod-name> -o yaml | grep -A 10 "readinessProbe:"

# Manually test readiness endpoint
kubectl port-forward pod/<pod-name> 8080:8080
curl http://localhost:8080/health/ready
# If not 200, the probe is correctly failing

# Check logs for why app isn't ready
kubectl logs <pod-name> --tail=100

# Possible causes:
# 1. Database not reachable
# 2. Dependencies starting up
# 3. Initial data load in progress
# 4. Readiness endpoint buggy

# Fix:
# - Increase initialDelaySeconds
# - Fix dependency connectivity
# - Fix readiness endpoint logic
# - Wait for startup to complete
```

---

## SCENARIOS

### Real-World Scenario 1: "Production App Down at 3AM"

**Timeline:**
- 3:00 AM: Alerting fires "No healthy pods"
- 3:02 AM: You get paged
- 3:05 AM: You investigate

**Investigation Steps:**

```bash
# 1. Confirm app is down
curl https://my-app.example.com
# → Connection timeout or 503

# 2. Check deployment
oc get deployment my-app -n production
# → Result:
# NAME     READY   UP-TO-DATE   AVAILABLE
# my-app   0/3     3            0
# Problem: 0 pods available

# 3. Check pod status
oc get pods -n production -l app=my-app
# → Result:
# my-app-5f4d8b9c-abc12   0/1 CrashLoopBackOff
# my-app-5f4d8b9c-abc23   0/1 CrashLoopBackOff
# my-app-5f4d8b9c-abc34   0/1 CrashLoopBackOff
# Problem: All 3 pods crashing

# 4. Check logs
oc logs my-app-5f4d8b9c-abc12 --tail=50 -n production
# → Logs show:
# ERROR: Database connection refused
# Problem: Database unreachable

# 5. Investigate database
oc get svc database -n production
oc describe svc database -n production
# → Endpoints: none!
# Problem: Database service has no healthy pods

# 6. Check database pod
oc get pods -n production -l app=database
# → Result:
# database-54f7a8d9-xyz1   0/1 CrashLoopBackOff
# Problem: Database pod also crashing

# 7. Check database logs
oc logs database-54f7a8d9-xyz1 --tail=100 -n production
# → Logs show:
# FATAL: Storage /data is full
# Problem: Database storage full

# 8. Check persistent volume
oc get pvc -n production
# → Result:
# database-pvc   Bound   pv-abc123   10Gi   RWO   2 months old
# Usage: 10Gi (100% full)

# ROOT CAUSE: Database storage full → crashes
# Application can't connect → crashes
```

**Solution:**

```bash
# Immediate (get service back up)
# 1. Scale down app temporarily
oc scale deployment my-app --replicas=0 -n production

# 2. Expand database storage (or clean up logs)
# Contact storage team OR manually clean logs in database pod

# 3. Restart database
oc rollout restart deployment database -n production
# Wait for it to be healthy
oc rollout status deployment database -n production

# 4. Scale app back up
oc scale deployment my-app --replicas=3 -n production
oc rollout status deployment my-app -n production

# 5. Verify service is up
curl https://my-app.example.com
# → Should be 200 OK

# Long-term (prevent recurrence)
# - Add monitoring on PVC usage (alert at 70%, 85%, 95%)
# - Implement log rotation in database
# - Increase PVC size
# - Add HPA to auto-scale if disk usage increases
```

---

### Real-World Scenario 2: "Rolling Update Gone Wrong - Database Stuck"

**Timeline:**
- 10:00 AM: You deploy new version v1.5
- 10:05 AM: Deployment stuck, some requests failing
- 10:06 AM: You investigate

**Investigation:**

```bash
# 1. Check deployment status
oc get deployment my-app -n production
# → Result:
# READY: 2/3
# UP-TO-DATE: 1/3
# Problem: Update stalled

# 2. Check rollout status
oc rollout status deployment my-app -n production
# → Shows: "waiting for rollout to finish: 2 out of 3 new replicas have been updated"

# 3. Check new pod
oc describe pod my-app-v15-abc123 -n production
# → Events show:
# Created container
# Started container
# ✓ Container started
# But Status: Running
# Check: Ready: False (readiness probe failing)

# 4. Check logs
oc logs my-app-v15-abc123 --tail=100 -n production
# → Logs show:
# ERROR: Database schema migration pending
# ERROR: Cannot start app until migration complete
# Problem: v1.5 requires new database schema

# 5. Check deployment manifest
oc get deployment my-app -o yaml | grep -A 20 "spec:"
# → Issue: No initContainer to run database migration
```

**Solution:**

```bash
# Immediate (stop the bad rollout)
# 1. Pause the rollout
oc rollout pause deployment my-app -n production

# 2. Rollback to previous version
oc rollout undo deployment my-app -n production
# Wait for it to stabilize
oc rollout status deployment my-app -n production

# 3. Service should be back online
curl https://my-app.example.com
# → Should be 200 OK

# Fix the deployment (long-term)
# Update deployment manifest:
```

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: db-migrate
        image: myregistry/my-app:v1.5
        command:
        - /bin/sh
        - -c
        - |
          #!/bin/bash
          set -e
          echo "Running database migration..."
          ./migrate-db.sh
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: my-app-secrets
              key: DATABASE_URL
      containers:
      - name: my-app
        # ... rest of spec
```

```bash
# Redeploy with fix
oc apply -f updated-deployment.yaml
oc rollout status deployment my-app -n production
```

---

### Real-World Scenario 3: "Memory Leak - Pods Getting Evicted"

**Timeline:**
- 11:00 AM: App running fine
- 1:00 PM: Notices high latency
- 1:30 PM: Pods keep restarting
- 2:00 PM: You investigate

**Investigation:**

```bash
# 1. Check pod restarts
oc get pods -n production -l app=my-app
# → RESTARTS column shows: 5, 7, 4
# Problem: Pods restarting frequently

# 2. Check events
oc describe pod <pod-name> -n production
# → Events show:
# FailedScheduling: "Pod evicted due to memory pressure"

# 3. Check node memory
oc top nodes
# → Shows one node at 95% memory

# 4. Check pod memory usage
oc top pods -n production -l app=my-app
# → Shows each pod using 750Mi (limits are 1Gi)
# Problem: Growing memory usage (memory leak)

# 5. Check app metrics (if Prometheus available)
# Graph memory over time → Should see upward trend
```

**Solution:**

```bash
# Immediate (relieve pressure)
# 1. Force new pod restarts (force garbage collection)
oc rollout restart deployment my-app -n production

# 2. If still having issues, scale down temporarily
oc scale deployment my-app --replicas=2 -n production

# 3. Add temporary monitoring
oc exec <pod-name> -n production -- ps aux
oc exec <pod-name> -n production -- free -h

# Long-term (fix leak)
# 1. Check app code for memory leak
# - Unclosed connections
# - Growing caches without eviction
# - Circular references in objects
# - Event listeners not unsubscribed

# 2. Increase memory limits temporarily while fixing
oc set resources deployment my-app \
  --limits=memory=2Gi \
  --requests=memory=1Gi \
  -n production

# 3. Once fixed, deploy new version
oc set image deployment/my-app \
  my-app=myregistry/my-app:v1.6-fixed \
  -n production

# 4. Monitor memory after deploy
oc top pods -n production -l app=my-app --watch
# Memory should stabilize, not climb

# 5. Once stable, reduce limits back
oc set resources deployment my-app \
  --limits=memory=1Gi \
  --requests=memory=512Mi \
  -n production
```

---

### Real-World Scenario 4: "Networking Issue - Can't Reach Database"

**Timeline:**
- 9:00 AM: Deploy new version with database access
- 9:02 AM: Requests failing with "database connection timeout"
- 9:05 AM: You investigate

**Investigation:**

```bash
# 1. Check app logs
oc logs <pod-name> -n production | head -50
# → ERROR: Cannot connect to database.production.svc.cluster.local:5432

# 2. Check if database service exists
oc get svc database -n production
# → Service exists: database.production.svc.cluster.local

# 3. Check service endpoints
oc get endpoints database -n production
# → Shows: (no endpoints)
# Problem: Service has no healthy pods

# 4. Check database pods
oc get pods -n production -l app=database
# → All pods are CrashLoopBackOff or Pending

# 5. Check network connectivity from app pod
oc exec -it <app-pod> -n production -- /bin/bash
# Inside pod:
$ nslookup database.production.svc.cluster.local
# Should resolve to service IP

# 6. Check NetworkPolicy
oc get networkpolicies -n production
oc describe networkpolicy <policy-name>
# Check if policy allows traffic to database port

# 7. Check if database is in different namespace
oc get pods -n other-namespace -l app=database
# If database is in different namespace, DNS would be:
# database.other-namespace.svc.cluster.local
```

**Solution:**

```bash
# Case 1: Database pods are down
# Fix database first (see Scenario 1 steps)

# Case 2: NetworkPolicy blocking traffic
# Update NetworkPolicy to allow database port:
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Allow database
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow external HTTPS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

```bash
# Case 3: Database in different namespace
# Update deployment connection string:
oc set env deployment/my-app \
  DATABASE_HOST=database.other-namespace.svc.cluster.local \
  -n production
```

---

## CHECKLISTS

### Daily Health Check (5 minutes)

```bash
# 1. Are all deployments healthy?
oc get deployments -n production
# All should show: READY X/X, UP-TO-DATE X, AVAILABLE X

# 2. Are all pods running?
oc get pods -n production
# All should show STATUS: Running

# 3. Any CrashLoopBackOff?
oc get pods -n production | grep -i crash
# Should be empty

# 4. Check recent events
oc get events -n production --sort-by='.lastTimestamp' | tail -20
# Look for any ERROR or WARNING events

# 5. Check node health
oc get nodes
# All should show STATUS: Ready

# 6. Check PVC usage
oc get pvc -n production
# Usage should be <80%
```

### Weekly Production Review (30 minutes)

```bash
# 1. Check resource utilization
oc top nodes
oc top pods -n production --sort-by=memory

# 2. Review recent rollouts
oc rollout history deployment/my-app -n production
# Any failed deployments in history?

# 3. Check certificate expiration (from earlier lesson)
oc get configmap openshiftca.crt -n production -o \
  jsonpath='{.data.service-ca\.crt}' | \
  openssl x509 -noout -dates

# 4. Review resource requests/limits
oc get deployment -n production -o yaml | \
  grep -A 5 "resources:"

# 5. Check scaling metrics
oc get hpa -n production

# 6. Review logs for errors
oc logs deployment/my-app -n production --tail=500 | \
  grep -i error | sort | uniq -c

# 7. Check backup status (if applicable)
oc get pvc -n production -o yaml | \
  grep -i snapshot
```

### Post-Deployment Checklist

```bash
# After deploying new version:

# 1. Verify deployment started
oc rollout status deployment/my-app -n production

# 2. All replicas healthy?
oc get deployment my-app -n production
# READY should equal DESIRED

# 3. Pods are running
oc get pods -n production -l app=my-app
# All STATUS: Running

# 4. Pod endpoints working
oc get endpoints my-app -n production
# Should show all pod IPs

# 5. Service responding
kubectl port-forward svc/my-app 8080:80 -n production &
curl http://localhost:8080/health/live
# Should be 200 OK

# 6. Check logs for errors
oc logs -l app=my-app -n production --tail=50
# Should be clean (no ERROR or FATAL)

# 7. Verify in-place upgrades completed
oc get pods -n production -l app=my-app -o yaml | \
  grep "image:" | sort | uniq
# All should be new image version

# 8. Test critical functionality
# Run smoke tests or manual verification

# 9. Monitor for 10-15 minutes
oc top pods -n production -l app=my-app --watch
# CPU/Memory should be stable
```

---

## QUICK REFERENCE

### Essential CLI Patterns

```bash
# ✓ = OpenShift command (oc)
# ✓ = Also works with kubectl

# Namespace operations
oc projects
oc project <name>
oc new-project <name>

# View resources
oc get pods -n production
oc get deployments -n production
oc get all -n production

# Detailed info
oc describe deployment my-app -n production
oc describe pod <pod-name> -n production

# Logs
oc logs <pod-name> -n production
oc logs deployment/my-app -n production
oc logs <pod-name> --previous -n production
oc logs <pod-name> -f -n production  # Follow

# Modify resources
oc set image deployment/my-app my-app=myregistry/my-app:v2 -n production
oc set env deployment/my-app VAR=value -n production
oc set resources deployment/my-app --limits=memory=1Gi -n production

# Scaling
oc scale deployment/my-app --replicas=5 -n production

# Rollout management
oc rollout status deployment/my-app -n production
oc rollout history deployment/my-app -n production
oc rollout undo deployment/my-app -n production

# Debugging
oc exec -it <pod-name> -- /bin/bash
oc exec <pod-name> -- ps aux
oc port-forward pod/<pod-name> 8080:8080

# Resource metrics
oc top nodes
oc top pods -n production

# View YAML
oc get deployment my-app -o yaml -n production
oc get pod <pod-name> -o yaml -n production

# Delete resources
oc delete pod <pod-name> -n production
oc delete deployment my-app -n production

# Apply manifests
oc apply -f deployment.yaml
oc apply -f *.yaml
```

### Health Check Endpoints (App Requirements)

Every production app MUST expose these endpoints:

```
/health/startup    (optional) - App is initializing - respond 200 when ready
/health/ready      (required) - App ready for traffic - respond 200 when DB OK
/health/live       (required) - App alive - respond 200 unless dead loop
/metrics           (recommended) - Prometheus metrics for monitoring
```

### Probes Configuration Template

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30      # 30 * 10 = 5 min max
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10   # Wait 10s before first check
  periodSeconds: 5          # Check every 5s
  timeoutSeconds: 3         # Timeout after 3s
  failureThreshold: 3       # 3 failures = not ready

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30   # Wait 30s before first check
  periodSeconds: 10         # Check every 10s
  timeoutSeconds: 3         # Timeout after 3s
  failureThreshold: 3       # 3 failures = restart pod
```

---

## SUMMARY

### You Now Know:

✓ How Kubernetes containers are organized (Pods → ReplicaSets → Deployments)
✓ How services discover and load-balance pods
✓ How OpenShift extends Kubernetes (Routes, BuildConfigs, Projects)
✓ How to read and write production-grade manifests
✓ The 5-step troubleshooting framework for any outage
✓ How to monitor health and catch issues early
✓ How to safely deploy updates without downtime
✓ Common production failure modes and fixes

### Key Takeaways for Day-to-Day Work:

1. **Probes are critical** - Liveness keeps app alive, readiness keeps traffic away from broken pods
2. **Logs are your best friend** - 80% of issues solved by reading logs
3. **Resource limits matter** - OOMKilled and evictions are silent killers
4. **Networks are isolate** - NetworkPolicies, DNS, port misconfigs cause unexpected outages
5. **Always have rollback plan** - Version control manifests, know how to `oc rollout undo`
6. **Monitoring > Debugging** - Catch issues before they impact users
7. **Manifest > Manual changes** - Everything in Git, reproducible, auditable

### Next Steps:

1. **Practice**: Spend 1 week deploying test apps, breaking them, fixing them
2. **Shadow**: Sit with oncall engineer during production incident
3. **Document**: Create runbooks for your company's specific applications
4. **Automate**: Write scripts to check health, alert on issues
5. **Train**: Teach developers how to write production-ready manifests

---

*This guide prepared for production support engineers starting OpenShift Kubernetes operations.*
*Last updated: 2026-04-15*
