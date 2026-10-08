---
title: "Revision — Quick Reference: Interview Cheat Sheet"
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
order: 13
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-12-System-Design-for-DevOps]]"
related:
  - "[[Interview-Prep-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Quick Reference: Interview Cheat Sheet

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-12-System-Design-for-DevOps|Revision — System Design for DevOps]]
<!-- /nav -->

## "Tell me about yourself" — DevOps Frame

> "I'm a Cloud and Automation Engineer with two years of production experience. I support JS7 JobScheduler systems, handle Keycloak/OIDC authentication issues, and do Linux administration and troubleshooting for enterprise clients. On the DevOps side, I've built end-to-end pipelines — Flask microservices containerized with Docker, deployed on Kubernetes with k3s, infrastructure provisioned with Terraform, CI/CD via GitHub Actions, GitOps with ArgoCD, and monitoring with Prometheus and Grafana. I'm also progressing toward AWS Solutions Architect Associate certification. I'm looking for a DevOps engineering role where I can own the full delivery pipeline."

---

## Must-Know Numbers

|Item|Value to Know|
|---|---|
|EC2 free tier|t3.micro, 750 hours/month|
|S3 max object size|5TB|
|S3 multipart upload required after|5GB|
|Default kubeconfig location|`~/.kube/config`|
|Prometheus default scrape interval|15s|
|Docker bridge network default subnet|172.17.0.0/16|
|etcd port|2379 (client), 2380 (peer)|
|kube-apiserver port|6443|
|kubelet port|10250|
|Grafana default port|3000|

---

## Your Projects — Quick Reference for Interviews

|Project|What it demonstrates|
|---|---|
|`flask-mongo-app`|Docker, GitHub Actions CI/CD, EC2 deployment|
|`fullstack-docker`|Multi-container, Docker Compose, networking|
|`k3s-fullstack`|Kubernetes, ConfigMap, Secrets, Service|
|`fullstack-chart`|Helm, ArgoCD, GitOps, full k8s stack|
|`terraform-aws`|Terraform, VPC, EC2, ECS Fargate, ALB|
|`taskmanager`|Flask REST API, MongoDB, Prometheus metrics|

---

## Session Summary

**Covered:** Full DevOps stack revision — Fundamentals, Linux/Bash, Git, Docker, Kubernetes, AWS, Terraform, CI/CD, GitOps, Monitoring, Deployment, System Design **Interview Q&A added:** Senior Tech Lead level — production scenarios, architecture decisions, debugging approaches **Next:** Practice speaking these answers out loud. Record yourself. The knowledge is there — fluency under pressure is the gap to close. **Progress Note:** Comprehensive revision complete — all major DevOps topics consolidated with senior-level interview Q&A.

> ⚠️ **Security reminder:** Never store AWS keys, GitHub PATs, Docker tokens, or DB passwords in Obsidian notes or plain text files. Always use your credentials manager or AWS Secrets Manager.

<!-- related -->
---
**Related:** [[Interview-Prep-MOC|Interview Prep]]
<!-- /related -->
