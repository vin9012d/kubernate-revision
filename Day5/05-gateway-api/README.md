# Step 5 – Gateway API (reading only, no hands-on yet)

## What is it?
**Gateway API** is the **newer, official replacement for Ingress** (made by the same Kubernetes networking group).
Same job: get outside traffic to your Services. It's more structured and more powerful.

Why learn Ingress first then? Most existing clusters (probably including your company's EKS) still use Ingress, and the concepts are the same.

## Why a replacement was needed
| Ingress problem | Gateway API answer |
|---|---|
| Extra features (rewrite, timeouts, headers) need **controller-specific annotations**: different on nginx, Traefik, AWS | Many of these are **built into the standard YAML** |
| **One object** mixes infra settings (ports, certificates) and app routes | **Split** into separate objects for the platform team and app teams |
| HTTP only (mostly) | HTTP, HTTPS, gRPC, TCP, UDP |
| Can't split traffic by weight (canary 90/10) without annotations | `weight` is a standard field |

## The 3 main objects (vs Ingress)

```
Ingress world                         Gateway API world

IngressClass   ───────────────►       GatewayClass   "which controller"         (installed once)
                                      Gateway        "the entrance: port 80/443, certificates"
Ingress        ──split into──►                       (owned by platform/infra team)
                                      HTTPRoute      "path/host → Service rules"
                                                     (owned by each app team)
```

## Same example in both (step 2's path routing)

**Ingress** (what we used):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-paths
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

**Gateway API** (same result, 2 objects):
```yaml
# 1. The entrance (platform team) - "listen on port 80"
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shop-gateway
spec:
  gatewayClassName: some-controller     # e.g. envoy-gateway, nginx, istio, aws...
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
# 2. The routes (app team) - "attach my rules to that entrance"
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-paths
spec:
  parentRefs:
    - name: shop-gateway                # attach to the Gateway above
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /orders
      backendRefs:
        - name: orders
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```

Bonus that Ingress can't do without annotations: a canary release, sending 90% of traffic to v1 and 10% to v2:
```yaml
      backendRefs:
        - name: orders-v1
          port: 80
          weight: 90
        - name: orders-v2
          port: 80
          weight: 10
```

## Same ideas, new names

| Concept | Ingress | Gateway API |
|---|---|---|
| Which controller | `IngressClass` | `GatewayClass` |
| Entry point, ports, TLS cert | inside `Ingress` (`tls:`) | `Gateway` (`listeners:`) |
| Host rule | `rules[].host` | `HTTPRoute.spec.hostnames` |
| Path rule | `paths[].path` + `pathType` | `matches[].path` (`PathPrefix` / `Exact`) |
| Target | `backend.service.name/port` | `backendRefs[].name/port` |
| Needs a controller installed? | yes (ingress-nginx...) | yes (Envoy Gateway, NGINX Gateway Fabric, Istio, AWS...) |

## Summary
- **Ingress** = simple, everywhere today, but limited without annotations. ingress-nginx is retired upstream.
- **Gateway API** = the future standard: split roles, more features built in.
- If you understand Ingress (steps 1–4), Gateway API is mostly **new names for the same ideas**.

Hands-on with Gateway API would need a Gateway controller installed (e.g. Envoy Gateway). That can be a later day.
