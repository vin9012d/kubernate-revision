# Step 8 (optional, advanced) – All Service types in one app

> Do steps 01–07 first. This combines everything into a mini "shop" app in its own namespace `shop`.
>
> ⚠️ First: `kubectl config current-context` → must be `kind-k8srevision`.
>
> ⚠️ This app also uses nodePort **30080**. Delete step 4's Service first: `kubectl delete -f ../04-nodeport/`

All outputs shown below are **real outputs** from running these files on our kind cluster.

---

## Part 0 – The problem Services solve (read this first)

On Day 3 we created a Deployment with 3 pods. Look at them:

```
NAME                         IP
orders-api-579c5d585-4sj4x   10.244.0.38
orders-api-579c5d585-pjc9g   10.244.0.37
orders-api-579c5d585-t8528   10.244.0.36
```

Now imagine the **frontend** needs to call this backend. Questions:

1. **Which IP should frontend use?** There are 3 pods. Hard-code `10.244.0.38`?
2. **What if that pod dies?** Deployment creates a new pod with a **new IP** (e.g. `10.244.0.51`). Frontend is now calling a dead IP → app broken.
3. **What if we scale to 10 pods?** Frontend must somehow learn 7 new IPs.
4. **How do users on the internet reach the frontend?** Pod IPs (`10.244.x.x`) only exist inside the cluster.

**Pod IPs are temporary. You must never use them directly.**

A **Service** solves all of this. It is a **permanent front door** for a group of pods:

```
                    BEFORE (no Service)                    AFTER (with Service)

   frontend ──► 10.244.0.38 ??? (dies tomorrow)   frontend ──► "orders-api:8080"  (never changes)
                                                                   │
                                                       ┌───────────┼───────────┐
                                                       ▼           ▼           ▼
                                                     pod-1       pod-2       pod-3
                                                  (whatever IPs they have today)
```

A Service gives you **3 things**:

| # | What | Example |
|---|---|---|
| 1 | **A stable name (DNS)** | `orders-api` or `orders-api.shop.svc.cluster.local` |
| 2 | **A stable IP** (ClusterIP) – never changes while Service exists | `10.96.180.97` |
| 3 | **Load balancing** – spreads requests across all healthy pods | request 1 → pod-1, request 2 → pod-3 ... |

---

## Part 1 – How does a Service find its pods? (LABELS)

A Service does **not** know pod names or pod IPs. It only knows a **label selector**.
It keeps asking: *"which running + ready pods have label `app=orders-api`?"* → those get traffic.

```yaml
# Deployment (pod template)              # Service
template:                                spec:
  metadata:                                selector:
    labels:                                  app: orders-api   ◄── must match exactly
      app: orders-api   ──────────────────►
```

- Pod dies → its IP is removed from the list automatically.
- New pod starts with the label → its IP is added automatically.
- Scale to 10 → all 10 IPs are in the list.

This list is stored in an object called **EndpointSlice**. You can see it:

```bash
kubectl -n shop get endpointslices
```
```
NAME                    ADDRESSTYPE   PORTS   ENDPOINTS
orders-api-tlvqt        IPv4          80      10.244.0.38,10.244.0.37,10.244.0.36   ◄ 3 backend pods
frontend-z75lj          IPv4          80      10.244.0.41,10.244.0.40               ◄ 2 frontend pods
redis-twlkx             IPv4          6379    10.244.0.39,10.244.0.42               ◄ 2 redis pods
```

> 🔑 **#1 debugging rule:** Service not working? Run `kubectl get endpointslices`.
> Empty ENDPOINTS = your `selector` doesn't match pod labels (typo!) or pods aren't Ready.

### What happens when you call a Service (step by step)

```
frontend pod runs:  curl http://orders-api:8080
   │
   │ 1. DNS: "what is orders-api?"
   ▼
CoreDNS (cluster's DNS server) ──► "orders-api = 10.96.180.97"   (the ClusterIP)
   │
   │ 2. packet sent to 10.96.180.97:8080
   ▼
kube-proxy rules on the node (iptables) ──► "10.96.180.97:8080 → pick a random pod: 10.244.0.37:80"
   │
   │ 3. packet delivered
   ▼
orders-api pod (10.244.0.37) listening on port 80
```

