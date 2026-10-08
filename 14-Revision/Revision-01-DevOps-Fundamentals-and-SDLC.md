---
title: "Revision — DevOps Fundamentals & SDLC"
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
order: 1
up: "[[Master-Revision-MOC]]"
next: "[[Revision-02-Linux-and-Bash-Scripting]]"
related:
  - "[[Fundamentals-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — DevOps Fundamentals & SDLC

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · → [[Revision-02-Linux-and-Bash-Scripting|Revision — Linux & Bash Scripting]]
<!-- /nav -->

## Concept

DevOps is a culture and set of practices that unify software Development (Dev) and IT Operations (Ops). The goal: **shorten the SDLC while delivering high-quality software continuously.**

Before DevOps, Dev teams threw code "over the wall" to Ops. Ops had no idea what changed. Deployments were quarterly events that caused incidents. DevOps fixes this by making both teams share responsibility across the full lifecycle.

**SDLC Phases:**

```
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → (back to Plan)
```

**Key DevOps Principles:**

- Everything as Code (IaC, PaC, config-as-code)
- Automate everything repeatable
- Shift Left — catch defects early (testing in CI, not after deployment)
- Continuous Feedback — monitoring feeds back into planning
- Fail Fast — detect issues at the earliest possible stage

**Types of Hypervisors:**

- **Type 1 (Bare-metal):** Runs directly on hardware. Examples: VMware ESXi, Hyper-V, KVM. Used in data centers, production. Better performance.
- **Type 2 (Hosted):** Runs on top of a host OS. Examples: VirtualBox, VMware Workstation. Used for development/learning. Your devops-labs VM uses VirtualBox (Type 2).

---

## Senior Interview Q&A

**Q: What is DevOps and how is it different from traditional IT?**

> "DevOps is fundamentally a cultural shift, not just a toolset. Traditional IT had hard walls between Dev and Ops — developers wrote code, threw it to operations, and operations deployed it without understanding what changed. This created slow releases, blame games, and high failure rates in production.
>
> DevOps breaks that wall. Developers own code from commit to production, and operations provides the platform but also codes — IaC, automation scripts, pipelines. The result is faster delivery with more stability. When I say faster: Netflix deploys hundreds of times a day. That's only possible with DevOps culture and the tooling that supports it.
>
> The key metrics that matter: DORA metrics — deployment frequency, lead time for changes, change failure rate, and time to restore service. DevOps organizations score better on all four."

---

**Q: What's the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment?**

> "CI is about automatically building and testing every code commit. The moment a developer pushes, a pipeline runs — compile, unit tests, integration tests. If it fails, the developer knows in minutes, not days.
>
> Continuous Delivery means the code is always in a deployable state after CI passes. It's ready to go to production, but a human still clicks 'deploy' — there's a manual gate. That gate exists for business reasons, not technical ones.
>
> Continuous Deployment removes that human gate entirely. Every green build goes to production automatically. You need high test coverage and feature flags to do this safely.
>
> Most companies I've worked with land on Continuous Delivery — CI all the way, automated staging deployments, manual production approval."

---

**Q: How does a Type 1 vs Type 2 hypervisor affect your infrastructure decisions?**

> "In production, you always use Type 1. KVM on Linux, ESXi for VMware environments — they sit on bare metal, no host OS overhead. You get near-native performance because the hypervisor talks directly to the hardware.
>
> Type 2 is for your laptop. VirtualBox, VMware Workstation — they run inside your OS, so you're paying a double virtualization tax. Fine for learning, never for production.
>
> In cloud context: AWS EC2 runs on KVM under the hood (Nitro hypervisor is a specialized KVM variant). So every EC2 instance you spin up is a Type 1 VM. This is why EC2 can offer consistent IOPS and network performance that you can SLA-guarantee."

<!-- related -->
---
**Related:** [[Fundamentals-MOC|Fundamentals]]
<!-- /related -->
