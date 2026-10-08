---
title: Lab — Prometheus + Grafana on Kubernetes
aliases:
  - Monitoring-Lab
type: lab
module: monitoring
status: ✅ Completed
tags:
  - monitoring
  - prometheus
  - grafana
  - helm
  - lab
  - day7
  - golden-thread
cluster: Minikube on bumblebee (192.168.1.18)
up: "[[Monitoring-MOC]]"
prev: "[[9.1-Prometheus-Grafana]]"
related:
  - "[[5.3-Kubernetes-Manifests-Lab]]"
origin: DevOps-Mastery/09-Monitoring/Labs/Monitoring-Lab.md
---

# Lab — Prometheus + Grafana on Kubernetes

<!-- nav -->
↑ [[Monitoring-MOC|Monitoring]] · ← [[9.1-Prometheus-Grafana|Prometheus & Grafana — Monitoring & Observability]]
<!-- /nav -->

## What Was Built

- Deployed kube-prometheus-stack via Helm to Minikube
- Accessed Grafana dashboards showing real cluster metrics
- Ran PromQL queries against Prometheus directly
- Added `/metrics` endpoint to Flask taskmanager app
- Verified Flask metrics capture both successful and failed requests
- CI pipeline updated Docker image with metrics support

---

## Setup Commands

```bash
# Start Minikube
minikube start

# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

# Verify pods
kubectl get pods -n monitoring
```

---

## Components Installed

| Component | Type | Purpose |
|-----------|------|---------|
| prometheus-grafana | Deployment | Dashboard UI |
| prometheus-kube-prometheus-operator | Deployment | Manages Prometheus/Alertmanager |
| prometheus-kube-state-metrics | Deployment | K8s object metrics |
| alertmanager | StatefulSet | Alert routing |
| prometheus | StatefulSet | Metrics storage |
| node-exporter | DaemonSet | Node CPU/memory metrics |

---

## Accessing UIs

```bash
# Grafana (keep terminal open)
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address=0.0.0.0 &
# http://192.168.1.18:3000  →  admin / admin123

# Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 --address=0.0.0.0 &
# http://192.168.1.18:9090
```

---

## PromQL Queries Run

```promql
# All pods in cluster with metadata
kube_pod_info

# Actual CPU usage %
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
# Result: ~10.4%

# Available memory %
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
# Result: ~50%

# Pod count by namespace
count by (namespace) (kube_pod_info)
# Result: kube-system=7, monitoring=6
```

---

## Flask Metrics Integration

Added to `app.py`:
```python
from prometheus_flask_exporter import PrometheusMetrics
metrics = PrometheusMetrics(app)
```

Install:
```bash
pip install prometheus-flask-exporter
pip freeze > requirements.txt
```

**Metrics generated:**
```
# Successful health checks
flask_http_request_duration_seconds_count{path="/health", status="200"} 2.0
flask_http_request_duration_seconds_sum{path="/health", status="200"} 0.00028s

# Failed tasks endpoint (MongoDB down)
flask_http_request_duration_seconds_count{path="/tasks", status="500"} 2.0
flask_http_request_duration_seconds_sum{path="/tasks", status="500"} 60.27s
```

The 60 second sum on `/tasks` immediately reveals MongoDB connection timeout as root cause.

---

## Grafana Dashboard Explored

**Kubernetes / Compute Resources / Cluster:**
- CPU Utilisation: 9.18% (real node data)
- Memory Utilisation: 48.9%
- CPU Requests Commitment: 18.8%
- CPU Quota table: kube-system (7 pods), monitoring (6 pods)

"No data" for CPU/Memory Utilisation initially → fixed by enabling node-exporter DaemonSet via `helm upgrade`.

---

## Managing the Stack

```bash
# Scale down to save RAM (3GB freed)
kubectl scale deployment prometheus-grafana --replicas=0 -n monitoring
kubectl scale deployment prometheus-kube-prometheus-operator --replicas=0 -n monitoring
kubectl scale deployment prometheus-kube-state-metrics --replicas=0 -n monitoring
kubectl scale statefulset alertmanager-prometheus-kube-prometheus-alertmanager --replicas=0 -n monitoring
kubectl scale statefulset prometheus-prometheus-kube-prometheus-prometheus --replicas=0 -n monitoring
minikube stop

# Resume next session
minikube start
kubectl scale deployment prometheus-grafana --replicas=1 -n monitoring
kubectl scale deployment prometheus-kube-prometheus-operator --replicas=1 -n monitoring
kubectl scale deployment prometheus-kube-state-metrics --replicas=1 -n monitoring
kubectl scale statefulset alertmanager-prometheus-kube-prometheus-alertmanager --replicas=1 -n monitoring
kubectl scale statefulset prometheus-prometheus-kube-prometheus-prometheus --replicas=1 -n monitoring
```

---

## Bugs / Gotchas Hit

| Issue | Cause | Fix |
|-------|-------|-----|
| `No data` on CPU/Memory utilisation | node-exporter DaemonSet missing | `helm upgrade` with `--set prometheus-node-exporter.enabled=true` |
| Port-forward not accessible from browser | Binds to localhost only | Add `--address=0.0.0.0` |
| Port-forward stops working | Terminal session ended | Re-run port-forward command |
| Flask hanging on `/metrics` | MongoDB not running, connection timeout | Start MongoDB first via docker compose |
| Port 5000 already in use | Docker container using port | Use `flask run --port 5001` |
| Minikube fails to start | Previous broken profile | `minikube delete` then `minikube start` |

---

## Related notes
- [[9.1-Prometheus-Grafana]]
- [[5.1-Kubernetes-Core-Concepts]]
- [[5.3-Kubernetes-Manifests-Lab]]]
- [[Lab-Flask-MongoDB-REST-API]]

<!-- related -->
---
**Related:** [[5.3-Kubernetes-Manifests-Lab|Lab — Deploying Taskmanager to Kubernetes]]
<!-- /related -->
