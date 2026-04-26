# 📊 Kubernetes Monitoring Stack (Prometheus + Grafana + InfluxDB)

This guide provides a **step-by-step setup** for:

* Prometheus + Grafana using **kube-prometheus-stack (rendered YAML, no Helm at runtime)**
* Minimal, noise-free configuration for **EKS / Kubernetes**
* Path-based routing support (e.g., `/grafana`, `/prometheus`)
* InfluxDB (for K6 metrics ingestion)

---

# 🧭 Architecture Overview

```
K8s Cluster
│
├── Prometheus (metrics collection)
├── Grafana (visualization)
├── kube-state-metrics (cluster metrics)
├── InfluxDB (K6 metrics storage)
│
└── Istio (optional)
     ├── /grafana
     └── /prometheus
```

---

# ⚙️ PART 1 — Prometheus + Grafana (WITHOUT Helm at runtime)

## Step 1 — Create values-minimal.yaml

This disables unnecessary components and prepares for Istio path-based routing.

```yaml
alertmanager:
  enabled: false

nodeExporter:
  enabled: false

defaultRules:
  create: false

kubeEtcd:
  enabled: false
kubeControllerManager:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false

grafana:
  enabled: true
  adminPassword: admin123

  service:
    type: ClusterIP

  grafana.ini:
    server:
      root_url: "%(protocol)s://%(domain)s/grafana/"
      serve_from_sub_path: true

prometheus:
  enabled: true
  prometheusSpec:
    replicas: 1
    retention: 7d

    externalUrl: /prometheus
    routePrefix: /prometheus

    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi

kubeStateMetrics:
  enabled: true

prometheusOperator:
  enabled: true
```

---

## Step 2 — Render YAML (on internet-enabled machine)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm template monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f values-minimal.yaml \
  > monitoring-stack.yaml
```

---

## Step 3 — Deploy to cluster (NO Helm required)

```bash
kubectl create namespace monitoring

kubectl apply -f monitoring-stack.yaml
```

---

## Step 4 — Verify

```bash
kubectl get pods -n monitoring
```

Expected components:

* Grafana
* Prometheus
* kube-state-metrics
* Prometheus Operator

NOT expected:

* Alertmanager
* node-exporter

---

## Step 5 — Access locally

Grafana:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Prometheus:

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

---

## Step 6 — Istio Path-Based Routing

```yaml
http:
- match:
  - uri:
      prefix: /grafana
  route:
  - destination:
      host: monitoring-grafana.monitoring.svc.cluster.local
      port:
        number: 80

- match:
  - uri:
      prefix: /prometheus
  route:
  - destination:
      host: monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local
      port:
        number: 9090
```

---

# 📦 PART 2 — InfluxDB (for K6)

## Step 1 — Apply InfluxDB YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: Service
metadata:
  name: influxdb
  namespace: monitoring
spec:
  clusterIP: None
  selector:
    app: influxdb
  ports:
    - port: 8086
---
apiVersion: v1
kind: Service
metadata:
  name: influxdb-svc
  namespace: monitoring
spec:
  type: ClusterIP
  selector:
    app: influxdb
  ports:
    - port: 8086
      targetPort: 8086
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: influxdb
  namespace: monitoring
spec:
  serviceName: influxdb
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      containers:
        - name: influxdb
          image: influxdb:1.12.4-alpine
          ports:
            - containerPort: 8086
          env:
            - name: INFLUXDB_DB
              value: k6
          volumeMounts:
            - name: influxdb-data
              mountPath: /var/lib/influxdb
          livenessProbe:
            httpGet:
              path: /ping
              port: 8086
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /ping
              port: 8086
            initialDelaySeconds: 10
  volumeClaimTemplates:
    - metadata:
        name: influxdb-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## Step 2 — Deploy

```bash
kubectl apply -f influxdb.yaml
```

---

## Step 3 — Verify DB

```bash
kubectl exec -n monitoring -it influxdb-0 -- influx
```

```sql
SHOW DATABASES;
```

Expected:

```
k6
_internal
```

---

## Step 4 — K6 Integration

```bash
k6 run \
  --out influxdb=http://influxdb-svc.monitoring:8086/k6 \
  script.js
```

---

# 🔗 PART 3 — Grafana Integration (InfluxDB)

Add datasource in Grafana:

* Type: InfluxDB
* URL: `http://influxdb-svc.monitoring:8086`
* Database: `k6`

---

# 🧠 Design Decisions

| Choice                         | Reason                           |
| ------------------------------ | -------------------------------- |
| No Helm at runtime             | Works in restricted environments |
| Disabled Alertmanager          | No alert noise                   |
| Disabled node-exporter         | Lean footprint                   |
| Disabled control-plane metrics | EKS compatibility                |
| Path-based routing             | Avoid subdomain cert complexity  |
| InfluxDB 1.12                  | Native K6 support                |

---

# 🧹 Cleanup

```bash
kubectl delete -f monitoring-stack.yaml
kubectl delete -f influxdb.yaml
kubectl delete namespace monitoring
```

---

# 🚀 Next Enhancements

* Add Prometheus authentication
* Secure Grafana with SSO
* Add K6 dashboards
* Integrate Loki for logs

---

**You now have a clean, minimal, production-aligned monitoring stack.**
