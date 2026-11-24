# Step 4 – NodePort (open a door on every node)

## Goal
Make the `web` pods (from step 1) reachable from **outside** the cluster.

## Idea
ClusterIP can't be reached from outside, but **nodes** (the servers) have real IPs.
NodePort opens the **same port (30080) on every node**. Traffic arriving there goes to the pods.

```
outside ──► <NodeIP>:30080 ──► Service ──► web pod:80
```

## The YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web               # same pods as step 2 – one app can have many Services
  ports:
    - port: 80             # inside cluster:  web-nodeport:80   (NodePort ALSO gets a ClusterIP)
      targetPort: 80       # pod port
      nodePort: 30080      # outside:         <NodeIP>:30080   (allowed range 30000-32767)
```

```bash
kubectl apply -f web-nodeport.yaml
kubectl get svc web-nodeport
```
```
NAME           TYPE       CLUSTER-IP      PORT(S)
web-nodeport   NodePort   10.96.100.137   80:30080/TCP
```
`80:30080` = Service port `80`, opened on every node as `30080`.

## The proof: call it from your Mac (outside the cluster)

Our kind cluster was created with [`kind-config.yaml`](../../kind-config.yaml), which connects
**Mac port 30080 → node port 30080**. So for us, "the node's address" is `localhost`.
(On a real cluster you'd use the node's real IP, e.g. `http://13.234.10.5:30080`.)

**From the Mac, call the NodePort:**
```bash
curl http://localhost:30080        # or open it in the browser
```
```
Hostname: web-844c647db8-fr4zp     ✅ an outside request reached a pod
```

**From the Mac, call the ClusterIP (for comparison):**
```bash
kubectl get svc web-nodeport       # note CLUSTER-IP, e.g. 10.96.61.186
curl -m4 http://10.96.61.186:80
```
```
curl: (28) timed out               ❌ ClusterIP is inside-only (step 3)
```

**Same Service, two doors:**
```
Mac ──► localhost:30080 (NodePort)  ──► web pod   ✅  outside door
Mac ──► 10.96.61.186:80 (ClusterIP) ──► ✖         ❌  inside-only door
client pod ──► web-nodeport:80      ──► web pod   ✅  NodePort also has a ClusterIP
```

## Problems with NodePort (why prod doesn't use it alone)
- Ugly ports: only 30000–32767. Users won't type `:30080`.
- Node IPs **change** (autoscaling, upgrades, crashes) → users would need a new address.
- Each nodePort number can be used by only one Service in the whole cluster.

NodePort gives a stable **port**, but not a stable **address**. Step 5 fixes that.

➜ Next: [05-loadbalancer](../05-loadbalancer/)
