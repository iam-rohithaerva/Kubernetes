# Kubernetes Production Issues (Real Scenarios)

> Cluster బాగా configure చేసి ఉంది: liveness, readiness, startup probes, rolling updates, HPA, resource requests/limits అన్నీ ఉన్నాయి. అయినా production లో issues వచ్చాయి. ఎందుకంటే code మారకపోయినా, **time, traffic, data, infra, external services** మారుతూనే ఉంటాయి.

---

## 1. Production Issue అంటే ఏమిటి?

| పదం | అర్థం |
|---|---|
| **Production Issue** | Live users వాడుతున్న system లో వచ్చే ఏ సమస్య అయినా. Real users, business, revenue పై impact |
| **Kubernetes Issue** | K8s cluster లోపల వచ్చే సమస్య (Pod, Node, Service). Dev, Test, Prod ఏ env లో అయినా రావచ్చు |
| **Kubernetes Production Issue** | Prod లో run అవుతున్న cluster లో వచ్చి, users కి impact చేసే సమస్య |

**గుర్తుపెట్టుకోండి:**
- ప్రతి Kubernetes issue production issue కాదు (dev లో కూడా రావచ్చు)
- ప్రతి production issue Kubernetes issue కాదు (DB, code, 3rd party వల్ల కూడా)

### Industry లో పిలిచే పేర్లు

| Term | అర్థం |
|---|---|
| **Incident** | Users కి impact ఉన్న ఏ unplanned event అయినా |
| **Outage** | Service పూర్తిగా down |
| **Degradation** | Service up, కానీ slow లేదా కొన్ని errors |
| **SEV1 / SEV2 / SEV3** | Severity levels (SEV1 = critical, మొత్తం down) |
| **RCA / Postmortem** | Root Cause Analysis document, incident తర్వాత |
| **MTTD / MTTR** | Mean Time To Detect / Mean Time To Recover |

---

## 2. Setup బాగున్నా Issues ఎందుకు వస్తాయి?

Lower env (test/stage) లో pass అయింది, prod లో నెలల తరబడి బాగా run అయింది. అయినా issues వచ్చాయి. కారణాలు:

| Category | Example |
|---|---|
| **Time** | Cert expiry, password rotation, memory leak, ECR lifecycle policy |
| **Traffic / Request weight** | Sale రోజు 10x traffic, heavy requests (పెద్ద reports) |
| **Data growth** | Traffic same, కానీ DB data సంవత్సరంలో 100x |
| **Infra events** | Node replace, spot interruption, AZ failure, autoscaling |
| **External** | Payment gateway down, Docker Hub rate limits, AWS outage |
| **Human changes** | IAM policy మార్చడం, SG rule తీసేయడం |

**Key pattern:** Running pods problem ని దాచేస్తాయి. **Pod restart అయినప్పుడు లేదా కొత్త node వచ్చినప్పుడే** issue బయటపడుతుంది.

---

## 3. నేను Face చేసిన Issues

### Issue 1: Sale రోజు Traffic Spike, HPA ఉన్నా Outage

**Situation:**
- HPA ఉంది: min 5, max 20, CPU 70%
- 10:00 AM కి sale start, traffic సెకన్లలో 10x

**ఏమి జరిగింది:**
1. HPA 20 pods కి scale చేసింది, కానీ 12 pods **Pending** (nodes లో space లేదు)
2. Cluster Autoscaler కొత్త nodes launch కి 4 నిమిషాలు పట్టింది
3. ఆలోపు 5 పాత pods overload, readiness probe fail, Service ఆ pods కి traffic ఆపింది, మిగతా pods మీద ఇంకా load
4. 10:05 కి 20 pods వచ్చాయి, కానీ సరిపోలేదు (max 20 మాత్రమే)
5. 20 pods x 10 connections = RDS connections పెరిగాయి, RDS CPU 100%, మొత్తం slow

**Root Cause:**
- HPA reactive. Metrics, decision, node launch, image pull, app start కలిపి 3 నుంచి 8 నిమిషాలు
- `maxReplicas` తక్కువ
- DB connection limit check చేయలేదు

