---
title: DevOps Mastery — Home
aliases:
  - DevOps Mastery — Dashboard
  - Dashboard
type: hub
module: home
tags:
  - home
  - hub
---

# DevOps Mastery — Home

**Goal:** Land a Cloud/DevOps Engineer role as fast as possible
**Timeline:** 7-day intensive bootcamp ✅ → Interview prep ongoing
**Server:** bumblebee (192.168.1.18) | devops-labs (192.168.56.101)

## Modules

- [[Fundamentals-MOC|Fundamentals]] — DevOps culture, SDLC, virtual machines and how cloud resources get provisioned.
- [[Linux-Bash-MOC|Linux & Bash]] — Linux internals, networking and performance troubleshooting; bash scripting from strict mode to a full reference; cron and log parsing.
- [[Python-DevOps-MOC|Python for DevOps]] — Python for automation: JSON/YAML, requests, subprocess, plus the Flask + MongoDB API that starts the golden-thread project.
- [[Git-MOC|Git & GitHub]] — Day-to-day Git workflow plus the complete Git & GitHub reference (13 parts).
- [[Docker-MOC|Docker]] — Architecture → images/containers → Dockerfile → networking → storage → Compose → interview Q&A, plus the Taskmanager and fullstack projects.
- [[Kubernetes-MOC|Kubernetes]] — Core concepts, manifests, k3s deployment, Helm, and EKS on Fargate with IRSA + ALB.
- [[AWS-MOC|AWS]] — Bootcamp notes (core services, EC2/ECS deployment) and the 20-part AWS Mastery reference.
- [[CICD-MOC|CI/CD]] — GitHub Actions, ArgoCD/GitOps, Jenkins (architecture, pipeline anatomy, Boardgame project) and the EKS GitOps labs.
- [[Terraform-MOC|Terraform]] — Bootcamp Terraform notes plus the 18-note Terraform Mastery course (state, modules, workspaces, VPC, projects, interview bank).
- [[Monitoring-MOC|Monitoring]] — Prometheus and Grafana concepts and the lab on Kubernetes.
- [[Deployment-MOC|Deployment (Nginx, PM2, Gunicorn, systemd)]] — Frontend/backend concepts, Nginx, PM2/Gunicorn/systemd, deployment debugging, framework deployment guide.
- [[Azure-MOC|Azure]] — Fifteen topic parts (fundamentals → networking) mapped forward from AWS, plus a cross-cloud interview Q&A.
- [[GCP-MOC|GCP]] — Quick orientation, sixteen topic parts (fundamentals → BigQuery) and a three-way cross-cloud interview Q&A.
- [[Interview-Prep-MOC|Interview Prep]] — Spoken answers, canonical answers, the concept bank, and links to every per-topic Q&A bank in the vault.
- [[Master-Revision-MOC|Master Revision]] — One-pass revision sheet: fundamentals through system design, plus an interview cheat sheet (13 parts).

## Tools & trackers

- [[Progress-Tracker]] — module status, interview-prep confidence, next steps
- [[Golden-Thread-Project]] — the Taskmanager app taken from Flask → Docker → K8s → AWS → Terraform → CI/CD → monitoring
- [[Quick-Commands]] — commands you re-type constantly
- [[Cross-Cloud-Service-Map]] — AWS ↔ Azure ↔ GCP equivalents
- [[Vault-Guide]] — how this vault is organised · [[Migration-Map]] — old file → new note · [[Backlog-Planned-Notes]] — notes you referenced but never wrote

## End-to-End Pipeline

```
git push → GitHub Actions → pytest → docker build → Docker Hub → K8s → Prometheus → Grafana
```

---
