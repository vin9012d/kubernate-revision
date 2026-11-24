# Step 7 – ExternalName (nickname for an outside service)

## Goal
Give something **outside** the cluster an internal name.

## Story
Your apps call a payment provider, e.g. `api.razorpay.com`. If every app has that hostname in its config, switching to a sandbox or a new provider means changing every app.
Instead: apps call `http://external-api`, and **one** Service decides where that really goes.

## The YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: httpbin.org      # pretend this is the payment provider
```
- **No selector, no pods, no ports, no IP.**
- It's only a DNS rule: `external-api` → "it's actually `httpbin.org`" (a **CNAME**).

```bash
kubectl apply -f external-api.yaml
kubectl get svc external-api
```
```
NAME           TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)
external-api   ExternalName   <none>       httpbin.org   <none>
```

## See the DNS alias
```bash
kubectl exec client -- nslookup external-api.default.svc.cluster.local
```
```
external-api.default.svc.cluster.local   canonical name = httpbin.org
```

## Call it
```bash
kubectl exec client -- wget -qO- http://external-api/get
```
```json
{
  "args": {},
  "headers": {
    "Host": "external-api",
    ...
```
The request went **out to the internet** to httpbin.org ✅

Notice `"Host": "external-api"`: the request still says "external-api" as the website name. httpbin.org doesn't care, but **many real sites reject it** (and HTTPS certificates won't match). Keep that in mind when using ExternalName with real APIs.

## Typical uses
- External DB: `orders-db` → `mydb.xxxx.ap-south-1.rds.amazonaws.com`
- Third-party APIs (payment, SMS, email)
- Migration: today the DB is outside (ExternalName); tomorrow you move it into the cluster and change the Service to ClusterIP. **Apps never change.**

## Rules
- `externalName` must be a **hostname**, not an IP.
- No port mapping. You call the real port of the outside service.

➜ Optional advanced: [08-all-together](../08-all-together/), a mini shop app using every type at once.