**Fix:**
- **Capacity planning:** ఒక pod load test చేశాను, safe capacity 100 RPS. Peak 4,000 RPS / 100 = 40 pods + 25% buffer = 50
- Sale ముందు రోజు `minReplicas: 40`, `maxReplicas: 60` (HPA మీద ఆధారపడకుండా)
- KEDA cron trigger తో 9:45 కి auto scale up
- Cluster Autoscaler నుంచి **Karpenter** కి మారాను (node 1 నిమిషం లోపు)
- Overprovisioning (low-priority placeholder pods) తో spare capacity
- **RDS Proxy** connection pooling, Redis cache, read replicas
- EC2 vCPU quota, subnet IPs ముందే పెంచాను

---

### Issue 2: Heavy Requests వల్ల OOMKilled

**Situation:**
- "Download all orders report" feature నెలలుగా బాగా పనిచేస్తోంది
- Memory request/limit: 512Mi

**ఏమి జరిగింది:**
- ఒక పెద్ద customer కి data 20 లక్షల orders కి పెరిగింది
- Report request వచ్చినప్పుడు మొత్తం data memory లోకి load, pod **OOMKilled**
- ఆ customer retry చేస్తూనే ఉన్నాడు, ఒక్కో pod ఒక్కోసారి kill

**Root Cause:**
- Traffic count మారలేదు, **request weight** (data size) మారింది
- Stage లో ఇంత data లేదు, reproduce అవ్వలేదు

**Fix:**
- Immediate: ఆ endpoint కి memory limit పెంచి, report pods వేరే Deployment లోకి వేరు చేశాను (main API safe)
- Permanent: Dev team తో pagination / streaming కి మార్చాము
- Heavy reports ని async job గా (SQS + worker pods) మార్చాము
- Memory 80% alert పెట్టాను

---

### Issue 3: Slow Memory Leak, Deploy Freeze లో బయటపడింది

**Situation:**
- ప్రతి వారం deploy ఉండేది, pods restart అయ్యేవి
- Deploy freeze వల్ల 4 వారాలు release లేదు

**ఏమి జరిగింది:**
- 3వ వారం నుంచి రోజుకి ఒకసారి pods **OOMKilled**, restart

**Root Cause:**
- రోజుకి ~20Mi memory leak. Weekly deploy దాన్ని దాచేసింది (Time based issue)

**Fix:**
- Immediate: Rolling restart (`kubectl rollout restart`) schedule చేశాను
- Grafana లో memory trend dashboard, leak కనిపించింది
- Dev team heap dump తో leak fix చేశారు
- Alert: memory 7 రోజుల్లో steady గా పెరిగితే notify

---

### Issue 4: DB Password Rotation తర్వాత CrashLoopBackOff

**Situation:**
- AWS Secrets Manager 90 రోజులకి DB password auto-rotate
- K8s secret manually create చేసినది

**ఏమి జరిగింది:**
- ఆదివారం రాత్రి password rotate అయింది. Running pods పాత connections తో బాగానే పనిచేశాయి
- సోమవారం node maintenance వల్ల pods restart, కొత్త pods పాత password తో DB login fail, **CrashLoopBackOff**

**Root Cause:**
- Secrets Manager మరియు K8s secret sync లో లేవు (Time based)

**Fix:**
- Immediate: K8s secret update చేసి rollout restart
- Permanent: **External Secrets Operator** తో auto sync
- Reloader తో secret మారగానే pods auto restart

---

### Issue 5: ECR Lifecycle Policy వల్ల ImagePullBackOff

**Situation:**
- Prod లో 3 నెలలుగా `v1.8` run అవుతోంది
- ECR lifecycle policy: last 10 images మాత్రమే ఉంచు

