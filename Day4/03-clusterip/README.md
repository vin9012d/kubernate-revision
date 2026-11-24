# Step 3 – ClusterIP (internal-only Service)

## Goal
Understand 3 things about ClusterIP:
1. `port` vs `targetPort`
2. It works **only inside** the cluster
3. DNS names (same namespace vs other namespace)

**Real-life use:** backend APIs, databases, caches: anything that only **other apps** call, not users. Most Services are ClusterIP.

## The YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP        # default (step 2 was also ClusterIP, just not written)
  selector:
    app: backend
  ports:
    - port: 8080         # clients call   backend:8080
      targetPort: 80     # Service sends → pod:80   (where the app really listens)
```

```bash
kubectl apply -f .                 # creates backend Deployment (3 pods) + Service
kubectl get svc backend
```
```
NAME      TYPE        CLUSTER-IP    PORT(S)
backend   ClusterIP   10.96.213.6   8080/TCP
```

---

## 1. `port` vs `targetPort`

```
client ──► backend:8080 ──► Service ──► pod:80
              (port)                   (targetPort)
```
- `targetPort` = the port the **app inside the container** listens on (80).
- `port` = the port **clients** use when calling the Service (8080).
- They can be the same (step 2) or different (here). The Service translates.

```bash
kubectl exec client -- wget -qO- http://backend:8080 | grep Hostname
```
```
Hostname: backend-5994c5c656-sg7c9          ✅
```
```bash
kubectl exec client -- wget -qO- -T3 http://backend:80
```
```
wget: download timed out                    ❌ the Service only listens on 8080
```

---

## 2. Only inside the cluster

From **inside** (client pod), using the Service IP:
```bash
kubectl exec client -- wget -qO- http://10.96.213.6:8080 | grep Hostname
```
```
Hostname: backend-5994c5c656-sg7c9          ✅
```
From your **Mac** (outside):
```bash
curl -m4 http://10.96.213.6:8080
```
```
curl failed (timeout)                       ❌
```
**Why?** `10.96.213.6` is not a real machine. It's a forwarding rule that exists only on the cluster's nodes. Your Mac doesn't know about it.
That's the point: your backend is **hidden from the outside world**.

To reach it from outside → step 4 (NodePort) / step 5 (LoadBalancer).

---

## 3. DNS names

Every Service gets a DNS name:
```
backend . default . svc . cluster.local
   │        │        │        │
service  namespace  fixed  cluster domain
```

| You are calling from | Use | Result |
|---|---|---|
| Same namespace (`default`) | `backend:8080` | ✅ |
| Same namespace, full name | `backend.default.svc.cluster.local:8080` | ✅ |
| **Other** namespace, short name | `backend:8080` | ❌ `bad address 'backend:8080'` |
| **Other** namespace | `backend.default:8080` | ✅ |

Try it from another namespace:
```bash
kubectl create ns other
kubectl -n other run c2 --image=busybox:1.36 --rm -it --restart=Never -- sh
# inside:
wget -qO- http://backend:8080            # ❌ bad address – looks for "backend" in namespace "other"
wget -qO- http://backend.default:8080    # ✅ works
exit
kubectl delete ns other
```

## What we learned
- ClusterIP = stable IP + DNS name + load balancing, **inside the cluster only**.
- `port` = what clients call. `targetPort` = where the app listens.
- Short name works in the same namespace. Otherwise add `.namespace`.

➜ Next: [04-nodeport](../04-nodeport/)