- **CoreDNS** = phone book. Service name → IP.
- **kube-proxy** = runs on every node, writes forwarding rules. Service IP → real pod IP.
- The ClusterIP `10.96.180.97` is **virtual** – no machine actually owns it. It only exists as a forwarding rule. (That's why `ping` to a Service IP doesn't work, but `curl` does.)

---

## Part 2 – The 3 ports (everyone gets confused here)

```yaml
ports:
  - port: 8080          # ①
    targetPort: 80      # ②
    nodePort: 30080     # ③  (only for NodePort / LoadBalancer)
```

| Port | Who uses it | Think of it as |
|---|---|---|
| ① `port` | **Clients inside cluster** call `orders-api:8080` | The Service's own door number |
| ② `targetPort` | Where the **container** actually listens | The pod's door number |
| ③ `nodePort` | **Outside world** calls `<NodeIP>:30080` | A door on every server (node) |

Full journey from outside:
```
browser ──► NodeIP:30080 ──► ServiceIP:80 ──► PodIP:80
            (③ nodePort)     (① port)        (② targetPort)
```

**Why can `port` and `targetPort` differ?** Your app might listen on 80, but you want clients to call 8080 (or the other way). The Service translates. Our `orders-api`: clients call **8080**, container listens on **80**.

**Named ports:** `targetPort: http` refers to the container port **named** `http`. If the app later moves to port 3000, you update only the Deployment; the Service keeps working.

---

## Part 3 – Our example app: "shop"

To understand **why** each Service type exists, we built a mini online shop. Each component needs a **different** kind of access, so each gets a **different** Service type.

```
                                    INTERNET / USERS
                                           │
                    ┌──────────────────────┴──────────────────────┐
                    │ (prod/EKS)                                  │ (dev/kind)
                    ▼                                             ▼
      ┌──────────────────────────────┐            ┌─────────────────────────────┐
      │ frontend-public              │            │ frontend                    │
      │ type: LoadBalancer           │            │ type: NodePort  :30080      │
      │ (AWS ELB, public DNS)        │            │                             │
      └──────────────┬───────────────┘            └──────────────┬──────────────┘
                     └──────────────────┬────────────────────────┘
                                        ▼
            ┌────────────────────────────────────────────────────┐
            │  frontend pods (nginx) x2                           │
            │    /      → reply itself                            │
            │    /api/  → http://orders-api:8080                  │
            │    /pay/  → http://payment-gateway                  │
            └───────────┬──────────────────────────┬──────────────┘
                        ▼                          ▼
       ┌───────────────────────────┐   ┌────────────────────────────────┐
       │ orders-api                │   │ payment-gateway                │
       │ type: ClusterIP (internal)│   │ type: ExternalName             │
       └─────────────┬─────────────┘   │ → CNAME httpbin.org            │
                     ▼                 └───────────────┬────────────────┘
       orders-api pods (Go) x3                         ▼
                     │                        httpbin.org (OUTSIDE cluster)
                     ▼                        (pretend: Stripe / Razorpay)
       ┌───────────────────────────┐
       │ redis                     │
       │ type: Headless (None)     │
       └─────────────┬─────────────┘
                     ▼
         redis-0 (primary)   redis-1 (replica)
```

| File | Component | Service type | Why this type? |
|---|---|---|---|
| [`01-namespace.yaml`](01-namespace.yaml) | namespace `shop` | – | Keeps all shop stuff together |
| [`02-orders-api.yaml`](02-orders-api.yaml) | Backend API | **ClusterIP** | Only frontend should call it. Must be hidden from internet. |
| [`03-redis-headless.yaml`](03-redis-headless.yaml) | Database | **Headless** | Need to reach a **specific** pod (`redis-0` primary), not a random one |
| [`04-payment-externalname.yaml`](04-payment-externalname.yaml) | Payment provider | **ExternalName** | It lives **outside** the cluster; want an internal alias |
| [`05-frontend.yaml`](05-frontend.yaml) | Website | **NodePort** | Users outside cluster must reach it (dev setup) |
| [`06-frontend-loadbalancer.yaml`](06-frontend-loadbalancer.yaml) | Website (prod) | **LoadBalancer** | Real cloud: one public address on port 80 |

Files are numbered because **order matters**: nginx (frontend) checks that `orders-api` and `payment-gateway` exist when it starts. `kubectl apply -f .` applies files alphabetically, so they're created first.

