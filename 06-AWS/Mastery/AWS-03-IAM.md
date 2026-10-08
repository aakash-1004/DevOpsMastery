---
title: "IAM — Identity & Access Management"
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
order: 3
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-02-AWS-Overview]]"
next: "[[AWS-04-EC2]]"
related:
  - "[[Lab-EKS-IRSA-Workload-Identity]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# IAM — Identity & Access Management

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-02-AWS-Overview|AWS Overview]] · → [[AWS-04-EC2|EC2 — Elastic Compute Cloud]]
<!-- /nav -->

## Concept

IAM controls **WHO** can access **WHAT** in your AWS account. It's the foundation of AWS security.

**Facts:**

- **Free service** — no additional cost
- **Global service** — not region-specific (unlike most AWS services)
- **Root account** created by default when you sign up — should NEVER be used for daily operations

## Four Core Entities

```
User
    ├── Individual identity for a person or application
    ├── Has permanent credentials (username/password, access keys)
    ├── Attached policies define what they can do
    └── Example: aakash-dev, jenkins-ci

Group
    ├── Collection of users sharing common permissions
    ├── Assign policy to group → all users in group inherit it
    ├── Simplifies management: 100 developers = one Developers group
    └── Example: Developers, Admins, ReadOnly

Role
    ├── Assumable identity — NO permanent credentials
    ├── Temporary access via STS tokens
    ├── Used by AWS services (EC2, Lambda) or cross-account access
    └── Example: EC2-S3-ReadOnly-Role, GitHub-Actions-Deploy-Role

Policy
    ├── JSON document defining permissions (Allow/Deny + Actions + Resources)
    ├── Attached to Users, Groups, or Roles
    ├── Two types: AWS Managed (predefined) and Customer Managed (you create)
    └── Example: AmazonS3ReadOnlyAccess (managed), MyCustomPolicy (customer)
```

## IAM Policy Structure (JSON)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

**Breaking it down:**

- **Effect**: `Allow` or `Deny` — Deny always wins over Allow
- **Action**: what operations (e.g., `s3:GetObject`, `ec2:StartInstances`)
- **Resource**: which resources (use ARN — Amazon Resource Name)
- **Condition** (optional): when the rule applies (IP restrictions, time of day, MFA required)

## MFA — Multi-Factor Authentication

Adds a second layer of security beyond password:

1. Something you **know** (password)
2. Something you **have** (phone with authenticator app)

```
Login flow with MFA:
Username + Password → Correct? → Prompt for MFA code → Correct? → Access granted
```

**Options:**

- **Virtual MFA** (Google Authenticator, Duo, Authy) — most common
- **U2F Security Key** (YubiKey) — hardware key
- **Hardware MFA** — physical device from AWS

**Rule:** ALWAYS enable MFA on:

- Root account (mandatory)
- All IAM users
- Especially anyone with admin/write permissions

## Ways to Access AWS

```
1. AWS Console (web UI)
   ├── Human access via browser
   └── Login with username/password + MFA

2. AWS CLI
   ├── Command line access from terminal
   ├── Requires Access Key ID + Secret Access Key
   └── aws configure to set up

3. AWS SDK
   ├── Programmatic access from applications
   ├── Available for Python (boto3), Node.js, Java, Go, etc.
   └── Uses credentials or IAM roles

4. AWS APIs (REST)
   ├── Direct HTTP calls to AWS APIs
   └── SDKs are wrappers around these
```

## Real Production Setup

**Your setup (as per notes):**

```
Root Account:  aakash@example.com (protected, MFA enabled, never used daily)
    ↓
IAM User:  devops-admin
    ├── Access Keys for CLI/programmatic
    ├── MFA enabled
    └── Attached policies: AdministratorAccess (for learning)

For real projects — create separate users:
    ├── developer (limited to dev environment)
    ├── jenkins-ci (limited to specific deployments)
    └── read-only-audit (view-only for audits)
```

## Least Privilege Principle

**Never grant `AdministratorAccess` in production.** Start with zero permissions, grant only what's needed.

**Bad:**

```json
{"Effect": "Allow", "Action": "*", "Resource": "*"}
```

**Good:**

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-app-bucket/*"
}
```

If compromised — attacker can only touch that one bucket. Blast radius contained.

## Memory Hints

- **User** = permanent employee badge
- **Role** = visitor pass (expires)
- **Policy** = the actual permissions written down
- **Group** = department (everyone in it gets the same access)
- **MFA** = double-lock on your door

## Interview Q&A — IAM

**Q: What's the difference between IAM User and IAM Role? When do you use each?**

> "IAM User is a permanent identity with long-lived credentials — access key and secret key. Used for humans logging in to console, or for applications with permanent credentials.
>
> IAM Role is an assumable identity — no permanent credentials. When assumed, it generates short-lived STS tokens, typically valid for 1 hour. Roles are used for:
>
> - EC2 instances (attached as instance profile) so apps can call AWS APIs without hardcoded keys
> - Lambda functions
> - Cross-account access
> - CI/CD systems like GitHub Actions via OIDC
>
> The golden rule: NEVER put access keys on an EC2 instance or in application code. Always use IAM Roles. If you see `aws_access_key_id` in a config file on a server, that's a security finding — flag it immediately."

**Q: Your EC2 app needs read access to S3. What's the correct way to grant it?**

> "Create an IAM Role with an S3 policy — specifically `s3:GetObject` on the target bucket ARN, not `s3:*`. Attach that role to the EC2 instance as an instance profile.
>
> The application code uses AWS SDK without any explicit credentials. The SDK automatically hits the EC2 instance metadata service at `169.254.169.254/latest/meta-data/iam/security-credentials/` to get temporary credentials from the role. It refreshes them before expiry.
>
> Never add credentials to code or environment variables on the instance. Roles handle authentication transparently and rotate automatically."

**Q: Explain least privilege and give me a real example.**

> "Least privilege means granting only the minimum permissions needed for the job, nothing more.
>
> Example: A Lambda function processes uploaded images from S3 and writes results to a specific DynamoDB table. Bad approach — attach `AmazonS3FullAccess` and `AmazonDynamoDBFullAccess`. Good approach — custom policy allowing only `s3:GetObject` on the specific bucket prefix, and `dynamodb:PutItem` on the specific table.
>
> If that Lambda is ever compromised, the attacker can only read one S3 prefix and write to one table. Blast radius contained. Without least privilege, they'd have full read/write access to every S3 bucket and every DynamoDB table in the account."

<!-- related -->
---
**Related:** [[Lab-EKS-IRSA-Workload-Identity|AWS EKS + IRSA — Secure Workload Identity]]
<!-- /related -->
