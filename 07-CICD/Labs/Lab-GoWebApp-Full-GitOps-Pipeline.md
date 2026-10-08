---
title: Go Web App — Full Containerization to GitOps CI/CD Pipeline
aliases:
  - GoWebApp-Full-GitOps-Pipeline
type: project
module: cicd
status: completed
interview-relevance: "5/5"
tags:
  - docker
  - kubernetes
  - eks
  - helm
  - argocd
  - gitops
  - github-actions
  - cicd
  - devops
  - interview-ready
  - project
date: 2026-07-26 to 2026-07-28
days: Day 23-26
up: "[[CICD-MOC]]"
prev: "[[Lab-GitOps-Go-App-on-EKS]]"
next: "[[Lab-EKS-IRSA-Workload-Identity]]"
related:
  - "[[7.3.2-ArgoCD-Application-Manifest-and-Commands]]"
  - "[[7.3.1-Helm-Chart-Anatomy-and-Commands]]"
  - "[[7.3-ArgoCD-and-GitOps]]"
  - "[[5.4-Helm]]"
  - "[[5.5-EKS-Fargate-IRSA-ALB-Project]]"
origin: DevOps-Mastery/07-CICD/Labs/GoWebApp-Full-GitOps-Pipeline.md.md
---

# Go Web App — Full Containerization to GitOps CI/CD Pipeline

<!-- nav -->
↑ [[CICD-MOC|CI/CD]] · ← [[Lab-GitOps-Go-App-on-EKS|GitOps Go App — End-to-End Pipeline on AWS EKS]] · → [[Lab-EKS-IRSA-Workload-Identity|AWS EKS + IRSA — Secure Workload Identity]]
<!-- /nav -->

## Project Summary (one-breath version)

Took a simple Go web app from source code to a fully automated GitOps deployment: containerized with a multi-stage distroless Docker build, deployed on two different Kubernetes environments (local k3s with Traefik, then AWS EKS with nginx Ingress), packaged as a Helm chart, wired up ArgoCD for GitOps-based continuous delivery, and closed the loop with a GitHub Actions CI/CD pipeline that tests, builds, pushes, and triggers automatic deployment on every commit — zero manual kubectl/helm commands needed after the initial setup.

---

## PART 1 — Docker: Multi-Stage Distroless Build

### Theory (short)

Multi-stage builds use two (or more) `FROM` statements in one Dockerfile: an early stage compiles the app with a full toolchain, a final stage copies only the compiled artifact into a minimal runtime image. Go's static binary compilation (no runtime dependencies) makes this especially clean — the final image needs nothing but the binary itself.

### Commands

```dockerfile
# Stage 1: Build
FROM golang:1.22.5-alpine AS base
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN go build -o main .

# Stage 2: Distroless runtime
FROM gcr.io/distroless/base
COPY --from=base /app/main .
COPY --from=base /app/static ./static
EXPOSE 8080
CMD ["./main"]
```

**Why `COPY go.mod` before `COPY . .`:** Docker layer caching — dependencies rarely change vs. source code; separating them means code-only changes don't force a full dependency re-download.

**Why `COPY --from=base`:** reaches into stage 1's _already-built_ filesystem and cherry-picks only the binary + static assets. Everything else (compiler, module cache, source) never enters the final image — this is the actual mechanism behind multi-stage size reduction, not compression or magic.

**Why distroless specifically (vs Alpine):** distroless ships **no shell, no package manager, no OS utilities** — just enough runtime to execute one binary. Massive attack-surface reduction: even with code execution inside the container, an attacker has no shell to pivot with. **Trade-off:** cannot `docker exec -it <container> sh` — no shell exists. Debugging relies on `docker logs` and external observability, not live shell access.

**Result:** final image ~13.7MB vs ~55MB for equivalent Python/Flask-based project images in the same portfolio — roughly 4x smaller.

### Real-world context

Distroless is what security-conscious companies use for production images specifically to minimize CVE exposure and attacker tooling. The trade-off (no shell) is a deliberate, known limitation — teams compensate with strong logging/observability rather than live debugging.

### Interview-ready answer

_"I used a multi-stage Docker build — compile in a full Go toolchain image, then copy just the binary into a distroless final stage. This got the image down to ~14MB and removed the shell entirely, so even if the app were compromised, there's no shell for an attacker to use."_

### Gaps identified (real, unresolved — good to mention proactively)

- No graceful shutdown handling in `main.go` (no SIGTERM handling) — pods would be hard-killed mid-request during rolling updates/scale-downs
- No `/healthz` endpoint — health probes had to point at an actual content page (`/home`), not ideal semantically
- App has zero request logging — combined with distroless's no-shell design, this means **zero runtime visibility** if something goes wrong in production

