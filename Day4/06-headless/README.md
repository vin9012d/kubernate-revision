# Step 6 – Headless Service

## The idea in one picture

A **normal Service** is like a **call center**:
you dial one number → you get connected to **any** free agent. You can't choose.
Perfect when every pod is the same (like our `web` pods).

A **headless Service** is like a **phone directory**:
every pod gets its **own direct number** → **you** choose who to call.
Needed when pods are **not** the same.

```
NORMAL Service "db-normal"                 HEADLESS Service "db"
                                          
app ──► db-normal ──► random pod           app ──► db-0.db ──► always db-0
                     (db-0 or db-1)        app ──► db-1.db ──► always db-1
```

---

## Our example: a database where pods are NOT the same

| Pod | Role | Accepts writes? |
|---|---|---|
| `db-0` | **PRIMARY** | ✅ yes |
| `db-1` | **REPLICA** (automatic copy of db-0) | ❌ no, read only |

This is how real databases run (Redis, MySQL, PostgreSQL, MongoDB): one primary takes the writes, replicas keep copies and help with reads.

**Rule: every write MUST go to db-0.** Let's see what happens with each kind of Service.

## Files

| File | What it creates |
|---|---|
| [`1-app-pod.yaml`](1-app-pod.yaml) | Pod `app`: plays the role of our application; has `redis-cli` to talk to the DB |
| [`2-database.yaml`](2-database.yaml) | Headless Service `db` + StatefulSet `db` (pods `db-0` primary, `db-1` replica) |
| [`3-normal-service.yaml`](3-normal-service.yaml) | A **normal** Service `db-normal` in front of the same 2 pods (the wrong choice, to see the problem) |

---

## Step 1 – Create everything

```bash
cd Day4/06-headless
kubectl apply -f .
kubectl get pods -l app=db -o wide
```
```
NAME   READY   STATUS    IP
db-0   1/1     Running   10.244.0.65
db-1   1/1     Running   10.244.0.66
```

Check each pod's role (from the startup logs):
```bash
kubectl logs db-0 | head -1      # I am db-0 -> PRIMARY
kubectl logs db-1 | head -1      # I am db-1 -> REPLICA of db-0.db
```

Look at the two Services:
```bash
kubectl get svc db db-normal
```
```
NAME        TYPE        CLUSTER-IP      PORT(S)
db          ClusterIP   None            6379/TCP     ← headless: NO IP
db-normal   ClusterIP   10.96.130.136   6379/TCP     ← normal: one IP, random pod
```

### Why `StatefulSet` and not `Deployment`?
- Deployment pods get **random** names: `db-7f9c8d-x2k4p`. Delete one → new random name.
- StatefulSet pods get **fixed** names: `db-0`, `db-1`. Delete `db-0` → it comes back as `db-0` again.

You can only have a fixed phone number for a pod if the pod has a fixed name. So headless + StatefulSet go together.

---

## Step 2 – ❌ The problem: write through the NORMAL Service

Our app saves 6 orders through `db-normal`:
```bash
for i in 1 2 3 4 5 6; do
  kubectl exec app -- redis-cli -h db-normal SET order:$i pizza
done
```
Real output:
```
OK
OK
READONLY You can't write against a read only replica.   ❌
READONLY You can't write against a read only replica.   ❌
READONLY You can't write against a read only replica.   ❌
OK
```
**Half the orders were lost!**

Why? Ask "who are you?" through `db-normal` a few times:
```bash
for i in 1 2 3 4; do
  kubectl exec app -- redis-cli -h db-normal INFO replication | grep role
done
```
```
role:master     ← db-0
role:master     ← db-0
role:slave      ← db-1  (replica)
role:master     ← db-0
```
The normal Service **randomly** sends us to db-0 **or** db-1. It doesn't know db-1 can't take writes. (Load balancing is great for identical pods, but bad here.)

---

## Step 3 – ✅ The fix: talk to the right pod by NAME (headless)

The headless Service `db` gives each pod its own DNS name:
```
db-0.db    → always the primary
db-1.db    → always the replica
```
(Full form: `<pod-name>.<headless-service>.<namespace>.svc.cluster.local` → `db-0.db.default.svc.cluster.local`)

**Writes → db-0.db**
```bash
for i in 1 2 3; do
  kubectl exec app -- redis-cli -h db-0.db SET order:$i pizza
done
```
```
OK
OK
OK          ✅ every write succeeds
```

**Reads → db-1.db** (the replica copied the data automatically)
```bash
kubectl exec app -- redis-cli -h db-1.db GET order:1
```
```
"pizza"     ✅
```

And writing to the replica is still refused, which proves `db-1.db` really is db-1:
```bash
kubectl exec app -- redis-cli -h db-1.db SET x y
# READONLY You can't write against a read only replica.
```

---

## Step 4 – What DNS returns (the technical difference)

```bash
kubectl exec app -- nslookup db-normal.default.svc.cluster.local
```
```
Address: 10.96.130.136          ← normal: ONE Service IP (Kubernetes picks a pod behind it)
```
```bash
kubectl exec app -- nslookup db.default.svc.cluster.local
```
```
Address: 10.244.0.65            ← headless: ALL pod IPs (db-0 and db-1)
Address: 10.244.0.66
```
```bash
kubectl exec app -- nslookup db-0.db.default.svc.cluster.local
```
```
Address: 10.244.0.65            ← one specific pod: db-0
```

---

## Step 5 – Pod restarts? The name still works

```bash
kubectl delete pod db-0
kubectl get pods -l app=db -o wide
```
```
NAME   IP
db-0   10.244.0.67      ← SAME name db-0, NEW IP (was .65)
db-1   10.244.0.66
```
```bash
kubectl exec app -- nslookup db-0.db.default.svc.cluster.local
# Address: 10.244.0.67   ✅ the name follows the pod to its new IP
```
The app keeps using `db-0.db` and never needs to know the IP.

The replica found the new primary by itself, because it also connects using the name `db-0.db` (see `2-database.yaml`):
```bash
kubectl exec app -- redis-cli -h db-0.db SET order:99 burger
kubectl exec app -- redis-cli -h db-1.db GET order:99
# "burger"   ✅ replication working again
```

⚠️ Old orders (`order:1`...) are **gone** after the restart. Our pods have no disk storage yet, so data lives only in memory. Real databases use **PersistentVolumes** (a later topic) so data survives restarts.

---

## Summary

| | Normal Service (`db-normal`) | Headless Service (`db`) |
|---|---|---|
| YAML difference | – | `clusterIP: None` |
| Service IP | `10.96.130.136` | **None** |
| DNS returns | 1 Service IP | every pod IP + a name per pod (`db-0.db`) |
| Who chooses the pod? | Kubernetes (random) | **Your app** |
| Our write test | ❌ 3 of 6 writes failed | ✅ 3 of 3 writes OK |
| Use when | All pods are identical (web, APIs) | Pods have **different roles/data** |

### Real-world uses of headless
- **Databases with primary/replica:** MySQL, PostgreSQL, MongoDB, Redis → write to the primary by name
- **Kafka:** each broker (`kafka-0`, `kafka-1`) holds different data → clients connect to a specific broker
- **Clusters that discover each other:** Elasticsearch, Cassandra, Zookeeper → "who else is in my cluster?" → DNS returns all pod IPs

**Rule of thumb:**
- All pods identical? → normal Service.
- Need to talk to a **specific** pod? → **headless Service + StatefulSet**.

## Clean up
```bash
kubectl delete -f .
```

➜ Next: [07-externalname](../07-externalname/)
