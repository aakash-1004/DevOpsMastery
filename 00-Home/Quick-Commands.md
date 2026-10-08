---
title: Quick Commands
aliases:
  - Key Commands — Quick Reference
type: cheatsheet
module: home
tags:
  - cheatsheet
  - commands
up: "[[Home]]"
---

# Quick Commands

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

```bash
# Start minikube
minikube start

# Port-forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address=0.0.0.0 &

# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 --address=0.0.0.0 &

# Scale up monitoring
kubectl scale deployment prometheus-grafana --replicas=1 -n monitoring
kubectl scale deployment prometheus-kube-prometheus-operator --replicas=1 -n monitoring
kubectl scale deployment prometheus-kube-state-metrics --replicas=1 -n monitoring
kubectl scale statefulset alertmanager-prometheus-kube-prometheus-alertmanager --replicas=1 -n monitoring
kubectl scale statefulset prometheus-prometheus-kube-prometheus-prometheus --replicas=1 -n monitoring

# Run taskmanager locally
cd ~/aakash/devops/taskmanager
source venv/bin/activate
docker compose up -d
python3 app.py

# AWS CLI check
aws sts get-caller-identity

# Terraform
cd ~/aakash/devops/bootcamp/terraform-aws
terraform init && terraform plan
```

---
