---
title: The Golden Thread Project — Taskmanager
aliases:
  - Golden Thread
type: project
module: home
tags:
  - golden-thread
  - project
up: "[[Home]]"
---

# The Golden Thread Project — Taskmanager

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

## The Golden Thread Project

**Taskmanager REST API** — Flask + MongoDB

| Stage | Status | Details |
|-------|--------|---------|
| Built locally | ✅ | Flask + MongoDB, full CRUD |
| Dockerized | ✅ | Dockerfile + Docker Compose |
| Kubernetes | ✅ | 2 replicas, NodePort :30000 |
| AWS EC2 | ✅ | Deployed via Docker Compose |
| Terraform | ✅ | Full infra as code |
| CI/CD | ✅ | GitHub Actions → Docker Hub |
| Monitoring | ✅ | /metrics → Prometheus → Grafana |

**Repo:** https://github.com/aakash-1004/taskmanager
**Docker Hub:** https://hub.docker.com/r/aakash0908/taskmanager

---

## End-to-End Pipeline

```
git push → GitHub Actions → pytest → docker build → Docker Hub → K8s → Prometheus → Grafana
```

---

## Repos

| Repo | Visibility | Stack |
|------|-----------|-------|
| [taskmanager](https://github.com/aakash-1004/taskmanager) | Public | Python, Docker, K8s, GitHub Actions |
| [terraform-aws](https://github.com/aakash-1004/terraform-aws) | Private | Terraform, AWS, HCL |

---

## Lab notes for each stage

1. **Built locally** → [[Lab-Flask-MongoDB-REST-API|Lab — Flask + MongoDB REST API]]
2. **Dockerized** → [[Lab-Taskmanager-Dockerization|Lab — Dockerizing the Taskmanager App]]
3. **Kubernetes** → [[5.3-Kubernetes-Manifests-Lab|Lab — Deploying Taskmanager to Kubernetes]]
4. **AWS EC2** → [[Lab-EC2-Deployment|Lab — Deploying Taskmanager to AWS EC2]]
5. **Terraform** → [[Lab-Terraform-AWS|Lab — Terraform AWS Infrastructure]]
6. **CI/CD** → [[Lab-Taskmanager-CICD|Lab — CI/CD Pipeline for Taskmanager]]
7. **Monitoring** → [[Lab-Monitoring-Prometheus-Grafana|Lab — Prometheus + Grafana on Kubernetes]]