### Run it
```bash
cd Day4/08-all-together
kubectl apply -f .
kubectl -n shop get pods,svc
```
Real output:
```
NAME                             READY   STATUS    IP
pod/frontend-86786dd75b-mckkv    1/1     Running   10.244.0.41
pod/frontend-86786dd75b-rr27z    1/1     Running   10.244.0.40
pod/orders-api-579c5d585-4sj4x   1/1     Running   10.244.0.38
pod/orders-api-579c5d585-pjc9g   1/1     Running   10.244.0.37
pod/orders-api-579c5d585-t8528   1/1     Running   10.244.0.36
pod/redis-0                      1/1     Running   10.244.0.39
pod/redis-1                      1/1     Running   10.244.0.42

NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
service/frontend          NodePort       10.96.1.70      <none>        80:30080/TCP
service/frontend-public   LoadBalancer   10.96.234.220   <pending>     80:30467/TCP
service/orders-api        ClusterIP      10.96.180.97    <none>        8080/TCP
service/payment-gateway   ExternalName   <none>          httpbin.org   <none>
service/redis             ClusterIP      None            <none>        6379/TCP
```

How to read `PORT(S)`:
- `8080/TCP` → Service port only (ClusterIP)
- `80:30080/TCP` → Service port `80`, opened on every node as `30080`

---

## Part 4 – Each type in detail

### 4.1 ClusterIP – "internal only" (default type)

**File:** `02-orders-api.yaml`

**Real-life use:** backend APIs, microservice-to-microservice calls, internal databases, caches, queues. **About 80% of all Services are ClusterIP.**

**Story:** `orders-api` holds business logic. Users must **never** call it directly; only the frontend should. ClusterIP gives it a name and IP that exist **only inside the cluster**, so it's invisible from outside.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-api            # → DNS name "orders-api"
  namespace: shop
spec:
  type: ClusterIP             # default, can be omitted
  selector:
    app: orders-api           # → pods with this label get traffic
  ports:
    - name: http
      port: 8080              # clients call orders-api:8080
      targetPort: http        # → container port named "http" (80)
```

**How frontend uses it:** in nginx config (`05-frontend.yaml`):
```nginx
location /api/ {
  proxy_pass http://orders-api:8080/;     # ← Service NAME, not a pod IP
}
```

**See load balancing yourself:**
```bash
kubectl -n shop port-forward svc/frontend 8080:80      # terminal 1
for i in 1 2 3 4; do curl -s localhost:8080/api/ | grep Hostname; done   # terminal 2
```
Real output – **different backend pod each time**:
```
Hostname: orders-api-579c5d585-pjc9g
Hostname: orders-api-579c5d585-t8528
Hostname: orders-api-579c5d585-4sj4x
Hostname: orders-api-579c5d585-pjc9g
```
Frontend always calls the same name `orders-api:8080`. The Service picks a pod.

**DNS names that work for a ClusterIP Service:**

| Calling from | Name you can use |
|---|---|
| Same namespace (`shop`) | `orders-api` |
| Other namespace | `orders-api.shop` |
| Anywhere (full name, FQDN) | `orders-api.shop.svc.cluster.local` |

```
orders-api   .  shop        .  svc   .  cluster.local
 └ service     └ namespace     └ fixed   └ cluster domain
```

**Prove it's NOT reachable from outside:** there is no port on your Mac or the node for it. Only pods inside the cluster can use `10.96.180.97`.

---

### 4.2 NodePort – "open a port on every node"

**File:** `05-frontend.yaml` (Service part)

**Real-life use:** quick dev/testing access, on-prem clusters without a cloud LB, or when you put your own external load balancer (F5, HAProxy) in front of the nodes.

**Story:** users must reach the website from **outside** the cluster. NodePort opens port **30080** on **every node**. Any `NodeIP:30080` → frontend pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: shop
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 80                # inside cluster:  frontend:80
      targetPort: http        # pod port:        80
      nodePort: 30080         # outside:         <NodeIP>:30080
```

**Important:** NodePort **also creates a ClusterIP** (`10.96.1.70`). So inside the cluster you can still call `frontend:80`. NodePort = ClusterIP + an extra door on every node.

```
          Node 1 (IP .2)            Node 2 (IP .3)            Node 3 (IP .4)
          :30080 open               :30080 open               :30080 open
              │                         │                         │
              └─────────────────────────┼─────────────────────────┘
                                        ▼
                         frontend Service (any pod, any node)
```
Even if no frontend pod runs on Node 3, `Node3-IP:30080` still works. kube-proxy forwards it to a node that has one.

