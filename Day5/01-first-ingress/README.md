# Step 1 – Your first Ingress

## Goal
Open `http://localhost` on the Mac and reach a pod, using Ingress.

## Files
| File | What |
|---|---|
| [`app.yaml`](app.yaml) | Deployment `web` (2 pods) + ClusterIP Service `web`. **Nothing new**, same as Day 4 step 2. |
| [`ingress.yaml`](ingress.yaml) | The Ingress rule |

## The Ingress YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx       # which controller handles this rule
  rules:
    - http:
        paths:
          - path: /             # URLs starting with "/" (= everything)
            pathType: Prefix
            backend:
              service:
                name: web       # → send to Service "web"
                port:
                  number: 80    #   on its port 80
```
In plain English: **"nginx, send every request to the Service `web` on port 80."**

Note: an Ingress points to a **Service**, never to pods directly.

---

## Try it: first the app only (no Ingress yet)
```bash
cd Day5/01-first-ingress
kubectl apply -f app.yaml
curl -i http://localhost/
```
```
HTTP/1.1 404 Not Found       ← app exists, but nginx has no rule for it
```

## Now add the Ingress
```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```
```
NAME   CLASS   HOSTS   ADDRESS     PORTS   AGE
web    nginx   *       localhost   80      13s
```
(`HOSTS *` = any domain. Wait ~10 seconds for nginx to load the rule.)

```bash
curl http://localhost/          # or open http://localhost in the browser
```
```
Name: web
Hostname: web-54bdd7fff5-6nh2j     ✅
...
```
Run it a few times: `Hostname` changes between the 2 pods (load balancing still works).

Any path goes to the app:
```bash
curl http://localhost/hello/world | grep GET
# GET /hello/world HTTP/1.1
```

## The full journey of the request
```
Mac: curl localhost:80
  → kind port mapping → node:80
  → ingress-nginx pod  (reads rule: "/ → Service web:80")
  → Service web (ClusterIP)
  → web pod :80
```
The app's Service is still **ClusterIP** (hidden). Only the Ingress controller is exposed.

## Clean up
```bash
kubectl delete -f .
```

➜ Next: [02-path-routing](../02-path-routing/)
