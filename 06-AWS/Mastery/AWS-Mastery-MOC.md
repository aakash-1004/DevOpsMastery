---
title: AWS Mastery — Complete DevOps Notes
aliases:
  - AWS-Mastery-Notes
type: moc
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
  - moc
level: DevOps Engineer / Cloud Engineer (2-3 YOE target)
up: "[[AWS-MOC]]"
related:
  - "[[6.1-AWS-Core-Services]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# AWS Mastery — Complete DevOps Notes

<!-- nav -->
↑ [[AWS-MOC|AWS]]
<!-- /nav -->

> **How to use this note:** Every service follows: Concept → Real Configuration → Production Use Case → Memory Hint → Interview Answer. Interview answers are framed as a senior DevOps engineer would give — grounded in production reality.

See the [[Cross-Cloud-Service-Map]] for the AWS ↔ Azure ↔ GCP equivalents.

## Contents

- [[AWS-01-Virtualization-and-Cloud-Foundation|1. Virtualization & Cloud Computing Foundation]] — Virtualization; Cloud Computing; Cloud Deployment Models; Interview Q&A
- [[AWS-02-AWS-Overview|2. AWS Overview]] — What is AWS?; AWS Global Infrastructure; Why AWS Dominates; Common AWS Career Roles
- [[AWS-03-IAM|3. IAM — Identity & Access Management]] — Concept; Four Core Entities; IAM Policy Structure (JSON); MFA — Multi-Factor Authentication; Ways to Access AWS; Real Production Setup
- [[AWS-04-EC2|4. EC2 — Elastic Compute Cloud]] — Concept; Configuration Options; Instance Type Families; User Data Script Example; Security Groups (SG); Common Ports to Know
- [[AWS-05-EBS|5. EBS — Elastic Block Store]] — Concept; EBS vs Instance Store; EBS Volume Types; Key Features; Practical: Attaching EBS to EC2; Snapshot Workflow
- [[AWS-06-AMI|6. AMI — Amazon Machine Image]] — Concept; Types of AMIs; Real Production Use Case — LAMP Stack Paid AMI; Creating a Custom AMI; Launch Template vs Custom AMI; EC2 Image Builder
- [[AWS-07-ELB|7. ELB — Elastic Load Balancer]] — Foundational Concepts First; Load Balancer — Concept; Types of AWS Load Balancers; ALB Advanced Features; Health Checks; Target Groups
- [[AWS-08-ASG|8. ASG — Auto Scaling Group]] — Concept; Key Functions; Steps to Create ASG; Scaling Policies Deep Dive; Real Production Patterns; ASG + ELB Integration
- [[AWS-09-S3|9. S3 — Simple Storage Service]] — Concept; Key Facts; Storage Types Comparison; Common Use Cases; S3 Versioning; S3 Replication
- [[AWS-10-RDS|10. RDS — Relational Database Service]] — Concept; Supported Database Engines; Amazon Aurora — The Star; Key Benefits of RDS; Multi-AZ vs Read Replicas; Practical Setup
- [[AWS-11-DynamoDB|11. DynamoDB — NoSQL]] — Concept; Data Model; Key Concepts; Capacity Modes; Serverless Benefits; Use Cases
- [[AWS-12-Lambda|12. Lambda — Serverless Compute]] — Concept; When to Use Lambda; Event-Driven Execution; Automatic Scaling; Pay-as-You-Go Pricing; Supported Languages
- [[AWS-13-CloudFormation|13. CloudFormation — IaC]] — Concept; Why IaC?; CloudFormation Concepts; Simple CloudFormation Template Example; Common Use Cases; CLI Commands
- [[AWS-14-Route-53|14. Route 53 — DNS]] — Concept; DNS Refresher; Route 53 Features; DNS Record Types Deep Dive; How Route 53 Works; Routing Policies — Use Cases
- [[AWS-15-CloudFront|15. CloudFront — CDN]] — Concept; How CloudFront Works; What Gets Cached; Cache Behavior Controls; Browser Cache vs CloudFront; Origin Types
- [[AWS-16-ECS|16. ECS — Elastic Container Service]] — Concept; Core ECS Terms; ECS Launch Types; ECS Architecture; Task Definition Example (JSON); ECS vs Kubernetes (EKS)
- [[AWS-17-EKS|17. EKS — Elastic Kubernetes Service]] — Concept; Kubernetes Refresher; EKS Node Options; Prerequisites — CLI Tools; Creating an EKS Cluster; After Cluster Creation
- [[AWS-18-Amplify|18. Amplify — Fullstack Platform]] — Concept; What Amplify Handles; What You Can Build; Practical Workflow; Amplify vs Other Options; Memory Hints
- [[AWS-19-Interview-Master-QA|19. Interview Master Q&A]] — Foundation Questions; IAM Deep Dive; Networking; Storage; Cost Optimization; Availability & Reliability
- [[AWS-20-Session-Wrap-up|20. Session Wrap-up]]

<!-- related -->
---
**Related:** [[6.1-AWS-Core-Services|AWS Core Services]]
<!-- /related -->
