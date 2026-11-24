# Step 0 – Setup: cluster ports + Ingress controller

## 1. Cluster with ports 80/443 open to the Mac

Already done. The cluster was recreated with [`kind-config.yaml`](../../kind-config.yaml) (in the repo root):

```yaml
extraPortMappings:
  - containerPort: 80       # Mac localhost:80    → node port 80    (Ingress HTTP)
    hostPort: 80
  - containerPort: 443      # Mac localhost:443   → node port 443   (Ingress HTTPS)
    hostPort: 443
  - containerPort: 30080    # Mac localhost:30080 → node port 30080 (Day 4 NodePort)
    hostPort: 30080
```
Why: the kind "node" is a Docker container. Without these mappings, the Mac can't reach it.
On a real cloud cluster you don't need this; the cloud load balancer is the entry point.

To recreate it yourself (⚠️ deletes everything in the cluster):
```bash
kind delete cluster --name k8srevision
kind create cluster --name k8srevision --config kind-config.yaml
```

## 2. Install the Ingress controller (ingress-nginx)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/kind/deploy.yaml

kubectl wait -n ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=240s
```

Check it:
```bash
kubectl -n ingress-nginx get pods
```
```
NAME                                      READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-f5784567-6rdzp   1/1     Running   0          41s
```
```bash
kubectl get ingressclass
```
```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       42s
```
`nginx` is the **IngressClass** name. Every Ingress we write says `ingressClassName: nginx` = "this rule is for the nginx controller".

## 3. Test: is the receptionist at the door?

```bash
curl -i http://localhost/
```
```
HTTP/1.1 404 Not Found
```
**404 is good news!** nginx answered. It just has no rules yet ("I'm here, but I don't know where to send you").
If nginx were NOT running, you'd get `Connection refused` instead.

## How the request reaches the controller

```
Mac browser → localhost:80 → (kind port mapping) → node:80 → ingress-nginx pod
```
On EKS it would be:
```
browser → shop.com → AWS LB → ingress-nginx pods (or AWS ALB directly with the AWS LB Controller)
```

The controller itself is exposed by a Service:
```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
```
```
NAME                       TYPE           EXTERNAL-IP   PORT(S)
ingress-nginx-controller   LoadBalancer   <pending>     80:30285/TCP,443:31217/TCP
```
**Only this one** LoadBalancer is needed for **all** your apps. (`<pending>` on kind is normal, as in Day 4 step 5.)

ℹ️ The ingress-nginx project has been retired upstream (no new features). It's still the most common controller in existing clusters and the easiest way to learn Ingress. New setups are moving to **Gateway API** (step 5).

➜ Next: [01-first-ingress](../01-first-ingress/)
