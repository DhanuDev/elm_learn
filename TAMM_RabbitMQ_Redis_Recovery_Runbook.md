# TAMM (tamt-prod) — RabbitMQ & Redis Storage Recovery Runbook

**Cluster:** prod2ocp4 (`console.apps.prod2ocp4.elm.sa`)
**Namespaces:** `dataservices-tamt-prod` (RabbitMQ/Redis), `apps-tamt-prod` (Tasks/backend/frontend)
**Situation:** Portworx (`px-csi-replicated`) PVC problem forced a hand-built workaround cluster on new (Nutanix) storage. The workaround RabbitMQ/Redis is now live but started **without the original data/topology**, so Tasks and users are failing. Old data is still on the **retained** Portworx PVCs.

**Your chosen strategy (from our Q&A):**
1. **Stabilize the live cluster first**, then recover old data (both, in order).
2. Old Portworx volumes are **intact & mountable** (Retain policy is set — confirmed in the YAMLs).
3. **No downtime** — recovery must be online (shovel/federation, rolling changes).

> ⚠️ **Two hard safety rules before you touch anything**
> 1. **Do NOT delete any PVC or PV** on `px-csi-replicated`. Both StatefulSets have `persistentVolumeClaimRetentionPolicy: {whenDeleted: Retain, whenScaled: Retain}` — good — but a manual `oc delete pvc` will still destroy the only copy of the old data.
> 2. The original `rabbitmq` and `redis`/`portowrx-redis-node` StatefulSets are **Helm/ArgoCD-managed** (`app.kubernetes.io/managed-by: Helm`). If ArgoCD **self-heal/auto-sync** is on, it can revert your manual scaling or re-create/prune resources mid-recovery. **Disable auto-sync on the ArgoCD app(s) first** (Section G0) and drive everything manually until you're done.

---

## A. What is actually broken (root causes)

You do not have one problem, you have **three overlapping ones**. Fixing storage alone will not make Tasks work.

### A1. RabbitMQ data is bound to the node name — a new StatefulSet name = empty broker
Your RabbitMQ pods set:

```
RABBITMQ_NODE_NAME  = rabbit@$(MY_POD_NAME).rabbitmq-headless.$(MY_POD_NAMESPACE).svc.cluster.local
RABBITMQ_MNESIA_DIR = /opt/bitnami/rabbitmq/.rabbitmq/mnesia/$(RABBITMQ_NODE_NAME)
```

The on-disk data (queues, bindings, messages, users, vhosts) lives in a directory **named after the node**. The original cluster's nodes are `rabbit@rabbitmq-0…`, `rabbit@rabbitmq-1…`, `rabbit@rabbitmq-2…`. The workaround cluster's pods are `rabbitmq-temp-2-0/1/2`, so its nodes are `rabbit@rabbitmq-temp-2-0…` etc.

**Consequence:** even if you mount the old Portworx PVC into the new pod, RabbitMQ looks for `mnesia/rabbit@rabbitmq-temp-2-0…`, doesn't find it, and boots **empty**. This is confirmed by the RabbitMQ maintainers: the old command that rewrote node identity inside mnesia was removed because it is *"not compatible with quorum queues and streams."* The supported way to recover messages after a node-name change is **blue-green**: start the original nodes under their original identity and **shovel** the messages across. This is exactly why "just re-attach the PVC" keeps failing — it can never work by design.

### A2. Split identity / overlapping label selectors (the "data not in sync" symptom)
The original `rabbitmq` StatefulSet and the workaround `rabbitmq-temp-2` **share the same pod labels and the same headless service**:

- Both pod templates carry `app.kubernetes.io/instance: tamt-paas-rabbitmq-prod` + `app.kubernetes.io/name: rabbitmq`.
- Both publish pod DNS under `serviceName: rabbitmq-headless`.

Someone already tried to fix this by editing the *StatefulSet's* top-level label to `rabbitmq-old` (visible in `rabbitmqPod.txt`, manager `Mozilla`, 2026-07-08). **That does nothing** — Services select **pods**, not StatefulSets, and the StatefulSet `selector`/pod-template labels are immutable. So any ClusterIP Service or peer-discovery lookup keyed on `{instance: tamt-paas-rabbitmq-prod, name: rabbitmq}` will match **both** clusters' pods the moment the old pods are running. Clients then land randomly on a broker that has the queue or one that doesn't → *"unable to process users / data not in sync."* The only reason it's not actively corrupting right now is that the old pods are scaled to 0.

