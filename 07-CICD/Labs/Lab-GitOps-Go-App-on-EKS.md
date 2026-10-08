---
title: GitOps Go App — End-to-End Pipeline on AWS EKS
aliases:
  - GitOps Go App — End-to-End Pipeline on EKS
type: project
module: cicd
status: ✅ Completed — live and verified
interview-relevance: "🔴 High — GitOps, ArgoCD, and EKS come up constantly in mid/senior DevOps interviews"
tags:
  - kubernetes
  - gitops
  - argocd
  - github-actions
  - eks
  - cert-manager
  - docker
  - helm
  - project
  - cicd
repo: "github.com/aakash-1004/gitops-go-app (built clean, not forked) **Live demo:** https://gitops-go-app.duckdns.org"
up: "[[CICD-MOC]]"
prev: "[[Lab-Taskmanager-CICD]]"
next: "[[Lab-GoWebApp-Full-GitOps-Pipeline]]"
related:
  - "[[7.3.2-ArgoCD-Application-Manifest-and-Commands]]"
  - "[[7.3-ArgoCD-and-GitOps]]"
  - "[[Lab-EKS-IRSA-Workload-Identity]]"
origin: DevOps-Mastery/07-CICD/Labs/GitOps Go App — End-to-End Pipeline on EKS.md
---

# GitOps Go App — End-to-End Pipeline on AWS EKS

<!-- nav -->
↑ [[CICD-MOC|CI/CD]] · ← [[Lab-Taskmanager-CICD|Lab — CI/CD Pipeline for Taskmanager]] · → [[Lab-GoWebApp-Full-GitOps-Pipeline|Go Web App — Full Containerization to GitOps CI/CD Pipeline]]
<!-- /nav -->

## Overview

Built a Go web app from scratch and deployed it to AWS EKS through a full GitOps pipeline. End state: push code → GitHub Actions builds/tags/pushes the image and bumps the Helm chart → ArgoCD detects the Git change and syncs it to the cluster → live, with HTTPS, with zero manual `kubectl`/`helm` commands after the initial push.

**Full stack:** Go → Docker (multi-stage, distroless) → Helm → AWS EKS → ingress-nginx (AWS NLB) → ArgoCD (GitOps) → GitHub Actions (CI) → cert-manager (HTTPS)

---

## 1. Multi-Stage Distroless Docker Build

```dockerfile
# Stage 1 — build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY app/go.mod ./
RUN go mod download
COPY app/ .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# Stage 2 — distroless final image
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /app/server .
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

|Line|What it does|
|---|---|
|`FROM golang:1.22-alpine AS builder`|Names this stage `builder` — a full Go toolchain, thrown away after build|
|`CGO_ENABLED=0`|Disables CGo (C library linking) — binary has zero external library dependencies|
|`GOOS=linux`|Cross-compiles for Linux explicitly, regardless of build host OS|
|`FROM gcr.io/distroless/static:nonroot`|Final image — no shell, no package manager, nothing but the binary|
|`COPY --from=builder`|Pulls only the compiled binary from Stage 1 — none of the Go toolchain ships in the final image|
|`USER nonroot:nonroot`|Container runs as an unprivileged user, not root|

**Why it matters:** A regular `golang:1.22` image is 800MB+ and contains a shell, package manager, and full OS — all attack surface if the container is ever compromised. The final distroless image here is a few MB and gives an attacker nothing to work with even with container access — no `sh`, no `apt`, no `curl`.

**Real-world use:** This is the standard pattern for any compiled-language production image (Go, Rust, C++). Interpreted languages (Python, Node) can't go fully distroless the same way since they need a runtime present, but the multi-stage _separation_ principle still applies (separate build deps from runtime deps).

---

## 2. Kubernetes `type: LoadBalancer` → Real AWS Infrastructure

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer
```

**The chain this triggers:**

```
Service type: LoadBalancer created
        |
        v
EKS's built-in AWS Cloud Controller Manager notices it
        |
        v
Calls the AWS API directly (uses IAM permissions baked into the node role)
        |
        v
Provisions a real AWS NLB, attaches worker nodes as targets
        |
        v
Writes the NLB's public DNS hostname into the Service's EXTERNAL-IP field
```

**Key concept:** `type: LoadBalancer` is a **Kubernetes-native abstraction**, not an AWS-specific feature. Same YAML field means something different depending on the cloud:

