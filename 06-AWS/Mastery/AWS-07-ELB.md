---
title: ELB — Elastic Load Balancer
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
order: 7
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-06-AMI]]"
next: "[[AWS-08-ASG]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# ELB — Elastic Load Balancer

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-06-AMI|AMI — Amazon Machine Image]] · → [[AWS-08-ASG|ASG — Auto Scaling Group]]
<!-- /nav -->

## Foundational Concepts First

Before ELB, understand three foundational terms:

**Scalability**

```
Ability to grow your system's resources when demand increases.

Vertical Scalability (Scaling Up):
    Add more power (CPU, RAM) to existing server
    Example: t2.micro → m5.large
    Limit: hardware maximum

Horizontal Scalability (Scaling Out):
    Add more instances behind a load balancer
    Example: 1 EC2 → 10 EC2 instances
    Limit: essentially unlimited in cloud
```

**High Availability (HA)**

```
Keeping your service running with minimal downtime.
Achieved by running resources across multiple AZs.
If one AZ fails, others take over.

Example: EC2 instances in AZ-a, AZ-b, and AZ-c behind ALB.
If AZ-a goes down → ALB routes traffic to AZ-b and AZ-c automatically.
```

**Elasticity**

```
Ability to automatically adjust resources based on demand.
Add resources during peak, remove during low usage.

Example: Auto Scaling Group scales from 2 to 20 instances during traffic spike.
Different from scalability — elasticity is automatic.
```

## Load Balancer — Concept

A **load balancer** distributes incoming traffic across multiple servers so no single server gets overloaded.

**Key benefits:**

- **Distributes traffic** — even load across instances
- **Improves availability** — routes around failed instances via health checks
- **Scales resources** — works with ASG for automatic scaling
- **Single point of access** — one DNS name for all your instances
- **HA across AZs** — spans multiple Availability Zones

## Types of AWS Load Balancers

```
1. Application Load Balancer (ALB) — Layer 7 (HTTP/HTTPS)
   ├── Content-based routing (path, host, headers)
   ├── SSL termination
   ├── WebSocket support
   ├── Integration with Cognito for auth
   └── Use for: web apps, microservices, APIs

2. Network Load Balancer (NLB) — Layer 4 (TCP/UDP)
   ├── Ultra-low latency (millions of requests per second)
   ├── Static IP + Elastic IP support
   ├── Preserves client source IP
   ├── TLS termination possible
   └── Use for: gaming, financial trading, IoT

3. Gateway Load Balancer (GWLB) — Layer 3 (IP packets)
   ├── For deploying third-party virtual appliances
   ├── Firewalls, intrusion detection systems
   ├── Traffic inspection and filtering
   └── Use for: enterprise security appliances

4. Classic Load Balancer (CLB) — Legacy, don't use
   ├── Layer 4 and Layer 7
   ├── Deprecated for new deployments
   └── Only for legacy applications
```

**Rule of thumb:** ALB for web apps, NLB for TCP/UDP, GWLB for security appliances.

## ALB Advanced Features

**Path-based routing:**

```
myapp.com/api/*  → API target group (Node.js instances)
myapp.com/admin/* → Admin target group (Django instances)
myapp.com/*      → Frontend target group (React instances)

ONE load balancer, MULTIPLE backend services
```

**Host-based routing:**

```
api.myapp.com     → API target group
admin.myapp.com   → Admin target group
www.myapp.com     → Frontend target group

ONE load balancer, MULTIPLE domains
```

**SSL Termination:**

- Upload SSL certificate to AWS Certificate Manager (ACM)
- ALB handles HTTPS decryption
- Traffic between ALB and instances is HTTP (internal network)
- Your app doesn't need to handle SSL — simpler

## Health Checks

**Configuration:**

- **Path**: `/health` or `/healthz` endpoint on your app
- **Protocol**: HTTP or HTTPS
- **Port**: usually same as app port (e.g., 5000)
- **Interval**: check every 30 seconds
- **Threshold**: fail 3 times before marking unhealthy

**Behavior:**

- Healthy instance → receives traffic
- Unhealthy → removed from rotation
- Becomes healthy again → automatically re-added

**Real world context:** Your app should implement a `/health` endpoint that:

- Returns 200 if the app is functional
- Returns 503 if a critical dependency (DB) is down
- Runs quickly (< 100ms) — the LB pings it frequently

## Target Groups

**Concept:** A target group is a collection of targets (EC2 instances, IP addresses, Lambda functions) that the ALB routes to.

