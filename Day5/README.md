# Day 5 – Ingress

Do the folders **in order**. Each one teaches **one idea**. Every output shown is a **real output** from our kind cluster.
Clean up each folder before the next one (each README ends with the cleanup command).

| Step | Folder | You will learn |
|---|---|---|
| 0 | [`00-setup`](00-setup/) | Cluster with ports 80/443 open + install the Ingress **controller** |
| 1 | [`01-first-ingress`](01-first-ingress/) | First Ingress: browser → `http://localhost` → your app |
| 2 | [`02-path-routing`](02-path-routing/) | One address, many apps by **URL path**: `/` → frontend, `/orders` → orders |
| 3 | [`03-host-routing`](03-host-routing/) | One address, many apps by **domain**: `shop.local`, `admin.local` |
| 4 | [`04-tls`](04-tls/) | **HTTPS** with a certificate |
| 5 | [`05-gateway-api`](05-gateway-api/) | *(Reading only)* Gateway API, the newer replacement for Ingress |

---

## Why Ingress? (the problem from Day 4)

You have 3 apps that users must reach: **website, orders API, admin panel**.

**With Day 4 tools:**
```
LoadBalancer Service #1 → website     → AWS LB #1  💰  a1.elb.amazonaws.com
LoadBalancer Service #2 → orders API  → AWS LB #2  💰  b2.elb.amazonaws.com
LoadBalancer Service #3 → admin       → AWS LB #3  💰  c3.elb.amazonaws.com
```
- 3 cloud load balancers = 3× the cost
- 3 different addresses for users
- HTTPS certificates configured in 3 places

**With Ingress:**
```
                        ┌──► shop.com/          → website    (ClusterIP)
users ──► 1 AWS LB ──► Ingress ──► shop.com/orders  → orders API (ClusterIP)
          💰 only one   (nginx)  └──► admin.shop.com    → admin      (ClusterIP)
```
- **One** entry point for all apps
- Routes by **URL path** and **domain name**
- HTTPS handled in **one** place
- All apps stay **ClusterIP** (hidden), only Ingress is exposed

## The 2 parts of Ingress (important!)

| Part | What it is | Analogy |
|---|---|---|
| **Ingress controller** | A real program (nginx) running as pods in the cluster. Receives all traffic. | The **receptionist** at the building entrance |
| **Ingress** (YAML object) | Routing **rules**: "path /orders → Service orders" | The **list of instructions** given to the receptionist |

**An Ingress YAML does nothing without a controller.** Kubernetes doesn't ship with one; you install it (step 0).
Common controllers: ingress-nginx, Traefik, HAProxy, AWS Load Balancer Controller (on EKS it creates an AWS ALB).

## Clean up the whole day
```bash
kubectl delete -f 01-first-ingress/ -f 02-path-routing/ -f 03-host-routing/ --ignore-not-found
kubectl delete -f 04-tls/ --ignore-not-found; kubectl delete secret shop-tls --ignore-not-found
```
