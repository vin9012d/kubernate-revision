# Step 4 – HTTPS (TLS) with Ingress

## Goal
Serve `https://shop.local` with a certificate. The app itself stays plain HTTP.

## Idea: nginx does the HTTPS work ("TLS termination")
```
browser ══ HTTPS (encrypted) ══► ingress-nginx ── HTTP (plain) ──► shop pod
                                 (has the certificate)
```
- The **certificate + private key** are stored in a Kubernetes **Secret** (type `kubernetes.io/tls`).
- The Ingress says: "for host `shop.local`, use the certificate in Secret `shop-tls`".
- Your apps don't need any HTTPS code.

## Files
| File | What |
|---|---|
| [`app.yaml`](app.yaml) | The shop app (Deployment + ClusterIP Service) |
| [`ingress-tls.yaml`](ingress-tls.yaml) | Ingress with a `tls` section |
| `shop.crt`, `shop.key` | You generate these in step 1 (git-ignored, never commit keys) |

## Step 1 – Create a certificate (self-signed, for learning)
```bash
cd Day5/04-tls
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout shop.key -out shop.crt \
  -subj "/CN=shop.local" -addext "subjectAltName=DNS:shop.local"
```
- `shop.crt` = certificate (public, "this is shop.local")
- `shop.key` = private key (secret!)
- **Self-signed** = we signed it ourselves, not a trusted authority. Browsers will warn. Fine for learning.
  In real life: **Let's Encrypt** via **cert-manager** (free, automatic), or AWS ACM on EKS.

## Step 2 – Store it in a Secret
```bash
kubectl create secret tls shop-tls --cert=shop.crt --key=shop.key
kubectl describe secret shop-tls
```
```
Type:  kubernetes.io/tls
Data
====
tls.crt:  1021 bytes
tls.key:  1704 bytes
```

## Step 3 – The Ingress with TLS
```yaml
spec:
  ingressClassName: nginx
  tls:                              # ← NEW
    - hosts:
        - shop.local                # HTTPS for this domain
      secretName: shop-tls          # certificate from this Secret
  rules:
    - host: shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop
                port:
                  number: 80        # app is still plain HTTP on port 80
```
```bash
kubectl apply -f app.yaml -f ingress-tls.yaml
kubectl get ingress shop-tls
```
```
NAME       CLASS   HOSTS        PORTS     AGE
shop-tls   nginx   shop.local   80, 443   14s       ← now 443 (HTTPS) too
```

## Step 4 – Test (wait ~10s)

**a) HTTPS, normal check → rejected (expected)**
```bash
curl --resolve shop.local:443:127.0.0.1 https://shop.local/
```
```
curl: (60) SSL certificate problem: self signed certificate
```
curl doesn't trust our homemade certificate. Good: that's real HTTPS security working.

**b) HTTPS, trust our certificate → works**
```bash
curl --cacert shop.crt --resolve shop.local:443:127.0.0.1 https://shop.local/ | grep Name
# Name: shop   ✅
```
(Quick alternative: `-k` = skip the check. Only for testing.)

**c) What the app sees**
```bash
curl -sk --resolve shop.local:443:127.0.0.1 https://shop.local/ | grep X-Forwarded-Proto
# X-Forwarded-Proto: https
```
The app got plain HTTP, but nginx tells it "the user came over HTTPS".

**d) HTTP automatically redirects to HTTPS**
```bash
curl -i --resolve shop.local:80:127.0.0.1 http://shop.local/ | grep -E "^HTTP|^Location"
```
```
HTTP/1.1 308 Permanent Redirect
Location: https://shop.local
```
ingress-nginx does this by default once `tls` is set.

**e) See which certificate nginx serves**
```bash
echo | openssl s_client -connect 127.0.0.1:443 -servername shop.local 2>/dev/null | openssl x509 -noout -subject -enddate
```
```
subject= /CN=shop.local
notAfter=Nov 24 09:21:17 2027 GMT
```

## Clean up
```bash
kubectl delete -f app.yaml -f ingress-tls.yaml
kubectl delete secret shop-tls
rm shop.crt shop.key
```

➜ Next: [05-gateway-api](../05-gateway-api/)
