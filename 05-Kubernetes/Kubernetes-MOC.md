---
title: Kubernetes — Map of Content
type: moc
module: kubernetes
tags:
  - moc
  - kubernetes
up: "[[Home]]"
related:
  - "[[Revision-05-Kubernetes]]"
---

# Kubernetes — Map of Content

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

Core concepts, manifests, k3s deployment, Helm, and EKS on Fargate with IRSA + ALB.

**Related modules:** [[Docker-MOC|Docker]] · [[CICD-MOC|CI/CD]] · [[AWS-MOC|AWS]] · [[Monitoring-MOC|Monitoring]]

## Concepts
- [[5.1-Kubernetes-Core-Concepts|Kubernetes Core Concepts]] — *concept* · interview: 🔴 High — K8s is the most tested DevOps skill in 2025
  - Covers: What Problem Kubernetes Solves; Architecture; Core Objects; Deployment YAML — Full Breakdown; Service YAML — Full Breakdown; Traffic Flow; ConfigMap
- [[5.4-Helm|Helm]] — *concept* · interview: ⭐⭐⭐⭐⭐ (Tested in CKA, MNC DevOps interviews, used in every production Kubernetes setup)
  - Covers: What Is Helm?; Core Concepts; Helm Chart Structure; The 5 Commands Used 90% of the Time; Chart We Built Today; Commands With Explanations; Helm Revision System

## Labs & projects
- [[5.2-K8s-Deployment-Express-Flask-k3s|Kubernetes Deployment — Express + Flask on k3s]] — *project* · interview: 🔴 High — Kubernetes is tested in every DevOps/Cloud interview
  - Covers: What We Built; Minikube vs k3s; Kubernetes Objects Used; ConfigMap vs Secret; Project Files; All Manifest Files; Commands Reference
- [[5.3-Kubernetes-Manifests-Lab|Lab — Deploying Taskmanager to Kubernetes]] — *lab*
  - Covers: What Was Built; Project Structure; Deploy Everything; namespace.yaml; mongodb.yaml; configmap.yaml; secret.yaml
- [[5.5-EKS-Fargate-IRSA-ALB-Project|EKS on Fargate + IRSA + ALB Ingress Controller — End to End]] — *project* · interview: 5/5
  - Covers: Theory (short); Commands (in order, with why); Real-world context; Interview-ready spoken answer (30-second version); Interview-ready spoken answer (deep dive / STAR); Anticipated follow-up questions

[[Home]] · [[Interview-Prep-MOC]]

<!-- related -->
---
**Related:** [[Revision-05-Kubernetes|Revision — Kubernetes]]
<!-- /related -->
