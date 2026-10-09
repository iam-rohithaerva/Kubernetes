# Kubernetes Production Issues (Real Scenarios)

> The cluster was well configured: liveness, readiness, and startup probes, rolling updates, HPA, and resource requests/limits were all in place. Production issues still happened, because even when the code does not change, **time, traffic, data, infrastructure, and external services** keep changing.

---

## 1. What Is a Production Issue?

| Term | Meaning |
|---|---|
| **Production Issue** | Any problem in the live system used by real users. It impacts users, business, and revenue |
| **Kubernetes Issue** | A problem inside the K8s cluster (Pod, Node, Service). It can happen in any environment: dev, test, or prod |
| **Kubernetes Production Issue** | A problem in the production cluster that impacts users |

**Remember:**
- Not every Kubernetes issue is a production issue (it can happen in dev too)
- Not every production issue is a Kubernetes issue (it can come from the database, code, or a third party)

### Industry Terms

| Term | Meaning |
|---|---|
| **Incident** | Any unplanned event that impacts users |
| **Outage** | The service is completely down |
| **Degradation** | The service is up, but slow or returning some errors |
| **SEV1 / SEV2 / SEV3** | Severity levels (SEV1 = critical, everything is down) |
| **RCA / Postmortem** | Root Cause Analysis document written after the incident |
| **MTTD / MTTR** | Mean Time To Detect / Mean Time To Recover |

---

## 2. Why Do Issues Happen Even With a Good Setup?

The release passed in lower environments (test/stage) and ran fine in prod for months. Issues still came up. The reasons:

| Category | Example |
|---|---|
| **Time** | Certificate expiry, password rotation, memory leaks, ECR lifecycle policy |
| **Traffic / Request weight** | 10x traffic on sale day, heavy requests (large reports) |
| **Data growth** | Same traffic, but database data grew 100x in a year |
| **Infrastructure events** | Node replacement, spot interruption, AZ failure, autoscaling |
| **External** | Payment gateway down, Docker Hub rate limits, AWS outage |
| **Human changes** | Someone changed an IAM policy or removed a security group rule |

**Key pattern:** Running pods hide the problem. The issue shows up **only when a pod restarts or a new node joins**.

---

## 3. Issues I Faced

### Issue 1: Sale Day Traffic Spike, Outage Even With HPA

**Situation:**
- HPA was configured: min 5, max 20, CPU target 70%
- The sale started at 10:00 AM and traffic went up 10x within seconds

**What happened:**
1. HPA scaled to 20 pods, but 12 pods stayed **Pending** (no space on the nodes)
2. Cluster Autoscaler took 4 minutes to launch new nodes
3. Meanwhile the 5 existing pods were overloaded, readiness probes failed, the Service stopped sending traffic to them, and the remaining pods got even more load
4. By 10:05, 20 pods were running, but that was not enough (max was only 20)
5. 20 pods x 10 connections each increased RDS connections, RDS CPU hit 100%, and everything slowed down

**Root cause:**
- HPA is reactive. Metrics collection, the scaling decision, node launch, image pull, and app startup together take 3 to 8 minutes
- `maxReplicas` was too low
- Database connection limits were not checked

**Fix:**
- **Capacity planning:** Load tested a single pod and found its safe capacity was 100 RPS. Peak 4,000 RPS / 100 = 40 pods, plus a 25% buffer = 50
- The day before the sale, set `minReplicas: 40` and `maxReplicas: 60` so we did not depend on HPA
- Used a KEDA cron trigger to scale up automatically at 9:45
- Moved from Cluster Autoscaler to **Karpenter** (new nodes in under 1 minute)
- Added overprovisioning (low-priority placeholder pods) to keep spare capacity
- Added **RDS Proxy** for connection pooling, a Redis cache, and read replicas
- Raised the EC2 vCPU quota and checked subnet IPs in advance

---

### Issue 2: OOMKilled Due to Heavy Requests

**Situation:**
- The "Download all orders report" feature had been working fine for months
- Memory request/limit: 512Mi

**What happened:**
- One large customer's data grew to 2 million orders
- When they requested the report, all the data was loaded into memory and the pod was **OOMKilled**
- The customer kept retrying, killing one pod after another

**Root cause:**
- The number of requests did not change, but the **request weight** (data size) did
- Stage did not have this much data, so it was never reproduced there

**Fix:**
- Immediate: Raised the memory limit for that endpoint and moved the report into a separate Deployment so the main API stayed safe
- Permanent: Worked with the dev team to switch to pagination / streaming
- Moved heavy reports to async jobs (SQS + worker pods)
- Added an alert at 80% memory usage

---

### Issue 3: Slow Memory Leak Exposed During a Deploy Freeze

**Situation:**
- We deployed every week, so pods restarted every week
- A deploy freeze meant no release for 4 weeks

**What happened:**
- From the third week, pods were **OOMKilled** and restarted about once a day

**Root cause:**
- A memory leak of about 20Mi per day. The weekly deploys had been hiding it (time based issue)

**Fix:**
- Immediate: Scheduled a rolling restart (`kubectl rollout restart`)
- Built a memory trend dashboard in Grafana, which made the leak visible
- The dev team found and fixed the leak using a heap dump
- Alert: notify if memory grows steadily over 7 days

---

### Issue 4: CrashLoopBackOff After a DB Password Rotation

