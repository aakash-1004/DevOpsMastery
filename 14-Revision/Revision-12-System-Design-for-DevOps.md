---
title: Revision — System Design for DevOps
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
order: 12
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-11-Deployment-Nginx-Gunicorn-systemd]]"
next: "[[Revision-13-Interview-Cheat-Sheet]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — System Design for DevOps

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-11-Deployment-Nginx-Gunicorn-systemd|Revision — Deployment: Nginx, Gunicorn, systemd]] · → [[Revision-13-Interview-Cheat-Sheet|Revision — Quick Reference: Interview Cheat Sheet]]
<!-- /nav -->

## The Core Patterns

**High Availability:**

- Eliminate single points of failure: everything critical has at least 2 instances across 2 AZs
- Load balancing: distribute traffic across instances
- Health checks: route traffic only to healthy instances
- Auto-scaling: handle load spikes automatically

**Deployment Strategies:**

- **Recreate**: Stop all old, start all new. Downtime. Never in production.
- **Rolling update**: Replace pods one by one. Default in Kubernetes Deployments. Zero downtime, brief period of mixed versions.
- **Blue/Green**: Two identical environments. Switch traffic instantly. Easy rollback (switch back). Cost: double infrastructure.
- **Canary**: Route small % of traffic to new version. Validate before full rollout. Best risk management, most complex.

**The 12-Factor App:**

1. Codebase — one repo per app
2. Dependencies — declare explicitly (requirements.txt, package.json)
3. Config — store in environment (never in code)
4. Backing services — treat as attached resources (DB URL in config, not hardcoded)
5. Build/Release/Run — strict separation
6. Processes — stateless (no local filesystem state)
7. Port binding — self-contained HTTP server
8. Concurrency — scale out via processes
9. Disposability — fast startup, graceful shutdown
10. Dev/Prod parity — keep environments as similar as possible
11. Logs — treat as event streams (stdout, not files)
12. Admin processes — run as one-off processes

---

## Senior Interview Q&A

**Q: Design a CI/CD pipeline for a microservices application. Walk me through the architecture.**

> "For a microservices app, I'd design around these principles: each service has its own pipeline, independent deployability, and central observability.
>
> Source: each microservice in its own directory in a monorepo, or separate repos. PRs trigger the respective service pipeline.
>
> CI pipeline per service: code checkout → dependency install → unit tests → integration tests (against test DB/services) → static analysis/security scan → Docker build → image pushed to ECR/Docker Hub with `git sha` tag.
>
> CD pipeline: pipeline updates the Helm values file in a 'config repo' with the new image tag, commits and pushes. ArgoCD detects the Git change, syncs the Kubernetes cluster.
>
> The separation is critical: the app code repo and the deployment config repo are separate. CI owns app code. ArgoCD owns cluster state. This means a cluster outage doesn't block CI, and a failed deploy doesn't affect other services.
>
> For staging: ArgoCD auto-syncs. For production: ArgoCD waits for manual sync approval (a PR approval in the config repo triggers sync).
>
> Observability layer: Prometheus scrapes all services, centralized Grafana, alerts route to PagerDuty for on-call. Logs flow to ELK or CloudWatch.
>
> One thing I always add: a smoke test job after deploy — runs a few key API calls against the deployed service to verify it's actually serving traffic correctly before marking the deploy complete."

---

**Q: Your application is running slow. How do you systematically debug it?**

> "The USE method for infrastructure + RED method for services.
>
> **USE (for every resource):** Utilization, Saturation, Errors. Is CPU at 90% (high utilization)? Is the request queue growing (saturation)? Are there errors in the logs?
>
> **RED (for every service):** Rate (requests per second), Errors (error rate), Duration (latency). These three metrics tell you the health of any service.
>
> Drill down approach:
>
> - Metrics first: Grafana dashboard. Where is CPU/memory high? Which service has high error rate or high latency?
> - Identify the bottleneck service. In a request trace: user → API gateway → service A → service B → DB. Which hop is slow?
> - If it's DB: slow query log. Missing index? N+1 query? Table scan?
> - If it's the app: profiling. Python cProfile, Java async-profiler. Where is the CPU time being spent?
> - If it's external: network latency to a third-party API? Add caching.
> - If it's Kubernetes: is the pod being throttled (CPU throttle)? Is it hitting memory limits and triggering GC constantly?
>
> The mistake people make: jumping to solutions before finding the actual bottleneck. Profile first, optimize second."