```
ALB
    ├── Listener (port 443, HTTPS)
    │   ├── Rule 1: path=/api/* → api-target-group
    │   ├── Rule 2: path=/admin/* → admin-target-group
    │   └── Default: → frontend-target-group
    │
    ├── api-target-group
    │   ├── i-01234 (EC2 instance)
    │   ├── i-05678 (EC2 instance)
    │   └── i-09abc (EC2 instance)
    │
    └── frontend-target-group
        ├── i-def01
        └── i-def02
```

## Practical: Creating an ALB

```bash
# Steps (via console):
1. Set Up EC2 Instances (2+ instances with a web server, tagged for identification)
2. Configure Security Groups (allow HTTP 80 and SSH 22)
3. Create the Load Balancer (EC2 → Load Balancers → Create → ALB)
4. Register Targets (add instances to the target group, configure health check)
5. Test the Load Balancer (access the DNS name — should see traffic distribution)

# The ALB DNS looks like:
myapp-alb-1234567890.ap-south-1.elb.amazonaws.com
```

## Real World Production Setup

```
Route 53 (DNS)
    ↓  (myapp.com → ALB DNS)
Internet-facing ALB
    ↓  Listener 443 (HTTPS with ACM cert)
    ├── /api/*   → API target group (3 EC2 in ASG)
    ├── /admin/* → Admin target group (2 EC2 in ASG)
    └── /*       → Frontend target group (5 EC2 in ASG)

All in private subnets, only ALB in public subnet.
```

## Memory Hints

- **ALB** = smart traffic cop (understands HTTP routes)
- **NLB** = raw pipe (fast, dumb, transport layer)
- **GWLB** = security checkpoint (traffic inspection)
- **Target Group** = pool of servers waiting for work
- **Health Check** = pulse check ("are you alive?")

## Interview Q&A — ELB

**Q: When would you choose ALB vs NLB?**

> "ALB for HTTP/HTTPS web apps and APIs — anything that speaks the web. ALB operates at Layer 7, so it understands paths, headers, host names. This unlocks powerful features: path-based routing to send /api to one service and /admin to another, host-based routing for multi-tenant apps, WebSocket support for real-time features, and integration with Cognito for authentication.
>
> NLB for anything that isn't HTTP or needs extreme performance. Layer 4 — just TCP/UDP with minimal overhead. Handles millions of requests per second with sub-millisecond latency. Use it for gaming servers, financial trading platforms, IoT gateways, or anywhere you need static IP addresses that don't change.
>
> One more distinction: NLB preserves the client's source IP by default, ALB requires X-Forwarded-For header. For apps that need real client IP (rate limiting, geo-blocking), NLB is simpler.
>
> Production example: web app on ALB with SSL termination and path routing, backend microservices communicating internally via NLB for speed."

**Q: Your ALB is returning 502 errors. How do you diagnose?**

> "502 Bad Gateway from ALB means the load balancer received a request but couldn't get a valid response from any backend instance.
>
> Systematic diagnosis: First, check target group health — CloudWatch or console. If targets are unhealthy, that's the immediate problem. Check the health check path — is `/health` returning 200? Common cause: app crashed and can't respond, or app is running but the health endpoint has a bug.
>
> Second, check application logs. `journalctl -u myapp` or CloudWatch Logs. Look for stack traces, connection errors, OOM kills.
>
> Third, verify Security Groups. ALB's security group must allow inbound 443 from 0.0.0.0/0. Instance security group must allow inbound from the ALB security group on the app's port.
>
> Fourth, check the app's response time. ALB has a default idle timeout of 60 seconds — if your app takes 90 seconds to respond, ALB gives up and returns 502. Fix: optimize slow queries or increase idle timeout.
>
> Fifth, if it's intermittent, could be capacity — instances overwhelmed during traffic spike, Auto Scaling not keeping up. Scale up manually and investigate ASG metrics."

**Q: How does path-based routing work in ALB?**

> "You define listener rules that inspect the request path and route to different target groups.
>
> Example: One ALB fronts a monolithic-looking domain but routes to multiple microservices behind the scenes.
>
> - `/api/v1/*` → api-v1 target group (3 EC2 instances running Node.js)
> - `/api/v2/*` → api-v2 target group (5 ECS tasks running Python)
> - `/admin/*` → admin target group (2 EC2 instances running Django)
> - `/*` → frontend target group (S3 bucket via CloudFront, or React on EC2)
>
> Rules are evaluated in priority order. First match wins. Default rule catches anything that doesn't match.
>
> Real benefit: users see one domain, one SSL certificate, one endpoint. Behind the scenes you have multiple independent services, each scalable independently. This is API gateway pattern without needing API Gateway service.
>
> Another use case: canary deployments. Route 10% of `/api/*` traffic to a new target group with the new version. Increase gradually. Rollback = update the rule to send 0%."
