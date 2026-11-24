# Day 2 – Pods

## 1. What is a Pod?

- **Pod = smallest deployable unit in Kubernetes.** You never run a container directly; you run a Pod that wraps one (or more) containers.
- All containers in a Pod share:
  - **Network** – same IP address, talk to each other via `localhost`.
  - **Storage** – can mount the same volumes.
- Each Pod gets its **own IP** inside the cluster (e.g. our pod got `10.244.0.5`).
- Pods are **ephemeral** (temporary). If a Pod dies, it is not "fixed" – a new one is created with a new IP. That's why in real apps we use **Deployments**, not bare Pods.
- Usually **1 container per Pod**. Multiple containers only when tightly coupled (sidecar pattern, e.g. log shipper next to app).

---

## 2. Our Hello World Pod

File: [`../hello-pod.yaml`](../hello-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app: hello
spec:
  restartPolicy: Never          # run once, don't restart after exit
  containers:
    - name: hello
      image: busybox:1.36       # tiny image from Docker Hub
      command: ["sh", "-c", "echo 'Hello World from Kubernetes!'"]
```

### YAML fields explained

| Field | Meaning |
|---|---|
| `apiVersion: v1` | API group/version for Pod object (core API = `v1`) |
| `kind: Pod` | Type of object we are creating |
| `metadata.name` | Unique name of pod in the namespace |
| `metadata.labels` | Key/value tags – used by Services/Deployments to select pods |
| `spec` | Desired state – what we want |
| `restartPolicy` | `Always` (default) / `OnFailure` / `Never` |
| `containers[].image` | Image pulled from registry (Docker Hub by default) |
| `containers[].command` | Overrides image's default ENTRYPOINT |

**Why `restartPolicy: Never`?** Our container prints one line and exits. With default `Always`, Kubernetes keeps restarting it → status goes into `CrashLoopBackOff`. `Never` lets it finish as `Completed`.

---

## 3. Pod Lifecycle

### Pod phases (`kubectl get pod` → STATUS column)

```
            kubectl apply
                 │
                 ▼
            ┌─────────┐   scheduled + image pulling
            │ Pending │ ─────────────────────────────┐
            └─────────┘                              │
                                                     ▼
                                               ┌─────────┐
                                               │ Running │
                                               └─────────┘
                                                │       │
                                   exit code 0  │       │ exit code ≠ 0
                                                ▼       ▼
                                        ┌───────────┐ ┌────────┐
                                        │ Succeeded │ │ Failed │
                                        └───────────┘ └────────┘
```

| Phase | Meaning |
|---|---|
| **Pending** | Accepted by cluster, but not running yet (waiting for scheduler to pick a node, or pulling image) |
| **Running** | Bound to a node, at least one container running |
| **Succeeded** | All containers exited with code 0 and won't restart (shows as `Completed`) |
| **Failed** | All containers stopped, at least one exited with error |
| **Unknown** | Control plane can't reach the node |

### Container states (inside the pod)

- **Waiting** – not started yet (e.g. `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`)
- **Running** – executing
- **Terminated** – finished (success or failure), has exit code + reason

### What happens behind the scenes on `kubectl apply`

1. `kubectl` sends YAML to **API Server**.
2. API Server validates and stores it in **etcd**.
3. **Scheduler** sees pod with no node → picks a node.
4. **Kubelet** on that node sees the pod assigned to it.
5. Kubelet asks container runtime (**containerd**) to pull image and start container.
6. Kubelet reports status back to API Server → we see it in `kubectl get pods`.

### Common STATUS values you'll see

| Status | Meaning / Fix |
|---|---|
| `ContainerCreating` | Pulling image / setting up – just wait |
| `Completed` | Container finished successfully (our hello pod) |
| `CrashLoopBackOff` | Container keeps crashing and K8s keeps restarting it – check `kubectl logs` |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag or no access to registry |
| `Error` | Container exited with non-zero code |
| `OOMKilled` | Container used more memory than its limit |

### Restart policies

| Policy | Behaviour | Used by |
|---|---|---|
| `Always` (default) | Restart whenever container exits | Deployments (long-running apps) |
| `OnFailure` | Restart only on non-zero exit | Jobs |
| `Never` | Never restart | One-shot tasks (our hello pod) |

---

## 4. Commands We Ran Today

### Environment check
```bash
kubectl version --client          # kubectl v1.34.1
kind version                      # kind v0.30.0
docker --version                  # Docker 28.4.0
kubectl config get-contexts       # list all clusters in kubeconfig
```

### Local cluster setup
```bash
open -a Docker                                # start Docker Desktop
kind create cluster --name k8srevision        # create local cluster
kubectl config use-context kind-k8srevision   # switch to local cluster
kubectl config current-context                # verify which cluster we're on
kubectl get nodes                             # check node is Ready
```

### Pod commands
```bash
kubectl apply -f hello-pod.yaml      # create pod from YAML
kubectl get pods                     # list pods
kubectl get pod hello-pod -o wide    # more info: IP, node
kubectl logs hello-pod               # see output → Hello World from Kubernetes!
kubectl delete -f hello-pod.yaml     # delete pod
```

---

## 5. Other Useful Pod Commands

```bash
# Create pod quickly without YAML
kubectl run nginx --image=nginx

# Generate YAML without creating (great for learning/exam)
kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx-pod.yaml

# Detailed info + Events (FIRST thing to check when pod is stuck)
kubectl describe pod hello-pod

# Full pod object as YAML (see status section too)
kubectl get pod hello-pod -o yaml

# Follow logs live
kubectl logs -f <pod-name>

# Logs of previous crashed container
kubectl logs <pod-name> --previous

# Get shell inside running container
kubectl exec -it <pod-name> -- sh

# Watch pod status change live
kubectl get pods -w

# All pods in all namespaces
kubectl get pods -A

# Delete immediately
kubectl delete pod <pod-name> --force --grace-period=0
```

---

## 6. Viewing in Lens

1. Open Lens → **Catalog / Kubernetes Clusters** → click `kind-k8srevision`.
2. Left menu → **Workloads → Pods**.
3. Namespace dropdown → `default` (or All namespaces).
4. Click pod → **Logs** icon to see output.

If cluster not visible → restart Lens (it reads `~/.kube/config` on start).

---

## 7. Important Points / Gotchas

- ⚠️ **Always check current context before `apply`/`delete`.** My kubeconfig also has company EKS cluster `dev-ap-south-1`. Use `kubectl config current-context`.
- Bare Pods are **not self-healing** – if node dies, pod is gone. Use Deployment for real apps.
- Pod IP **changes** when pod is recreated → use a **Service** for stable access.
- Most pod fields **can't be edited** after creation (only a few like `image` can). To change `command`, ports, etc. → delete and recreate.
- `kubectl describe pod` → **Events** section at bottom tells why a pod is stuck.
- Image without tag = `:latest` – avoid in real work, always pin a version (`busybox:1.36`).
