# Step 2 – Path-based routing (one address, many apps)

## Goal
One address `http://localhost`, two apps, chosen by the **URL path**:
```
http://localhost/          → frontend   (the website)
http://localhost/orders    → orders     (the backend API)
```
This is how most companies expose a website and its API on one domain (`shop.com` and `shop.com/orders`).

## Files
| File | What |
|---|---|
| [`apps.yaml`](apps.yaml) | 2 apps: `frontend` and `orders`, each Deployment + ClusterIP Service. Replies say `Name: frontend` or `Name: orders`. |
| [`ingress.yaml`](ingress.yaml) | One Ingress with 2 path rules |

## The Ingress YAML
```yaml
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /orders            # URL starts with /orders → orders Service
            pathType: Prefix
            backend:
              service:
                name: orders
                port:
                  number: 80
          - path: /                  # everything else → frontend Service
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

## Try it
```bash
cd Day5/02-path-routing
kubectl apply -f .
kubectl describe ingress shop-paths
```
```
Rules:
  Host        Path      Backends
  ----        ----      --------
  *
              /orders   orders:80 (10.244.0.20:80,10.244.0.21:80)
              /         frontend:80 (10.244.0.18:80,10.244.0.19:80)
```
`describe` shows each rule **and the pod IPs behind each Service**. Great for debugging.

Wait ~10 seconds, then:
```bash
curl -s http://localhost/            | grep -E "^(Name|GET)"
curl -s http://localhost/products    | grep -E "^(Name|GET)"
curl -s http://localhost/orders      | grep -E "^(Name|GET)"
curl -s http://localhost/orders/123  | grep -E "^(Name|GET)"
```
Real results:

| URL | Went to | Why |
|---|---|---|
| `/` | **frontend** | matches `/` |
| `/products` | **frontend** | matches `/` (no other rule fits) |
| `/orders` | **orders** | matches `/orders` |
| `/orders/123` | **orders** | starts with `/orders/` |
| `/ordersXYZ` | **frontend** ⚠️ | Prefix matches whole **path parts** (split by `/`). `ordersXYZ` ≠ `orders` |
| `/shop/orders` | **frontend** | must **start** with `/orders` |

## Rules for matching
- **Longest matching path wins.** `/orders/123` matches both `/` and `/orders`; `/orders` is longer, so it wins. The order in the YAML doesn't matter.
- `pathType: Prefix` → matches the path and everything under it (`/orders`, `/orders/`, `/orders/123`).
- `pathType: Exact` → only that exact path (`/orders`, not `/orders/123`).

## Note: the app sees the full path
The orders app receives `GET /orders/123`, **including** `/orders`. Your app must expect that.
(If an app expects `/123` instead, nginx can strip the prefix with the `nginx.ingress.kubernetes.io/rewrite-target` annotation. That's an advanced topic.)

## Clean up
```bash
kubectl delete -f .
```

➜ Next: [03-host-routing](../03-host-routing/)