- AWS EKS → NLB or Classic ELB
- GCP GKE → Google Cloud Load Balancer
- Azure AKS → Azure Load Balancer
- Bare-metal / k3s with no cloud provider → stays `<pending>` forever — nothing to call

**Real-world use:** This is exactly why the home k3s setup uses Traefik + NodePort instead — there's no cloud API underneath it to fulfill a LoadBalancer request. On EKS, the "magic" isn't magic — it's a controller that's just watching and calling an API.

---

## 3. ArgoCD — GitOps Reconciliation Loop

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-go-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/aakash-1004/gitops-go-app.git
    targetRevision: main
    path: helm/gitops-go-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

|Field|What it does|
|---|---|
|`repoURL` + `path`|Exact location of the Helm chart ArgoCD should watch|
|`targetRevision: main`|Which branch to track|
|`destination.server`|`https://kubernetes.default.svc` = deploy to the same cluster ArgoCD runs in|
|`automated.selfHeal: true`|If cluster state drifts from Git (e.g. manual `kubectl edit`), ArgoCD reverts it|
|`automated.prune: true`|If something is removed from the Helm chart, ArgoCD deletes it from the cluster too|

**Why it matters:** Without `syncPolicy.automated`, ArgoCD still shows drift in the UI but waits for a human to click "Sync." Automated sync + selfHeal is what makes this _true_ GitOps rather than just "Git-based deployment with a manual trigger."

**Real-world use / proof:** Tested this directly —

```bash
kubectl scale deployment gitops-go-app --replicas=5
```

Result: 3 new pods spun up, but ArgoCD's reconciliation loop caught the drift from the Git-declared `replicaCount: 2` fast enough to kill the extra pods **while they were still `ContainerCreating`** — before they ever became `Ready`. Confirmed final state back to exactly 2 pods, `Synced + Healthy`.

---

## 4. GitHub Actions — Closing the CI Loop

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'app/**'
      - 'Dockerfile'

jobs:
  build-and-push:
    steps:
      - uses: actions/checkout@v4
      - id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: aakash0908/gitops-go-app:${{ steps.vars.outputs.sha_short }}
      - run: sed -i "s|tag: .*|tag: ${{ steps.vars.outputs.sha_short }}|" helm/gitops-go-app/values.yaml
      - run: |
          git config user.name "github-actions[bot]"
          git commit -am "ci: bump image tag" || echo "No changes"
          git push
