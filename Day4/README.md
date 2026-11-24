# Day 4 – Kubernetes Services (step by step)

Do the folders **in order**. Each one teaches **one idea** with tiny YAML files.
Every output shown in the READMEs is a **real output** from our kind cluster.

| Step | Folder | You will learn |
|---|---|---|
| 0 | [`00-setup`](00-setup/) | Create a `client` pod to run test commands from inside the cluster |
| 1 | [`01-problem-without-service`](01-problem-without-service/) | **The problem:** pod IPs change → calling pods directly breaks |
| 2 | [`02-first-service`](02-first-service/) | **The fix:** your first Service → stable name + load balancing |
| 3 | [`03-clusterip`](03-clusterip/) | **ClusterIP:** internal only, `port` vs `targetPort`, DNS names |
| 4 | [`04-nodeport`](04-nodeport/) | **NodePort:** open a port on every node for outside access |
| 5 | [`05-loadbalancer`](05-loadbalancer/) | **LoadBalancer:** cloud load balancer in front (EKS) |
| 6 | [`06-headless`](06-headless/) | **Headless:** talk to a SPECIFIC pod (database primary vs replica) |
| 7 | [`07-externalname`](07-externalname/) | **ExternalName:** a nickname for something outside the cluster |
| 8 | [`08-all-together`](08-all-together/) | *(Optional, advanced)* a mini shop app using all types together |

> ⚠️ Before starting: `kubectl config current-context` → must be `kind-k8srevision`.

## The whole topic in 5 lines
1. Pods die and come back with **new IPs** → never call pods by IP.
2. A **Service** = a permanent **name + IP** in front of a group of pods (found by **label**).
3. **ClusterIP** (default) → reachable only inside the cluster.
4. **NodePort** / **LoadBalancer** → reachable from outside.
5. **Headless** and **ExternalName** → special DNS-only cases.

## Clean up everything from steps 0–7
```bash
kubectl delete -f 00-setup/ -f 01-problem-without-service/ -f 02-first-service/ -f 03-clusterip/ \
  -f 04-nodeport/ -f 05-loadbalancer/ -f 06-headless/ -f 07-externalname/
```