The same overlap exists on Redis (`redis-node` / `redis-node-temp` / `portowrx-redis-node` all share `instance: tamt-paas-redis-prod`, `name: redis`).

### A3. The workaround broker started with no topology/state
`RABBITMQ_LOAD_DEFINITIONS: 'no'`. So `rabbitmq-temp-2` only has the exchanges/queues that reconnecting clients happened to re-declare (the UI shows 204 exchanges / 119 queues / 894 messages — a *partial* topology). Any **durable** exchange/queue/binding that no client re-declares is simply **missing**, so those task flows publish into the void or fail to consume. Redis likewise came up empty, so any session/lookup/state the app expected in cache is gone. **This — not the raw storage — is what's making Tasks fail.**

### Current-state map (verify against your cluster)

| Object | NS | Role | Storage | State |
|---|---|---|---|---|
| `rabbitmq` (labelled `rabbitmq-old`) | dataservices-tamt-prod | **Original** — holds old data | `px-csi-replicated` (Portworx), PVCs `data-rabbitmq-0/1/2` (Retained) | scaled down |
| `rabbitmq-temp` | dataservices-tamt-prod | abandoned workaround | — | 0/0 |
| `rabbitmq-temp-2` | dataservices-tamt-prod | **LIVE** workaround (serving now) | Nutanix (new) | 3/3 |
| `portowrx-redis-node` | dataservices-tamt-prod | original redis identity (Helm, migrated) | `px-csi-replicated` | 1/1 |
| `redis-node` | dataservices-tamt-prod | original redis | `px-csi-replicated` | 1/1 |
| `redis-node-temp` | dataservices-tamt-prod | **LIVE** workaround redis | Nutanix (new) | 3/3 |
| `tasks` | apps-tamt-prod | consumer app | — | 9/10 ready (1 failing) |

> The exact storage-class/PVC mapping per StatefulSet must be confirmed with the Phase-0 commands — treat the table as the working model, not gospel.

---

## B. Phase 0 — Read-only triage (run first, change nothing)

Log in and set context:

```bash
oc login https://api.prod2ocp4.elm.sa:6443     # cluster-admin or storage-admin
DS=dataservices-tamt-prod
APP=apps-tamt-prod
```

**B1. StatefulSets, pods, and the labels that actually route traffic**

```bash
oc -n $DS get statefulset -o wide
oc -n $DS get pods -o wide --show-labels | egrep 'rabbitmq|redis'
# The distinguishing label between old and live is the goal here.
# Expected: live temp pods = app.kubernetes.io/managed-by=Manual ; original = Helm
```

**B2. Which Service the app really connects to (this decides everything in Phase 1)**

```bash
# Services + their selectors + current endpoints in the data namespace
oc -n $DS get svc -l app.kubernetes.io/name=rabbitmq -o wide
oc -n $DS get svc | egrep 'rabbitmq|redis'
oc -n $DS get endpoints | egrep 'rabbitmq|redis'      # <-- do endpoints include BOTH clusters?
oc -n $DS describe svc rabbitmq        # note the selector

# What the Tasks app is actually configured with (host/uri):
oc -n $APP get deploy tasks -o yaml | grep -iE 'rabbit|amqp|redis|sentinel' -n
oc -n $APP get secret tasks-env-vars -o jsonpath='{.data}' | \
  tr ',' '\n'    # then base64 -d the AMQP_URL / REDIS_* keys you care about
```