---

## PART 2 — Kubernetes Core Objects (Written From Scratch)

### Theory (short)

Every K8s manifest follows the same shape: **apiVersion → kind → metadata → spec** ("AKMS"). Three core objects were used: Deployment (manages pods), Service (stable network identity for ephemeral pods), Ingress (HTTP-aware routing, requires a controller to do anything).

### Deployment — mnemonic "RST" (Replicas, Selector, Template)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: go-web-app
  template:
    metadata:
      labels:
        app: go-web-app    # MUST exactly match selector.matchLabels
    spec:
      containers:
        - name: go-web-app
          image: aakash0908/go-web-app:v1
          ports:
            - containerPort: 8080
```

**Critical rule:** `selector.matchLabels` and `template.metadata.labels` must be identical, or Kubernetes rejects the manifest outright. Write the label once, copy-paste both places — don't retype.

### Service — mnemonic "STP" (Selector, Type, Ports) — the 3-port relay race

```yaml
apiVersion: v1
kind: Service
metadata:
  name: go-web-app-service
spec:
  type: ClusterIP
  selector:
    app: go-web-app       # matches POD labels, never the Deployment's name
  ports:
    - port: 80             # Service's own front-door port
      targetPort: 8080      # actual container port — must match app's real listen port
```

**Mental model:** traffic enters at `nodePort` (if NodePort type) → hits `port` (Service identity) → forwarded to `targetPort` (container). Three hops, three port numbers, easy to confuse.

### Ingress — mnemonic "CRB" (Class, Rules, Backend)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-web-app-ingress
spec:
  ingressClassName: nginx    # or traefik — must match the actual running controller
  rules:
    - host: go-web-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: go-web-app-service   # must match actual Service name exactly
                port:
                  number: 80
```

### Real bugs hit and fixed while learning this (all genuinely instructive)

1. **YAML indentation mismatch** — sibling keys must share identical indentation; off-by-one-space breaks parsing entirely (`did not find expected key` errors)
2. **Missing space after list-item dash** — `-containerPort: 8080` parses as one malformed token; must be `- containerPort: 8080`. Single most common YAML bug across all K8s object types.
3. **Case sensitivity** — `ClusterIp` vs `ClusterIP`, `matchlabels` vs `matchLabels` — Kubernetes YAML fields are case-sensitive, no fuzzy matching
4. **Wrong field name** — `-hosts:` should be `host:` (singular)
5. **Real logic bug, not syntax** — Ingress backend referenced Service name `go-web-app` when actual Service was named `go-web-app-service`. Kubernetes doesn't error on this — it just silently never routes correctly. No crash, no error, just doesn't work.

**Validation habit established:** `kubectl apply --dry-run=client -f file.yaml` catches schema/syntax errors before touching the cluster — became a standing pre-flight check.

### Interview-ready answer

_"I hand-wrote Deployment, Service, and Ingress manifests and hit real bugs along the way — indentation errors, case-sensitivity issues, and one logic bug where my Ingress pointed at the wrong Service name. That last one is the sneaky kind: Kubernetes doesn't error, it just silently fails to route. It taught me to always verify with a real curl test, not just `kubectl get` showing objects as 'created'."_

---

## PART 3 — Ingress / Ingress Controller / IngressClass — Proven on Two Different Controllers

### Theory (short)

Three distinct concepts, commonly conflated:

1. **Ingress** — just a YAML declaration sitting in etcd. Does nothing by itself.
2. **Ingress Controller** — real running pods (Traefik, nginx, AWS ALB Controller) that watch for Ingress objects and physically implement the routing.
3. **IngressClass** — the pairing mechanism (`ingressClassName: nginx`) telling Kubernetes which controller should handle a given Ingress, when multiple might exist.

### Proven twice, two different controllers

- **k3s + Traefik**: ships built-in by default. Address populated as the node's own IP (Traefik binds to host network directly — no separate cloud load balancer object created).
- **EKS + ingress-nginx** (installed via Helm): EKS ships with _zero_ Ingress controller by default. Installing `ingress-nginx` created a `LoadBalancer`-type Service, which triggered EKS's cloud-controller-manager to automatically provision a real AWS **Network Load Balancer (NLB)**.

### Load Balancer vs Ingress — not interchangeable

- **Load Balancer** = Layer 4 (TCP/UDP), no understanding of HTTP paths/hostnames, purely spreads traffic across targets
- **Ingress** = Layer 7 (HTTP-aware), understands hostnames/paths, can route to different backend Services based on rules