**Situation:**
- AWS Secrets Manager rotated the DB password automatically every 90 days
- The Kubernetes secret had been created manually

**What happened:**
- The password rotated on Sunday night. Running pods kept working with their existing connections
- On Monday, node maintenance restarted the pods. The new pods used the old password, the DB login failed, and they went into **CrashLoopBackOff**

**Root cause:**
- Secrets Manager and the Kubernetes secret were out of sync (time based issue)

**Fix:**
- Immediate: Updated the Kubernetes secret and ran a rollout restart
- Permanent: **External Secrets Operator** for automatic sync
- **Reloader** to restart pods automatically when a secret changes

---

### Issue 5: ImagePullBackOff Caused by the ECR Lifecycle Policy

**Situation:**
- Prod had been running `v1.8` for 3 months
- ECR lifecycle policy: keep only the last 10 images

**What happened:**
- After many new builds, `v1.8` was deleted from ECR
- Existing nodes had the image cached, so their pods kept running
- At night, traffic increased and the autoscaler added a new node. The image pull failed there, giving **ImagePullBackOff**
- The service could not scale exactly when it needed to

**Fix:**
- Immediate: Rebuilt `v1.8` from the Git tag and pushed it
- Permanent: Excluded `prod-*` tags from the lifecycle policy
- Deploy by image digest, and keep a separate repository for prod images

---

### Issue 6: Node Disk Full, Pods Evicted, New Pods Pending

**Situation:**
- A bug in one service printed DEBUG logs (large JSON) on every request
- Node root disk was 50GB

**What happened:**
1. Sale traffic produced 15GB of logs per hour, and within 3 hours the disk was 92% full
2. The node got `DiskPressure=True` and a taint was added
3. kubelet deleted unused images, then started **evicting** pods
4. The ReplicaSet created new pods, but the other nodes were filling up too, so the new pods stayed **Pending**
5. Cascade: a bug in one service took down other services too (payment, cart)

**Note:** Pods do use disk. Images, container logs, the writable layer, and emptyDir volumes all live on the node disk. When the node disk is full, you get **Evicted + Pending**, not CrashLoopBackOff. (CrashLoopBackOff happens only when a PVC fills up and the app itself crashes.)

**Fix:**
- Immediate: Changed the log level to INFO, redeployed, and cleaned up evicted pods
- Set `ephemeral-storage` requests/limits on every pod
- Enabled containerd log rotation (max 10Mi, 5 files)
- Shipped logs to CloudWatch with Fluent Bit
- Increased the node root volume to 100GB and added an alert at 75% disk usage

```yaml
resources:
  requests:
    cpu: "400m"
    memory: "512Mi"
    ephemeral-storage: "1Gi"
  limits:
    memory: "512Mi"
    ephemeral-storage: "2Gi"
```

---

### Issue 7: Internal TLS Certificate Expiry

**Situation:**
- The internal service-to-service TLS certificate was valid for 1 year and had been created manually

**What happened:**
- At midnight the certificate expired, all internal API calls failed, and checkout went down

**Fix:**
- Immediate: Generated a new certificate, updated the secret, and ran a rollout restart
- Permanent: **cert-manager** for automatic renewal
- Alert 30 days before certificate expiry

---

## 4. Troubleshooting Flow

1. **Check the impact:** How many users, which service, what severity
2. **Mitigate first:** Rollback, scale up, or shift traffic
3. **Go layer by layer:** Ingress → Service → Pod → Node → External (DB, APIs)
4. **Find the root cause:** `kubectl describe`, `logs --previous`, events, metrics
5. **RCA + prevention:** Postmortem document, alerts, automation

```bash
kubectl get pods -A | grep -v Running
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl describe node <node>
kubectl get events --sort-by=.lastTimestamp
kubectl top pods / kubectl top nodes
```

---

## 5. Summary Table

| Issue | Trigger | Status | Fix |
|---|---|---|---|
| Traffic spike | Sale event | Pending, slow | Capacity planning, pre-scaling, Karpenter, RDS Proxy |
| Heavy request | Data growth | OOMKilled | Pagination, async jobs, separate deployment |
| Memory leak | Time (no deploys) | OOMKilled | Heap dump fix, trend alerts |
| Password rotation | Time | CrashLoopBackOff | External Secrets Operator, Reloader |
| ECR lifecycle | Time + new node | ImagePullBackOff | Protect prod tags |
| Disk full | Logs + traffic | Evicted, Pending | ephemeral-storage limits, log rotation |
| Certificate expiry | Time | Calls failing | cert-manager, expiry alerts |

---

## 6. Interview Answer

> "Our cluster was well configured with liveness, readiness, and startup probes, rolling updates, HPA, and proper resource requests. Still, production issues came from things that change over time: traffic and request weight, data growth, secret rotation, certificate expiry, and infrastructure events. For example, during a sale HPA reacted too slowly and the database ran out of connections, so we moved to capacity planning with load tests, pre-scaling, Karpenter, and RDS Proxy. We also hit CrashLoopBackOff after a DB password rotation, which we solved with External Secrets Operator, and node DiskPressure from debug logs, which we fixed with ephemeral-storage limits and log rotation. The pattern I learned is that running pods hide problems until a restart or a new node, so we focus on resilience, automation, and early alerts rather than expecting zero failures."
