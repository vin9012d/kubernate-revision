# Step 5 – LoadBalancer (cloud gives a public address)

## Goal
Give users **one stable public address** instead of `<NodeIP>:30080`.

## Idea
A LoadBalancer Service asks the **cloud provider** (AWS, GCP, Azure) to create a real load balancer.
That LB has a fixed public address and sends traffic to the nodes. Kubernetes **keeps the LB's node list up to date automatically**, so node IP changes don't matter anymore.

```
browser ──► AWS LB (abc123.elb.amazonaws.com:80)  ← stable address
               └──► any node:<auto nodePort> ──► web pod:80
```
**LoadBalancer = NodePort + a cloud LB in front.**

## The YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web               # same web pods again
  ports:
    - port: 80             # port on the load balancer
      targetPort: 80       # pod port
```

```bash
kubectl apply -f web-loadbalancer.yaml
kubectl get svc web-lb
```

## On kind (our laptop)
```
NAME     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
web-lb   LoadBalancer   10.96.129.177   <pending>     80:32439/TCP
```
- `EXTERNAL-IP <pending>` **forever**: there is no cloud to create an LB. This is normal on kind.
- `80:32439`: Kubernetes **auto-created a NodePort** (32439). Proof it's NodePort underneath:
  ```bash
  docker exec k8srevision-control-plane sh -c 'curl -s $(hostname -i):32439 | grep Hostname'
  # Hostname: web-844c647db8-rj7pv  ✅      (use YOUR port number)
  ```

## On AWS EKS (what you'd see at work)
```
NAME     TYPE           EXTERNAL-IP                                        PORT(S)
web-lb   LoadBalancer   a1b2c3d4...ap-south-1.elb.amazonaws.com            80:32439/TCP
```
Open `http://a1b2c3d4...elb.amazonaws.com` in the browser → works. Point `shop.com` DNS to it.

## Downside
**1 LoadBalancer Service = 1 cloud LB = money.** 20 apps = 20 LBs.
Real companies use **one** LB + **Ingress** (routes by URL/host to many ClusterIP Services). That's a later topic.

➜ Next: [06-headless](../06-headless/)
