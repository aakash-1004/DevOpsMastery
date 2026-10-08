---
title: ASG — Auto Scaling Group
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
order: 8
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-07-ELB]]"
next: "[[AWS-09-S3]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# ASG — Auto Scaling Group

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-07-ELB|ELB — Elastic Load Balancer]] · → [[AWS-09-S3|S3 — Simple Storage Service]]
<!-- /nav -->

## Concept

**ASG (Auto Scaling Group)** is a service that automatically adds or removes EC2 instances based on demand.

**What it does:**

- Scale **up** when demand increases (more instances)
- Scale **down** when demand decreases (fewer instances)
- Maintain a **desired number** of healthy instances at all times
- Replace unhealthy instances automatically

## Key Functions

```
1. Automatic Scaling
   ├── Add/remove instances based on demand
   ├── Triggered by CloudWatch metrics (CPU, network, custom metrics)
   └── Cooldown period prevents flapping

2. Maintain Instance Health
   ├── Health checks (EC2 status or ELB target group)
   ├── Unhealthy instance → terminate → launch replacement
   └── Self-healing without human intervention

3. Scaling Policies
   ├── Target tracking (keep CPU at 60%)
   ├── Step scaling (add 2 when CPU > 70%, add 5 when CPU > 90%)
   ├── Simple scaling (add 1 when metric threshold breached)
   └── Predictive scaling (ML predicts future demand)

4. Ensure Availability
   ├── Minimum size: never fewer than X instances
   ├── Maximum size: never more than Y instances
   ├── Desired capacity: current target
   └── Handles instance failures automatically

5. Scheduled Scaling
   ├── Pre-configure scaling for known patterns
   ├── Example: scale up every weekday at 9AM (office hours traffic)
   └── Example: scale down every weekend

6. Multi-AZ Distribution
   ├── Spread instances across multiple AZs
   ├── AZ failure = only lose 1/N instances
   └── ASG automatically launches replacements in healthy AZs

7. Cost Optimization
   ├── Scale down during low demand = save money
   ├── Mix of On-Demand and Spot Instances
   └── Only pay for what you need
```

## Steps to Create ASG

```
1. Launch Template or Launch Configuration
   ├── Defines HOW instances are launched
   ├── AMI, instance type, security group, IAM role, user_data
   └── Reusable — one template, many ASGs

2. Create Auto Scaling Group
   ├── Select the Launch Template
   ├── Set min/max/desired instances
   └── Name the ASG

3. Select VPC and Subnets
   ├── Choose which subnets to launch in
   ├── Select multiple AZs for HA
   └── Public or private subnets based on architecture

4. Attach Load Balancer (Optional but common)
   ├── ASG registers instances with ALB/NLB automatically
   ├── ASG uses ELB health checks
   └── Enables health-based replacement

5. Configure Scaling Policies
   ├── Target tracking is the easiest
   ├── "Keep average CPU at 60%"
   └── ASG handles the rest

6. Health Checks
   ├── EC2 health checks (instance status)
   ├── ELB health checks (application-level)
   └── Grace period before checks start

7. Add Notifications (Optional)
   ├── SNS topic for scale up/down events
   └── Alerts your team, feeds monitoring

8. Review and Create
```

## Scaling Policies Deep Dive

**Target Tracking (recommended for most cases):**

```
Target: Average CPU utilization = 60%

If CPU rises above 60% → ASG adds instances until average drops to 60%
If CPU falls below 60% → ASG removes instances until average rises to 60%

Simple, self-tuning, no threshold math needed.
```

**Step Scaling:**

```
CPU 50-70%    → Add 1 instance
CPU 70-90%    → Add 2 instances
CPU 90%+      → Add 3 instances

More aggressive scaling for severe spikes.
```

**Scheduled Scaling:**

```
Every Monday 9AM       → Scale to 10 instances (office hours)
Every Monday 6PM       → Scale to 3 instances (evening)
Every Saturday morning → Scale to 2 instances (weekend traffic)

Predictable patterns handled without waiting for metrics to trigger.
```

## Real Production Patterns

**E-commerce site:**

```
Baseline:  5 instances (normal traffic)
Peak:      50 instances (Black Friday)
Overnight: 3 instances (low traffic)

ASG min: 3, max: 60, target CPU: 60%
Result: automatic scaling handles 10x traffic, saves 40% cost overnight
```

**Batch processing:**

```
Baseline:  1 instance (always available)
Job queue length increases:
    5 jobs   → 3 instances
    50 jobs  → 20 instances
    500 jobs → 100 instances (up to max)

Custom CloudWatch metric = queue depth
```