### Testing quirk (host-based routing without real DNS)

```bash
curl -H "Host: go-web-app.local" http://<real-LB-address>/home
```

Since there's no real DNS entry, the `Host` header is forged manually while genuinely connecting to the real load balancer address. **Analogy: the LB address is the building's real street address; the Host header is the tenant name on the package** — same building, different recipient until they match.

Alternative for browser-based testing: add an `/etc/hosts` entry mapping the hostname to the LB's resolved IP.

### Interview-ready answer

_"Ingress is just a declaration — it does nothing without a controller actually watching for it and implementing the routing. I proved this twice: on k3s with Traefik (built-in, binds directly to the node), and on EKS with nginx (installed via Helm, which triggered AWS to provision a real Network Load Balancer automatically). The controller is the actual router; Ingress and IngressClass are just how you tell Kubernetes which router handles which rules."_

---

## PART 4 — AWS EKS Specifics

### Control plane vs worker nodes — the fundamental split

- **Control plane** (API server, etcd, scheduler): fully AWS-managed, invisible, in AWS's own infrastructure — never provisioned, patched, or scaled by the user
- **Worker nodes**: plain EC2 instances in _your_ account/VPC, provisioned via `eksctl create nodegroup` (or Fargate profiles for serverless compute)

**Contrast with k3s:** k3s runs control plane + worker on a single node — appropriate for single-machine/edge use cases, not the HA model EKS targets.

### Fargate vs EC2 Managed Nodegroup — compute model trade-offs

||Fargate|EC2 Nodegroup|
|---|---|---|
|Isolation|1 pod = 1 micro-VM|Many pods share a node|
|Cost model|Per-pod, per-second|Per-node, pack many pods|
|Startup|Slower (fresh micro-VM per pod)|Fast (existing node capacity)|
|Best for|Spiky/unpredictable workloads|Steady-state workloads|
|Ops overhead|Zero node management|AWS-managed patching (with "managed" nodegroups)|

### Real constraint hit: Free Tier instance-type restriction

Attempting `t3.medium` on a Free Tier account failed outright:

```
InvalidParameterCombination - The specified instance type is not eligible for Free Tier
```

Free Tier accounts are hard-restricted to specific instance types (`t2.micro`/`t3.micro`, and — verified via AWS's own instance-type search — `t3.small` in this account's case).

### Real constraint hit: Pod-density-per-node is an ENI/networking limit, not just CPU/RAM

**The critical EKS-specific gotcha:** the AWS VPC CNI assigns each pod its own ENI-attached IP — pods aren't just processes sharing a node's IP, they consume real VPC IP address slots. **Max pods per node is capped by ENI + secondary-IP capacity for that specific instance type**, following an AWS-published formula — completely independent of whether the node has spare CPU/memory.

- `t3.micro`: max ~4 pods per node (2 ENIs × 2 IPs) — already consumed by baseline system pods (`aws-node`, `kube-proxy`, `coredns`), leaving near-zero headroom for anything else
- `t3.small`: significantly higher ceiling — resolved the issue

**Symptom when hit:** `FailedScheduling` event: _"Too many pods"_ — distinct from memory-pressure errors, easy to misdiagnose as a resource problem when it's actually a networking/IP-allocation ceiling.

### Interview-ready answer

_"EKS control plane is fully AWS-managed — I only provision worker capacity, either via EC2 managed nodegroups or Fargate. One non-obvious gotcha I hit firsthand: max pods per node on EKS is capped by the VPC CNI's ENI/IP allocation for that instance type, not just CPU or memory. A t3.micro only supports about 4 pods total per node — I hit a real 'Too many pods' scheduling failure that had nothing to do with resource limits and everything to do with networking capacity."_

---

## PART 5 — Helm Chart Conversion

### Theory (short)

Helm charts parameterize Kubernetes manifests via templating (`{{ .Values.x }}`) against a `values.yaml` file, rendered with `helm template`/`helm install`. `helm create <name>` scaffolds a full, production-tested chart structure — the correct starting point, not hand-written files.

### Real bugs hit — the scaffold isn't automatically correct for your app

1. **Default liveness/readiness probes point at `/`** — this app returns 404 on root (no route registered there). Left as default, Kubernetes would treat every probe response as "unhealthy" and **crash-loop the pods indefinitely**, even though the app works fine. Fixed by pointing probes at `/home` instead.
2. **`containerPort` templated from `.Values.service.port` (80) instead of the actual app listen port (8080).** The default scaffold assumes container port = Service port, true for many simple apps (like nginx, which the scaffold is modeled on) but not this one. Silent failure mode: Service would exist, pods would run, but traffic would never reach the app — connection refused.

