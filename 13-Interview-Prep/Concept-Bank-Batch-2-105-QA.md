---
title: "DevOps Interview Prep — Batch 2 (Concept Bank, ~105 Q&A)"
type: interview
module: interview
tags:
  - interview
  - devops
  - concept-bank
up: "[[Interview-Prep-MOC]]"
prev: "[[Spoken-Answers-Batch-1-28-Questions]]"
next: "[[Canonical-Answer-01-Deployment-Strategies]]"
related:
  - "[[4.9-Docker-Interview-QA]]"
origin: "OnlyQuestions/DevOps Interview Prep — Batch 2 (Concept Bank, ~105 Q&A).md"
---

# DevOps Interview Prep — Batch 2 (Concept Bank, ~105 Q&A)

<!-- nav -->
↑ [[Interview-Prep-MOC|Interview Prep]] · ← [[Spoken-Answers-Batch-1-28-Questions|DevOps Interview Prep — Batch 1 (28 Questions)]] · → [[Canonical-Answer-01-Deployment-Strategies|Canonical Answer #1 — Deployment Strategies]]
<!-- /nav -->

These are mostly "what is X" definitional questions — an interviewer at 3 YOE level wants a clean definition, why it matters, and ideally a one-line tie to real usage. Where it naturally maps to your actual stack (Terraform, Helm, ArgoCD, GitHub Actions, EKS/IRSA, Prometheus/Grafana, Git, JS7/Keycloak) I've woven that in — say it if it's true for you, drop it if it feels forced. Where a tool is outside what you've touched (Ansible/Puppet/Chef, Istio, ELK, Azure/GCP specifics, Tekton, Kong, chaos tooling), the answer is straight conceptual knowledge — fine to state matter-of-factly, no need to claim hands-on you don't have.

---

## DevOps Fundamentals

### What is DevOps?

A set of practices and culture that merges Dev and Ops so software gets built, tested, and released faster and more reliably. The core idea is collapsing the wall between "writing code" and "running code" — same team owns both, backed by automation (CI/CD, IaC) and continuous feedback (monitoring). It's not a tool, it's a way of working.

### What are the benefits of DevOps?

Faster, more frequent releases; fewer failed deployments because things are automated and tested consistently; shorter recovery time when something breaks (MTTR); better cross-team communication since Dev and Ops share ownership; and more engineering time spent building instead of firefighting.

### What is Continuous Integration?

Developers merge code into a shared branch frequently — ideally multiple times a day — and every merge triggers an automated build and test run. The goal is catching integration bugs within minutes of them being introduced, not weeks later during a "big merge." In my GitHub Actions pipeline, every push to main triggers exactly this: build, test, then the Docker build stage.

---

## Docker

### What is Docker?

A platform for packaging an application with everything it needs — runtime, libraries, config — into a single portable image that runs identically across any machine with Docker installed. It solves "works on my machine" by shipping the machine along with the code.

### What is the difference between Docker Image and Docker Container?

An image is the immutable, read-only template — built from a Dockerfile, made of layers. A container is a running instance of that image, with its own writable layer and isolated process/network/filesystem namespace. Same image, many independent containers.

### What is Dockerfile?

A text file with the step-by-step instructions Docker follows to build an image — base image, dependencies, source, build command, entrypoint. Mine for the Go app is a two-stage build: compile in a `golang:alpine` stage, then copy only the binary into a `distroless` final stage so the shipped image has no shell, no package manager, minimal attack surface.

---

## Kubernetes

### What is Kubernetes?

An open-source container orchestration platform — it automates deploying, scaling, healing, and networking containers across a cluster of machines, so you don't manually decide which node runs which container or manually restart crashed ones. I run it both as k3s in a home lab and as EKS on AWS.

### What are the main components of Kubernetes architecture?

Control plane: API server (front door for all cluster operations), etcd (the cluster's source-of-truth key-value store), controller manager (drives actual state toward desired state), scheduler (decides which node a pod lands on). Worker node: kubelet (talks to the API server, manages pods on that node), container runtime (actually runs containers), kube-proxy (handles Service networking rules).

### What is a Pod in Kubernetes?

The smallest deployable unit — one or more tightly coupled containers that share a network namespace (same IP, can talk over localhost) and can share storage volumes. Usually one container per pod unless you have a genuine sidecar case.

---

## CI/CD

### What is a CI/CD Pipeline?

A defined sequence of automated stages — build, test, package, deploy — that code goes through from commit to production. Mine is: GitHub Actions builds and pushes a Docker image → bumps the tag in a Helm values file → ArgoCD detects the Git change and syncs it to the cluster.

### What is Jenkins?

An open-source automation server, historically the default CI/CD tool — you define pipelines (via a Jenkinsfile) that build, test, and deploy, with a master–agent architecture to distribute work and a huge plugin ecosystem. My primary stack is GitHub Actions + ArgoCD, but I've built Jenkins declarative pipelines too.

---

## Cloud Platforms

### What is Cloud Computing?

Delivering compute, storage, networking, and other IT services over the internet on demand, rather than owning physical hardware — you pay for what you use and scale up or down instantly.

### What is AWS?

The market-leading cloud platform, 200+ services. The ones I actually work in day to day: EC2, S3, IAM, VPC, EKS, ALB, CloudWatch — plus ECR for images and Route 53 for DNS.

### What is Azure?

Microsoft's cloud platform — conceptually parallel to AWS: Virtual Machines and App Services for compute, Blob Storage for object storage, Virtual Network and Application Gateway for networking. I know it at a working-knowledge level, AWS is my primary.

### What are the different types of cloud services?

IaaS gives you raw virtualized infrastructure — you manage the OS up (EC2, Azure VMs). PaaS gives you a platform to run code without managing servers (Heroku, Elastic Beanstalk). SaaS is a finished application delivered over the internet (Salesforce, Gmail). FaaS is serverless — you deploy a function, the platform handles everything else (Lambda).

---

## Infrastructure as Code

### What is Infrastructure as Code?

Defining infrastructure in version-controlled code instead of clicking through a console — so it's repeatable, reviewable, and auditable the same way application code is. I use Terraform for AWS infra like VPC, ALB, and ECS — modules plus remote state.

### What is Terraform?

A declarative IaC tool — you describe the desired end state in HCL, and Terraform figures out the diff between that and current reality, then creates a plan of exactly what to add/change/destroy. I keep state in an S3 backend with a DynamoDB table for locking so two people (or two pipeline runs) can't corrupt state by applying at the same time.

### What is Ansible?

A configuration management and automation tool that uses YAML playbooks to describe tasks — install a package, start a service, push a config file — and runs them over SSH with no agent required on the target machine. I know the model well; my own environment work leans more on Terraform for provisioning and Helm/Kubernetes-native config rather than Ansible for post-provision config, but the concepts (idempotent tasks, inventory, roles) are the same discipline.

---

## Monitoring & Logging

### What is monitoring in DevOps?

Continuously collecting metrics and health signals from infrastructure and applications — CPU, memory, error rates, latency — so problems get caught by an alert, not by a user complaint. I run Prometheus/Grafana dashboards for JS7 job monitoring in my current role.

### What is ELK Stack?

Elasticsearch (search/storage), Logstash (ingestion/processing pipeline), Kibana (visualization) — together, a log aggregation and analysis stack. Common for centralizing logs from many services into one searchable place. I haven't run ELK myself in production — my monitoring stack is Prometheus/Grafana — but the pattern (ship logs centrally, search/alert on them) is the same idea.

### What is Prometheus?

A pull-based time-series monitoring system with its own query language (PromQL) — it scrapes metrics endpoints on a schedule, stores them, and can fire alerts off threshold rules. This is what I use to track JS7 job health metrics.

### What is Grafana?

The visualization layer on top of metrics sources like Prometheus — dashboards, graphs, and alerting on top of whatever data source you point it at. I build Grafana dashboards on top of the Prometheus metrics I collect for JS7.

### Explain the difference between monitoring and logging.

Monitoring is aggregated, numerical, and forward-looking — "is CPU over 80%, is error rate spiking" — good for alerting on trends. Logging is discrete, detailed, event-level records — good for figuring out exactly _why_ something failed after the fact. You use monitoring to know something's wrong, and logs to find out what actually happened.

---

## Security & Compliance

### What is DevSecOps?

Baking security checks into every stage of the pipeline instead of bolting them on at the end — static code scanning at commit, dependency/image scanning at build, policy checks before deploy. "Shift security left" so issues are caught before they're in production, not after.

### What is Infrastructure Security?

Layered security across network (security groups, NACLs, VPNs), cloud/identity (IAM least privilege, encryption, KMS), and host (OS hardening, patching). In my EKS+IRSA project this is the whole point — pods get scoped IAM permissions via OIDC/STS instead of long-lived AWS keys sitting in the cluster.

---

## Linux

### What are the basic Linux commands every DevOps engineer should know?

File ops (`ls`, `cd`, `cp`, `mv`, `rm`, `mkdir`), system info (`top`, `df`, `free`, `ps`), text processing (`grep`, `sed`, `awk`, `cat`), and networking (`curl`, `netstat`/`ss`, `dig`). This is daily-driver stuff from my support background — a lot of JS7/Keycloak incident diagnosis starts with `grep`-ing logs and checking `top`/`df` before anything else.

---

## Version Control

### What is Git?

A distributed version control system — every clone has full history, branching is cheap, and it's the backbone of pretty much every modern CI/CD or GitOps workflow. In my ArgoCD setup, Git literally _is_ the source of truth for what's deployed.

### What is Git Branching Strategy?

A convention for how branches are created and merged. Git Flow uses long-lived `main`/`develop` plus `feature`/`release`/`hotfix` branches — more structure, more overhead. Trunk-based development keeps one main branch with short-lived feature branches merged frequently — faster, less overhead, needs strong CI to back it up. For a solo project I lean toward trunk-based; a larger team benefits from more structure.

---

## Configuration Management

### What is Configuration Management?

Keeping systems in a known, desired state over time — so a server provisioned today looks the same as one provisioned six months from now, and drift gets caught and corrected rather than accumulating silently.

### What is Puppet?

A configuration management tool using a declarative language and a client-server model — agents pull their desired config from a Puppet master. Same category as Ansible/Chef; I know the model conceptually, haven't run Puppet specifically in production.

---

## Scalability & High Availability

### What is Scalability in DevOps?

The ability to handle more load by adding resources. Vertical scaling = bigger machine (more CPU/RAM on the same box). Horizontal scaling = more machines. Horizontal is generally preferred in cloud-native setups because it avoids a hard ceiling and plays well with auto-scaling and load balancing.

### What is High Availability?

Designing so there's no single point of failure — redundant instances across multiple AZs, health checks with automated failover, and load balancing to spread traffic. My EKS setup with an ALB in front of multiple pod replicas across nodes is exactly this pattern.

### What is Load Balancing?

Distributing incoming traffic across multiple backend targets so no single instance gets overloaded, and unhealthy targets get automatically taken out of rotation. I've worked with ALB (Layer 7, HTTP-aware routing) fronting EKS workloads via the AWS Load Balancer Controller.

### What is Auto Scaling?

Automatically adjusting the number of running instances based on demand — scale out under load, scale back in when it drops, based on metrics like CPU or request count. Saves cost during quiet periods and absorbs spikes without manual intervention.

---

## Backup & Disaster Recovery

### What is Backup and Disaster Recovery?

Backup is making copies of data so it can be restored if lost. Disaster Recovery is the broader plan — procedures, failover systems, RTO/RPO targets — for getting the whole service back up after a major outage, not just the data.

### What are different types of backups?

Full backup — complete copy, slowest to take, fastest to restore. Incremental — only what changed since the last backup, fast and space-efficient, slower to restore since you need the whole chain. Differential — changes since the last _full_ backup, a middle ground between the two.

---

## Cloud Native Architecture

### What is Cloud Native Architecture?

Building applications specifically to exploit what cloud gives you — containerization, orchestration, microservices, automation, designed-in resilience — rather than just lifting an old monolith onto a VM in the cloud.

### What are Microservices?

An architectural style where an app is split into small, independently deployable services, each owned around a business capability, communicating over APIs or events rather than in-process calls. My current project is a single service, but I apply the same patterns — independent Helm chart, independent CI/CD trigger, service discovery via Kubernetes DNS — that would extend cleanly to a multi-service setup.

### What is Service Mesh?

An infrastructure layer, usually sidecar proxies, that handles service-to-service traffic — routing, retries, mTLS, telemetry — without the application code having to implement any of it. Istio and Linkerd are the common examples; I know the pattern conceptually, haven't run one in production myself.

---

## Performance Testing

### What is Performance Testing?

Testing how a system behaves under load — response time, throughput, resource usage — to find bottlenecks and confirm it meets requirements before real users hit it.

### What are different types of Performance Tests?

Load testing — expected traffic, does it perform to spec. Stress testing — push past expected limits to find the breaking point. Endurance/soak testing — sustained load over hours/days to catch memory leaks and slow resource exhaustion that only show up over time.

---

## API Gateway & Related

### What is an API Gateway?

A single entry point that sits in front of your services, handling cross-cutting concerns — auth, rate limiting, routing, SSL termination — so individual services don't each have to implement them.

### What are the benefits of using an API Gateway?

Centralized auth/SSL instead of duplicating it per service, built-in caching and load balancing, and centralized monitoring/logging across all APIs in one place instead of scattered across services.

### What is API Security?

Protecting APIs via authentication (API keys, OAuth2, JWT) and authorization (RBAC, scoped permissions) so only the right callers can hit the right endpoints with the right level of access.

### What is Rate Limiting?

Capping how many requests a client can make in a given time window, to protect backend services from being overwhelmed — whether by abuse or just an unexpected traffic spike. Token bucket and leaky bucket are the two classic algorithms behind most implementations.

### What is API Documentation?

The reference material describing how to use an API — endpoints, request/response formats, auth requirements, example calls — usually auto-generated from an OpenAPI/Swagger spec so it stays in sync with the actual API.

---

## Container Orchestration — Advanced K8s

### What are StatefulSets in Kubernetes?

Used for stateful apps — databases, message queues — that need a stable network identity and stable storage per replica, plus ordered, predictable creation/scaling/deletion. Regular Deployments give you identical, interchangeable pods; StatefulSets don't.

### What are DaemonSets in Kubernetes?

Ensures exactly one copy of a pod runs on every node (or a selected subset) — the standard pattern for node-level agents like log collectors or monitoring exporters that need to be everywhere.

### What is Helm?

The package manager for Kubernetes — a chart bundles your manifests as templates plus a values file, so the same chart deploys consistently across dev/staging/prod just by swapping values. This is central to how I deploy — my Go app and my Jenkins-on-EKS knowledge both go through Helm charts, and my GitOps loop works by ArgoCD syncing a Helm values change.

### What is Istio?

A service mesh implementation — traffic management (routing, retries, fault injection), security (mTLS between services), and observability (automatic telemetry), all via sidecar proxies with no app code changes. Conceptual knowledge only on my end, no hands-on.

### What is Container Runtime Interface (CRI)?

The API contract between Kubernetes (kubelet) and whatever actually runs containers underneath — lets Kubernetes stay runtime-agnostic (containerd, CRI-O, etc.) instead of being hardwired to one implementation.

---

## DevOps Tools & Automation

### What is Infrastructure Automation?

Scripting the entire lifecycle of an environment — provisioning, configuration, deployment — so none of it depends on someone doing it by hand and getting it slightly different each time.

### What is GitOps?

Using a Git repository as the single source of truth for what should be running, with an automated agent continuously reconciling the live system to match Git — pull-based, not push-based. This is exactly my ArgoCD setup: I don't push deployments; ArgoCD watches the repo and pulls the desired state, then keeps enforcing it even if someone changes something manually.

### What is ArgoCD?

A GitOps continuous delivery tool for Kubernetes — it watches a Git repo, compares it to what's actually running in the cluster, and syncs any drift. I've proven self-heal live on this: deleted a deployment manually, and ArgoCD put it right back within seconds because Git still said it should exist.

### What is Tekton?

A Kubernetes-native CI/CD framework — pipelines are defined as Kubernetes custom resources and run as pods, so it's cloud-native and extensible by design. Conceptual knowledge, haven't run it myself — my CI is GitHub Actions.

### What are Deployment Strategies?

Rolling update — replace old pods with new ones gradually, some downtime risk if done carelessly but resource-cheap. Blue-Green — two full environments, switch traffic all at once, easy instant rollback, costs double the resources while both exist. Canary — send a small % of traffic to the new version first, watch metrics, then ramp up — lowest blast radius, more moving parts to set up. Recreate — kill everything old, then start new — simplest, has real downtime.

---

## Cloud Cost Optimization

### What is Cloud Cost Optimization?

Reducing cloud spend without hurting performance — right-sizing instances, killing unused resources, using auto-scaling instead of static overprovisioning, and picking the right pricing model (on-demand vs. reserved vs. spot) for each workload's actual usage pattern.

### What are Reserved Instances?

A commitment to a specific instance type for a 1- or 3-year term in exchange for a big discount versus on-demand — best for steady, predictable workloads. Convertible RIs trade some discount for flexibility to change instance family later; Spot Instances are the other end of the spectrum — deep discount, but the provider can reclaim the instance anytime, so only for interruption-tolerant workloads.

---

## Site Reliability Engineering (SRE)

### What is Site Reliability Engineering?

Applying software-engineering discipline to operations — using error budgets to balance reliability work against feature velocity, automating away repetitive manual work (toil), and treating reliability as a measurable, engineered property rather than a vague goal.

### What is Toil in SRE?

Manual, repetitive, automatable operational work that scales linearly with the size of the service and adds no lasting value — the stuff you should be scripting away, not doing by hand every time. SRE teams track and actively budget time to eliminate it.

_(SLA/SLO/SLI/Error Budget covered in full under "Advanced" below — same concept whichever section it's asked from.)_

---

## DevOps Metrics & KPIs

### What are DevOps Metrics?

The numbers that tell you if your delivery process is actually working — deployment frequency, lead time for changes, change failure rate, and MTTR are the four most commonly cited (the DORA metrics).

### What is Mean Time to Recovery (MTTR)?

Average time from a failure occurring to service being restored — `total recovery time ÷ number of incidents`. Breaks down into detection time (how fast you noticed), response time (how fast someone started working it), and resolution time (how fast it was actually fixed).

---

## Serverless

### What is Serverless Computing?

You deploy a function, the cloud provider handles provisioning, scaling, and patching the underlying servers entirely — you're billed per invocation/execution time, not for idle capacity. AWS Lambda is the standard example; good fit for event-driven, spiky, or infrequent workloads.

---

## Database in DevOps

### What is Database DevOps?

Applying the same DevOps discipline to database changes as to application code — version-controlled schema migrations, automated testing of migrations, and automated, repeatable deployment of schema changes instead of manual DBA scripts run by hand.

---

## Network Security

### What is Network Security in DevOps?

Security controls woven through the pipeline and infra — security groups/NACLs at the infra layer, TLS everywhere in transit, and auth/authz built into the application layer — so security isn't a separate afterthought bolted on post-deploy.

### What is Zero Trust Security?

"Never trust, always verify" — every request gets authenticated and authorized regardless of whether it originated inside or outside the network perimeter. In practice: strong identity verification, least-privilege access, and micro-segmentation instead of one big trusted internal network.

### What is SSL/TLS?

The cryptographic protocol securing data in transit between client and server — encrypts the payload and authenticates the server's identity via a certificate. This is what ACM issues and what an ALB HTTPS listener terminates.

### What is a Web Application Firewall (WAF)?

Sits in front of a web app and inspects incoming HTTP traffic for known attack patterns — SQL injection, XSS, bad bot traffic — blocking malicious requests before they reach the app. AWS WAF integrates directly in front of an ALB or CloudFront.

### What is Network Segmentation?

Dividing a network into isolated zones so a breach in one segment doesn't automatically give an attacker access to everything — public subnet for load balancers, private subnet for app/DB tiers, is the classic AWS VPC version of this.

---

## Incident Management

### What is Incident Management?

The structured process of detecting, responding to, and resolving service disruptions — detect (monitoring/alerts), respond (acknowledge, assess severity, notify), resolve (find root cause, fix, verify), then usually a postmortem to prevent recurrence.

---

## DevOps Culture

### What is DevOps Culture?

Shared ownership between Dev and Ops instead of a "you build it, we run it and blame you when it breaks" divide — collaboration, blameless postmortems, and continuous improvement over rigid process for its own sake.

### What are DevOps Best Practices?

Technical: IaC, CI, automated testing, continuous deployment, solid monitoring. Cultural: shared responsibility, blameless postmortems, continuous learning. Process: version control everything, proper release management, structured incident response. The common thread across all three is reducing manual, error-prone human steps.

---

## Infra Monitoring & Observability Basics

### What is Infrastructure Monitoring?

Tracking the health of the infra itself — CPU, memory, disk, network — separate from application-level metrics, so you know whether a problem is "the box is out of resources" versus "the app has a bug."

### What are Monitoring Tools?

Infra-focused: Prometheus, Nagios, Zabbix, Datadog. Application-focused (APM): New Relic, AppDynamics, Dynatrace — these add transaction tracing and error tracking on top of raw metrics. I run Prometheus + Grafana for JS7 job monitoring.

### What are Monitoring Best Practices?

Alert on symptoms that actually affect users, not on every metric that moves — alert fatigue kills responsiveness. Set thresholds based on real historical baselines, not guesses. Make dashboards for humans to scan quickly during an incident, not just data dumps. And treat monitoring config itself as code, versioned like everything else.

### What is Application Performance Monitoring?

APM specifically instruments the application layer — transaction traces, error rates by endpoint, dependency call latency — to answer "why is this specific request slow" rather than just "is the server healthy."

### What is Log Management?

Centralizing, indexing, and making logs searchable across every service, so debugging doesn't mean SSHing into ten different boxes to `grep` locally. Comes with retention, aggregation, and usually a search UI like Kibana or CloudWatch Logs Insights.

---

## Cloud Migration

### What is Cloud Migration?

Moving apps, data, and infra from on-prem (or one cloud) to another cloud environment — involves assessment, a migration strategy, execution, and validation before cutover.

### What are Cloud Migration Strategies?

The 6 R's, most commonly cited three: Rehost (lift-and-shift, fastest, least optimized), Replatform (lift-tinker-shift, small cloud-specific improvements without a rearchitect), Refactor/Re-architect (rebuild for cloud-native, most effort, best long-term payoff).

### What is Cloud Assessment?

Evaluating whether and how a workload should move to the cloud — cost-benefit analysis, security/compliance fit, performance requirements — before committing to a migration strategy.

### What is Application Modernization?

Transforming an existing app to actually use cloud-native patterns — breaking a monolith into services, containerizing it, adopting managed services — instead of just relocating the same architecture onto cloud VMs.

### What are Cloud Migration Tools?

Database migration tools (AWS DMS-style), application packaging/containerization tools, and migration orchestration tools that sequence and automate the whole cutover process.

---

## Advanced DevOps & Cloud

### What is Platform Engineering?

Building an Internal Developer Platform — a self-service layer so app teams can provision infra, spin up CI/CD, and deploy without needing deep platform expertise themselves. "Golden paths" for common tasks, standardized and compliant by default, treated like a product with developers as its users.

### What is FinOps?

Cloud financial management as a cross-team discipline — engineering, finance, and product collaborating on cost visibility and decisions instead of cost being a surprise finance discovers at month-end. The lifecycle is Inform (visibility/tagging) → Optimize (right-sizing, RIs/spot, killing waste) → Operate (budgets, alerts, ongoing review).

### What is Policy as Code?

Writing infrastructure/security policies as code — checked into Git, versioned, automatically enforced in CI/CD or at admission time — instead of manually configuring rules through a UI. Open Policy Agent and HashiCorp Sentinel are the common engines; typical use: blocking a Kubernetes deployment that pulls an image from an untrusted registry.

### What is Chaos Engineering?

Deliberately injecting failure into a system — kill an instance, add network latency, take down a dependency — to verify it actually degrades gracefully instead of assuming it does. You define expected "steady state" behavior first, form a hypothesis, run the experiment (ideally in production, starting small), then fix what you find. Chaos Monkey, Gremlin, and AWS Fault Injection Simulator are the common tools.

### What is Blue/Green Deployment?

Two identical environments — only one takes live traffic at a time. Deploy the new version to the idle one, test it fully, then switch the load balancer/router to point at it all at once. Rollback is just switching back. Costs double the infra while both environments exist, and DB schema compatibility across both versions needs planning.

### What is Feature Flagging?

Wrapping a new code path in a runtime-toggleable condition so a feature can be turned on/off — or shown to a subset of users — without a new deployment. Decouples "code is deployed" from "feature is live," which is what makes safe canary rollouts and fast kill-switches possible.

### What is a Service Catalog?

A self-service portal listing the standardized, pre-approved services and templates developers can request — a new microservice template, a managed database, a test environment — so they don't wait on a manual ticket to IT ops for routine things. Backstage is the common open-source example.

### What is a Service Level Agreement (SLA)?

The external, formal commitment to customers — e.g., "99.9% uptime" — usually with financial consequences (service credits) if missed.

### What is a Service Level Objective (SLO)?

The internal target a team sets for itself, deliberately stricter than the SLA, to keep a safety margin. E.g., SLA says 99.9% to the customer; internal SLO might be 99.95% so there's buffer before you'd actually breach the customer-facing commitment.

### What is a Service Level Indicator (SLI)?

The actual measured metric — e.g., "percentage of requests returning 2xx" — that tells you where you stand against the SLO. SLA is the promise, SLO is the internal target, SLI is the real number you're tracking against both.

### What is Error Budget?

The allowed amount of unreliability, derived directly from the SLO — a 99.9% SLO over 30 days gives you about 43 minutes of allowed downtime. Burn through it too fast, and the team pauses new feature work to focus on reliability instead; if you're well within budget, that's the green light to ship faster.

### What is a Runbook?

A documented, step-by-step procedure for a specific operational task or known failure mode — trigger condition, diagnosis steps, remediation steps, verification, escalation path. The end goal is usually automating the runbook away entirely once it's proven and repeatable.

### What is a Playbook in Incident Response?

Like a runbook, but specifically for security incidents or major outages — includes roles (who's Incident Commander, who handles comms), containment/eradication/recovery steps, and a mandatory postmortem afterward. More process-and-people focused than a routine ops runbook.

### What is Observability?

The ability to understand _why_ a system is behaving a certain way just from its external outputs — logs, metrics, traces — even for failure modes you never explicitly built a dashboard for. Monitoring answers questions you already knew to ask ("is CPU high"); observability lets you investigate questions you didn't anticipate.

### What is Tracing in Observability?

Following a single request's full path through a distributed system, span by span, across every service it touches — the tool for finding exactly which hop in a multi-service call chain is actually slow. Jaeger and Zipkin are the standard tools.

### What is a Sidecar Pattern?

Running a helper container alongside your main app container in the same pod, sharing network and storage — used for things like log shipping, metrics export, or a service mesh proxy — without baking that functionality into the app itself.

### What is a Service Mesh Control Plane?

The management layer that configures and monitors the sidecar proxies (the data plane) — pushes routing rules, distributes mTLS certificates, and defines policy, without touching actual request traffic itself. `istiod` in Istio is the concrete example.

### What is GitHub Actions?

GitHub's built-in CI/CD and automation platform — workflows defined as YAML in the repo, triggered by events like push or PR, running in hosted or self-hosted runners. This is my primary CI tool — it's what builds, tests, and pushes my Docker images before ArgoCD takes over for deployment.

### What is a Self-Healing System?

A system that detects and automatically corrects failures without a human stepping in — Kubernetes restarting a crashed pod, or ArgoCD reconciling a manually deleted deployment back to match Git, are both self-healing in practice, not just theory, for me.

### What is Canary Analysis?

Automatically evaluating the health metrics of a canary release — error rate, latency — against the baseline before deciding whether to promote it to full rollout or auto-rollback. The "analysis" part specifically means it's a data-driven gate, not just "wait an hour and hope."

### What is Infrastructure Drift?

When the real state of infrastructure no longer matches what's defined in code — usually from a manual console change someone made "just this once." Terraform's `plan` surfaces drift by diffing state against reality; in a GitOps setup like mine, ArgoCD actively corrects drift by re-syncing to Git automatically instead of just reporting it.

---

_That's the full concept bank. Next: send me real interview questions as they come up, or another batch of rough material and I'll do the same treatment._

<!-- related -->
---
**Related:** [[4.9-Docker-Interview-QA|Docker — Interview Q&A]]
<!-- /related -->
