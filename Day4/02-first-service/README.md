# Step 2 – Your first Service (the fix)

## Goal
Put a Service in front of the `web` pods from step 1 and see what it solves.

## The YAML – only 3 important things
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web          # 1. NAME        → other pods call http://web
spec:
  selector:
    app: web         # 2. WHICH PODS  → every pod with label app=web
  ports:
    - port: 80       # 3. PORTS       → Service listens on 80 ...
      targetPort: 80 #                  ... and sends to pod port 80
```

How does the Service find the pods? **By label.** In step 1 the pods got `app: web`:
```
Deployment pod label:   app: web   ◄──── matches ────►   Service selector:  app: web
```

## Create it
```bash
kubectl apply -f web-service.yaml
kubectl get svc web
```
```
NAME   TYPE        CLUSTER-IP      PORT(S)
web    ClusterIP   10.96.224.104   80/TCP
```
The Service got its own IP (`10.96.224.104`). **This IP never changes** while the Service exists.

## Which pods is it sending to?
```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```
```
NAME        PORTS   ENDPOINTS
web-kqxgz   80      10.244.0.46,10.244.0.48      ← the 2 web pod IPs
```

## ✅ Fix 1: call by NAME, not IP
```bash
kubectl exec client -- wget -qO- http://web
```
`web` works like a website name inside the cluster.

## ✅ Fix 2: load balancing – run it 4 times
```bash
for i in 1 2 3 4; do kubectl exec client -- wget -qO- http://web | grep Hostname; done
```
```
Hostname: web-844c647db8-q279j
Hostname: web-844c647db8-q279j
Hostname: web-844c647db8-rj7pv      ← different pod
Hostname: web-844c647db8-rj7pv
```
Same name `web`, but requests are spread across both pods.

## ✅ Fix 3: pod dies → still works
```bash
kubectl delete pod web-844c647db8-q279j        # use YOUR pod name
for i in 1 2 3; do kubectl exec client -- wget -qO- http://web | grep Hostname; done
```
```
Hostname: web-844c647db8-4bv9l      ← brand-new pod, picked up automatically
Hostname: web-844c647db8-rj7pv
Hostname: web-844c647db8-4bv9l
```
```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```
```
ENDPOINTS
10.244.0.46,10.244.0.49      ← old IP (.48) removed, new IP (.49) added automatically
```
The client didn't change anything. It still calls `http://web`.

## What we learned
| Problem in step 1 | Solved by Service |
|---|---|
| Pod IP changes | Call the **name** `web` (or Service IP), which never changes |
| Which pod to call? | Service **load balances** across all pods |
| New/deleted pods | Service updates its pod list **automatically** (by label) |

The Service we made is type **ClusterIP** (the default). Next we look at it closely.

➜ Next: [03-clusterip](../03-clusterip/)
