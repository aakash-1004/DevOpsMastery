---
title: CloudFormation — IaC
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
order: 13
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-12-Lambda]]"
next: "[[AWS-14-Route-53]]"
related:
  - "[[Terraform-MOC]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# CloudFormation — IaC

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-12-Lambda|Lambda — Serverless Compute]] · → [[AWS-14-Route-53|Route 53 — DNS]]
<!-- /nav -->

## Concept

**AWS CloudFormation** is Infrastructure as Code (IaC) — define AWS resources in YAML or JSON templates, and CloudFormation creates, updates, and manages them for you.

**Key characteristics:**

- **Declarative language** — you define the desired end state
- **Template-based** — YAML or JSON files
- **AWS Native** — supports all AWS resources
- **Cost-free tool** — you only pay for resources you create
- **State management** — CloudFormation tracks state internally (no external state file like Terraform)

## Why IaC?

```
Consistency:     Same template = same infrastructure every time
Automation:      No manual clicking in console
Repeatable:      Deploy to dev, staging, prod with same code
Version control: Templates in Git = full change history
Rollback:        Failed deployment auto-rolls back
Auditability:    Every change is recorded
```

## CloudFormation Concepts

```
Template
    ├── YAML or JSON file defining resources
    └── Reusable across environments

Stack
    ├── A collection of resources created from a template
    └── Manage all resources as one unit

Stack Set
    ├── Deploy same template across multiple accounts/regions
    └── Great for enterprise multi-account setups

Change Set
    ├── Preview changes before applying (like Terraform plan)
    └── See what will be created/updated/deleted

Drift Detection
    ├── Detects resources changed outside CloudFormation
    └── Reports differences from template
```

## Simple CloudFormation Template Example

```yaml
---
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Simple EC2 instance'

Resources:
  SimpleEC2Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: t3.micro
      ImageId: ami-02db68a01488594c5
      Tags:
        - Key: Name
          Value: MySimpleInstance
```

## Common Use Cases

```
1. Create EC2 Instances with Security Groups + Elastic IPs
2. Provision S3 Buckets with configurations
3. Set up VPCs with Subnets and Route Tables
4. Create RDS databases with automated backups
5. Deploy Lambda functions with API Gateway
6. Set up ELB and ASG for web apps
7. Deploy IAM roles, policies, users
```

## CLI Commands

```bash
# Create a stack
aws cloudformation create-stack \
  --stack-name MyStack \
  --template-body file://template.yaml

# Update stack
aws cloudformation update-stack \
  --stack-name MyStack \
  --template-body file://template.yaml

# Delete stack (removes ALL resources)
aws cloudformation delete-stack --stack-name MyStack

# See what will change (preview)
aws cloudformation create-change-set \
  --stack-name MyStack \
  --change-set-name my-changes \
  --template-body file://template.yaml
```

## CloudFormation vs Terraform

```
CloudFormation                     Terraform
────────────────────────────────   ────────────────────────────────
AWS only                           Multi-cloud (AWS, Azure, GCP)
YAML/JSON                          HCL (HashiCorp)
State managed by AWS               State file (you manage)
Free                               Free (Terraform Cloud has pricing)
Better AWS integration             More flexible, better community
Native drift detection             terraform plan
Change Sets                        terraform plan
```

**Real world:** Most modern teams use Terraform because of multi-cloud support and better tooling. CloudFormation is common in AWS-only shops or enterprises with existing AWS investments.

## Memory Hint

- **CloudFormation** = "AWS's built-in Terraform"
- **Template** = "recipe for infrastructure"
- **Stack** = "the actual cooked meal" (created from template)
- **Change Set** = "preview of what will change"

## Interview Q&A — CloudFormation

**Q: CloudFormation vs Terraform — which do you prefer?**

> "Depends on the context.
>
> CloudFormation: AWS-native, no external state to manage, integrates deeply with AWS services (Change Sets, drift detection built-in). Free service. Better for AWS-only shops or when you want minimal external dependencies.
>
> Terraform: multi-cloud (AWS, Azure, GCP, Kubernetes, and thousands of providers via community). Better developer experience with HCL. Larger ecosystem — pre-built modules for everything. State management is more explicit which is powerful but requires care.
>
> My preference: Terraform for most modern projects. Reasons: multi-cloud portability, better modularity, larger community, and it's the industry standard for infrastructure code in 2026. Also, learning Terraform gives you skills that transfer across all clouds.
>
> I'd choose CloudFormation when: the team is AWS-exclusive and has existing CloudFormation investment, or when the workload uses AWS-specific features that Terraform lags on."

<!-- related -->
---
**Related:** [[Terraform-MOC|Terraform]]
<!-- /related -->
