---
title: ECS — Elastic Container Service
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
order: 16
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-15-CloudFront]]"
next: "[[AWS-17-EKS]]"
related:
  - "[[6.2-AWS-Deployment-EC2-ECS]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# ECS — Elastic Container Service

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-15-CloudFront|CloudFront — CDN]] · → [[AWS-17-EKS|EKS — Elastic Kubernetes Service]]
<!-- /nav -->

## Concept

**AWS ECS (Elastic Container Service)** is a cloud-based container management service that runs and manages Docker containers on a cluster of virtual servers.

**What it handles for you:**

- Container **creation** (spins up containers based on your task definition)
- Container **management** (health monitoring, restart on failure)
- Container **updating** (rolling deployments)
- **Scaling** (horizontal scaling based on demand)

## Core ECS Terms

```
Cluster
    ├── Logical grouping of tasks and services
    ├── Hosts all resources and infrastructure
    └── One cluster can have many services

Task Definition
    ├── Blueprint for how a container runs
    ├── Similar to a Kubernetes Deployment spec
    ├── Specifies: image, CPU, memory, ports, env vars, IAM role
    └── Versioned (task-def:1, task-def:2, etc.)

Task
    ├── Running instance of a task definition
    ├── Represents actual running containers
    └── Each task = 1+ containers running together (like a K8s pod)

Service
    ├── Long-running task manager
    ├── Handles scaling, load balancing, health
    ├── Ensures N tasks are always running
    └── Similar to a K8s Deployment
```

## ECS Launch Types

**Two ways to run containers on ECS:**

```
1. EC2 Launch Type
   ├── You manage the underlying EC2 instances
   ├── More control, more responsibility
   ├── Cheaper for high steady load
   └── Best when you need custom AMIs, GPUs, or specific instance types

2. Fargate Launch Type (Serverless)
   ├── AWS manages the underlying infrastructure
   ├── You just define containers — no servers to manage
   ├── Pay per task (vCPU + memory + time)
   ├── More expensive per unit but zero ops
   └── Best for variable workloads, microservices
```

**Decision:**

- **High steady workload** (containers running 24/7 fully utilized): EC2 launch type
- **Variable, sporadic, or microservices**: Fargate

## ECS Architecture

```
ECS Cluster
    ├── EC2 Instances (or Fargate — no instances visible)
    │   └── ECS agent running on each instance
    │
    ├── Service: web-app
    │   ├── Task Definition: web-app:v3
    │   ├── Desired count: 5 tasks
    │   └── Load balancer: attached to ALB target group
    │       ├── Task 1 (Container: nginx + node.js)
    │       ├── Task 2 (Container: nginx + node.js)
    │       ├── Task 3 (Container: nginx + node.js)
    │       ├── Task 4 (Container: nginx + node.js)
    │       └── Task 5 (Container: nginx + node.js)
    │
    └── Service: worker
        ├── Task Definition: worker:v2
        ├── Desired count: 3 tasks
        └── No load balancer (background worker)
```

## Task Definition Example (JSON)

```json
{
  "family": "flask-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::xxx:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "flask",
      "image": "aakash0908/flask-app:v2",
      "portMappings": [
        {
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "FLASK_ENV", "value": "production"}
      ],
      "secrets": [
        {
          "name": "MONGO_URI",
          "valueFrom": "arn:aws:secretsmanager:xxx:secret:mongo-uri"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-app",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "flask"
        }
      }
    }
  ]
}
```

## ECS vs Kubernetes (EKS)

```
ECS                                    EKS (Kubernetes)
────────────────────────────────       ────────────────────────────────
AWS proprietary                        Open-source, portable
Simpler to learn                       Steeper learning curve
Deep AWS integration                   Cloud-agnostic
No K8s features (Ingress, etc.)        Full K8s ecosystem
Free (pay for underlying compute)      $0.10/hour for control plane
Best for AWS-only shops                Best for multi-cloud, K8s expertise
```

**Rule of thumb:**

- **ECS**: startups on AWS, teams new to containers, simple use cases
- **EKS**: enterprises, teams with K8s expertise, multi-cloud requirements

## Real Production Setup

**Microservices on ECS Fargate:**

```
Application Load Balancer
    ↓
ECS Cluster (Fargate)
    ├── Service: auth-service (3 tasks)
    ├── Service: user-service (5 tasks)
    ├── Service: payment-service (3 tasks)
    └── Service: notification-worker (2 tasks — no LB)
    ↓
RDS (managed database)
DynamoDB (session storage)
S3 (file storage)
```

**Deployment flow:**

```
1. Developer pushes code
2. CI (GitHub Actions) builds Docker image
3. Push image to ECR (AWS's Docker registry)
4. Update task definition to new image tag
5. ECS rolling deployment:
   ├── Start new tasks with v2
   ├── Wait for health checks
   ├── Deregister old v1 tasks from ALB
   └── Stop v1 tasks
6. Zero downtime deployment complete
```

## Memory Hints

- **ECS** = "AWS's container platform" (simpler than K8s)
- **Cluster** = "the party venue" (where everything happens)
- **Task Definition** = "recipe for a task"
- **Task** = "one running instance" (like a K8s pod)
- **Service** = "keeps N tasks running always"
- **Fargate** = "serverless containers" (no VMs to manage)

## Interview Q&A — ECS

**Q: When would you choose ECS Fargate over ECS EC2 launch type?**

> "Depends on workload characteristics and operational preferences.
>
> Fargate when: you want zero infrastructure management, workload has variable or unpredictable traffic, you're running microservices with independent scaling needs, or you're a small team without capacity to manage EC2. Pay only for what you use, no server patching.
>
> EC2 launch type when: you have steady high load (24/7 fully utilized containers) — Fargate becomes expensive at high sustained utilization. Need specific instance types (GPU, high-memory). Need custom AMIs or specific OS-level configurations. Already have EC2 fleet you want to leverage.
>
> Cost comparison: at high utilization (>80% for months), EC2 with Reserved Instances can be 50% cheaper than Fargate. But Fargate eliminates operational overhead — worth the premium for many teams.
>
> Modern preference: Fargate for most microservices. EC2 for specialized workloads. This is what most teams I've worked with settle on."

**Q: ECS vs EKS — how do you decide?**

> "Depends on team skills, ecosystem needs, and portability requirements.
>
> ECS when: AWS-only strategy, team new to containers, want simplest possible container orchestration, deep AWS service integration matters (IAM per task, native ALB integration, CloudWatch out of the box).
>
> EKS when: multi-cloud portability needed, existing K8s expertise on team, need Kubernetes ecosystem tools (Helm, ArgoCD, Istio), running complex workloads that benefit from K8s features (StatefulSets, CronJobs, Ingress with path-based routing, service mesh).
>
> Cost consideration: EKS control plane is $0.10/hour ($73/month) regardless of usage. ECS control plane is free. For small workloads, ECS is cheaper.
>
> My default recommendation for a team starting fresh: ECS Fargate if they're AWS-only and want simplicity. EKS if they have K8s expertise or expect to grow into complex workloads. The knowledge from either transfers reasonably well to the other if requirements change later."

<!-- related -->
---
**Related:** [[6.2-AWS-Deployment-EC2-ECS|AWS Deployment — EC2 & ECS]]
<!-- /related -->
