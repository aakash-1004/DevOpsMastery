---
title: "IaC — Deployment Manager, Config Connector, Terraform"
type: reference
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
order: 12
up: "[[GCP-MOC]]"
prev: "[[GCP-10-Cloud-Functions-and-Cloud-Run]]"
next: "[[GCP-12-Cloud-DNS]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# IaC — Deployment Manager, Config Connector, Terraform

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-10-Cloud-Functions-and-Cloud-Run|Cloud Functions & Cloud Run]] · → [[GCP-12-Cloud-DNS|Cloud DNS]]
<!-- /nav -->

## Concept

Infrastructure as Code on GCP.

## AWS Equivalent

**Deployment Manager ≈ CloudFormation** (both being deprecated in favor of Terraform-first approaches)

## Azure Equivalent

**Deployment Manager ≈ ARM templates** (raw); **Terraform is the modern choice on all three**

## The Reality: Use Terraform

GCP's Deployment Manager was Google's original IaC tool. It's **being deprecated** — end of life announced for 2025-2026. Not a good choice for new projects.

**Modern GCP IaC options:**
1. **Terraform** (with GCP provider) — the practical choice, works cross-cloud
2. **Config Connector** — Kubernetes-based IaC, provisions GCP resources via K8s CRDs (unique but niche)
3. **Cloud Foundation Toolkit** — Terraform modules maintained by Google for common patterns
4. **Deployment Manager** — legacy, don't start here

## Terraform on GCP — The Practical Choice

Same HCL syntax as AWS/Azure. Provider is `google` (or `google-beta` for newer features).

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  
  backend "gcs" {
    bucket = "myapp-terraform-state"
    prefix = "prod"
  }
}

provider "google" {
  project = "myapp-prod"
  region  = "us-central1"
}

resource "google_storage_bucket" "uploads" {
  name          = "myapp-prod-user-uploads"
  location      = "us-central1"
  storage_class = "STANDARD"
  
  uniform_bucket_level_access = true
  
  lifecycle_rule {
    action { type = "SetStorageClass"; storage_class = "NEARLINE" }
    condition { age = 30 }
  }
}

resource "google_cloud_run_v2_service" "api" {
  name     = "api"
  location = "us-central1"
  
  template {
    containers {
      image = "us-central1-docker.pkg.dev/myapp-prod/repo/api:v1"
      resources {
        limits = { cpu = "1", memory = "512Mi" }
      }
    }
    service_account = google_service_account.api.email
    scaling {
      max_instance_count = 100
    }
  }
}
```

Same workflow: `terraform init`, `plan`, `apply`, `destroy`. State stored in a GCS bucket (like S3 backend for AWS Terraform, or Azure Storage for Azure).

## Config Connector — The Niche But Cool Option

Config Connector lets you manage GCP resources via Kubernetes CRDs (Custom Resource Definitions). Install it in your GKE cluster, and you can write:

```yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: myapp-prod-user-uploads
  namespace: default
spec:
  location: us-central1
  storageClass: STANDARD
  uniformBucketLevelAccess: true
```

Apply with `kubectl apply -f`, GCP resources get created. GitOps for GCP infrastructure — same tools you use for K8s manifests (ArgoCD, Flux) now manage GCP resources.

Niche but powerful for GKE-centric teams. AWS has Controllers for Kubernetes (ACK), Azure has ASO (Azure Service Operator). Same pattern.

## Three-Way IaC Comparison

| Approach | AWS | Azure | GCP |
|---|---|---|---|
| Native declarative | CloudFormation | ARM / Bicep | Deployment Manager (dying) |
| Native CDK-style | CDK (multi-lang) | Bicep | (nothing well-adopted) |
| Terraform | Excellent support | Excellent support | Excellent support |
| K8s-based | Controllers for Kubernetes (ACK) | Azure Service Operator | Config Connector |

**Recommendation across all three clouds: Terraform.** Best cross-cloud investment. Ecosystem is huge. Portable skills.

## Interview Q&A — IaC

**Q: What's your IaC recommendation for a team doing multi-cloud on AWS, Azure, and GCP?**

> "Terraform, unambiguously. Zero debate.
>
> Reasoning: any multi-cloud team benefits from one tool that speaks all three clouds. Terraform's provider ecosystem covers not just AWS/Azure/GCP but also GitHub, Cloudflare, Datadog, Snowflake, and hundreds of others. State management, modules, and workflows work identically across providers.
>
> Alternative approaches all have gaps:
> - CloudFormation is AWS-only
> - Bicep is Azure-only
> - Deployment Manager is GCP-only AND being deprecated
> - CDK is AWS-first with Azure/GCP support that's less mature
>
> Pulumi is worth mentioning as a Terraform alternative — real programming languages instead of HCL. Some teams love it. Terraform is more established and has larger community.
>
> Structure recommendation for multi-cloud Terraform:
> - Separate root modules per cloud per environment: `envs/prod/aws/`, `envs/prod/azure/`, `envs/prod/gcp/`
> - Shared modules for common patterns (bucket, DB, cluster)
> - State backends per cloud in that cloud's native storage (S3, Azure Storage, GCS)
> - Automation via Atlantis, Terraform Cloud, or Spacelift for PR-based workflows
>
> Interview signal: candidates who dogmatically insist on native tools (CloudFormation-only, Bicep-only) look inflexible. Senior answer acknowledges native tools have their place for single-cloud shops but Terraform wins for anyone crossing clouds."
