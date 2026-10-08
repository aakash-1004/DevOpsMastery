---
title: Lambda — Serverless Compute
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
order: 12
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-11-DynamoDB]]"
next: "[[AWS-13-CloudFormation]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# Lambda — Serverless Compute

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-11-DynamoDB|DynamoDB — NoSQL]] · → [[AWS-13-CloudFormation|CloudFormation — IaC]]
<!-- /nav -->

## Concept

**AWS Lambda** is a serverless computing service — run code in response to events without managing servers.

**You upload code. AWS handles:**

- Server provisioning
- Scaling (from 0 to thousands of concurrent executions)
- Availability
- OS patching
- Monitoring

**You pay only for:**

- Number of executions
- Duration of each execution (in milliseconds)

**No charge when idle.** Empty Lambda = $0/month.

## When to Use Lambda

**Perfect for event-driven, short-lived tasks:**

```
1. Image Processing
   User uploads image → Lambda triggered → resizes/compresses → saves back to S3

2. Data Transformation
   Data arrives in S3 → Lambda cleans it → writes to DynamoDB/RDS

3. Real-Time Notifications
   User signs up → Lambda sends welcome email via SES

4. Scheduled Tasks (Cron-like)
   Every hour → CloudWatch Events triggers Lambda → runs cleanup script

5. API Backend
   API Gateway → Lambda → returns response
   (Serverless REST APIs)

6. Stream Processing
   Kinesis stream → Lambda processes each record → stores results

7. Chatbots
   Slack/Teams webhook → Lambda handles command → posts response
```

## Event-Driven Execution

**Lambda runs in response to triggers.** Common event sources:

```
S3               → File uploads, deletions
DynamoDB Streams → Table changes (inserts, updates, deletes)
API Gateway      → HTTP requests
CloudWatch       → Scheduled events (cron), alarms
SNS              → Message published to topic
SQS              → Message in queue
Kinesis          → Stream records
EventBridge      → Custom events, third-party integrations
CodeCommit       → Git push
Cognito          → User signup, login
```

## Automatic Scaling

- **1 request:** 1 Lambda instance runs
- **1,000 concurrent requests:** 1,000 Lambda instances run in parallel
- **No configuration needed** — AWS handles it

**Concurrency limit:** 1,000 concurrent executions per region by default (can request increase).

## Pay-as-You-Go Pricing

**Billed on:**

- **Requests:** $0.20 per 1 million requests (first 1M free per month)
- **Duration:** $0.0000166667 per GB-second

**Example:** Lambda with 512MB memory, runs for 200ms, called 1 million times/month:

- Requests: $0.20
- Duration: 1M × 0.5GB × 0.2s × $0.0000167 = ~$1.67
- **Total: ~$1.87/month for 1 million executions**

**Comparison:** Same workload on EC2 (t3.small running 24/7) = ~$15/month + you manage the server.

## Supported Languages

```
Node.js (JavaScript)
Python
Java
C# (.NET Core)
Go
Ruby
Custom Runtime API (bring any language via container image)
```

**Most popular:** Python and Node.js for their fast cold start and rich ecosystem.

## Lambda Limits

```
Memory:           128 MB to 10,240 MB (10 GB)
CPU:              Scales with memory (proportionally)
Timeout:          15 minutes maximum
Deployment size:  50 MB zipped, 250 MB unzipped
Container image:  10 GB
Concurrent execs: 1,000 default (soft limit)
Temp storage:     512 MB (/tmp) — expandable to 10 GB
```

## Cold Start vs Warm Start

**Cold Start:** First invocation of a Lambda (or after idle period). AWS spins up a new container. Takes 100ms - 2 seconds depending on runtime.

**Warm Start:** Reusing an existing container. Sub-millisecond overhead.

**Mitigation for critical low-latency apps:**

- **Provisioned Concurrency**: keep N containers always warm (extra cost)
- **Language choice**: Python and Node.js have faster cold starts than Java or .NET

## Real Production Use Cases

**Image Processing Pipeline:**

```
User uploads image to S3 bucket "raw-images"
    ↓
S3 event triggers Lambda "image-processor"
    ↓
Lambda:
    ├── Reads image from S3
    ├── Resizes to multiple sizes (thumbnail, medium, large)
    ├── Compresses
    ├── Applies watermark
    └── Uploads results to S3 bucket "processed-images"
    ↓
DynamoDB record updated with URLs
    ↓
SNS notification sent to user
```