**Test on kind:** our "node" is a Docker container (IP `172.19.0.2`), not reachable from Mac directly. Test from inside the node:
```bash
docker exec k8srevision-control-plane sh -c 'curl -s $(hostname -i):30080/; curl -s $(hostname -i):30080/api/ | grep Hostname'
```
Real output:
```
Hello from FRONTEND pod: frontend-86786dd75b-rr27z
Hostname: orders-api-579c5d585-4sj4x
```
(The full chain works: **NodePort → frontend → ClusterIP → orders-api**.)

**Downsides (why it's not used for production websites):**
- Ugly ports: only **30000–32767** allowed. Users won't type `:30080`.
- Users need to know node IPs, and nodes get replaced (IPs change).
- Each port can be used by only **one** Service in the whole cluster.

---

### 4.3 LoadBalancer – "give me a real public address" (cloud)

**File:** `06-frontend-loadbalancer.yaml`

**Real-life use:** exposing an app to the internet on AWS EKS / GKE / AKS.

**Story:** in production we want users to open `http://shop.com`, not `node-ip:30080`. A LoadBalancer Service tells the **cloud provider**: *"create a real load balancer and point it at my nodes"*.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-public
  namespace: shop
  # annotations:
  #   service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: frontend             # SAME pods as the NodePort service
  ports:
    - port: 80
      targetPort: http
```

**LoadBalancer = NodePort + cloud LB in front.** Look at the output: `80:30467/TCP`. K8s auto-created nodePort `30467`. The cloud LB sends traffic to `every-node:30467`.

```
user ──► AWS ELB (abc123.elb.ap-south-1.amazonaws.com:80)
              │
              ▼
        any node :30467  (auto NodePort)
              │
              ▼
        ClusterIP :80 ──► frontend pod :80
```

**On EKS (e.g. your company's `dev-ap-south-1`):**
```
NAME              TYPE           EXTERNAL-IP
frontend-public   LoadBalancer   a1b2c3...ap-south-1.elb.amazonaws.com
```
**On kind (our cluster):**
```
frontend-public   LoadBalancer   <pending>      ← forever; no cloud to ask
```
That's expected. It still works as a NodePort. (To get a real IP locally, install `cloud-provider-kind` or MetalLB.)

**Downsides:**
- **1 Service = 1 cloud load balancer = $$$**. 20 microservices exposed = 20 AWS LBs.
- That's why real setups use **one** LoadBalancer + **Ingress** (routes `shop.com/api`, `shop.com/admin` to different Services). Ingress is the next topic.
- Annotations are cloud-specific (AWS annotations differ from GCP ones).

---

### 4.4 Headless – "no load balancing, give me the pods themselves"

**File:** `03-redis-headless.yaml`

**Real-life use:** databases and clustered systems (Redis, MongoDB, Kafka, Cassandra, Elasticsearch, Zookeeper) running as **StatefulSets**.

**Story:** our Redis has 2 pods. `redis-0` is the **primary** (accepts writes), `redis-1` is a **replica** (read-only copy).
If our app writes through a normal ClusterIP Service, the Service picks a **random** pod. 50% of writes would hit `redis-1` and fail.
We need to say **"talk to redis-0 specifically"**. Pod IPs change, so we need a **stable name per pod**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: shop
spec:
  clusterIP: None             # ◄── this one line makes it "headless"
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet             # like a Deployment, but pods get FIXED names: redis-0, redis-1
spec:
  serviceName: redis          # ◄── links pods to the headless Service → per-pod DNS
  replicas: 2
  ...
```

**What `clusterIP: None` changes:**

| | Normal ClusterIP | Headless (`None`) |
|---|---|---|
| Virtual IP | ✅ `10.96.x.x` | ❌ none |
| DNS `redis` returns | 1 IP (the Service IP) | **all pod IPs** |
| Load balancing by K8s | ✅ | ❌ (client chooses) |
| Per-pod DNS (`redis-0.redis`) | ❌ | ✅ (with StatefulSet) |

**See the difference in DNS** (real output):
```bash
kubectl -n shop run dns-test --rm -it --restart=Never --image=redis:7.4-alpine -- sh
```
```
# normal ClusterIP → ONE virtual IP
$ nslookup orders-api.shop.svc.cluster.local
Address: 10.96.180.97

# headless → ALL pod IPs (compare with `kubectl -n shop get pods -o wide`)
$ nslookup redis.shop.svc.cluster.local
Address: 10.244.0.39
Address: 10.244.0.42

# per-pod name → exactly ONE pod, always redis-0 even if it restarts with a new IP
$ nslookup redis-0.redis.shop.svc.cluster.local
Address: 10.244.0.39

# write to the primary by name
$ redis-cli -h redis-0.redis SET order:1 "pizza"
OK
$ redis-cli -h redis-0.redis GET order:1
"pizza"
```

**Per-pod DNS format:** `<pod-name>.<headless-service>.<namespace>.svc.cluster.local` → `redis-0.redis.shop.svc.cluster.local`

**Try it:** delete `redis-0` → StatefulSet recreates it with the **same name** `redis-0` (new IP). `redis-0.redis` still works. A Deployment pod would get a random new name.
```bash
kubectl -n shop delete pod redis-0
kubectl -n shop get pods -w        # redis-0 comes back with the same name
```

(The apply warning `spec.SessionAffinity is ignored for headless services` is harmless.)

---

### 4.5 ExternalName – "an alias for something outside the cluster"

**File:** `04-payment-externalname.yaml`

**Real-life use:** external databases (AWS RDS), 3rd-party APIs (Stripe, Razorpay, Twilio), services in another cluster, migrations.

**Story:** our shop uses an external payment provider. We pretend `httpbin.org` is it. Without ExternalName, every app has `https://api.razorpay.com` hard-coded in its config. Want to switch to the sandbox URL in dev, or to another provider? Change and redeploy every app.

With ExternalName, apps call `http://payment-gateway`. The real hostname lives in **one** Service file.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-gateway
  namespace: shop
spec:
  type: ExternalName
  externalName: httpbin.org   # in prod: api.razorpay.com / mydb.xxxx.ap-south-1.rds.amazonaws.com
```

- **No selector, no pods, no ClusterIP, no ports, no proxying.**
- It's purely **DNS**: CoreDNS answers `payment-gateway` → `CNAME httpbin.org`.

Real output:
```
$ nslookup payment-gateway.shop.svc.cluster.local
payment-gateway.shop.svc.cluster.local   canonical name = httpbin.org
```
And through the frontend (`/pay/` → `payment-gateway`):
```bash
curl localhost:8080/pay/get        # (with port-forward running)
```
```json
{ "headers": { "Host": "httpbin.org", ... }, "method": "GET", ... }
```

**Common use – migration story:**
```
Phase 1: DB is outside (RDS)     → Service "orders-db" type ExternalName → mydb.rds.amazonaws.com
Phase 2: move DB into cluster    → change "orders-db" to ClusterIP with a selector
Apps keep calling "orders-db" the whole time. Zero app config changes.
```

**Gotchas:**
- `externalName` must be a **hostname**, not an IP. (For an external IP, use a Service without a selector plus a manual EndpointSlice.)
- The HTTP `Host` header your app sends will be `payment-gateway`, not the real hostname. httpbin.org accepts any Host, but many real providers (behind CDNs or HTTPS with SNI/certificates) **reject** it. That's why our nginx sets `proxy_set_header Host httpbin.org;`. With HTTPS, the certificate is for `api.razorpay.com`, not `payment-gateway`, so TLS checks fail unless the client uses the real name for TLS.
- Port is **not** translated – you call the external service's real port.

---

## Part 5 – Side-by-side summary

| | ClusterIP | NodePort | LoadBalancer | Headless | ExternalName |
|---|---|---|---|---|---|
| **One-liner** | Internal door | Door on every node | Cloud public door | Direct to pods | DNS alias outside |
| **Our example** | orders-api | frontend | frontend-public | redis | payment-gateway |
| Has ClusterIP? | ✅ | ✅ | ✅ | ❌ `None` | ❌ |
| Has selector/pods? | ✅ | ✅ | ✅ | ✅ | ❌ |
| Reachable from outside? | ❌ | ✅ NodeIP:30000-32767 | ✅ LB address | ❌ | n/a |
| K8s load balances? | ✅ | ✅ | ✅ | ❌ | ❌ |
| DNS answer | Service IP | Service IP | Service IP | Pod IPs | CNAME |
| Production use | ⭐ Most common | Dev / on-prem | Public apps (with Ingress) | Databases, StatefulSets | External deps |

**How they build on each other:**
```
ClusterIP           = virtual IP + DNS + load balancing (inside)
NodePort            = ClusterIP    + port on every node
LoadBalancer        = NodePort     + cloud load balancer
Headless            = DNS only, no IP, returns pod IPs
ExternalName        = DNS CNAME only, no pods at all
```

### Which type should I use? (decision guide)
```
Is the thing running OUTSIDE the cluster?
 ├─ yes → ExternalName
 └─ no  → Do clients need a SPECIFIC pod (DB primary, Kafka broker)?
           ├─ yes → Headless (+ StatefulSet)
           └─ no  → Must users outside the cluster reach it?
                     ├─ no  → ClusterIP   ⭐ (default; most services)
                     └─ yes → On cloud (EKS)?
                               ├─ yes → LoadBalancer (or better: Ingress)
                               └─ no  → NodePort
```

---

## Part 6 – Practice exercises (do these!)

```bash
# 1. Break the selector → watch endpoints go empty → /api/ fails (502 Bad Gateway)
kubectl -n shop patch svc orders-api -p '{"spec":{"selector":{"app":"orders-apiX"}}}'
kubectl -n shop get endpointslices -l kubernetes.io/service-name=orders-api
#    fix it:
kubectl -n shop patch svc orders-api -p '{"spec":{"selector":{"app":"orders-api"}}}'

# 2. Scale backend, watch endpoints grow automatically
kubectl -n shop scale deploy orders-api --replicas=5
kubectl -n shop get endpointslices -l kubernetes.io/service-name=orders-api

# 3. Delete a backend pod while calling /api/ in a loop → no errors, new pod joins
while true; do curl -s localhost:8080/api/ | grep Hostname; sleep 0.5; done   # (port-forward running)
kubectl -n shop delete pod <one-orders-api-pod>

# 4. Call a Service from ANOTHER namespace (default) → must use name.namespace
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- http://orders-api.shop:8080
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- http://orders-api:8080   # fails: wrong namespace

# 5. Change payment provider without touching the frontend
#    edit 04-payment-externalname.yaml: externalName: postman-echo.com
#    (and the Host header in 05-frontend.yaml), apply, restart frontend:
kubectl -n shop rollout restart deploy frontend

# 6. Look at everything in Lens: Network → Services, Network → Endpoints (namespace: shop)
```

---

## Part 7 – Useful commands

```bash
kubectl -n shop get svc                       # list services
kubectl -n shop get svc -o wide               # + SELECTOR column
kubectl -n shop describe svc orders-api       # selector, ports, endpoints, events
kubectl -n shop get endpointslices            # which pod IPs each service sends to
kubectl get svc -A                            # all namespaces (see kube-dns in kube-system)

# create a Service the quick (imperative) way
kubectl -n shop expose deploy orders-api --name=orders-api-2 --port=8080 --target-port=80

# generate YAML without creating (great for learning/exams)
kubectl -n shop expose deploy frontend --type=NodePort --port=80 --dry-run=client -o yaml

# reach any Service from Mac (debugging; sticks to ONE pod, no load balancing)
kubectl -n shop port-forward svc/frontend 8080:80

kubectl explain service.spec                  # built-in docs for every field
```

## Part 8 – Gotchas checklist

- ❌ Never use pod IPs in app config → always use the **Service name**.
- Service selects pods by **labels only**. Typo = empty endpoints = `connection refused` / `502`.
- Only **Ready** pods get traffic (readiness probes = later topic).
- Service names are **per namespace**. From another namespace use `svc.namespace`.
- ClusterIP can't be `ping`ed – it's a forwarding rule, not a machine. Use `curl`/`wget`/`nc`.
- `nodePort` range 30000–32767, unique across the whole cluster.
- LoadBalancer on kind = `<pending>` forever; that's normal.
- 1 LoadBalancer = 1 cloud LB = cost → use **Ingress** for many apps (next topic).
- ExternalName = DNS only: no port mapping; watch out for Host header / TLS certificate names.
- `kubectl port-forward` sticks to **one pod**, so it doesn't show load balancing. Test LB from inside the cluster.

## Clean up
```bash
kubectl delete -f .          # or: kubectl delete ns shop  (deletes everything inside)
```