### The stray-file gotcha (genuinely subtle, real production lesson)

**Helm renders every single file inside `templates/`, regardless of filename or extension.** A leftover nano backup file (`deployment.yaml.save`) containing the _pre-fix_ version of the Deployment template was silently rendered **alongside** the corrected `deployment.yaml` — producing two competing Deployment definitions in the same release, both trying to own the same object name.

**How it manifested:** a confusing `Duplicate value: "http"` error on `ports[1].name` — traced back to Kubernetes' **strategic merge patch** behavior: array merges for container ports are keyed by `containerPort` (not `name`). Since the old file said port 80 and the new one said port 8080, Kubernetes treated them as two _different_ ports to add (not one to replace) — both named `http` — hence the naming collision on validation.

**Fix:** delete stray files from `templates/`; `rm deployment.yaml.save`. **Lesson:** keep `templates/` directories completely clean of anything not meant to be rendered — editor backups, `.bak` files, anything.

### Idempotent install pattern

```bash
helm upgrade --install go-web-app .
```

**Why this over plain `helm install`:** handles both "fresh install" and "fix/update an existing release" in one command — the actual pattern used in CI/CD automation, since a pipeline shouldn't need to know in advance whether a release already exists.

### Interview-ready answer

_"Converting to Helm surfaced two real bugs the default scaffold introduced for this specific app — wrong probe path and wrong container port — both would've caused silent or crash-looping failures in production. The trickiest issue was a stale editor backup file sitting in the templates folder: Helm renders every file there regardless of name, so it was silently applying an old, buggy version of my Deployment alongside the fixed one, causing a confusing duplicate-port error. It taught me that Helm's template directory needs to be as clean as source code — no stray files, ever."_

---

## PART 6 — ArgoCD & GitOps

### Theory (short)

GitOps principle: **Git is the single source of truth for cluster state.** ArgoCD runs _inside_ the cluster, continuously polling a Git repo path, diffing declared state (in Git) against live cluster state, and reconciling any drift — either by installing missing resources or correcting manual changes back to match Git.

### ArgoCD Application manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/aakash-1004/go-web-app.git
    targetRevision: main
    path: go-web-app
  destination:
    server: https://kubernetes.default.svc   # magic constant = "this same cluster"
    namespace: default
  syncPolicy:
    automated:
      prune: true      # delete cluster objects removed from Git
      selfHeal: true   # revert manual drift back to match Git
```

### Self-healing — proven live, not just theoretical

Manually ran `kubectl scale deployment go-web-app --replicas=5` (bypassing Git entirely). ArgoCD detected the drift (live state ≠ Git's declared `replicaCount: 2`) and automatically scaled back down to 2 — no human intervention, no re-applying anything. **Concrete, demoable proof of GitOps self-healing.**

### Key realization: ArgoCD's operation is fully decoupled from any viewing client

`kubectl port-forward` (used to view the ArgoCD UI) is purely a local viewing tunnel — when it drops (VM network flakiness), **ArgoCD's actual sync/reconciliation continues uninterrupted in AWS**, completely unaffected. The UI is just a window into a process that runs independently; losing the window doesn't stop the process.

### Interview-ready answer

_"I proved GitOps self-healing hands-on: manually scaled a Deployment outside of Git to 5 replicas, and watched ArgoCD detect the drift and automatically revert it back to 2 — matching what Git declared. It's a good concrete example of the core GitOps principle: Git is the source of truth, and the cluster continuously reconciles toward it, regardless of manual changes."_

---

## PART 7 — GitHub Actions CI/CD — Closing the Loop

### Theory (short)

CI (GitHub Actions) never touches the cluster directly. Instead: test → build image → push to registry → **update the Git repo's Helm values with the new image tag** → commit. ArgoCD (already watching Git) picks up that commit and deploys automatically. This keeps "what triggers a deploy" strictly Git-based, never an external pipeline reaching into the cluster.

### Workflow file

```yaml
name: CI-CD

permissions:
  contents: write     # explicit opt-in for GITHUB_TOKEN to push commits

on:
  push:
    branches: [main]
    paths-ignore:
      - 'go-web-app/values.yaml'   # CRITICAL — prevents infinite trigger loop

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22.5'
      - run: go test ./...
      - id: vars
        run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: aakash0908/go-web-app:${{ steps.vars.outputs.tag }}
      - run: |
          sed -i "s/tag: .*/tag: \"${{ steps.vars.outputs.tag }}\"/" go-web-app/values.yaml
      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add go-web-app/values.yaml
          git commit -m "ci: bump image tag to ${{ steps.vars.outputs.tag }}" || echo "No changes"
          git push
