# Day 3 – ReplicaSet & Deployment

Image used: **`traefik/whoami`** – tiny Go HTTP server (~7MB) on port 80.
It never exits (unlike Day 2's busybox), and every response shows the **pod hostname + IP** → easy to see which replica answered.

Files:
- [`replicaset.yaml`](replicaset.yaml) – 3 replicas, managed by a ReplicaSet directly
- [`deployment.yaml`](deployment.yaml) – 3 replicas, managed by a Deployment (which creates a ReplicaSet for us)

> ⚠️ Before anything: `kubectl config current-context` → must be `kind-k8srevision`, not the EKS cluster.

---

## 1. Why not just Pods?

Day 2 problem: a bare Pod has **no self-healing**. Delete it / node dies → it's gone forever.
We need something that watches pods and says *"I want 3 running, always"*.

```
Deployment  ──manages──►  ReplicaSet  ──manages──►  Pods
(versions, rollout,        (keep N copies           (actual
 rollback)                  running)                 containers)
```

---

## 2. ReplicaSet

**Job:** make sure **exactly N pods** matching a label selector are running at all times.

- Pod deleted / crashed → RS creates a new one.
- Extra pod with same label appears → RS deletes one (it counts by label!).
- It runs a **control loop**: `desired state (replicas: 3)` vs `actual state (pods running)` → fix the difference.

### Key fields

| Field | Meaning |
|---|---|
| `apiVersion: apps/v1` | RS & Deployment are in `apps` group (Pod is core `v1`) |
| `replicas` | How many pods we want |
| `selector.matchLabels` | Which pods this RS owns/counts |
| `template` | Pod blueprint used when creating new pods |
| `template.metadata.labels` | **Must match** `selector` – otherwise API rejects the YAML |

### Limitation of ReplicaSet
If you change the image in the RS template and re-apply → **existing pods are NOT updated**.
Only *new* pods (created after a delete) get the new image. No rolling update, no rollback.
👉 That's why we almost never create ReplicaSets directly – we use **Deployments**.

### Hands-on: ReplicaSet

```bash
kubectl apply -f replicaset.yaml
kubectl get rs                                # DESIRED / CURRENT / READY = 3
kubectl get pods -l app=whoami-rs -o wide     # 3 pods, random suffix names: whoami-rs-xxxxx

# Self-healing test: delete one pod, watch RS recreate it
kubectl delete pod <one-whoami-rs-pod-name>
kubectl get pods -l app=whoami-rs -w          # new pod appears (Ctrl+C to stop)

# Scale
kubectl scale rs whoami-rs --replicas=5
kubectl get pods -l app=whoami-rs

# See who owns a pod
kubectl describe pod <whoami-rs-pod> | grep "Controlled By"   # → ReplicaSet/whoami-rs

# Clean up (deleting RS deletes its pods)
kubectl delete -f replicaset.yaml
```

---

## 3. Deployment

**Job:** manage ReplicaSets for you + handle **versioned updates**.

- Gives everything RS gives (replicas, self-healing) **plus**:
  - **Rolling update** – replace pods gradually, zero downtime
  - **Rollback** – go back to previous version with one command
  - **Rollout history** – each change = new revision
  - Pause / resume rollouts
- Every time the **pod template** changes (image, env, etc.), the Deployment creates a **new ReplicaSet** and shifts pods from old RS → new RS.
- Old RS is kept with `0` replicas (for rollback) – `revisionHistoryLimit` controls how many.

### Naming chain (look at this in `kubectl get`)
```
Deployment:  whoami-deploy
ReplicaSet:  whoami-deploy-6d8f7c9b5d          ← pod-template-hash
Pod:         whoami-deploy-6d8f7c9b5d-x7k2p    ← random suffix
```

### Update strategies

| Strategy | How | Downtime? | Use when |
|---|---|---|---|
| **RollingUpdate** (default) | Replace pods a few at a time | No | Most apps |
| **Recreate** | Kill all old pods, then start new ones | Yes | App can't run 2 versions at once (e.g. DB schema lock) |

**RollingUpdate knobs** (our file):
- `maxSurge: 1` → may temporarily run 3 + 1 = **4** pods
- `maxUnavailable: 0` → never fewer than **3** ready pods
- Result: start 1 new → when ready, kill 1 old → repeat. Zero downtime.

### Rolling update flow (3 replicas, v1 → v2)
```
old RS (v1): ●●●        new RS (v2):
old RS (v1): ●●●        new RS (v2): ○          ← surge: create 1 new
old RS (v1): ●●         new RS (v2): ●          ← new ready → kill 1 old
old RS (v1): ●          new RS (v2): ●●
old RS (v1):            new RS (v2): ●●●        ← done, old RS kept at 0
```

### Hands-on: Deployment

```bash
# Create
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods -l app=whoami-deploy
kubectl rollout status deploy/whoami-deploy

# Hit the Go server (port-forward to the deployment, open 2nd terminal)
kubectl port-forward deploy/whoami-deploy 8080:80
curl localhost:8080             # shows Hostname: whoami-deploy-xxxx-yyyy, IP, headers

# Scale
kubectl scale deploy whoami-deploy --replicas=5

# Rolling update – option A: edit YAML image to v1.10.2, then
kubectl apply -f deployment.yaml
# Rolling update – option B: imperative
kubectl set image deploy/whoami-deploy whoami=traefik/whoami:v1.10.2

# Watch it happen (run in another terminal BEFORE updating)
kubectl get pods -l app=whoami-deploy -w
kubectl get rs -l app=whoami-deploy           # 2 RS now: old (0 pods), new (3 pods)

# Add reason to history
kubectl annotate deploy/whoami-deploy kubernetes.io/change-cause="upgrade to v1.10.2"

# History & rollback
kubectl rollout history deploy/whoami-deploy
kubectl rollout history deploy/whoami-deploy --revision=2   # details of a revision
kubectl rollout undo deploy/whoami-deploy                   # back to previous
kubectl rollout undo deploy/whoami-deploy --to-revision=1   # back to specific one

# Pause / resume (batch several changes into one rollout)
kubectl rollout pause deploy/whoami-deploy
kubectl rollout resume deploy/whoami-deploy

# Restart all pods (e.g. to pick up new ConfigMap) – no YAML change needed
kubectl rollout restart deploy/whoami-deploy

# Break it on purpose: bad image → see rollout get stuck, old pods stay alive
kubectl set image deploy/whoami-deploy whoami=traefik/whoami:does-not-exist
kubectl get pods -l app=whoami-deploy         # new pod: ImagePullBackOff, old 3 still Running (maxUnavailable: 0!)
kubectl rollout undo deploy/whoami-deploy     # fix

# Clean up (deletes RS + pods too)
kubectl delete -f deployment.yaml
```

---

## 4. ReplicaSet vs Deployment

| | ReplicaSet | Deployment |
|---|---|---|
| Keeps N pods running | ✅ | ✅ (via RS) |
| Self-healing | ✅ | ✅ |
| Scaling | ✅ | ✅ |
| Rolling update on image change | ❌ | ✅ |
| Rollback / history | ❌ | ✅ |
| Create it directly? | Rarely | **Yes – default choice for stateless apps** |

---

## 5. Important Points / Gotchas

- `selector` **must match** `template.metadata.labels`, and `selector` is **immutable** after creation.
- Don't give two controllers the same labels – they'll fight over pods. (That's why RS uses `app: whoami-rs` and Deployment uses `app: whoami-deploy`.)
- **Never edit the ReplicaSet created by a Deployment** – the Deployment will overwrite it. Change the Deployment.
- Only changes to `spec.template` trigger a new rollout. Changing `replicas` = just scaling, no new revision.
- `kubectl scale` is imperative – next `kubectl apply -f` resets replicas to what's in YAML. Prefer editing YAML.
- Pods from a Deployment still get **new IPs** when recreated → next topic: **Services** (stable IP/DNS + load balancing across replicas).
- Without readiness probes, K8s treats a pod as "ready" as soon as container starts – rolling update may send traffic too early. (Probes = later topic.)
- `kubectl get all` shows deploy + rs + pods + services in one shot.
