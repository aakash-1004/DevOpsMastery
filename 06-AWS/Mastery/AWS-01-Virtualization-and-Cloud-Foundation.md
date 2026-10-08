---
title: "Virtualization & Cloud Computing Foundation"
type: reference
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
order: 1
up: "[[AWS-Mastery-MOC]]"
next: "[[AWS-02-AWS-Overview]]"
related:
  - "[[0.3-Virtual-Machines]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# Virtualization & Cloud Computing Foundation

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · → [[AWS-02-AWS-Overview|AWS Overview]]
<!-- /nav -->

## Virtualization

**Concept:** Creating multiple simulated environments (virtual machines) from a single physical hardware system. One physical server hosts many virtual servers, each isolated with its own OS.

**Why it exists:** Before virtualization — one physical server ran one OS ran one app. Servers utilized only 5-15% of capacity. Waste. With virtualization — one physical server runs 20+ VMs. Utilization jumps to 70-80%. This is the foundation of cloud computing.

**Hypervisor** — the software that creates and manages VMs.

```
Type 1 (Bare-metal Hypervisor)         Type 2 (Hosted Hypervisor)
─────────────────────────────          ─────────────────────────────
Runs directly on hardware               Runs on top of a host OS
No host OS                              Requires host OS
Better performance                      Easier to install
Examples: VMware ESXi, KVM,             Examples: VirtualBox,
Hyper-V, Xen                            VMware Workstation
Used in: production data centers        Used in: dev laptops, learning
                                        (your devops-labs VM = Type 2)
```

**Real world context:** AWS EC2 runs on the **Nitro hypervisor** — a specialized KVM variant (Type 1). Every EC2 instance you spin up is a VM running on Nitro. That's why AWS can offer consistent IOPS and network performance with SLA guarantees.

**Memory hint:**

- Type 1 = "sits on the metal" — no host OS
- Type 2 = "sits on the OS" — needs Windows/Linux underneath

---

## Cloud Computing

**Definition:** On-demand delivery of IT resources over the internet with pay-as-you-go pricing.

**Instead of:** Buying and maintaining physical hardware **You get:** Servers, storage, databases, networking rented from a provider

**Three Service Models:**

```
IaaS (Infrastructure as a Service)
    ├── You manage: OS, runtime, apps, data
    ├── Provider manages: physical hardware, networking, virtualization
    └── Example: AWS EC2, Azure VMs

PaaS (Platform as a Service)
    ├── You manage: apps, data
    ├── Provider manages: OS, runtime, hardware
    └── Example: AWS Elastic Beanstalk, Heroku, App Engine

SaaS (Software as a Service)
    ├── You manage: nothing except your usage
    ├── Provider manages: everything
    └── Example: Gmail, Salesforce, Office 365
```

**Memory hint — "Pizza as a Service":**

- **On-premises** = You make pizza at home (buy everything)
- **IaaS** = Take & bake pizza (they give ingredients, you cook)
- **PaaS** = Pizza delivery (they cook, you set the table)
- **SaaS** = Dine at restaurant (they do everything)

---

## Cloud Deployment Models

```
Public Cloud
    Shared cloud environment
    Multiple users share the underlying infrastructure
    Examples: AWS, Azure, GCP
    Best for: startups, most workloads

Private Cloud
    Dedicated environment for ONE organization
    More control, more privacy, more expensive
    Examples: VMware Private Cloud, OpenStack
    Best for: banks, government, highly regulated industries

Hybrid Cloud
    Mix of public and private
    Sensitive data on private, elastic workloads on public
    Best for: enterprises transitioning to cloud
```

**Real world context:** Most modern companies use hybrid or multi-cloud. Bank keeps customer data on private, uses AWS for the mobile app. Netflix uses AWS but keeps some workloads on their own hardware. Multi-cloud (AWS + GCP + Azure) is common for avoiding vendor lock-in in large enterprises.

---

## Interview Q&A

**Q: What is cloud computing and why has it become dominant?**

> "Cloud computing is on-demand delivery of IT resources over the internet with pay-as-you-go pricing. Instead of buying and maintaining hardware, you rent it from providers like AWS.
>
> It became dominant for three reasons. First, CapEx to OpEx conversion — no huge upfront hardware investment, you pay only for what you use. Second, elasticity — need 100 servers for Black Friday? Scale up in minutes, scale down after. Third, global reach — deploy to 30+ regions worldwide without building data centers.
>
> For developers and DevOps engineers, cloud enabled the entire DevOps movement — Infrastructure as Code, immutable infrastructure, and rapid iteration are only possible because you can create and destroy infrastructure programmatically."

**Q: Difference between IaaS, PaaS, and SaaS with real examples?**

> "IaaS gives you raw infrastructure — VMs, storage, networking. You manage the OS and everything above it. AWS EC2 is IaaS — you get a server, you install what you want.
>
> PaaS gives you a platform to deploy code — you don't touch OS or servers. AWS Elastic Beanstalk is PaaS — push your code, it handles servers, load balancers, scaling. Great for developers who don't want to manage infrastructure.
>
> SaaS is fully managed software you just use. Gmail, Salesforce — you don't manage anything, just use the app.
>
> As a DevOps engineer I work primarily with IaaS (EC2, EBS, VPC) and increasingly with PaaS-like services (ECS, Lambda) that reduce operational overhead."

<!-- related -->
---
**Related:** [[0.3-Virtual-Machines|Virtual Machines]]
<!-- /related -->
