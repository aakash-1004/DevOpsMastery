---
title: CI/CD — Map of Content
type: moc
module: cicd
tags:
  - moc
  - cicd
up: "[[Home]]"
related:
  - "[[Revision-08-CICD-GitHub-Actions-and-Jenkins]]"
---

# CI/CD — Map of Content

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

GitHub Actions, ArgoCD/GitOps, Jenkins (architecture, pipeline anatomy, Boardgame project) and the EKS GitOps labs.

**Related modules:** [[Git-MOC|Git & GitHub]] · [[Docker-MOC|Docker]] · [[Kubernetes-MOC|Kubernetes]] · [[Terraform-MOC|Terraform]]

## GitHub Actions
- [[7.1-GitHub-Actions|GitHub Actions — CI/CD]] — *concept* · interview: 🔴 High — CI/CD is asked in every DevOps interview
  - Covers: What is CI/CD?; What is GitHub Actions?; Workflow File Structure; Key YAML Fields Explained; The Pipeline We Built; Pipeline Gate — Proven in Lab; Secrets Setup
- [[7.1.1-GitHub-Actions-Quick-Reference|GitHub Actions CI/CD — Quick Reference & Mnemonics]] — *cheat sheet* · interview: 5/5
  - Covers: The scaffold — say this before writing any workflow; The skeleton; Standard 3-stage pattern (test → build-and-push → deploy); Four rules — each one is a real bug hit and fixed this cycle, not theory; The secrets-hygiene lesson — worth its own section, recurring across every project this cycle; Small things worth remembering, each from a real moment
- [[7.2-GitHub-Actions-Pipeline|GitHub Actions CI/CD Pipeline]] — *project* · interview: ⭐⭐⭐⭐⭐ (Asked in almost every MNC DevOps interview)
  - Covers: What Is GitHub Actions?; Jenkins → GitHub Actions Mental Map; The 4 Core Building Blocks; GitHub Actions Superpower — `uses:`; Pipeline We Built Today; Secrets Management; Dockerfile Created for flask-mongo-app

## GitOps
- [[7.3-ArgoCD-and-GitOps|CI/CD: ArgoCD & GitOps]] — *concept* · interview: ⭐⭐⭐⭐⭐ (Asked in every MNC DevOps interview, industry standard for Kubernetes deployments)
  - Covers: What is GitOps?; Push vs Pull — Key Differences; What is Drift?; What is ArgoCD?; What is a CRD?; ArgoCD Components; Installation
- [[7.3.1-Helm-Chart-Anatomy-and-Commands|Helm — Chart Anatomy, Templating & Commands (GitOps Lab)]] — *concept* · interview: ⭐⭐⭐⭐⭐ (Asked in every MNC DevOps interview, industry standard for Kubernetes deployments)
  - Covers: Theory (short); Chart anatomy — the mnemonic: "CVT" (Chart, Values, Templates); Creating a chart; Writing manifests — templating syntax you'll actually use; The `required` function — a real lesson worth internalizing; Real lesson: secrets probably shouldn't be in the chart at all; Namespace — two separate concepts, easy to conflate
- [[7.3.2-ArgoCD-Application-Manifest-and-Commands|ArgoCD — Application Manifest, Commands & Interview Answers]] — *concept* · interview: ⭐⭐⭐⭐⭐ (Asked in every MNC DevOps interview, industry standard for Kubernetes deployments)
  - Covers: Theory (short); The Application manifest — full field-by-field breakdown; Commands — applying and inspecting an Application; Cross-cutting lesson: Helm and ArgoCD's secret-handling philosophies actually agree; Interview-Ready Spoken Answers; What Comes Next

## Jenkins
- [[7.4-Jenkins-CICD-Pipeline|Jenkins CI/CD — Automated Deployment Pipeline]] — *project* · interview: 🔴 High — CI/CD pipelines are tested in every DevOps interview
  - Covers: What We Built; Architecture; Jenkins Deep Dive; Writing Jenkinsfiles — Complete Guide; Key Concepts; Installation Steps; Setting Up Applications
- [[7.4.1-Jenkins-Controller-Agent-Architecture|Jenkins — Controller/Agent Architecture]] — *concept*
  - Covers: Theory; Topology (this environment); Lab — commands, with why; Verification pipeline; Real-world context; Interview-ready spoken answers
- [[7.4.2-Jenkins-Declarative-Pipeline-Anatomy|Jenkins — Declarative Pipeline Anatomy]] — *concept* · interview: high
  - Covers: Theory; Lab — practice repo; The debugging saga (the actual learning, not the happy path); Real-world gotchas found in the SUCCESSFUL build's own log; Interview-ready spoken answers

## Labs & projects
- [[Lab-Taskmanager-CICD|Lab — CI/CD Pipeline for Taskmanager]] — *lab*
  - Covers: What Was Built; Files Created; ci.yml — Full Pipeline; test_app.py; GitHub Secrets Added; Pipeline Runs; Pipeline Gate Demo
- [[Lab-GitOps-Go-App-on-EKS|GitOps Go App — End-to-End Pipeline on AWS EKS]] — *project* · interview: 🔴 High — GitOps, ArgoCD, and EKS come up constantly in mid/senior DevOps interviews
  - Covers: Overview; Multi-Stage Distroless Docker Build; Kubernetes `type: LoadBalancer` → Real AWS Infrastructure; ArgoCD — GitOps Reconciliation Loop; GitHub Actions — Closing the CI Loop; cert-manager — Kubernetes-Native HTTPS; Topology Spread Constraints — Real Pod-Level HA
- [[Lab-GoWebApp-Full-GitOps-Pipeline|Go Web App — Full Containerization to GitOps CI/CD Pipeline]] — *project* · interview: 5/5
  - Covers: Project Summary (one-breath version); PART 1 — Docker: Multi-Stage Distroless Build; PART 2 — Kubernetes Core Objects (Written From Scratch); PART 3 — Ingress / Ingress Controller / IngressClass — Proven on Two Different Controllers; PART 4 — AWS EKS Specifics; PART 5 — Helm Chart Conversion; PART 6 — ArgoCD & GitOps
- [[Lab-EKS-IRSA-Workload-Identity|AWS EKS + IRSA — Secure Workload Identity]] — *project* · interview: High — IRSA/OIDC trust chains are a strong differentiator, directly maps to Keycloak/OIDC background
  - Covers: Overview; The Problem IRSA Solves; Step 1 - IAM OIDC Provider; Step 2 - IAM Policy (Pure AWS, No Kubernetes Yet); Step 3 - The Actual IRSA Binding; Step 4 - Deploy the Controller Using the IRSA Identity; Step 5 - Proving It: A Real ALB, Not Just a Running Pod
- [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] — *hub* · 17 notes

[[Home]] · [[Interview-Prep-MOC]]

<!-- related -->
---
**Related:** [[Revision-08-CICD-GitHub-Actions-and-Jenkins|Revision — CI/CD: GitHub Actions & Jenkins]]
<!-- /related -->
