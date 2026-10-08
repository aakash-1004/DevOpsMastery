---
title: "Canonical Answer #1 — Deployment Strategies"
aliases:
  - Canonical Answer - Deployment Strategies
type: interview
module: interview
tags:
  - interview
  - canonical-answer
  - deployment-strategies
up: "[[Interview-Prep-MOC]]"
prev: "[[Concept-Bank-Batch-2-105-QA]]"
origin: OnlyQuestions/Canonical Answer - Deployment Strategies.md
---

# Canonical Answer #1 — Deployment Strategies

<!-- nav -->
↑ [[Interview-Prep-MOC|Interview Prep]] · ← [[Concept-Bank-Batch-2-105-QA|DevOps Interview Prep — Batch 2 (Concept Bank, ~105 Q&A)]]
<!-- /nav -->

_Covers every phrasing this shows up as: "what deployment strategies have you used," "how do you achieve zero-downtime deployment," "explain blue-green vs canary," "design a CI/CD pipeline for zero-downtime K8s deployment."_

---

## 1. The quick definition layer (say this first, always)

"There are four common strategies. **Recreate** — kill everything old, then start new; simplest, but real downtime. **Rolling update** — replace pods gradually while some old ones keep serving traffic; no downtime if configured right, this is Kubernetes' default. **Blue-green** — two full identical environments, only one live at a time, switch all traffic at once; instant rollback, costs double the infra while both exist. **Canary** — send a small percentage of traffic to the new version first, watch metrics, then ramp up; lowest blast radius, most moving parts to set up."

---

## 2. What you've actually built (your real anchor — lead with this)

This is your strongest ground because it's real, not theoretical:

"In my GitOps project, every deployment goes through a Kubernetes rolling update, orchestrated by ArgoCD. The flow is: GitHub Actions builds the image, bumps the tag in the Helm values file, commits it — ArgoCD picks up that diff and syncs it to the cluster. The rolling update itself is what actually gives me zero downtime: I set `maxUnavailable: 0` so Kubernetes is never allowed to drop below full serving capacity, and `maxSurge: 1` so it spins up one new pod _before_ it retires an old one. The new pod doesn't get traffic until its readiness probe passes — so if the new version is broken, it just never enters rotation and the rollout stalls safely instead of taking the app down. On top of that, I've got topology spread constraints across nodes, so pods aren't all landing on the same node — a single node going down mid-rollout doesn't kill the whole deployment. And ArgoCD's self-heal means if anything drifts from what's in Git — someone deletes a pod, a deployment gets manually edited — it gets reconciled back automatically."

**Config you should be able to reference from memory (the "why" matters more than reciting YAML):**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # spin up 1 extra pod above desired count during rollout
    maxUnavailable: 0  # never allow capacity to drop below 100% — this is the actual zero-downtime lever
```

_Why `maxUnavailable: 0` specifically:_ if you leave it at the default (25%), Kubernetes is allowed to take pods out of rotation before new ones are confirmed healthy — under real traffic that's a capacity dip, not true zero-downtime. Setting it to 0 forces Kubernetes to always add new capacity before removing old capacity.

---

## 3. Blue-green and canary — honest framing

Don't overclaim these. You understand them cold and can explain exactly how you'd implement them with your stack, but your production experience is rolling update via K8s + ArgoCD. Say it straight:

**If asked "have you done blue-green or canary":** "My production deployments run rolling updates via Kubernetes and ArgoCD — that's what I've actually built and proven self-heal on. I haven't needed blue-green or canary specifically because rolling update with strict readiness gating has covered my zero-downtime requirement so far. But I know exactly how I'd extend to it: for blue-green, I'd run two full Deployments/Services behind the same Ingress or ALB, and cut traffic over by switching the ALB target group — or the Service selector if I wanted it K8s-native — instead of replacing pods in place. For canary, the clean way in Kubernetes is Argo Rollouts — it replaces the standard Deployment resource and gives you weighted traffic splitting plus automated analysis steps, so you can gate promotion on real metrics instead of eyeballing it."

**If pushed on "why didn't you use canary if it's lower-risk":** "Canary buys you lower blast radius at the cost of more infrastructure to manage — traffic splitting, a metrics-based promotion gate, usually a tool like Argo Rollouts or a service mesh. For a single service without complex traffic-shaping needs, rolling update with a strict `maxUnavailable: 0` and solid readiness probes gets you the same zero-downtime outcome with a lot less operational overhead. I'd reach for canary specifically when the blast radius of a bad release is high enough to justify that extra complexity — think a payment service, not a single internal Go app."

---

## 4. Senior-differentiation angle (for the 13-30 Q senior batches)

If the question is framed like "design a zero-downtime CI/CD pipeline for Kubernetes microservices" (Batch 13-level), layer this on top of section 2:

"Beyond the rolling update mechanics, the pipeline itself needs to protect the deploy: SonarQube and Trivy scans as blocking gates _before_ the image is even eligible to deploy — I've built exactly this in a separate Jenkins pipeline, 13 stages, with dual Trivy scans and a SonarQube quality gate that fails the build rather than shipping a vulnerable image. Combine that with namespace-scoped RBAC so the CI pipeline's ServiceAccount can only touch what it needs to, and you've got a deploy path where a bad image is unlikely to even reach the point of a rolling update mattering."

---

## 5. STAR script — "tell me about a time you implemented zero-downtime deployment"

**Situation:** "I was building out a GitOps pipeline for a Go application I wanted to run production-grade, not just as a demo — deployed to both a home-lab k3s cluster and AWS EKS."

**Task:** "I needed deployments to roll out with zero downtime, and I wanted the system to self-heal if something drifted outside of Git — without me manually intervening."

**Action:** "I set the Deployment's rolling update strategy to `maxUnavailable: 0` and `maxSurge: 1`, so new pods always came up before old ones went away, gated by readiness probes so a broken new version would never take traffic. I added topology spread constraints so replicas landed across different nodes, not stacked on one. For the GitOps side, I wired GitHub Actions to build the image and bump the Helm values tag, and let ArgoCD own the actual sync — Git became the single source of truth."

**Result:** "I proved the self-heal live — manually deleted a deployment mid-testing, and ArgoCD reconciled it back within seconds with zero manual steps. And because of the rolling update config, a normal image bump never dropped serving capacity at any point during rollout."

---

## 6. Anticipated follow-ups

- **"What if the new version passes readiness but is still broken?"** → "Readiness only checks that the pod can serve traffic, not that the business logic is correct — that's exactly why I'd add canary-style metrics analysis for anything higher-stakes than what I've built so far. Right now my safety net is: readiness gates entry, and if it's bad, I roll back via a Git revert, and ArgoCD syncs the rollback the same way it syncs a forward deploy."
- **"How do you roll back?"** → "Since it's GitOps, a rollback is just a Git revert of the values.yaml commit — ArgoCD sees the diff and syncs to the previous image tag exactly like any other deploy. No separate rollback tooling needed."
- **"Why ArgoCD and not just `kubectl rollout undo`?"** → "`kubectl rollout undo` works, but it changes cluster state without changing Git — so now Git and the cluster disagree, and the next ArgoCD sync would silently overwrite my manual rollback. Reverting in Git keeps the single source of truth intact."

---

_Next up: Terraform drift. Say the word and I'll build that one the same way._
