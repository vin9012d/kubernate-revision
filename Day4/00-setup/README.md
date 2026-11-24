# Step 0 – A client pod for testing

Services like ClusterIP only work **inside** the cluster. Our Mac is outside.
So we create a small pod called `client` and run commands **from inside it**.

```bash
kubectl apply -f client.yaml
kubectl get pod client        # wait for STATUS = Running
```

How we'll use it in every step:
```bash
kubectl exec client -- wget -qO- http://<address>     # like curl, from inside the cluster
kubectl exec client -- nslookup <name>                # DNS lookup from inside the cluster
```
`kubectl exec client -- <command>` = "run this command inside the `client` pod".

Keep it running for all steps. ➜ Next: [01-problem-without-service](../01-problem-without-service/)