**ఏమి జరిగింది:**
- చాలా builds వల్ల `v1.8` ECR నుంచి delete అయింది
- పాత nodes మీద image cache లో ఉంది, pods బాగానే ఉన్నాయి
- రాత్రి traffic పెరిగి autoscaler కొత్త node add చేసింది, అక్కడ pull fail, **ImagePullBackOff**
- Scale అవ్వాల్సిన సమయానికి scale అవ్వలేదు

**Fix:**
- Immediate: Git tag నుంచి `v1.8` rebuild చేసి push
- Permanent: Lifecycle policy లో `prod-*` tags exclude
- Image digest తో deploy, prod images కి separate repo

---

### Issue 6: Node Disk Full, Pods Evicted, కొత్త Pods Pending

**Situation:**
- ఒక service bug వల్ల ప్రతి request కి DEBUG logs (పెద్ద JSON)
- Node root disk 50GB

**ఏమి జరిగింది:**
1. Sale traffic వల్ల logs గంటకి 15GB, 3 గంటల్లో disk 92%
2. Node `DiskPressure=True`, taint add
3. kubelet unused images delete చేసింది, తర్వాత pods **Evicted**
4. ReplicaSet కొత్త pods create చేసింది, వేరే nodes కూడా నిండుతున్నాయి, కొత్త pods **Pending**
5. Cascade: ఒక service bug, వేరే services (payment, cart) కూడా down

**గమనిక:** Pods disk వాడతాయి: images, container logs, writable layer, emptyDir అన్నీ node disk మీదే. Node disk full అయితే **Evicted + Pending**, CrashLoopBackOff కాదు. (PVC full అయి app crash అయితేనే CrashLoopBackOff)

**Fix:**
- Immediate: Log level INFO కి మార్చి redeploy, evicted pods clean
- `ephemeral-storage` requests/limits ప్రతి pod కి
- containerd log rotation (max 10Mi, 5 files)
- Fluent Bit తో logs CloudWatch కి
- Node root volume 100GB, disk 75% alert

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
- Internal service-to-service TLS cert 1 సంవత్సరం valid, manual గా create చేసినది

**ఏమి జరిగింది:**
- అర్ధరాత్రి 12 కి cert expire, అన్ని internal API calls fail, checkout down

**Fix:**
- Immediate: కొత్త cert generate చేసి secret update, rollout restart
- Permanent: **cert-manager** తో auto renewal
- Cert expiry కి 30 రోజుల ముందు alert

---

## 4. Troubleshooting Flow

1. **Impact చూడు:** ఎంత మంది users, ఏ service, severity
2. **Mitigate first:** Rollback, scale up, traffic shift
3. **Layer by layer:** Ingress → Service → Pod → Node → External (DB, API)
4. **Root cause:** `kubectl describe`, `logs --previous`, events, metrics
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
| Traffic spike | Sale event | Pending, slow | Capacity planning, pre-scale, Karpenter, RDS Proxy |
| Heavy request | Data growth | OOMKilled | Pagination, async jobs, separate deployment |
| Memory leak | Time (no deploys) | OOMKilled | Heap dump fix, trend alerts |
| Password rotation | Time | CrashLoopBackOff | External Secrets Operator, Reloader |
| ECR lifecycle | Time + new node | ImagePullBackOff | Protect prod tags |
| Disk full | Logs + traffic | Evicted, Pending | ephemeral-storage limits, log rotation |
| Cert expiry | Time | Calls failing | cert-manager, expiry alerts |

---

## 6. Interview Answer (English)

> "Our cluster was well configured with liveness, readiness, and startup probes, rolling updates, HPA, and proper resource requests. Still, production issues came from things that change over time: traffic and request weight, data growth, secret rotation, certificate expiry, and infrastructure events. For example, during a sale HPA reacted too slowly and the database ran out of connections, so we moved to capacity planning with load tests, pre-scaling, Karpenter, and RDS Proxy. We also hit CrashLoopBackOff after a DB password rotation, which we solved with External Secrets Operator, and node DiskPressure from debug logs, which we fixed with ephemeral-storage limits and log rotation. The pattern I learned is that running pods hide problems until a restart or a new node, so we focus on resilience, automation, and early alerts rather than expecting zero failures."
