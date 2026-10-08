---
title: Azure Mastery — Complete DevOps Notes
aliases:
  - Azure Mastery
  - Azure-Mastery-Notes
type: moc
module: azure
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - azure
  - devops
  - cloud
  - multicloud
  - interview
  - masterreview
  - moc
level: "DevOps Engineer / Cloud Engineer (2-3 YOE, cross-cloud)"
prerequisites: "[[AWS-Mastery-MOC]] — this doc assumes AWS fluency and maps forward"
up: "[[Home]]"
related:
  - "[[AWS-MOC]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Azure Mastery — Complete DevOps Notes

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

> **How to use this note:**
> Every service follows: Concept → AWS Equivalent → Key Differences → Real Config → Production Use Case → Console Path → Memory Hint → Interview Q&A.
> The AWS mapping is deliberate — you're not learning Azure from scratch, you're translating what you already know. When the mental model is different (RBAC vs IAM policies, resource groups vs tags), the "Key Differences" section calls it out explicitly.

See the [[Cross-Cloud-Service-Map]] for the AWS ↔ Azure ↔ GCP equivalents.

*Fifteen topic parts (fundamentals → networking) mapped forward from AWS, plus a cross-cloud interview Q&A.*  
**Related modules:** [[AWS-MOC|AWS]] · [[GCP-MOC|GCP]]

## Contents

- [[Azure-01-Fundamentals|1. Azure Fundamentals]] — Azure Fundamentals — Subscriptions, Resource Groups, Tenants
- [[Azure-02-Entra-ID-and-RBAC|2. Entra ID & RBAC]] — Entra ID & RBAC — Identity and Access
- [[Azure-03-Virtual-Machines|3. Virtual Machines]] — Virtual Machines — Compute
- [[Azure-04-Managed-Disks|4. Managed Disks]] — Managed Disks — Block Storage
- [[Azure-05-Blob-Storage|5. Blob Storage]] — Blob Storage — Object Storage
- [[Azure-06-Load-Balancer-App-Gateway-Front-Door|6. Load Balancer, App Gateway, Front Door]] — Load Balancer, Application Gateway, Front Door
- [[Azure-07-VMSS-Auto-Scaling|7. VMSS — Auto Scaling]] — Virtual Machine Scale Sets (VMSS)
- [[Azure-08-Azure-SQL-Database|8. Azure SQL Database]] — Azure SQL Database — Managed Relational
- [[Azure-09-Cosmos-DB|9. Cosmos DB]] — Cosmos DB — Multi-Model NoSQL
- [[Azure-10-Azure-Functions|10. Azure Functions]] — Azure Functions — Serverless
- [[Azure-11-ARM-and-Bicep|11. ARM & Bicep]] — ARM Templates & Bicep — IaC
- [[Azure-12-Azure-DNS|12. Azure DNS]] — Concept; AWS Equivalent; Key Differences from AWS; Real Configuration; Production Use Case; Memory Hint
- [[Azure-13-Azure-CDN|13. Azure CDN]] — Azure CDN & Front Door — Content Delivery
- [[Azure-14-AKS|14. AKS — Azure Kubernetes Service]] — Concept; AWS Equivalent; Key Differences from AWS; Real Configuration; Workload Identity — Managed Identity for Pods; Production Use Case
- [[Azure-15-Networking-VNets-Subnets-NSGs|15. Networking — VNets, Subnets, NSGs]] — Concept; AWS Equivalent; Key Differences from AWS; Real Configuration; Production Use Case — 3-Tier VNet Design; Console Path
- [[Azure-16-Cross-Cloud-Interview-Master-QA|16. Cross-Cloud Interview Master Q&A]] — "You're an AWS shop being asked to also support Azure — what changes in your mental model?"; AWS ↔ Azure Terminology Cheat Sheet; Cost Model Differences Worth Knowing; Design Question: 3-Tier Web Application on Azure
- [[Azure-17-Session-Wrap-up|17. Session Wrap-up]]

<!-- related -->
---
**Related:** [[AWS-MOC|AWS]]
<!-- /related -->
