# Step 3 – Host-based routing (by domain name)

## Goal
Same entry point, different apps by **domain name**:
```
http://shop.local    → shop app
http://admin.local   → admin app
```
Real life: `shop.com`, `admin.shop.com`, `api.shop.com` all point to the **same** load balancer, and Ingress decides by name.

## How does nginx know the domain?
Every HTTP request carries a **`Host` header** with the domain the user typed:
```
GET / HTTP/1.1
Host: admin.local          ← nginx reads this
```
All domains go to the same IP. The `Host` header is what tells them apart.

## The Ingress YAML
```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: shop.local            # ← NEW: match the domain name
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop
                port:
                  number: 80
    - host: admin.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin
                port:
                  number: 80
```
(You can combine both: `host` + several `paths` per host.)

## Try it
```bash
cd Day5/03-host-routing
kubectl apply -f .
kubectl get ingress sites
```
```
NAME    CLASS   HOSTS                    PORTS
sites   nginx   shop.local,admin.local   80
```

`shop.local` isn't a real domain, so we tell curl which Host to send:
```bash
curl -s -H "Host: shop.local"  http://localhost/ | grep Name      # Name: shop   ✅
curl -s -H "Host: admin.local" http://localhost/ | grep Name      # Name: admin  ✅
curl -s -i -H "Host: other.local" http://localhost/ | head -1     # HTTP/1.1 404 ❌ no rule for this host
curl -s -i http://localhost/ | head -1                            # HTTP/1.1 404 ❌ "localhost" matches no rule
```
Same IP, same port. **Only the Host header is different**, and it decides the app.

Another way (makes curl believe `shop.local` = 127.0.0.1):
```bash
curl --resolve shop.local:80:127.0.0.1 http://shop.local/
```

### Open in the browser (optional)
The browser needs `shop.local` to point to your Mac. Add it to `/etc/hosts` (needs your password):
```bash
sudo sh -c 'echo "127.0.0.1 shop.local admin.local" >> /etc/hosts'
```
Open `http://shop.local` and `http://admin.local`. Remove that line from `/etc/hosts` when done.

## Path vs host routing

| | Path routing (step 2) | Host routing (step 3) |
|---|---|---|
| Decides by | URL path: `/orders` | Domain: `admin.local` |
| Example | `shop.com/orders` | `admin.shop.com` |
| YAML field | `paths[].path` | `rules[].host` |

## Clean up
```bash
kubectl delete -f .
```

➜ Next: [04-tls](../04-tls/)