> Key question B2 answers: does the app connect via a **stable Service name** (good) or via **per-pod DNS like `rabbitmq-0.rabbitmq-headless…`** (bad — those names don't exist on the temp cluster, which explains the 1 failing Tasks pod and intermittent failures).

**B3. Portworx / PV state for the old data (confirm it's really intact)**

```bash
oc -n $DS get pvc | egrep 'rabbitmq|redis'
oc get pv | egrep 'rabbitmq|redis|tamt'          # look for STATUS Bound/Released + RECLAIM Retain
# Portworx health (needs pxctl access, usually via the portworx namespace):
PX=$(oc -n portworx get pods -l name=portworx -o name | head -1)
oc -n portworx exec $PX -- /opt/pwx/bin/pxctl status
oc -n portworx exec $PX -- /opt/pwx/bin/pxctl volume list | egrep 'rabbitmq|redis'
# For any volume that won't attach later:
# oc -n portworx exec $PX -- /opt/pwx/bin/pxctl volume inspect <vol-id>
```

**B4. Live RabbitMQ cluster health & topology gap**

```bash
POD=rabbitmq-temp-2-0
oc -n $DS exec $POD -c rabbitmq -- rabbitmqctl cluster_status
oc -n $DS exec $POD -c rabbitmq -- rabbitmqctl list_queues name type durable messages messages_ready --no-table-headers | sort
oc -n $DS exec $POD -c rabbitmq -- rabbitmqctl list_exchanges name type durable --no-table-headers | wc -l
oc -n $DS exec $POD -c rabbitmq -- rabbitmqctl list_vhosts
# Save this as the "AFTER" topology to compare against the original in Phase 1.
```

**B5. The failing Tasks pod — get the real error**

```bash
oc -n $APP get pods -l app.kubernetes.io/instance=tasks
BAD=$(oc -n $APP get pods -l app.kubernetes.io/instance=tasks --field-selector=status.phase!=Running -o name | head -1)
oc -n $APP describe $BAD | sed -n '/Events/,$p'
oc -n $APP logs $BAD -c tasks --tail=200 | egrep -i 'amqp|rabbit|redis|timeout|refused|NOT_FOUND|resource_locked|queue|exchange'
```

Collect B1–B5 output before proceeding — it confirms the distinguishing label (B1), the routing (B2), storage integrity (B3), the topology gap (B4), and the concrete failure (B5).

---

## C. Phase 1 — Stabilize the LIVE cluster (online, no data recovery yet)

Goal: make `rabbitmq-temp-2` + `redis-node-temp` the single, unambiguous, complete brokers the app talks to.

### C1. Pin the client Services to the live cluster only
Using the distinguishing label found in B1 (assumed `app.kubernetes.io/managed-by: Manual` for the temp/live pods), narrow every **client-facing** Service selector so it can never route to the old pods — even after you boot them in Phase 2.

```bash
# RabbitMQ client service (repeat for the AMQP service the app uses, per B2)
oc -n $DS patch svc rabbitmq --type merge \
  -p '{"spec":{"selector":{"app.kubernetes.io/instance":"tamt-paas-rabbitmq-prod","app.kubernetes.io/name":"rabbitmq","app.kubernetes.io/managed-by":"Manual"}}}'

oc -n $DS get endpoints rabbitmq -o wide     # must now list ONLY rabbitmq-temp-2-* IPs
```

Do the same for the Redis **and** Redis-Sentinel client Services (pin to the `redis-node-temp` pods). If the temp pods do **not** have a unique label, add one to the *live* StatefulSet's pod template (this triggers a safe rolling restart of the temp cluster, which keeps quorum):

```bash
oc -n $DS patch statefulset rabbitmq-temp-2 --type merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"tamm.elm.sa/role":"active"}}}}}'
# then use tamm.elm.sa/role=active in the Service selector instead.
```

### C2. Make sure the app connects via the Service, not per-pod DNS
If B2 showed the app pointing at `rabbitmq-0.rabbitmq-headless…` (original pod names), change the connection to the **stable ClusterIP service** so it always reaches healthy nodes of the live cluster:

```
amqp://<user>:<pass>@rabbitmq.dataservices-tamt-prod.svc.cluster.local:5672/<vhost>
redis (sentinel):  master=<name>  sentinel host = <redis-sentinel-svc>.dataservices-tamt-prod.svc.cluster.local:26379
```

Update the `tasks-env-vars` secret (and the same secret for backend/SDK/other consumers) and roll the deployments:

```bash
oc -n $APP rollout restart deploy/tasks
oc -n $APP rollout status  deploy/tasks
```

### C3. Restore the missing topology onto the live broker (fixes "Tasks failing")
This is the step that most directly unblocks users. You need the **definitions** (vhosts, users, exchanges, queues, bindings, policies) that the workaround broker never loaded.

**Preferred — you already have a definitions file/secret.** The pod env references `RABBITMQ_DEFINITIONS_FILE: /app/load_definition.json`. Check for it:

```bash
oc -n $DS get secret,configmap | egrep -i 'definition|load_definition|rabbitmq-config'
# If found, extract the JSON, then import into the LIVE broker (idempotent, online):
oc -n $DS cp ./load_definition.json rabbitmq-temp-2-0:/tmp/defs.json -c rabbitmq
oc -n $DS exec rabbitmq-temp-2-0 -c rabbitmq -- rabbitmqctl import_definitions /tmp/defs.json
```

**If there is no definitions file**, you'll get the topology from the original cluster during Phase 2 (export from old, import into live) — see D4. Import of *definitions only* creates queues/exchanges/bindings without moving messages and is safe to run on the live broker.

After C1–C3, verify the live broker now has the full exchange/queue set (compare against B4) and that Tasks pods go 10/10 Ready and stop erroring. **Users should be served at this point**, even though old in-flight messages aren't back yet.

---

## D. Phase 2 — Online recovery of old RabbitMQ messages (blue-green + shovel)

Because the old PVCs are intact and you must stay live, **do not** try to graft the old PVCs onto the temp StatefulSet. Instead, boot the **original** StatefulSet on its own retained PVCs in isolation, then shovel its messages into the live cluster.

### D1. Boot the original nodes under their original identity, isolated from clients
The original `rabbitmq` StatefulSet still exists and its PVCs `data-rabbitmq-0/1/2` are retained, so scaling it up **reuses the old data** (pods `rabbitmq-0/1/2` → nodes `rabbit@rabbitmq-0…` → their own mnesia). Client traffic can't reach it because you pinned the Service in C1.

Set `RABBITMQ_FORCE_BOOT: 'yes'` so the nodes boot straight from their retained mnesia instead of running k8s peer discovery (peer discovery would see the shared `rabbitmq-headless` endpoints and could try to entangle with the live cluster). Then scale up:

```bash
# add/patch FORCE_BOOT on the ORIGINAL sts (env currently 'no')
oc -n $DS set env statefulset/rabbitmq RABBITMQ_FORCE_BOOT=yes
# bring the full original quorum up (3 nodes) so quorum queues can recover
oc -n $DS scale statefulset/rabbitmq --replicas=3
oc -n $DS rollout status statefulset/rabbitmq --timeout=10m
oc -n $DS exec rabbitmq-0 -c rabbitmq -- rabbitmqctl cluster_status   # expect rabbit@rabbitmq-0/1/2 only
```

> Bring up **all three** old nodes: RabbitMQ 4.0 has no classic mirrored queues, so quorum queues need a majority and some classic queues are pinned to node 1 or 2. A single node 0 would miss them.
>
> Re-confirm `oc -n $DS get endpoints rabbitmq` still shows **only** the temp pods after the old pods are Running. If old IPs appear, your Service selector (C1) isn't specific enough — fix before continuing.

### D2. Reach the old broker without a shared-label service
Create a **dedicated, uniquely-selected** service for the old cluster so the shovel can address it (and nothing else routes there):

```bash
cat <<'EOF' | oc -n dataservices-tamt-prod apply -f -
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-old-src
spec:
  clusterIP: None
  selector:
    statefulset.kubernetes.io/pod-name: rabbitmq-0   # exact old pod, no label overlap
  ports: [{name: amqp, port: 5672, targetPort: 5672}]
EOF
```

(Or simply target the old pod's stable DNS `rabbitmq-0.rabbitmq-headless.dataservices-tamt-prod.svc.cluster.local` — it resolves because the pod name and headless service match the original node identity.)

### D3. Drain old → live with dynamic shovels (online, message-safe)
Declare shovels **on the live cluster** that pull each old queue into the same-named queue on the live broker. `ack-mode: on-confirm` guarantees no message is removed from the source until safely confirmed on the destination; `src-delete-after: queue-length` drains the current backlog and then the shovel auto-removes itself.

Per-queue example:

```bash
oc -n $DS exec rabbitmq-temp-2-0 -c rabbitmq -- \
  rabbitmqctl set_parameter shovel drain-<queue> '{
    "src-protocol":"amqp091",
    "src-uri":"amqp://tamm:'"$OLDPASS"'@rabbitmq-0.rabbitmq-headless.dataservices-tamt-prod.svc.cluster.local:5672/%2f",
    "src-queue":"<queue>",
    "dest-protocol":"amqp091",
    "dest-uri":"amqp://tamm:'"$NEWPASS"'@localhost:5672/%2f",
    "dest-queue":"<queue>",
    "ack-mode":"on-confirm",
    "src-delete-after":"queue-length"
  }'
```

Drain **all** old queues in one shot (script it):

```bash
# 1) enumerate old queues (vhost + name) from the old broker
oc -n $DS exec rabbitmq-0 -c rabbitmq -- \
  rabbitmqctl list_queues -p / name messages --no-table-headers | awk '$2>0{print $1}' > /tmp/oldq.txt

# 2) create one shovel per queue on the LIVE broker
while read Q; do
  oc -n $DS exec rabbitmq-temp-2-0 -c rabbitmq -- \
    rabbitmqctl set_parameter shovel "drain-$Q" \
    "{\"src-protocol\":\"amqp091\",\"src-uri\":\"amqp://tamm:$OLDPASS@rabbitmq-0.rabbitmq-headless.dataservices-tamt-prod.svc.cluster.local:5672/%2f\",\"src-queue\":\"$Q\",\"dest-protocol\":\"amqp091\",\"dest-uri\":\"amqp://tamm:$NEWPASS@localhost:5672/%2f\",\"dest-queue\":\"$Q\",\"ack-mode\":\"on-confirm\",\"src-delete-after\":\"queue-length\"}"
done < /tmp/oldq.txt
```

Monitor and confirm the drain, then clean up shovels:

```bash
oc -n $DS exec rabbitmq-temp-2-0 -c rabbitmq -- rabbitmqctl shovel_status
watch "oc -n $DS exec rabbitmq-0 -c rabbitmq -- rabbitmqctl list_queues name messages --no-table-headers | awk '\$2>0'"
# when a shovel shows terminated / source drained:
oc -n $DS exec rabbitmq-temp-2-0 -c rabbitmq -- rabbitmqctl clear_parameter shovel drain-<queue>
```

> Repeat for **each vhost** (`-p <vhost>`) if the app uses more than the default `/`. If a queue with the same name already has live messages on the destination, shovel **appends** — ordering across the two sources is not preserved, which is normal and expected for a recovery merge.

### D4. If you had no definitions file (C3 fallback)
Export topology from the now-running original cluster and import into the live one **before** draining messages, so every destination queue/exchange/binding exists:

```bash
oc -n $DS exec rabbitmq-0 -c rabbitmq -- rabbitmqctl export_definitions /tmp/old-defs.json
oc -n $DS cp rabbitmq-0:/tmp/old-defs.json ./old-defs.json -c rabbitmq
oc -n $DS cp ./old-defs.json rabbitmq-temp-2-0:/tmp/old-defs.json -c rabbitmq
oc -n $DS exec rabbitmq-temp-2-0 -c rabbitmq -- rabbitmqctl import_definitions /tmp/old-defs.json
```

### D5. Stand down the original cluster
Once queues read 0 and consumers on the live cluster have processed the drained backlog:

```bash
oc -n $DS scale statefulset/rabbitmq --replicas=0
oc -n $DS set env statefulset/rabbitmq RABBITMQ_FORCE_BOOT=no
# keep the PVCs (retained) until you've signed off recovery, THEN clean up (Section G).
```

---

## E. Phase 3 — Redis recovery (online, selective)

Redis in TAMM is used behind Sentinel (`redis-node-temp` = live master/replicas/sentinels). Two facts drive the approach:

- Redis persistence (RDB/AOF in `/data`) is **not** tied to node name, so the old dump is loadable anywhere — easier than RabbitMQ.
- But you **cannot cleanly "merge"** two live Redis datasets; blindly loading the old RDB over the live one would roll back everything written since cutover. So recover **selectively**.

### E1. First decide: is any of it durable, or is it a cache?
```bash
oc -n $DS exec redis-node-temp-0 -c redis -- redis-cli -a "$REDIS_PW" info keyspace
oc -n $DS exec redis-node-temp-0 -c redis -- redis-cli -a "$REDIS_PW" --scan --count 100 | head
# sample TTLs — if nearly everything has a short TTL, it's a cache and needs no recovery.
```
If it's purely a cache/session store, **skip recovery** — just confirm the app points at the live Sentinel (C1/C2) and let it repopulate. Users may need to re-authenticate; that's expected.

### E2. If specific durable keys must come back — read the old data in isolation
Bring the old Redis data up as a **throwaway, standalone** instance (no Sentinel, unique labels/service) mounted on the old retained PVC, then copy only the keys you need into the live master.

```bash
# clone the old PVC so you never risk the original (Portworx CSI snapshot/clone), or mount read-mostly:
# then run a standalone redis pod using the old redis-data PVC, service name redis-old-src (unique selector)

# copy selected keys online from old -> live master using MIGRATE (per key/pattern):
oc -n $DS exec redis-old-src-0 -- redis-cli -a "$OLD_PW" --scan --pattern 'user:*' | while read K; do
  oc -n $DS exec redis-old-src-0 -- redis-cli -a "$OLD_PW" \
    MIGRATE redis-node-temp-0.<headless>.dataservices-tamt-prod.svc.cluster.local 6379 "$K" 0 5000 COPY REPLACE AUTH "$LIVE_PW"
done
```

`MIGRATE … COPY REPLACE` moves individual keys over the wire into the live master with no downtime. Scope the `--pattern` to only the durable key families the app needs (sessions, tokens, task state), never the whole keyspace.

### E3. Stand down the old redis identity
Scale `redis-node` / `portowrx-redis-node` back to 0 and remove the throwaway `redis-old-src` once done. Keep PVCs until sign-off.

---

## F. Why "attach it back on OCP" keeps failing (and the correct way)

You most likely hit one of these. For your chosen path (blue-green + shovel) you **do not need** to re-attach anything — scaling the original StatefulSet on its retained PVCs (D1) sidesteps all of it. This section is for completeness / if the original StatefulSet was deleted.

**F1. `volumeClaimTemplates` is immutable.** Editing a StatefulSet to point at the old PVC or a different storage class fails with:
```
Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals',
'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
```
You cannot re-home an existing StatefulSet's storage in place. Correct pattern — **orphan-delete and recreate**, which keeps the pods and PVCs:
```bash
oc -n $DS get statefulset rabbitmq -o yaml > rabbitmq-sts.yaml   # back it up
oc -n $DS delete statefulset rabbitmq --cascade=orphan            # pods + PVCs stay running
# edit rabbitmq-sts.yaml (fix volumeClaimTemplates / storageClassName), then:
oc -n $DS apply -f rabbitmq-sts.yaml                              # re-adopts existing pods & PVCs
```
A StatefulSet **reuses** a PVC that already exists with the expected name (`data-rabbitmq-0`, …); it only creates one when missing. So pre-creating correctly-named PVCs bound to the old PVs is how you "attach the old data" to a (re)created StatefulSet.

**F2. The old PV is `Released` and won't rebind.** If a PVC was deleted, its Retained PV goes `Released` and refuses new claims because of a stale `claimRef`. Clear it, then create a matching PVC:
```bash
oc get pv <pv-name> -o jsonpath='{.status.phase}{"\n"}'          # Released
oc patch pv <pv-name> --type merge -p '{"spec":{"claimRef":null}}'   # -> Available
# recreate the PVC with the EXACT name/namespace/size/storageClass the STS expects:
#   name: data-rabbitmq-0, storageClassName: px-csi-replicated, storage: 10Gi, accessModes: [ReadWriteOnce]
oc -n $DS apply -f pvc-data-rabbitmq-0.yaml                       # binds to the freed PV
```

**F3. Portworx multi-attach / "volume already attached to another node."** RWO volumes attach to one node at a time; a stuck attachment (from the failed pod/node) blocks the new pod:
```
Multi-Attach error for volume "pvc-…"  /  MountVolume.MountDevice failed … volume is attached to another node
```
```bash
oc -n portworx exec $PX -- /opt/pwx/bin/pxctl volume inspect <vol-id>   # see attached-on node
# ensure the old pod is fully deleted first; if the holding node is gone/NotReady, force-detach:
oc -n portworx exec $PX -- /opt/pwx/bin/pxctl host detach --volume <vol-id>
# clear any leftover VolumeAttachment object:
oc get volumeattachment | grep <pv-name>
oc delete volumeattachment <va-name>
```

**F4. Storage-class mismatch.** A PV provisioned by `px-csi-replicated` cannot bind a PVC that requests the Nutanix class (and vice-versa). The PVC's `storageClassName`, `accessModes`, and `resources.requests.storage` must all match the target PV exactly.

---

## G. Cleanup, ArgoCD, and prevention

**G0. Before Phase 1 — quiesce GitOps.** The original `rabbitmq`/`redis` are Helm/ArgoCD-managed; the temp ones are `managed-by: Manual` (out-of-band). With auto-sync on, ArgoCD may revert your scaling or prune the manual temp StatefulSets mid-recovery.
```bash
# identify the app(s), then disable auto-sync / self-heal for the duration:
argocd app list | egrep 'tamt|rabbitmq|redis'
argocd app set <app> --sync-policy none
# (or in the UI: App → Details → Sync Policy → Disable Auto-Sync + Self-Heal)
```
Re-enable only after you've reconciled Git to the intended final state.

**G1. After sign-off — collapse to one cluster.** Keep exactly one RabbitMQ and one Redis owning the `instance/name` labels.
```bash
oc -n $DS delete statefulset rabbitmq-temp        # 0/0 abandoned
# once recovery is signed off and PVCs backed up:
oc -n $DS delete statefulset rabbitmq --cascade=orphan   # then delete its drained PVCs deliberately
oc -n $DS delete pvc data-rabbitmq-0 data-rabbitmq-1 data-rabbitmq-2   # ONLY after backup + sign-off
```
Do the equivalent for the redundant Redis StatefulSets (`redis-node`, `portowrx-redis-node`) once their data is recovered/confirmed unneeded.

**G2. Make the workaround the real deployment.** The live cluster is currently hand-managed (`Manual`) with a fragile `-temp-2` name. Fold it back into Helm/ArgoCD under a **clean, unique** name + its **own headless service**, so you never again have two clusters sharing `rabbitmq-headless` and one label set. Update the Helm values to the Nutanix storage class and reconcile Git so ArgoCD owns it.

**G3. Prevention checklist**
- Never run two StatefulSets that share a headless service **and** pod labels — node identity + Service routing both break. For storage swaps, do a **named blue-green** (distinct service/labels) or **rebind PVs to the same-named StatefulSet** (F1/F2).
- Enable **RabbitMQ definitions export** on a schedule and **Redis RDB/AOF backups** to the S3/NSF bucket already in your architecture, so topology loss can't happen again.
- Keep `persistentVolumeClaimRetentionPolicy: Retain` (already set) on stateful data.
- Document the Portworx→Nutanix storage-class migration as the permanent fix for the data-services tier, and validate PVC provisioning on the new class before cutover.

---

## H. Verification checklist (exit criteria)

1. `oc -n $DS get endpoints rabbitmq` and the Redis/Sentinel services list **only** live-cluster pod IPs.
2. Live broker exchange/queue/binding set matches the original (compare B4 vs. imported defs); no `NOT_FOUND` / `resource_locked` in consumer logs.
3. `oc -n $APP get deploy tasks` → **10/10** Ready; B5 error gone; a synthetic user request succeeds end-to-end.
4. Old queues drained to 0 (`rabbitmqctl list_queues … messages`), all shovels cleared (`shovel_status` empty).
5. Required Redis keys present on live master (`redis-cli … exists <key>` / `info keyspace`).
6. Original `rabbitmq`/`redis` StatefulSets scaled to 0; ArgoCD reconciled to the intended final state; PVC backups taken before any deletion.
7. One RabbitMQ + one Redis cluster remain, uniquely labelled, owned by GitOps.

---

## I. What I still need from you to make this exact

1. **B2 output** — the AMQP/Redis connection string the app uses (Service name vs. per-pod DNS), and the selector of the `rabbitmq`/`redis` client Services. This decides whether C1 alone fixes routing or C2 is also required.
2. **B1 output** — confirmation of the label that distinguishes live (`temp`) pods from original pods (I'm assuming `managed-by: Manual` vs `Helm`).
3. **B4 vs. definitions** — whether a `load_definition.json` secret/configmap exists (drives C3 vs. D4).
4. **B5 output** — the actual error on the failing Tasks pod, to confirm the cause is missing topology/Redis vs. a bad endpoint.
5. **Queue types** — are the important queues **quorum** or **classic**? (Confirms whether all 3 old nodes are required in D1 — assume yes.)
6. **ArgoCD app name(s)** and whether auto-sync/self-heal is currently on.

Paste those and I'll turn Sections C–E into an exact, copy-paste sequence with your real names, vhosts, and queue list.
