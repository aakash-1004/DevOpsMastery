---
title: AWS Overview
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
order: 2
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-01-Virtualization-and-Cloud-Foundation]]"
next: "[[AWS-03-IAM]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# AWS Overview

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-01-Virtualization-and-Cloud-Foundation|Virtualization & Cloud Computing Foundation]] · → [[AWS-03-IAM|IAM — Identity & Access Management]]
<!-- /nav -->

## What is AWS?

**Amazon Web Services** — launched in 2006 as the first public cloud platform. Started with S3 (storage) and EC2 (compute). Now offers 200+ services across compute, storage, networking, databases, AI/ML, IoT, and more.

**Pay-as-you-go model** — you pay only for what you use, billed per second/hour/GB.

## AWS Global Infrastructure

```
Regions (33+ worldwide)
    ├── Physical locations around the world
    ├── Each region is INDEPENDENT
    ├── You choose which region(s) to deploy in
    └── Example: ap-south-1 (Mumbai), us-east-1 (N. Virginia)

Availability Zones (AZs)
    ├── Each region has 3+ AZs
    ├── Each AZ = one or more physical data centers
    ├── AZs in the same region are connected by low-latency links
    └── Isolated from each other — one AZ failure doesn't affect others

Edge Locations (400+)
    ├── Used by CloudFront (CDN) and Route 53
    ├── Closer to end users than regions
    └── Deliver content with low latency
```

**Real world context:** Your AWS setup: `ap-south-1` (Mumbai) region, account `956179206096`, IAM user `devops-admin`. Mumbai has 3 AZs — `ap-south-1a`, `ap-south-1b`, `ap-south-1c`. For high availability, always deploy across at least 2 AZs.

## Why AWS Dominates

1. **Scalability** — scale from 1 to 1 million users
2. **Global reach** — 33+ regions, 400+ edge locations
3. **Reliability** — 99.99% SLA on many services
4. **Security** — compliance with SOC, ISO, PCI DSS, HIPAA
5. **Ecosystem** — huge partner network, tools, expertise available

## Common AWS Career Roles

- Cloud Engineer / Solutions Architect
- **DevOps Engineer** ← your target
- Cloud Developer / Data Engineer
- Site Reliability Engineer (SRE)
- Cloud Security Specialist
- Cloud Support Associate

**Memory hint:** AWS launched with S3 and EC2 — **storage + compute** are the two pillars of cloud. Everything else is built on top.
