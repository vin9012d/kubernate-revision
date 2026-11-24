# Step 1 – The problem: calling pods directly

## Goal
See **why** we need Services. No Service in this step on purpose.

## Create 2 web pods
```bash
kubectl apply -f web-deployment.yaml
kubectl get pods -l app=web -o wide
```
```
NAME                   STATUS    IP
web-844c647db8-jkcmz   Running   10.244.0.47
web-844c647db8-rj7pv   Running   10.244.0.46
```
The app (`traefik/whoami`) replies with its own pod name, so we can see who answered.

## Call a pod by its IP – works
```bash
kubectl exec client -- wget -qO- http://10.244.0.47      # use YOUR pod IP
```
```
Hostname: web-844c647db8-jkcmz
```

## Now the pod dies (crash, node failure, update...)
```bash
kubectl delete pod web-844c647db8-jkcmz                  # use YOUR pod name
kubectl get pods -l app=web -o wide
```
```
NAME                   STATUS    IP
web-844c647db8-q279j   Running   10.244.0.48     ← Deployment made a NEW pod, NEW IP
web-844c647db8-rj7pv   Running   10.244.0.46
```

## Call the old IP again – broken
```bash
kubectl exec client -- wget -qO- -T3 http://10.244.0.47
```
```
wget: download timed out
```

## What we learned
- Deployment **heals** the app (new pod created) ✅
- But the new pod has a **new IP** ❌ → anyone using the old IP is broken.
- Also: with 2 pods, which IP should a client use? Who spreads traffic between them?

**We need a fixed address that always points to whatever pods are alive right now.**
That is a **Service**.

Keep these pods running. ➜ Next: [02-first-service](../02-first-service/)
