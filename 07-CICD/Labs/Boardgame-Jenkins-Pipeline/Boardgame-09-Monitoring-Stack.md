---
title: PHASE 9 — Monitoring Stack
type: project
module: cicd
status: completed
tags:
  - jenkins
  - kubeadm
  - kubernetes
  - rbac
  - sonarqube
  - nexus
  - trivy
  - prometheus
  - grafana
  - blackbox-exporter
  - cicd
  - devops
  - notes
  - project
order: 15
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-08-Pipeline-Runs-and-Real-Debugging]]"
next: "[[Boardgame-10-Repo-Polish]]"
related:
  - "[[9.1-Prometheus-Grafana]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 9 — Monitoring Stack

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-08-Pipeline-Runs-and-Real-Debugging|PHASE 8 — Pipeline Runs and Real Debugging]] · → [[Boardgame-10-Repo-Polish|PHASE 10 — Repo Polish]]
<!-- /nav -->

## Launched monitoring server

```bash
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-0aa761682283b4cc8 \
  --instance-type c7i-flex.large \
  --key-name myWebServer \
  --security-group-ids sg-09c50e5f8249e552e \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=monitoring-server}]' \
  --query "Instances[0].InstanceId" --output text
# Result: i-0964449c2cec9ab33
```

## Prometheus installation (step by step, raw binary — not apt)

1. Created dedicated system user:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prometheus
```

Why --shell /usr/sbin/nologin: this account can never be logged into — no password exists, no shell access possible. Exists purely as an OS-level identity for file ownership and process attribution.

2. Downloaded and extracted:

```bash
cd ~
wget https://github.com/prometheus/prometheus/releases/download/v3.14.0/prometheus-3.14.0.linux-amd64.tar.gz
tar -xvzf prometheus-3.14.0.linux-amd64.tar.gz
```

3. Moved binaries into place:

```bash
sudo mv prometheus-3.14.0.linux-amd64/prometheus /usr/local/bin/
sudo mv prometheus-3.14.0.linux-amd64/promtool /usr/local/bin/
```

4. Created config and data directories:

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

/etc/prometheus: config files (standard Linux convention for config). /var/lib/prometheus: time-series database storage (standard for application data that changes over time). Empty until Prometheus actually starts.

5. Moved bundled config file:

```bash
cd prometheus-3.14.0.linux-amd64
sudo mv prometheus.yml /etc/prometheus/
```

6. Set ownership:

```bash
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

7. Edited config file:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Content (initial, later expanded):

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

8. Created systemd service file:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus/

[Install]
WantedBy=multi-user.target
```

Why a manual service file is needed: Prometheus was installed from a raw GitHub binary tarball, not an apt package — no .deb package means no pre-written systemd service included. apt-installed tools (Jenkins, Docker, Grafana) get this automatically; manually-downloaded binaries require you to write it yourself.

9. Started the service:

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus --no-pager
```

## YAML config bug hit twice

First time: missing closing bracket `]` on a targets line — Prometheus failed to start with INVALIDARGUMENT. Second time: indentation mismatch on two new job blocks (8 spaces where 4 were expected) — same failure.

Both diagnosed via `cat -A /etc/prometheus/prometheus.yml` (shows invisible whitespace characters) + `sudo journalctl -u prometheus --no-pager -n 30` (shows the full, untruncated error that systemctl status cuts off).

## Node Exporter installation (on monitoring server, then on Jenkins server)

Same manual-binary pattern as Prometheus:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
tar -xvzf node_exporter-1.12.1.linux-amd64.tar.gz
sudo mv node_exporter-1.12.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Service file (`/etc/systemd/system/node_exporter.service`):

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Simpler than Prometheus's service — no config file or storage path flags needed. Node Exporter reads live OS stats on demand (from /proc, /sys) and serves them on port 9100 whenever scraped.

## Grafana installation (via apt — auto-creates systemd service)

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Why Grafana is different from Prometheus/Node Exporter: ships as a proper .deb package via apt — auto-creates its own system user and systemd service during install. Service name is `grafana-server`, not `grafana`.

Accessed at `http://<monitoring-ip>:3000`. Default login: admin / admin (forced password change on first login).

Connected to Prometheus data source: Connections → Data sources → Add → Prometheus → URL: `http://localhost:9090` → Save & Test.

## Blackbox Exporter installation

Same manual-binary pattern:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin blackbox_exporter
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.28.0/blackbox_exporter-0.28.0.linux-amd64.tar.gz
tar -xvzf blackbox_exporter-0.28.0.linux-amd64.tar.gz
sudo mv blackbox_exporter-0.28.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo mkdir -p /etc/blackbox_exporter
sudo mv blackbox_exporter-0.28.0.linux-amd64/blackbox.yml /etc/blackbox_exporter/
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chown -R blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
```

Service file (`/etc/systemd/system/blackbox_exporter.service`):

```ini
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target
```

Blackbox needs a config file (unlike Node Exporter) because it needs to know what kinds of probes to support (HTTP, TCP, ICMP) — the config defines probe "modules." Default bundled config includes `http_2xx` (the one we actually use), plus TCP/SSH/gRPC/ICMP modules we don't need.

## Final prometheus.yml (all scrape targets)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://<jenkins-ip>:8080/login
          - http://<sonarqube-ip>:9000
          - http://<app-nodeport-ip>:31540
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

  - job_name: 'jenkins_node_exporter'
    static_configs:
      - targets: ['<jenkins-ip>:9100']

  - job_name: 'jenkins_app_metrics'
    metrics_path: /prometheus
    static_configs:
      - targets: ['<jenkins-ip>:8080']
```

Why Blackbox's config shape is different (the relabel trick): Prometheus normally scrapes targets directly. For Blackbox, we rewrite the request so Prometheus scrapes Blackbox itself (localhost:9115/probe?target=...), telling Blackbox which real target to probe via a query parameter. This relabeling pattern is standard for every Blackbox+Prometheus integration.

Why Jenkins uses /login for Blackbox probing: Jenkins requires authentication for most endpoints — an unauthenticated http_2xx probe gets a 403 (correctly), which Blackbox marks as failure. /login responds with a 200 without authentication, genuinely proving "Jenkins is alive" without needing to solve auth in the probe.

Why jenkins_app_metrics uses metrics_path: /prometheus: the Jenkins Prometheus plugin exposes metrics at this non-standard path, not the default /metrics — must be specified explicitly.

## Grafana dashboards imported

- Blackbox Exporter dashboard — imported from Grafana.com by dashboard ID
- Node Exporter Full (ID 1860) — imported similarly, visualizes Jenkins server's OS-level metrics

<!-- related -->
---
**Related:** [[9.1-Prometheus-Grafana|Prometheus & Grafana — Monitoring & Observability]]
<!-- /related -->