## ASG + ELB Integration

```
Client Request
    ↓
Route 53 (DNS)
    ↓
Application Load Balancer (target group)
    ↓
ASG (min=3, max=20)
    ├── Instance 1 (AZ-a) — healthy
    ├── Instance 2 (AZ-b) — healthy
    └── Instance 3 (AZ-c) — healthy
```

**Flow:**

1. ASG launches instances per Launch Template
2. Instances auto-register with ALB target group
3. ALB starts sending traffic once health check passes
4. If instance becomes unhealthy → ALB stops sending traffic → ASG terminates and replaces
5. New instance boots → registers with ALB → traffic resumes

## Memory Hints

- **ASG** = "smart HR that hires/fires workers based on workload"
- **Launch Template** = "job posting" (how new hires are onboarded)
- **Scaling Policy** = "hiring rules" (when to hire more or fewer)
- **Health Check** = "attendance check" (are workers showing up?)
- **Min/Max/Desired** = "workforce boundaries" (never fewer than X, never more than Y)

## Interview Q&A — ASG

**Q: Explain how ASG maintains availability during an AZ outage.**

> "ASG is designed for multi-AZ deployments. When you create an ASG, you specify subnets across multiple Availability Zones. ASG spreads instances evenly — if you have 6 instances and 3 AZs, it launches 2 per AZ.
>
> When an AZ fails: ASG's EC2 health checks detect that instances in that AZ are unreachable. ASG terminates the failed instances and immediately launches replacements — but only in HEALTHY AZs. So if AZ-a fails, ASG launches 2 more instances split between AZ-b and AZ-c to maintain the desired count.
>
> During the outage, you're temporarily running with 2/3 of your normal capacity in 2 AZs. If the ALB was multi-AZ and had cross-zone load balancing enabled, traffic automatically shifted away from AZ-a with no manual intervention.
>
> Recovery: when AZ-a comes back, ASG rebalances by launching instances there and terminating some from AZ-b and AZ-c to restore even distribution.
>
> Key takeaway: multi-AZ ASG plus ALB with cross-zone load balancing = automatic failover with zero downtime. This is why 'HA across AZs' is a core cloud principle."

**Q: What's the difference between Target Tracking and Step Scaling?**

> "Both are scaling policies but with different mental models.
>
> Target Tracking is set-and-forget. You define a target metric like 'keep average CPU at 60%'. ASG continuously monitors and adjusts instance count to maintain that target. Uses two CloudWatch alarms internally (scale up, scale down) but you don't manage them. Self-tuning, handles most workloads well. Best for steady traffic patterns.
>
> Step Scaling gives you granular control. You define specific thresholds and how many instances to add or remove at each threshold. Example: CPU 50-70% add 1, CPU 70-90% add 2, CPU 90%+ add 3. More responsive to severe spikes because you're explicitly saying 'when things get really bad, add a lot at once'.
>
> When to use which: Target Tracking for 90% of cases — it's simpler and works well. Step Scaling when you know your traffic patterns are spiky and you want aggressive response, or for batch workloads where scaling behavior needs precise tuning.
>
> Advanced option: Predictive Scaling uses machine learning on historical CloudWatch data to forecast future demand and pre-scale before the spike hits. Great for predictable patterns like daily/weekly cycles."

**Q: How do you prevent ASG from thrashing (constantly scaling up and down)?**

> "Three mechanisms.
>
> First, cooldown periods. After a scaling action, ASG waits for the cooldown to expire before evaluating scale actions again. Default is 300 seconds. This gives time for the metric to stabilize after new instances start receiving traffic. Prevents adding 10 instances when 3 would have been enough.
>
> Second, appropriate target thresholds. If you set CPU target at 90%, you'll be constantly at the edge of scaling. Better to target 60-70% — provides buffer for spikes without triggering unnecessary scale actions.
>
> Third, instance warmup time. New instances need time to boot, register with LB, and start serving traffic. Configure the warmup period — during warmup, the new instance's metrics don't count toward scaling decisions. Prevents 'we just added an instance but CPU is still high, add another' loop.
>
> Fourth, health check grace period. When a new instance launches, ASG waits before starting health checks. Prevents premature termination of instances that are still booting.
>
> Fifth, use target tracking rather than step scaling for most cases — it has built-in stabilization logic.
>
> If you're still seeing thrashing: your scaling metric might be too volatile. Try scaling on a smoothed metric (5-minute average) rather than instant values."
