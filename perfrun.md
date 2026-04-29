# 50K TPS Architecture Tuning Guide
## NATS Request/Reply — Spring Boot + Fastify on AWS m6id.4xlarge

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [AZ Co-location — Confirmed + Lock In](#az-co-location--confirmed--lock-in)
3. [Node Layout](#node-layout)
4. [JVM Tuning — lltm Server](#jvm-tuning--lltm-server)
5. [JVM Tuning — lltm Client (Spring Boot)](#jvm-tuning--lltm-client-spring-boot)
6. [FBP NATS Client — EDN Config Analysis](#fbp-nats-client--edn-config-analysis)
7. [NATS Server Tuning](#nats-server-tuning)
8. [Hikari Connection Pool Tuning](#hikari-connection-pool-tuning)
9. [PostgreSQL Diagnostic Queries](#postgresql-diagnostic-queries)
10. [PostgreSQL Tuning Guide](#postgresql-tuning-guide)
11. [LMAX Disruptor](#lmax-disruptor)
12. [Chronicle Queue + NVMe](#chronicle-queue--nvme)
13. [Fastify Node.js Client](#fastify-nodejs-client-replacement)
14. [k6 Load Test Configuration](#k6-load-test-configuration)
15. [Kubernetes Resource Sizing](#kubernetes-resource-sizing)
16. [Pre-Test Checklist](#pre-test-checklist)
17. [Monitoring During Test](#monitoring-during-test)
18. [Decision Tree — Sync vs Async DB Write](#decision-tree--sync-vs-async-db-write)

---

## Architecture Overview

```
k6 pods (external)
    │
    │ HTTP
    ▼
lltm-client pods (external node)
    │
    │ VPC network ~0.3ms (must be same AZ)
    ▼
NATS pod (m6id.4xlarge — 4 CPU / 8 Gi)
    │
    │ pod network (localhost speed)
    ▼
lltm server pod (m6id.4xlarge — 8 CPU / 20 Gi)
    │
    ├──▶ LMAX Disruptor (RAM)         ~microseconds
    ├──▶ Chronicle Queue (NVMe)       ~50–100µs
    └──▶ PostgreSQL RDS (network)     ~1–5ms  ← ceiling
```

---

## AZ Co-location — Confirmed + Lock In

### Confirmed Final Topology

```
eu-north-1a — Node 1 (m6id.4xlarge — NVMe SSD — TAINTED)
  ├── lltm server    (taint keeps all other pods off this node)
  └── nats           (taint keeps all other pods off this node)

eu-north-1a — Node 2 (separate node, same AZ)
  ├── lltm-client × 10
  └── k6 × 10–15
```

This is the **ideal layout**:
- Server + NATS share intra-node pod network (~0.05ms)
- Client has dedicated CPU/RAM — no resource competition with server
- k6 and client on same node is fine — different workload profiles
- Taint on m6id node already prevents client/k6 from landing there

---

### Measured Network Latency

```
Client → NATS ping results:
  mdev : 0.071ms   (extremely stable — no jitter)
  max  : 0.555ms   (worst case single packet)
  avg  : ~0.3ms    (estimated)

Verdict: same AZ confirmed ✓
         Cross-AZ would show max > 2ms — well within budget
```

### Latency Budget with Real Numbers

```
Client → NATS (measured, intra-AZ)       ~0.3ms  ✓
NATS → Server (intra-node pod network)   ~0.05ms ✓
FBP pipeline + server processing          ~?ms   ← measure via baseline run
LMAX + Chronicle NVMe write              ~0.1ms  ✓
Reply path                               ~0.3ms  ✓
─────────────────────────────────────────────────
Network overhead total                   ~0.75ms ✓

Processing budget remaining:
  At 350 VUs to hit 50K TPS:
    Required avg latency = 350 / 50,000  = 7ms total
    Network consumes                     = ~0.75ms
    FBP + server processing budget       = 6.25ms ← must stay under this
```

### Why You Still Need Affinity on Client

The taint on the m6id node protects server and NATS automatically. But without
affinity on the client deployment, a pod restart could schedule client pods to
`eu-north-1c` — adding 1–3ms cross-AZ latency silently.

`nodeAffinity` on the client locks it to `eu-north-1a` permanently. No
`podAffinity` needed — the taint handles node separation.

> lltm server and NATS do **not** need affinity patches — the node taint
> already guarantees they stay on the m6id.4xlarge in eu-north-1a.

---

### Step 1 — Confirm Current State

```bash
# Confirm taint on server node
kubectl describe node <server-node> | grep -i taint

# Check AZ of all nodes
kubectl get nodes -o custom-columns=\
"NAME:.metadata.name,\
AZ:.metadata.labels.topology\.kubernetes\.io/zone,\
TYPE:.metadata.labels.node\.kubernetes\.io/instance-type"

# Confirm current pod placement
kubectl get pods -o wide -n lltm | grep -E "lltm|nats|k6"
```

---

### Step 2 — Apply nodeAffinity to Client and k6 Only

```bash
# ── lltm-client Deployment ──────────────────────────────────────
kubectl patch deployment lltm-client -n lltm --type=merge -p '{
  "spec": {
    "template": {
      "spec": {
        "affinity": {
          "nodeAffinity": {
            "requiredDuringSchedulingIgnoredDuringExecution": {
              "nodeSelectorTerms": [{
                "matchExpressions": [{
                  "key": "topology.kubernetes.io/zone",
                  "operator": "In",
                  "values": ["eu-north-1a"]
                }]
              }]
            }
          }
        }
      }
    }
  }
}'

# ── k6 Deployment ───────────────────────────────────────────────
kubectl patch deployment k6 -n lltm --type=merge -p '{
  "spec": {
    "template": {
      "spec": {
        "affinity": {
          "nodeAffinity": {
            "requiredDuringSchedulingIgnoredDuringExecution": {
              "nodeSelectorTerms": [{
                "matchExpressions": [{
                  "key": "topology.kubernetes.io/zone",
                  "operator": "In",
                  "values": ["eu-north-1a"]
                }]
              }]
            }
          }
        }
      }
    }
  }
}'
```

---

### Step 3 — Verify After Patch

```bash
# Confirm affinity set on client
kubectl get pod -l app.kubernetes.io/name=lltm-client -n lltm \
  -o jsonpath='{.items[0].spec.affinity}' | jq .

# Confirm affinity set on k6
kubectl get pod -l app=k6 -n lltm \
  -o jsonpath='{.items[0].spec.affinity}' | jq .

# Confirm all pods still in eu-north-1a after rollout
kubectl get pods -o wide -n lltm | grep -E "lltm|nats|k6"
# NODE column should all resolve to eu-north-1a nodes

# Re-run ping to confirm latency unchanged after rollout
kubectl exec -it lltm-client-<pod> -n lltm -- \
  ping -c 100 <nats-pod-ip> | tail -3
# Expect: mdev < 0.1ms, max < 1ms
```

---

### Step 4 — Baseline Run Before Full Test

With topology locked, run a low-VU baseline to measure actual FBP processing time:

```javascript
// baseline.js
export const options = {
  stages: [
    { duration: '1m',  target: 50 },
    { duration: '5m',  target: 50 },  // ← read avg latency here
    { duration: '30s', target: 0  },
  ],
  thresholds: {
    http_req_duration: ['p(50)<100', 'p(99)<500'],
  },
  discardResponseBodies: true,
}
```

From k6 summary — use `http_req_duration avg`:

```
avg latency from k6   = end-to-end including network
Subtract ~0.75ms      = actual FBP + server processing time

VUs needed for 50K TPS = 50,000 × avg_total_latency_seconds

Examples:
  avg = 4ms  → VUs needed =  200  ✓ well within your 350
  avg = 7ms  → VUs needed =  350  ✓ exactly your current config
  avg = 10ms → VUs needed =  500  → scale k6 to 500 VUs across pods
  avg = 15ms → VUs needed =  750  → 75 VUs per pod across 10 pods
```

---

## Node Layout

### Server Node — m6id.4xlarge (TAINTED — lltm + NATS only)

| Resource | Total | lltm Server | NATS | kube-system | Free |
|---|---|---|---|---|---|
| vCPU | 16 | 8 | 4 | ~1 | ~3 |
| RAM | 64 Gi | 20 Gi | 8 Gi | ~1 Gi | ~35 Gi |
| NVMe | 2×474 GB | Chronicle | — | — | — |
| Network | 12.5 Gbps | shared | shared | — | ~94% |

### Client Node (separate — eu-north-1a, affinity locked)

| Component | Replicas | CPU Limit | RAM Limit |
|---|---|---|---|
| lltm-client | 10 | 4 | 4 Gi |
| k6 | 10–15 | 4 | 4 Gi |

```bash
# Verify node capacity available for client pods
kubectl describe node <client-node> | grep -A10 "Allocated resources"
```


---

## JVM Tuning — lltm Server

### ConfigMap

```yaml
_JAVA_OPTIONS: >-
  -Xms2g
  -Xmx16g
  -XX:+UseZGC
  -XX:+ZGenerational
  -XX:ConcGCThreads=4
  -XX:ZAllocationSpikeTolerance=4
  -XX:MaxMetaspaceSize=512m
  -XX:ReservedCodeCacheSize=256m
  -XX:MaxDirectMemorySize=2g
  -XX:+UseContainerSupport
  -XX:ActiveProcessorCount=8
  -XX:TieredStopAtLevel=4
  -Djdk.tracePinnedThreads=short
```

### Memory Budget (20 Gi container limit)

| Region | Size | Notes |
|---|---|---|
| Heap (Xmx) | 16384 Mi | ZGC manages this |
| Metaspace | 512 Mi | Spring context + classes |
| Code cache | 256 Mi | JIT compiled paths |
| Direct memory | 2048 Mi | Chronicle mmap + NATS buffers |
| ZGC internal | ~300 Mi | ~1–2% of heap |
| Thread stacks | ~400 Mi | ~400 virtual threads |
| **Total** | **~19900 Mi** | |
| **OOMKill buffer** | **~100 Mi** | ⚠ tight — monitor |

> If OOMKill occurs under load, reduce `Xmx` to `15g`.

### Flag Rationale

| Flag | Value | Reason |
|---|---|---|
| Xms | 2g | LMAX pre-allocates 131072 slots at startup — needs committed heap |
| Xmx | 16g | 20Gi limit minus off-heap overhead |
| ZGenerational | on | Separates short-lived NATS message objects from long-lived Chronicle state |
| ConcGCThreads | 4 | ZGC concurrent threads — half of ActiveProcessorCount |
| ZAllocationSpikeTolerance | 4 | Handles burst allocation during LMAX ring buffer fill |
| MaxDirectMemorySize | 2g | Chronicle memory-mapped files + NATS NIO buffers |
| UseContainerSupport | on | **Critical** — reads cgroup limits not host RAM/CPU |
| ActiveProcessorCount | 8 | Explicit CPU visibility — prevents JVM using all 16 host vCPUs |
| TieredStopAtLevel | 4 | Full JIT compilation — server does real compute |
| tracePinnedThreads | short | Detects virtual thread pinning — remove after perf test |

### Virtual Threads Config

```yaml
spring.threads.virtual.enabled: "true"
SERVER_TOMCAT_THREADS_MAX: "200"        # safety cap only — virtual threads do real work
spring.task.execution.pool.core-size: "8"
spring.task.execution.pool.max-size: "8"
spring.task.execution.pool.queue-capacity: "10000"
```

### Verify After Restart

```bash
kubectl exec -it lltm-0 -- \
  java -XshowSettings:all -version 2>&1 | grep -i "heap\|processor"

# Expected:
# Min. Heap Size: 2.00G
# Max. Heap Size: 16.00G

# Check for virtual thread pinning during test
kubectl logs lltm-0 -f | grep -i "pinned\|VirtualThread"
```

---

## JVM Tuning — lltm Client (Spring Boot)

### ConfigMap

```yaml
_JAVA_OPTIONS: >-
  -Xms512m
  -Xmx2560m
  -XX:+UseZGC
  -XX:+ZGenerational
  -XX:ConcGCThreads=2
  -XX:ZAllocationSpikeTolerance=2
  -XX:MaxMetaspaceSize=256m
  -XX:ReservedCodeCacheSize=128m
  -XX:MaxDirectMemorySize=512m
  -XX:+UseContainerSupport
  -XX:ActiveProcessorCount=4
  -Djdk.tracePinnedThreads=short
```

### Memory Budget (4 Gi container limit)

| Region | Size |
|---|---|
| Heap (Xmx) | 2560 Mi |
| Metaspace | 256 Mi |
| Code cache | 128 Mi |
| Direct memory | 512 Mi |
| ZGC + stacks | ~350 Mi |
| **Total** | ~3806 Mi |
| **OOMKill buffer** | ~290 Mi ✓ |

### Flags Removed vs Original

| Flag | Action | Reason |
|---|---|---|
| `-XX:+AlwaysPreTouch` | **Removed** | Causes slow startup + OOMKill risk on constrained nodes |
| `-XX:MaxGCPauseMillis=5` | **Removed** | ZGC ignores this — G1GC flag only |
| `THREADS_MAX: 2000` | **Reduced to 200** | Virtual threads make this a safety cap only |

---

## FBP NATS Client — EDN Config Analysis

### Architecture Reality

The lltm-client uses a **Flow Based Programming (FBP) engine** with a Clojure library that owns the NATS connection entirely. The connection is managed as a closure inside the library — **not configurable via application.yml or JVM flags**.

The library exposes only four EDN parameters:

```clojure
{:urls     ["nats://nats-headless:4222"]
 :secure?  false
 :req-timeout  5     ; request timeout in seconds
 :fetch-expiry 10}   ; fetch expiry in seconds
```

Everything else — socket buffers, reconnect policy, ping interval, pending buffer size, connection pool — is **hardcoded inside the Clojure library**. You cannot change these without the library team making changes.

---

### Behaviour Pipeline (for context)

```clojure
{:config {
  "nats-http"   {:request [["loggingBehaviour" "payloadValidationBehaviour" ...]]}
  "lltm-client" {:request [["isoToLLTMRequestBehaviour" "respondToRequestBehaviour"]]}
  "pump"        {:request [["isoToLLTMRequestBehaviour" "publishBehaviour"]]}
}}
```

---

### Your Entire Tuning Surface

```
YOU CAN CHANGE:
  :req-timeout    ← only timing parameter you own
  :fetch-expiry   ← only expiry parameter you own

YOU CANNOT CHANGE (library owns these):
  socket send buffer size     ← OS default ~212KB — likely too small at 50K TPS
  socket recv buffer size     ← OS default ~212KB
  max pending buffer          ← hardcoded
  reconnect wait              ← NATS Java default 2000ms
  ping interval               ← NATS Java default 2min
  connection pool             ← single connection, intentional
```

---

### The Two Changes You Can Make Right Now

```clojure
;; Current
{:urls ["nats://nats-headless:4222"]
 :secure?      false
 :req-timeout  5
 :fetch-expiry 10}

;; Recommended
{:urls ["nats://nats-headless:4222"]
 :secure?      false
 :req-timeout  2     ; ← fail fast — prevents virtual thread stack-up at 5s each
 :fetch-expiry 4}    ; ← slightly under req-timeout — fetch cannot outlive request
```

**Why req-timeout matters at 50K TPS:**
```
If server slows down and req-timeout = 5s:
  5,000 concurrent requests × 5s wait = 25,000 thread-seconds of backlog
  Virtual threads stack up holding heap → OOM before timeout fires

With req-timeout = 2s:
  Load shed happens faster
  System recovers instead of cascading
```

---

### What to Ask the Library Team

Ranked by impact on 50K TPS:

**Ask 1 — Socket buffer size (highest impact)**
```
Can you expose :socket-send-buffer-size or :write-buffer-size
as an EDN parameter?

OS default is ~212KB per connection.
At 50K TPS on a single connection we need 2–4MB.
Without this, TCP send buffer saturation becomes the ceiling
before any other component is exhausted.

Target value: 4194304 (4MB)
```

**Ask 2 — Reconnect wait (operational reliability)**
```
Can you expose :reconnect-wait-ms?

NATS Java client default is 2000ms between reconnect attempts.
On Kubernetes, NATS pod restarts complete in ~5s.
With 2s reconnect wait, you lose ~5-10 reconnect windows.
With 100ms reconnect wait, reconnection is near-instant.

Target value: 100
```

**Ask 3 — NATS Java client version**
```
What version of nats.java is the library using?

nats.java >= 2.16 supports full Options.Builder with socket tuning.
If you are already on this version, exposing these as EDN params
is a small wrapper change — no core library rewrite needed.
```

---

### How to Check What the Library Actually Configures

```bash
# Find NATS client jar version bundled in the app
kubectl exec -it lltm-client-<pod> -- \
  find /app -name "jnats*.jar" -o -name "nats-*.jar" 2>/dev/null

# Alternatively check all jar manifests
kubectl exec -it lltm-client-<pod> -- \
  find /app -name "*.jar" -exec \
  sh -c 'jar tf "$1" 2>/dev/null | grep -qi nats && echo "$1"' _ {} \;
```

---

### Monitor for Socket Buffer Saturation During Test

Since you cannot set client socket buffers, watch for TCP-level pressure:

```bash
# Run this on client pod during k6 test
# retrans > 0 = TCP retransmitting = socket buffer saturated
kubectl exec -it lltm-client-<pod> -- \
  ss -tmi dst <nats-pod-ip> | \
  grep -E "retrans|snd_buf|rcv_buf|cwnd"

# What to look for:
#   snd_buf close to ~212KB AND retrans > 0  → library buffer ceiling hit
#   snd_buf ~4MB                              → OS expanded buffer, healthy
#   retrans = 0                               → no saturation ✓
```

If `retrans` climbs during the test — you have concrete data to take to the library team showing socket buffer is the bottleneck.

---

### Compensation — NATS Server Side

Since client socket buffers are library-controlled, compensate from the server side in nats.conf. The server-side settings push larger OS buffers on the server end of the same TCP connection:

```conf
socket_send_buffer_size: 4194304   # 4MB server → client send buffer
socket_recv_buffer_size: 4194304   # 4MB server ← client recv buffer
max_pending_size: 134217728        # 128MB server-side pending queue per connection
```

This is asymmetric — client send buffer is still OS default — but it significantly helps the receive path and gives NATS more pending headroom on the server side.

---

## NATS Server Tuning

### Kubernetes Resource

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

### nats.conf — Full Tuned Config

The existing config only had `server_name`, `lame_duck` settings, `port`, and `http_port`.
Everything below is **new and must be added**.

```conf
server_name: nats-server
lame_duck_grace_period: 10s
lame_duck_duration: 30s
pid_file: /var/run/nats/nats.pid

# Client connections
port: 4222

# Monitoring
http_port: 8222

# ── NEW TUNING BELOW ──────────────────────────────────

# Connection limits
max_connections: 500          # protect from connection floods
max_subs: 100000              # subscription table cap

# Payload
max_payload: 1048576          # 1MB — matches ISO 8583 message sizes

# Keepalive — detect dead connections faster
ping_interval: "30s"          # default is 2min — too slow
max_pings_out: 5              # default is 2 — more tolerance

# Write deadline
write_deadline: "2s"

# Pending buffer per connection
# Default 64MB → 128MB — handles burst at 50K TPS
# Safe on m6id.4xlarge with 35Gi free RAM
max_pending_size: 134217728   # 128MB

# Socket buffers — OS level
# Default ~212KB is too small for 50K TPS single connection
socket_send_buffer_size: 4194304   # 4MB
socket_recv_buffer_size: 4194304   # 4MB
```

### Verify NATS Health During Test

```bash
# Connection count and message rates
curl http://<nats-pod-ip>:8222/varz | \
  jq '{connections:.connections, in_msgs:.in_msgs, out_msgs:.out_msgs, in_bytes:.in_bytes}'

# Subscription count — should be LOW with shared reply pattern
curl http://<nats-pod-ip>:8222/subsz | jq '.num_subscriptions'

# Check socket buffer pressure from server node
kubectl exec -it lltm-0 -- \
  ss -tip | grep nats
```

### _INBOX Churn — Shared Reply Subject Pattern

Replace default per-request `_INBOX` subjects with a persistent shared reply subject per pod.
This reduces NATS subscription table from **50,000 creates/sec** to **10 total** (one per client pod).

```kotlin
// Client-side — one persistent subscription per pod
@Component
class NatsRequestRouter(
    private val nats: Connection,
    @Value("\${POD_NAME}") podName: String
) {
    private val replySubject = "replies.$podName"
    private val pending = ConcurrentHashMap<String, CompletableFuture<Message>>()

    @PostConstruct
    fun init() {
        nats.createDispatcher { msg ->
            val correlationId = msg.headers?.getFirst("X-Correlation-Id") ?: return@createDispatcher
            pending.remove(correlationId)?.complete(msg)
        }.subscribe(replySubject)
    }

    fun request(subject: String, payload: ByteArray): CompletableFuture<Message> {
        val correlationId = UUID.randomUUID().toString()
        val future = CompletableFuture<Message>()
        pending[correlationId] = future

        nats.publish(
            NatsMessage.builder()
                .subject(subject)
                .replyTo(replySubject)
                .headers(Headers().add("X-Correlation-Id", correlationId))
                .data(payload)
                .build()
        )
        return future
    }
}
```

---

## Hikari Connection Pool Tuning

### Step 1 — Check RDS max_connections

```sql
SELECT 
    name, 
    setting::int AS value,
    unit
FROM pg_settings
WHERE name IN (
    'max_connections',
    'superuser_reserved_connections'
);
```

### Step 2 — Calculate Safe Pool Size

```
Formula:
  safe_connections = max_connections - superuser_reserved - 10 (admin buffer)
  per_pod_max      = safe_connections / number_of_server_pods

RDS sizing reference:
  db.t3.medium   4 Gi  → max_connections ~170  → per pod (1 pod): 150
  db.r5.large   16 Gi  → max_connections ~500  → per pod (1 pod): 480, (4 pods): 120
  db.r5.xlarge  32 Gi  → max_connections ~1000 → per pod (1 pod): 980, (4 pods): 245
  db.r5.2xlarge 64 Gi  → max_connections ~2000 → per pod (1 pod): 980, (4 pods): 490
```

### Step 3 — Hikari ConfigMap Settings

```yaml
# For single server pod
spring.datasource.hikari.maximum-pool-size: "150"
spring.datasource.hikari.minimum-idle: "20"
spring.datasource.hikari.connection-timeout: "2000"
spring.datasource.hikari.validation-timeout: "1000"
spring.datasource.hikari.idle-timeout: "300000"
spring.datasource.hikari.max-lifetime: "1200000"
spring.datasource.hikari.keepalive-time: "60000"
spring.datasource.hikari.leak-detection-threshold: "10000"
spring.datasource.hikari.data-source-properties.reWriteBatchedInserts: "true"
```

### Step 4 — Connections Needed at 50K TPS

```
Required connections = TPS × avg_db_latency_seconds

At 1ms DB latency:  50,000 × 0.001 = 50  connections minimum
At 2ms DB latency:  50,000 × 0.002 = 100 connections minimum
At 5ms DB latency:  50,000 × 0.005 = 250 connections minimum
At 10ms DB latency: 50,000 × 0.010 = 500 connections minimum
```

> **Key insight:** If NATS reply is sent **before** DB write (async path),  
> Hikari pool size does not affect TPS — only affects persistence throughput.  
> If NATS reply waits for DB commit (sync path), Hikari becomes your TPS ceiling.

### PgBouncer — Add If Scaling to Multiple Server Pods

```ini
[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
min_pool_size = 10
reserve_pool_size = 10
reserve_pool_timeout = 3
server_idle_timeout = 300
server_connect_timeout = 5
```

```
Without PgBouncer (4 pods × 150 Hikari):
  600 real DB connections → risky on db.r5.large

With PgBouncer (4 pods × 150 Hikari → PgBouncer → 50 real):
  50 real DB connections → safe on any RDS instance
```

---

## PostgreSQL Diagnostic Queries

### Connection Overview

```sql
-- Overall connection picture
SELECT
    max_conn,
    used_conn,
    res_conn,
    (max_conn - used_conn - res_conn) AS available_conn
FROM (
    SELECT
        (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_conn,
        COUNT(*) AS used_conn,
        (SELECT setting::int FROM pg_settings WHERE name = 'superuser_reserved_connections') AS res_conn
    FROM pg_stat_activity
) t;
```

### Connections by Application and State

```sql
SELECT
    application_name,
    state,
    wait_event_type,
    wait_event,
    COUNT(*) AS count
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY 1, 2, 3, 4
ORDER BY count DESC;
```

### Hikari Pool Per Pod

```sql
SELECT
    application_name,
    client_addr,
    state,
    COUNT(*) AS connections,
    MAX(now() - state_change) AS longest_in_state
FROM pg_stat_activity
WHERE datname = current_database()
  AND application_name LIKE '%lltm%'
GROUP BY 1, 2, 3
ORDER BY connections DESC;
```

### Current TPS and Cache Hit Rate

```sql
SELECT
    datname,
    xact_commit + xact_rollback AS total_tx,
    xact_commit,
    xact_rollback,
    tup_inserted,
    tup_updated,
    tup_deleted,
    ROUND(blks_hit::numeric / NULLIF(blks_hit + blks_read, 0) * 100, 2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

### Long Running Queries (run during load test)

```sql
SELECT
    pid,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 120) AS query_snippet
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '100ms'
ORDER BY duration DESC
LIMIT 20;
```

### Lock Waits (critical during load test)

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.application_name,
    blocked_locks.locktype,
    blocked_locks.relation::regclass AS locked_table,
    blocking.pid AS blocking_pid,
    LEFT(blocking.query, 100) AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked
    ON blocked.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking
    ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Write Throughput Per Table

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_tup_ins + n_tup_upd DESC
LIMIT 20;
```

### Index Usage (ensure writes aren't hitting too many indexes)

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;
```

### Real-Time Connection Monitor (run every 10s during test)

```sql
SELECT
    state,
    wait_event_type,
    wait_event,
    COUNT(*) AS count
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY 1, 2, 3
ORDER BY count DESC;
```

### Checkpoint and WAL Pressure

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,
    buffers_alloc
FROM pg_stat_bgwriter;
```

> High `buffers_backend` means PostgreSQL is doing synchronous writes —  
> consider increasing `checkpoint_completion_target` and `max_wal_size`.

### Shell Watch Command (during k6 test)

```bash
watch -n5 'psql -U pgadmin -h <rds-endpoint> -d <dbname> -c "
SELECT state, wait_event_type, wait_event, COUNT(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY 1,2,3
ORDER BY count DESC;"'
```

---

## PostgreSQL Tuning Guide

### RDS Parameter Group Recommendations

```ini
# WAL and checkpoint — reduce I/O spikes
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB
wal_compression = on

# Memory
shared_buffers = 25% of RDS instance RAM
effective_cache_size = 75% of RDS instance RAM
work_mem = 16MB                    # per sort/hash — careful with many connections
maintenance_work_mem = 256MB

# Write performance
synchronous_commit = off           # async commit — safe if Chronicle is durable buffer
                                   # ONLY if NATS reply is sent before DB write
wal_writer_delay = 10ms
commit_delay = 1000                # microseconds — batch commits

# Connection
max_connections = <set by instance>
idle_in_transaction_session_timeout = 30000   # kill stuck transactions

# Autovacuum — tune for high write workload
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms
autovacuum_max_workers = 4
```

> ⚠ `synchronous_commit = off` is safe **only** if Chronicle Queue is your  
> durable write-ahead log and NATS reply does not depend on DB commit.

### Hibernate Batch Config (already good — confirm these are set)

```yaml
spring.jpa.properties.hibernate.jdbc.batch_size: "100"
spring.jpa.properties.hibernate.order_inserts: "true"
spring.jpa.properties.hibernate.order_updates: "true"
spring.datasource.hikari.data-source-properties.reWriteBatchedInserts: "true"
```

---

## LMAX Disruptor

### Hard Ceiling — 131072 Slots

```
131072 = 2^17 — maximum before pod CrashLoopBackOff
Reason: LMAX pre-allocates ALL slots at startup
        Above this threshold → heap pressure during init → timeout

Current config:
  lmax.buffer.size: 131072   ✓ keep at this value
  fbp.lmax.batch.size: 500   ✓ good batch size

At 50K TPS:
  131072 slots / 50,000 TPS = 2.6 seconds of buffer
  Sufficient for transient DB slowdowns
```

### Verify Slot Memory Cost

```bash
# Check heap usage right after startup
kubectl exec -it lltm-0 -- jcmd 1 GC.heap_info

# Check full native memory breakdown
kubectl exec -it lltm-0 -- \
  jcmd 1 VM.native_memory summary scale=MB | \
  grep -E "Java Heap|Class|Thread|Code|GC"
```

---

## Chronicle Queue + NVMe

### Verify NVMe Mount

```bash
# Is Chronicle on NVMe or EBS?
kubectl exec -it lltm-0 -- df -h /data/chronicle
kubectl exec -it lltm-0 -- lsblk -d -o NAME,SIZE,ROTA,TYPE

# ROTA=0 means SSD/NVMe ✓
# ROTA=1 means spinning disk ✗

# Benchmark write speed
kubectl exec -it lltm-0 -- \
  dd if=/dev/zero of=/data/chronicle/speedtest \
  bs=1M count=2000 oflag=direct 2>&1
rm /data/chronicle/speedtest
# NVMe: > 1 GB/s ✓
# EBS gp3: ~250 MB/s
```

### Optional — RAID-0 Across Both NVMe Drives

```bash
# Check if already striped
cat /proc/mdstat

# If not striped — one-time node setup for ~2x write throughput
mdadm --create /dev/md0 --level=0 --raid-devices=2 \
  /dev/nvme1n1 /dev/nvme2n1
mkfs.xfs -f /dev/md0
mount /dev/md0 /data/chronicle
```

> RAID-0 gives ~3 GB/s vs ~1.5 GB/s single NVMe.  
> Chronicle batch size of 10,000 at 50K TPS is well within single NVMe capacity.  
> Only needed if you scale beyond 100K TPS.

---

## Fastify Node.js Client Replacement

If Spring Boot client is replaced with Fastify (recommended for better I/O efficiency):

### server.js

```javascript
import Fastify from 'fastify'
import { connect, StringCodec } from 'nats'

const fastify = Fastify({ logger: false })
const sc = StringCodec()
const SHARD_COUNT = parseInt(process.env.SHARD_COUNT || '4')

// FNV-1a hash — deterministic account → shard routing
function getSubject(accountId) {
  let h = 2166136261n
  for (const char of accountId) {
    h ^= BigInt(char.charCodeAt(0))
    h = BigInt.asUintN(32, h * 16777619n)
  }
  const shard = Number(h % BigInt(SHARD_COUNT))
  return `transactions.shard.${shard}`
}

let natsClient

async function init() {
  natsClient = await connect({
    servers: process.env.NATS_URL || 'nats://nats-service:4222',
    reconnect: true,
    maxReconnectAttempts: -1,
    reconnectTimeWait: 100,
  })
}

fastify.post('/process', async (request, reply) => {
  const { accountId } = request.body
  if (!accountId) return reply.code(400).send({ error: 'accountId required' })

  const subject = getSubject(accountId)
  const msg = await natsClient.request(
    subject,
    sc.encode(JSON.stringify(request.body)),
    { timeout: 5000 }
  )
  return reply.send(sc.decode(msg.data))
})

fastify.get('/health', async () => ({ status: 'ok' }))

process.on('SIGTERM', async () => {
  await natsClient.drain()
  await fastify.close()
  process.exit(0)
})

await init()
await fastify.listen({ port: 3000, host: '0.0.0.0', backlog: 4096 })
```

### Kubernetes Resources for Fastify

```yaml
resources:
  requests:
    cpu: "1"
    memory: "256Mi"
  limits:
    cpu: "4"
    memory: "4Gi"

env:
  - name: SHARD_COUNT
    value: "4"
  - name: NATS_URL
    value: "nats://nats-headless:4222"
  - name: NODE_OPTIONS
    value: "--max-old-space-size=3072"
```

> Fastify throughput: ~15,000–25,000 RPS per pod at 4 CPU  
> vs Spring Boot sync: ~3,000–5,000 RPS per pod at 4 CPU

---

## k6 Load Test Configuration

### Critical Issues with Current Config

```
Current:  { duration: '30s', target: 350 }, { duration: '30s', target: 350 }
Problem:  60 second total — no steady state
          System never warms up before test ends
          JVM JIT not warm, NATS connections not settled
          Results measure ramp noise, not throughput
```

### Correct Test Structure

```javascript
export const options = {
  scenarios: {
    load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50  },   // warm up JVM + connections
        { duration: '3m', target: 50  },   // baseline latency measurement
        { duration: '2m', target: 200 },   // step up
        { duration: '3m', target: 200 },   // measure
        { duration: '2m', target: 350 },   // target VUs
        { duration: '5m', target: 350 },   // ← THIS is your measurement window
        { duration: '1m', target: 0   },   // graceful ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    http_req_failed:   ['rate<0.01'],
  },
  discardResponseBodies: true,   // saves CPU and memory on k6 pods
}
```

### VUs Needed to Hit 50K TPS

```
TPS  = VUs / avg_response_time_seconds

Target: 50,000 TPS
At 5ms  avg latency → need 250 VUs  (25 per pod across 10 pods)
At 10ms avg latency → need 500 VUs  (50 per pod)
At 20ms avg latency → need 1000 VUs (100 per pod)
At 50ms avg latency → need 2500 VUs (250 per pod)
```

### k6 Diagnostic Commands

```bash
# Check if k6 pods are CPU-throttled (silent killer)
kubectl top pods -l app=k6 --containers
# If CPU shows exactly 4000m → throttled → add more pods

# Check k6 pod logs for errors
kubectl logs <k6-pod> -f | grep -E "error|failed|timeout"
```

---

## Kubernetes Resource Sizing

### Server Node (m6id.4xlarge)

```yaml
# lltm server — StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: lltm
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: lltm
          resources:
            requests:
              cpu: "6"
              memory: "12Gi"
            limits:
              cpu: "8"
              memory: "20Gi"
```

```yaml
# NATS
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

### Client Node

```yaml
# lltm-client (Spring Boot)
resources:
  requests:
    cpu: "2"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "4Gi"

# k6
resources:
  requests:
    cpu: "2"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "4Gi"
```

### Node Affinity — Enforce Same AZ

```yaml
# Server pod
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: ["eu-north-1a"]   # match your AZ

# Client pod — same zone label
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: ["eu-north-1a"]   # same AZ as server
```

---

## Pre-Test Checklist

```
AZ Co-location
✓ Confirmed same AZ — ping max 0.555ms, mdev 0.071ms
□ Affinity patch applied to lltm-client, nats, lltm StatefulSet
□ Pods verified still on same AZ nodes after patch rollout
□ Baseline 50 VU run completed — FBP processing time measured

Infrastructure
□ NVMe mount verified at /data/chronicle (not EBS)
□ NATS pod confirmed on same node as lltm server

JVM — Server
□ _JAVA_OPTIONS applied and verified (jcmd check)
□ Min Heap: 2.00G, Max Heap: 16.00G confirmed
□ tracePinnedThreads active — watching for pinning warnings

JVM — Client
□ _JAVA_OPTIONS applied (not JDK_JAVA_OPTIONS if Chronicle conflict)
□ MaxDirectMemorySize: 512m confirmed in /proc/1/cmdline
□ THREADS_MAX: 200 confirmed

FBP NATS Client (EDN Config — only tunable surface)
□ req-timeout reduced from 5 → 2 in nats.config EDN
□ fetch-expiry reduced from 10 → 4
□ Library team contacted re: socket buffer exposure
□ NATS Java client version confirmed (target >= 2.16)
□ Socket saturation monitoring command ready (ss -tmi)

NATS Server
□ nats.conf updated with socket_send/recv_buffer_size: 4MB
□ max_pending_size: 128MB applied
□ ping_interval: 30s applied
□ NATS memory limit bumped to 8Gi
□ NATS monitoring port 8222 accessible
□ Verify via /varz endpoint after restart

PostgreSQL
□ Run connection overview query — know your baseline
□ Run max_connections query — know your ceiling
□ Hikari maximum-pool-size set correctly per formula
□ reWriteBatchedInserts: true confirmed

k6
□ Test duration extended to minimum 18 minutes total
□ discardResponseBodies: true set
□ Stages include 5-minute steady state window
□ k6 pods not CPU-throttled at idle (kubectl top pods)

Chronicle
□ Write speed benchmarked > 500 MB/s
□ /data/chronicle is on NVMe (ROTA=0)
```

---

## Monitoring During Test

### Watch All Key Metrics Simultaneously

```bash
# Terminal 1 — Pod resource usage
watch -n3 'kubectl top pods | grep -E "lltm|nats"'

# Terminal 2 — NATS message rates + slow consumers
watch -n3 'curl -s http://<nats-pod-ip>:8222/varz | \
  jq "{in_msgs_rate:.in_msgs_rate, out_msgs_rate:.out_msgs_rate, \
       connections:.connections, slow_consumers:.slow_consumers, \
       max_pending:.max_pending_size}"'

# Terminal 3 — PostgreSQL connections
watch -n5 'psql -U pgadmin -h <rds-endpoint> -c \
  "SELECT state, count(*) FROM pg_stat_activity \
   WHERE datname=current_database() GROUP BY state;"'

# Terminal 4 — Virtual thread pinning (both client and server)
kubectl logs lltm-0 -f | grep -i "pinned\|VirtualThread\|carrier" &
kubectl logs <lltm-client-pod> -f | grep -i "pinned\|VirtualThread\|carrier"

# Terminal 5 — JVM heap live on server
watch -n3 'kubectl exec lltm-0 -- \
  curl -s localhost:8080/actuator/metrics/jvm.memory.used | \
  jq ".measurements[0].value / 1024 / 1024 | floor | tostring + \" MB\""'

# Terminal 6 — FBP pipeline errors + timeouts
kubectl logs <lltm-client-pod> -f | \
  grep -iE "timeout|req-timeout|behaviour|pipeline|error|slow|exception"
```

### NATS Slow Consumer — Critical Alert

```bash
# slow_consumers > 0 means server cannot keep up with NATS publish rate
# This is your earliest warning of server-side backpressure
curl -s http://<nats-pod-ip>:8222/varz | jq '.slow_consumers'
```

---

## Decision Tree — Sync vs Async DB Write

```
Does NATS reply wait for PostgreSQL commit?
│
├── YES (sync path)
│     50K TPS requires:
│       DB latency < 0.02ms avg     ← not achievable on RDS
│     Recommendation:
│       Switch to async — reply from LMAX/Chronicle
│       Persist to DB via queue drainer (already exists)
│       Chronicle is your durability guarantee
│       50K TPS becomes achievable ✓
│
└── NO (async path)
      NATS reply sent after LMAX/Chronicle write
      DB write happens via queue drainer asynchronously
      Chronicle = durable buffer between NATS and DB
      
      50K TPS is achievable on current hardware ✓
      
      Tune queue.drainer.poll.interval.ms: 50
      (reduce from 1000ms for aggregate drainer)
      Monitor Chronicle queue depth during test
```

---

## Summary — Priority Order of Changes

| Priority | Change | Impact |
|---|---|---|
| 1 | **AZ co-location confirmed** (0.555ms max) — apply affinity patch to lock it | Prevents cross-AZ regression after pod restart |
| 2 | Run **baseline k6 at 50 VUs** to measure FBP processing time | Determines exact VU count needed for 50K TPS |
| 3 | Extend full k6 test to **18 min with 5 min steady state** | Get real measurements not warmup noise |
| 4 | Apply server `_JAVA_OPTIONS` with `Xms2g Xmx16g` | Fixes LMAX startup + GC headroom |
| 5 | Reduce FBP **req-timeout 5→2, fetch-expiry 10→4** in EDN | Prevents backpressure cascade at scale |
| 6 | Update **nats.conf** — socket buffers 4MB + max_pending 128MB | Compensates for library-locked client buffers |
| 7 | Bump NATS memory limit to **8Gi** | Prevents NATS becoming bottleneck |
| 8 | Check sync vs async DB write path | Determines if 50K TPS is architecturally possible |
| 9 | Set Hikari pool size per formula | Prevents DB connection exhaustion |
| 10 | Add `tracePinnedThreads` temporarily to both pods | Finds hidden virtual thread pinning |
| 11 | Run PostgreSQL diagnostic queries during test | Locate actual DB ceiling |
| 12 | Monitor `ss -tmi` for TCP retransmits on client pod | Detect library socket buffer saturation |
| 13 | Ask library team to expose `:socket-send-buffer-size` | Unlocks single biggest remaining tuning lever |
| 14 | Ask library team to expose `:reconnect-wait-ms` | Improves pod restart recovery time |
| 15 | Replace Spring Boot client with Fastify | 3–5x client throughput — bypasses library constraint entirely |

---

*Generated: 2026-04-29 | Architecture: FBP/NATS Request-Reply | Target: 50K TPS | Node: AWS m6id.4xlarge | NATS: 2.12.4*