**Serverless API:**

```
Client HTTP request
    ↓
API Gateway (REST or HTTP API)
    ↓
Lambda function (handles route)
    ↓
DynamoDB (data storage)
    ↓
Response back to client

Zero servers to manage. Scales from 1 to millions of requests.
```

**Scheduled Cleanup:**

```
CloudWatch Events (every day at 2AM)
    ↓
Lambda "cleanup-old-data"
    ↓
Scans DynamoDB for records older than 90 days
    ↓
Archives to S3 Glacier
    ↓
Deletes from DynamoDB
```

## Memory Hints

- **Lambda** = "code that runs when something happens"
- **Serverless** = "no server for YOU to manage" (AWS still uses servers)
- **Cold Start** = "first-time wake up" (small delay)
- **Concurrency** = "how many can run at the same time"

## Interview Q&A — Lambda

**Q: When would you choose Lambda over EC2 or containers?**

> "Lambda is ideal for event-driven, short-lived, sporadic workloads. Three main scenarios:
>
> First, event processing: S3 uploads, DynamoDB stream changes, API calls, scheduled tasks. Lambda's event integration is native — no polling, no infrastructure setup.
>
> Second, unpredictable traffic: workloads that go from 0 to 1000s of requests unpredictably. Lambda scales instantly with zero management. On EC2 you'd over-provision to handle spikes, wasting money during idle.
>
> Third, cost optimization for low-traffic APIs: if your API gets 100 requests per day, running EC2 24/7 is wasteful. Lambda costs pennies for the same workload.
>
> When NOT to use Lambda: long-running processes (>15 min timeout), stateful workloads, applications requiring persistent connections (WebSockets — though Lambda + API Gateway can do this), or workloads with strict latency requirements where cold starts matter.
>
> Modern architecture: use Lambda for event handlers and light APIs, ECS/EKS for stateful services and heavy compute, EC2 for legacy or specialized needs. Lambda often glues these together."

**Q: How do you handle Lambda cold starts?**

> "Cold starts happen when Lambda has to spin up a new container to handle a request. Duration varies by runtime — Python and Node.js are around 100-300ms, Java can be 1-2 seconds because JVM initialization.
>
> Strategies: First, optimize package size. Smaller deployment = faster init. Remove unnecessary dependencies, use Lambda Layers for shared libraries.
>
> Second, choose runtime carefully. If cold start matters, use Python or Node.js over Java/.NET.
>
> Third, use Provisioned Concurrency for critical low-latency workloads. Keeps N containers pre-warmed at all times. Costs extra but eliminates cold starts. Good for user-facing APIs where 2-second cold start is unacceptable.
>
> Fourth, warm-up strategy: schedule CloudWatch Events to invoke Lambda every 5 minutes to keep containers warm. Cheaper than Provisioned Concurrency but less reliable.
>
> Fifth, architectural pattern: use SQS or SNS in front of Lambda for async workloads where cold start doesn't matter. Queue absorbs the burst, Lambda processes at its natural pace.
>
> Real-world: for most workloads, cold start doesn't matter — the first user might see 300ms extra, subsequent users are instant. Optimize only when you measure it as a problem."

**Q: How does Lambda pricing work?**

> "Two components: requests and duration.
>
> Requests: $0.20 per million requests. First 1 million free per month.
>
> Duration: measured in GB-seconds. Memory allocated × execution time. Priced at $0.0000167 per GB-second. First 400,000 GB-seconds free per month.
>
> Example calculation: Lambda with 1GB memory runs for 500ms, called 1M times per month: Requests: 1M × $0.20/M = $0.20 Duration: 1M × 1GB × 0.5s = 500K GB-seconds. First 400K free, pay for 100K. 100K × $0.0000167 = $1.67 Total: about $1.87/month.
>
> Compare to always-on EC2 running the same workload — $15-20/month plus operational overhead. For event-driven workloads with idle time, Lambda is often 10x cheaper.
>
> Cost optimization tips: allocate the RIGHT amount of memory. More memory = more CPU = faster execution = potentially cheaper. Use AWS Lambda Power Tuning tool to find optimal memory setting. Also, minimize package size to reduce cold start duration, and cache values across invocations (module-level variables persist between warm invocations)."
