---
title: "Revision — Monitoring: Prometheus & Grafana"
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 10
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-09-GitOps-and-ArgoCD]]"
next: "[[Revision-11-Deployment-Nginx-Gunicorn-systemd]]"
related:
  - "[[Monitoring-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Monitoring: Prometheus & Grafana

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-09-GitOps-and-ArgoCD|Revision — GitOps & ArgoCD]] · → [[Revision-11-Deployment-Nginx-Gunicorn-systemd|Revision — Deployment: Nginx, Gunicorn, systemd]]
<!-- /nav -->

## Concept

Observability has three pillars: metrics (Prometheus), logs (ELK/Loki), traces (Jaeger/Tempo). Prometheus + Grafana is the de facto Kubernetes monitoring stack.

**How Prometheus works:**

1. Prometheus **scrapes** metrics from targets (HTTP GET to `/metrics` endpoint)
2. Targets are discovered via static config, service discovery, or ServiceMonitor CRDs
3. Metrics are stored in a time-series database
4. PromQL is used to query and compute on those metrics
5. Alertmanager handles alert routing (PagerDuty, Slack, email)

**Metric types:**

- **Counter**: Only goes up. Total requests, total errors. Use `rate()` to get per-second rate.
- **Gauge**: Can go up or down. Memory usage, active connections, queue size.
- **Histogram**: Samples observations into buckets. Used for request latency distributions. Exposes `_bucket`, `_count`, `_sum`.
- **Summary**: Client-side percentiles. Similar to histogram but less flexible for aggregation.

---

## Key Commands & PromQL

```bash
# Helm installation (kube-prometheus-stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Access UIs
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
# Grafana default: admin / prom-operator

# Useful PromQL queries
rate(http_requests_total[5m])                          # Request rate over 5 minutes
rate(http_requests_total{status=~"5.."}[5m])           # 5xx error rate
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))  # p99 latency
container_memory_usage_bytes{namespace="production"}   # Memory per container
kube_pod_container_status_restarts_total               # Pod restart count (alert on this!)

# For JS7/JMX metrics (your specific setup):
java_lang_Memory_HeapMemoryUsage_used                 # JVM heap usage
js7_order_count_total                                  # JS7 order count
```

**ServiceMonitor (for scraping custom apps):**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flask-monitor
  namespace: monitoring
  labels:
    release: prometheus     # Must match Helm release label selector
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: flask-app
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

---

## Senior Interview Q&A

**Q: What's the difference between a Counter and a Gauge? Give examples.**

> "A Counter only ever increases — it counts events. HTTP request count, error count, bytes sent. It resets to zero only on process restart. You never query a raw counter value in PromQL — you use `rate()` to get the per-second rate of increase, or `increase()` for total increase over a range.
>
> A Gauge can go up or down — it represents a current state. CPU utilization, memory usage, active connections, number of pods running, queue depth. You query gauges directly.
>
> Why the distinction matters: if your metric counts total requests since startup (counter), and you want requests per minute, you need `rate(http_requests_total[1m]) * 60`. If you mistakenly modeled this as a gauge and reset it each minute, you lose the ability to aggregate across multiple replicas and you have inconsistent behavior across restarts.
>
> Common mistake: modeling something as a gauge when it should be a counter. If it's a thing that's happened (event), it's a counter. If it's a thing that is (state), it's a gauge."

---

**Q: How do you set up alerting? Walk through the full chain.**

> "The chain: metric condition → Prometheus alert rule → Alertmanager → notification channel.
>
> In the PrometheusRule CRD:
>
> ```yaml
> - alert: HighErrorRate
>   expr: rate(http_requests_total{status=~'5..'}[5m]) > 0.1
>   for: 5m            # Must be true for 5 minutes before firing
>   labels:
>     severity: critical
>   annotations:
>     summary: 'High 5xx error rate on {{ $labels.service }}'
> ```
>
> Prometheus evaluates this every `evaluationInterval` (default: 15s). If the expression is true for the `for` duration, the alert fires to Alertmanager.
>
> Alertmanager handles routing: which alert goes to which team (based on labels), grouping (batch 50 alerts into one Slack message instead of 50), silencing (suppress alerts during planned maintenance), and inhibition (if the cluster is down, suppress all service alerts).
>
> The `for` duration is critical. Without it, a single spike fires the alert. 5 minutes of sustained high error rate is a real problem; a 10-second spike might be a batch job."

---

**Q: Your Prometheus targets show a service as DOWN. How do you debug?**

> "Check the Prometheus targets page (`/targets` in the UI) — it shows the exact error: connection refused, 404, no route to host.
>
> Connection refused: the app is not running on the scraped port. Check if the pod is up and the port is correct in the ServiceMonitor.
>
> No route to host / DNS resolution failure: the service name in the scrape config doesn't resolve. Check the ServiceMonitor selector labels match the Service labels.
>
> 403 Forbidden: the app has auth on `/metrics`. Either remove auth on that path or configure Prometheus with the right auth headers.
>
> 404 Not Found: the app doesn't expose `/metrics` at that path. Either it's not instrumented (needs the prometheus client library added) or the path is different.
>
> For Flask apps: you need `prometheus_flask_exporter` and `PrometheusMetrics(app)` initialized. Then the `/metrics` endpoint exists automatically.
>
> The most common issue I see: ServiceMonitor `release` label doesn't match the Prometheus Helm release name. kube-prometheus-stack looks for ServiceMonitors with `release: prometheus` by default — if yours has `release: myrelease`, it's ignored silently."

<!-- related -->
---
**Related:** [[Monitoring-MOC|Monitoring]]
<!-- /related -->