```

|Design choice|Why|
|---|---|
|`paths: ['app/**', 'Dockerfile']`|Scopes the trigger so the workflow's own commit (to `values.yaml`) doesn't re-trigger itself — prevents an infinite CI loop|
|`git rev-parse --short HEAD` for the tag|Every image is traceable to an exact commit. A static `latest` tag never changes, so `values.yaml` never gets a real diff, so ArgoCD has nothing new to sync even after a fresh image push|
|Commit back to `values.yaml` in-pipeline|This is the actual handoff point between CI and CD — GitHub Actions' job ends the moment this commit lands; ArgoCD picks it up independently|
|Repo `workflow` scope on PAT|GitHub requires the `workflow` PAT scope specifically to push changes to `.github/workflows/*` — a security guard since workflow files can execute with repo secrets|

**Real-world use / proof:** Edited `main.go`, pushed, watched Actions build, tag as `c64c804`, push to Docker Hub, bump `values.yaml`, and commit — all in under a minute — then ArgoCD synced it and the new text was live in the browser with zero manual steps.

---

## 5. cert-manager — Kubernetes-Native HTTPS

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: aakashrao0908@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

```yaml
# Ingress annotation that wires it all together:
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [gitops-go-app.duckdns.org]
      secretName: gitops-go-app-tls
```

**cert-manager vs certbot — the actual difference:**

||certbot|cert-manager|
|---|---|---|
|Execution model|Runs on a single server, spins up a temp webserver or writes to a webroot|Runs as a controller inside the cluster|
|Renewal|Needs a cron job|Automatic — the controller runs continuously|
|Fits this architecture?|No single server is in the traffic path here|Yes — designed for exactly this (Ingress-based routing, pods that move)|

**The chain reaction:**

```
Ingress has cert-manager annotation
        |
        v
cert-manager sees tls.secretName doesn't exist yet
        |
        v
Creates a Certificate object, triggers ACME HTTP-01 challenge
        |
        v
Challenge solved through the EXISTING ingress-nginx controller
        |
        v
Let's Encrypt issues the cert, stored in the named Secret
        |
        v
ingress-nginx picks it up, starts serving HTTPS
```

**Real-world use / proof:** `kubectl get certificate` showed `READY: True` within about 2 minutes of the Ingress change syncing. Cert valid Aug 21 to Nov 19 2026, with **auto-renewal scheduled for Oct 20** — 30 days before expiry, entirely hands-off.

---

## 6. Topology Spread Constraints — Real Pod-Level HA

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: gitops-go-app
```

|Field|What it does|
|---|---|
|`maxSkew: 1`|Max allowed difference in pod count between any two nodes — with 2 replicas, this forces 1-per-node|
|`topologyKey: kubernetes.io/hostname`|Defines "spread across" as "spread across nodes" (vs. `topology.kubernetes.io/zone` for multi-AZ spread)|
|`whenUnsatisfiable: DoNotSchedule`|Hard constraint — refuses to schedule rather than silently violating the spread (soft alternative: `ScheduleAnyway`)|

**Why this was needed — a real gap found during a full-system audit:** Kubernetes' default scheduler does not spread replicas across nodes on its own. A `kubectl get pods -o wide` audit showed both replicas had landed on the same node — meaning 2 replicas gave zero actual protection against a single node failure. This constraint fixed that: verified afterward that each replica lands on a different node.

**Interview point:** "Replica count is not the same thing as high availability" unless you also control placement. This is a subtle, easy-to-miss gap worth naming directly in an interview — it shows you check assumptions rather than just reading dashboard numbers.

---

## Engineering decisions & bugs fixed (chronological)

1. **Fork vs. clean repo** — originally built on a forked tutorial repo. A visible "forked from" banner undersells real engineering work even when 100% of the pipeline is original. Deleted the fork, rebuilt fresh with clean commit history in `gitops-go-app`.
2. **`/health` endpoint designed in from day one** — direct lesson from a past project where probes pointed at `/`, which 404'd, causing crash-loops. Built the correct probe target into the app from the start this time instead of fixing it after the fact.
3. **CI self-trigger loop** — solved via `paths:` filter on the workflow trigger, scoping it away from the file the pipeline itself writes to.
4. **`latest` tag would have broken the GitOps loop entirely** — switched to commit-hash tagging specifically because a static tag produces no Git diff, so ArgoCD would never see a reason to sync a "new" image.
5. **DuckDNS only accepts IPs, not hostnames** — resolved the NLB hostname via `nslookup` and pointed DuckDNS at the raw IP. Known limitation: NLB IPs can rotate, so this would need Route53 alias records for a production setup (skipped here — not worth ~$0.50/mo for a portfolio project).
6. **Silent single-node pod placement** — found via full-system audit (`kubectl get pods -o wide` across everything), fixed with topology spread constraints.

---

## Interview-Ready Spoken Answer

"I built a GitOps pipeline for a Go app on AWS EKS. A code push triggers GitHub Actions, which builds the image, tags it by commit hash for traceability, pushes it to Docker Hub, and bumps the tag in my Helm chart's values file. ArgoCD watches that repo and auto-syncs the change, so there's no manual kubectl or helm step after the initial push. I proved the self-healing side directly — I manually scaled the deployment up, and ArgoCD reverted it within seconds, actually killing the extra pods before they even finished starting. During a full audit I also noticed both replicas had landed on the same node by default, so I added topology spread constraints to force real node-level redundancy. And I wired up automatic HTTPS with cert-manager and Let's Encrypt — same underlying idea as certbot, but built to work with Kubernetes Ingress instead of a single server, so it renews itself with zero manual intervention."

---

## Related notes

- [[7.3-ArgoCD-and-GitOps]]
- Kubernetes - Scheduling and Topology Spread
- [[Lab-GoWebApp-Full-GitOps-Pipeline|Docker - Multi-stage Builds]]
- [[7.1-GitHub-Actions|GitHub Actions]]
- cert-manager and TLS
- EKS - Cloud Controller Manager
- [[5.4-Helm|Helm Charts]]
- [[5.5-EKS-Fargate-IRSA-ALB-Project|OIDC and Trust Chains]]

<!-- related -->
---
**Related:** [[7.3.2-ArgoCD-Application-Manifest-and-Commands|ArgoCD — Application Manifest, Commands & Interview Answers]] · [[7.3-ArgoCD-and-GitOps|CI/CD: ArgoCD & GitOps]] · [[Lab-EKS-IRSA-Workload-Identity|AWS EKS + IRSA — Secure Workload Identity]]
<!-- /related -->