```

### Why the image tag is a commit hash, not manual versioning

`git rev-parse --short HEAD` ties every deployed image directly and traceably back to the exact commit that produced it — far superior to manually incrementing `v1`/`v2`/`v3`, which loses that direct traceability.

### Real bug hit: GITHUB_TOKEN permission denial (two-layer permission system)

```
remote: Permission to aakash-1004/go-web-app.git denied to github-actions[bot].
```

GitHub's default `GITHUB_TOKEN` is read-only unless explicitly granted write access. **This requires fixing in two separate places, not just one:**

1. **Workflow file:** `permissions: contents: write` block
2. **Repo-level setting:** Settings → Actions → General → Workflow permissions → "Read and write permissions"

**Key insight:** the repo-level setting acts as a **ceiling** — the workflow file can only request up to what the repo-level setting allows, never beyond it. Fixing only the workflow file (without the repo setting) silently caps back down to read-only.

### The `paths-ignore` self-trigger trap

Without excluding the file the pipeline itself writes back to (`values.yaml`), every automated commit would re-trigger the same workflow, which bumps the tag again, triggering again — an infinite loop. Standard CI/CD hygiene: **a pipeline should never react to its own output.**

### Full loop proven live, end to end

Edited `static/about.html` (added a visible credit line) → committed → pushed → GitHub Actions ran (test, build, push new image, bump `values.yaml`) → ArgoCD detected the new tag in Git → auto-synced → new pods deployed → refreshed `go-web-app.local/about` in browser → **new content visibly live**, with zero manual deployment commands after the initial `git push`.

### Interview-ready answer

_"I built a full GitOps CI/CD loop: GitHub Actions tests and builds on every push, tags the image with the commit hash for traceability, pushes to Docker Hub, then updates the Helm chart's values.yaml with the new tag and commits that back to the repo — critically excluding that same file from re-triggering the pipeline, to avoid an infinite loop. ArgoCD, already watching the repo, picks up that commit and deploys automatically. I proved the whole thing end-to-end by editing a static HTML file and watching it go live with zero manual kubectl or helm commands after the push. I also hit a real permissions issue — GitHub's default token is read-only, and it turns out there are two separate permission layers (the workflow file's request, and a repo-level ceiling setting) that both need aligning."_

---

## Cross-Cutting Lessons & Debugging Patterns

- **"Declared vs actually working" is a recurring theme across this entire project** — Ingress objects, Fargate profiles, Helm releases, and CI pipeline runs can all report success/exist while not actually functioning end-to-end. The only real proof is a genuine `curl` test hitting real content, not just `kubectl get` showing `Running`/`Created`.
- **kubeconfig context management** — multiple clusters (k3s, EKS) coexist peacefully in one `~/.kube/config`; `KUBECONFIG` env var overrides can cause confusing permission errors if pointed at a root-owned file (e.g., k3s's default `/etc/rancher/k3s/k3s.yaml`).
- **Working-directory-relative paths** bit twice with Helm commands (`.` vs `./go-web-app` depending on `pwd`) — habit: always `pwd` before Helm commands.
- **Free Tier / cost-aware infrastructure choices are real architectural constraints**, not just billing concerns — they can structurally block certain instance types or pod densities regardless of workload needs.
- **Async AWS operations** (`eksctl delete nodegroup [async]`) can return control to the terminal before the underlying operation actually completes — interrupting or immediately re-running commands based on the CLI's return can create genuinely confusing intermediate states.

## Related notes

- [[5.5-EKS-Fargate-IRSA-ALB-Project]]
- Kubernetes-Pod-Lifecycle
- Docker-Security-Hardening

<!-- related -->
---
**Related:** [[7.3.2-ArgoCD-Application-Manifest-and-Commands|ArgoCD — Application Manifest, Commands & Interview Answers]] · [[7.3.1-Helm-Chart-Anatomy-and-Commands|Helm — Chart Anatomy, Templating & Commands (GitOps Lab)]] · [[7.3-ArgoCD-and-GitOps|CI/CD: ArgoCD & GitOps]] · [[5.4-Helm|Helm]] · [[5.5-EKS-Fargate-IRSA-ALB-Project|EKS on Fargate + IRSA + ALB Ingress Controller — End to End]]
<!-- /related -->
