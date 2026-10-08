---
title: GCP Mastery — Complete DevOps Notes
aliases:
  - GCP Mastery
  - GCP-Mastery-Notes
type: moc
module: gcp
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - gcp
  - devops
  - cloud
  - multicloud
  - interview
  - masterreview
  - moc
level: "DevOps Engineer / Cloud Engineer (2-3 YOE, cross-cloud)"
prerequisites: "[[AWS-Mastery-MOC]] and [[Azure-MOC]] — this doc assumes fluency in both and maps forward"
up: "[[Home]]"
related:
  - "[[AWS-MOC]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# GCP Mastery — Complete DevOps Notes

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

> **How to use this note:**
> Every service follows: Concept → AWS Equivalent → Azure Equivalent → Key Differences → Real Config → Production Use Case → Console Path → Memory Hint → Interview Q&A.
> By mapping to BOTH AWS and Azure, you build trilateral fluency — the goal is that when an interviewer asks "have you worked across clouds," you can answer with concrete architectural comparisons across all three.

See the [[Cross-Cloud-Service-Map]] for the AWS ↔ Azure ↔ GCP equivalents.

*Quick orientation, sixteen topic parts (fundamentals → BigQuery) and a three-way cross-cloud interview Q&A.*  
**Related modules:** [[AWS-MOC|AWS]] · [[Azure-MOC|Azure]]

## Contents

- [[GCP-00-Quick-Orientation|0. GCP Basics — Quick Orientation]] — Global Infrastructure; Product Naming Convention
- [[GCP-01-Fundamentals|1. GCP Fundamentals]] — GCP Fundamentals — Organization, Folders, Projects
- [[GCP-02-IAM-Service-Accounts-Roles|2. IAM — Identity, Service Accounts, Roles]] — Concept; AWS Equivalent; Azure Equivalent; Three-Way Comparison; Key Differences from AWS AND Azure; Real Configuration
- [[GCP-03-Compute-Engine|3. Compute Engine]] — Compute Engine — Virtual Machines
- [[GCP-04-Persistent-Disk|4. Persistent Disk]] — Persistent Disk — Block Storage
- [[GCP-05-Cloud-Storage|5. Cloud Storage]] — Cloud Storage — Object Storage
- [[GCP-06-Cloud-Load-Balancing|6. Cloud Load Balancing]] — Cloud Load Balancing — The One-Product-Many-Modes Model
- [[GCP-07-MIG-Managed-Instance-Groups|7. MIG — Managed Instance Groups]] — Managed Instance Groups (MIG) — Autoscaling
- [[GCP-08-Cloud-SQL|8. Cloud SQL]] — Cloud SQL — Managed Relational
- [[GCP-09-Firestore-Bigtable-Spanner|9. Firestore, Bigtable, Spanner]] — Firestore, Bigtable, Spanner — Three Distinct NoSQL/NewSQL Products
- [[GCP-10-Cloud-Functions-and-Cloud-Run|10. Cloud Functions & Cloud Run]] — Cloud Functions & Cloud Run — Serverless
- [[GCP-11-IaC-Deployment-Manager-Config-Connector-Terraform|11. IaC — Deployment Manager, Config Connector, Terraform]] — Concept; AWS Equivalent; Azure Equivalent; The Reality: Use Terraform; Terraform on GCP — The Practical Choice; Config Connector — The Niche But Cool Option
- [[GCP-12-Cloud-DNS|12. Cloud DNS]] — Concept; AWS Equivalent; Azure Equivalent; Three-Way Comparison; Key Differences from BOTH; Real Configuration
- [[GCP-13-Cloud-CDN|13. Cloud CDN]] — Concept; AWS Equivalent; Azure Equivalent; Key Differences from BOTH; Cloud CDN Configuration; Console Path
- [[GCP-14-GKE|14. GKE — Google Kubernetes Engine]] — Concept; AWS Equivalent; Azure Equivalent; Key Differences from BOTH; Real Configuration; Production Use Case
- [[GCP-15-VPC-Networking|15. VPC Networking]] — VPC Networking — Global VPCs and Firewall Rules
- [[GCP-16-BigQuery|16. BigQuery]] — BigQuery — Analytics Warehouse
- [[GCP-17-Three-Way-Cross-Cloud-Interview-QA|17. Three-Way Cross-Cloud Interview Master Q&A]] — The Terminology Cheat Sheet (Three-Way); Cost Model Differences (One-Liner Each); Master Q&A — Multi-Cloud Senior Interview Questions
- [[GCP-18-Wrap-Up-Whats-Next|18. Wrap-Up — What's Next]] — Practical Next Steps (Not Just More Notes); Priority Order; What NOT to Do; Security Reminder; Cross-References

<!-- related -->
---
**Related:** [[AWS-MOC|AWS]]
<!-- /related -->
