---
title: Azure Functions
type: reference
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
order: 10
up: "[[Azure-MOC]]"
prev: "[[Azure-09-Cosmos-DB]]"
next: "[[Azure-11-ARM-and-Bicep]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Azure Functions

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-09-Cosmos-DB|Cosmos DB]] · → [[Azure-11-ARM-and-Bicep|ARM & Bicep]]
<!-- /nav -->

## Concept

Azure Functions is serverless compute — run code in response to events without managing servers.

## AWS Equivalent

**Azure Functions = AWS Lambda**

Same category, similar developer experience. Different hosting plans and pricing model.

## Key Differences from AWS

**1. Hosting Plans (Not Just One Model)**

AWS Lambda: one pricing model — requests + duration. That's it.

Azure Functions: three hosting plans:

| Plan | AWS Analogue | Use When |
|---|---|---|
| **Consumption** | Lambda default | Sporadic events, pay-per-execution |
| **Premium** | Lambda Provisioned Concurrency | Warm instances, VNet integration, no cold starts |
| **App Service Plan** | (no direct AWS parallel) | Function runs on your existing Web App VMs — you already pay for them |

For most workloads → Consumption. Latency-sensitive → Premium. Already using App Service → App Service plan.

**2. Bindings — Declarative Integration**

Lambda uses SDK calls in code to interact with S3, DynamoDB, etc.

Functions has **input and output bindings** — declare in config that "this function's input is a Blob at this path, output is a Cosmos document." The runtime handles I/O plumbing. Less code, more declarative.

Example — a function triggered by blob upload, writing to Cosmos:
```json
{
  "bindings": [
    {
      "type": "blobTrigger",
      "direction": "in",
      "name": "myBlob",
      "path": "uploads/{name}"
    },
    {
      "type": "cosmosDB",
      "direction": "out",
      "name": "outDoc",
      "databaseName": "AppDB",
      "collectionName": "processed"
    }
  ]
}
```
Your function code just receives `myBlob` as a parameter and returns a value assigned to `outDoc`. Runtime handles reading blob and writing to Cosmos.

**3. Durable Functions**

Serverless stateful orchestration — chained functions, fan-out/fan-in, human interaction, timers. AWS equivalent is Step Functions, but Durable Functions are code-first (write orchestrator in C#/Python/JS) vs Step Functions' JSON state machines. Different flavor of the same idea.

**4. Function App Wraps Multiple Functions**

Lambda: one function per resource.
Functions: a "Function App" contains multiple related functions sharing runtime, config, and plan. Feels more like a small microservice with several routes/triggers.

## Real Configuration

```bash
# Create a storage account (Functions needs one for state)
az storage account create \
  --name stfuncdevops01 \
  --resource-group rg-func-lab \
  --location eastus \
  --sku Standard_LRS

# Create a Function App on Consumption plan (Python 3.11)
az functionapp create \
  --resource-group rg-func-lab \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name func-devops-aakash \
  --storage-account stfuncdevops01 \
  --os-type Linux

# Deploy code (from local project directory)
func azure functionapp publish func-devops-aakash

# Enable Managed Identity for the function (best practice)
az functionapp identity assign \
  --name func-devops-aakash \
  --resource-group rg-func-lab

# Grant it access to a Cosmos DB
az cosmosdb sql role assignment create \
  --account-name cosmos-prod-aakash \
  --resource-group rg-prod-db-eastus \
  --role-definition-id 00000000-0000-0000-0000-000000000002 \
  --principal-id <function-managed-identity-principal-id> \
  --scope /
```

## Production Use Case

**Image processing pipeline (same as classic Lambda example):**

```
User uploads image → Blob Storage (uploads container)
    ↓ (blob trigger)
Function: process-image
    ├── Reads blob, resizes to thumbnail + medium + large
    ├── Writes 3 blobs to processed container
    ├── Writes metadata to Cosmos DB via output binding
    └── Sends notification via Service Bus output binding
    ↓
Function: send-email
    ├── Triggered by Service Bus message
    ├── Sends email via SendGrid connector
    └── Logs to Application Insights
```

Both functions live in one Function App on Consumption plan. Pay only for actual invocations. Managed Identity for Cosmos access, no keys.

## Console Path

- Portal → **Function App** (search)
- Function App detail:
  - **Functions** blade → list of functions in this app
  - **Configuration** → app settings (env vars), connection strings
  - **Identity** → enable/manage Managed Identity
  - **Deployment Center** → CI/CD from GitHub/Azure DevOps
  - **Networking** → VNet integration, private endpoints (Premium+ plans only)
  - **Log stream** → live log tail
  - Individual function → **Code + Test** for in-portal editing

## Memory Hint

- **Function App** = container for one or more Functions (unlike Lambda's per-function model)
- **Consumption** = pay-per-execution (default choice)
- **Premium** = pre-warmed + VNet (no cold starts)
- **Bindings** = declarative I/O (less code than SDK calls)
- **Durable Functions** = Step Functions equivalent, code-first

## Interview Q&A — Azure Functions

**Q: A team is moving from AWS Lambda to Azure Functions. What's the mental shift?**

> "Three main shifts.
>
> First, hosting plans matter. Lambda is one thing — Azure has Consumption (pay-per-execution, default), Premium (warm, VNet integration, no cold starts), and App Service (runs on existing web app compute). Consumption is closest to Lambda default; picking the right plan for a workload is a decision Lambda doesn't force.
>
> Second, bindings. Lambda code explicitly calls AWS SDK. Functions can declare bindings — 'my input is this blob, my output goes to Cosmos DB' — and the runtime handles the I/O. It's declarative vs imperative. Some teams love it, some prefer explicit SDK calls. Once you internalize bindings, they reduce a lot of boilerplate.
>
> Third, Function Apps group functions. Lambda is one function per resource. In Azure, a Function App is a deployable unit containing multiple related functions sharing runtime, settings, and hosting plan. Feels more like a microservice than a single-purpose function.
>
> Cold starts: similar issue in both. Python and Node cold-start fast (100-300ms). C# and Java slower (1-2s). Mitigation on Azure is the Premium plan, on AWS it's Provisioned Concurrency — same idea, different plumbing.
>
> Cost model: Consumption charges per execution + GB-seconds — nearly identical to Lambda. Premium is a per-vCPU-hour flat rate — different math, need to compare based on actual invocation volume."

**Q: When would you use Durable Functions vs a chain of regular Functions?**

> "Durable Functions when the workflow has state, orchestration, or long-running steps that outlast a single function invocation.
>
> Regular chained Functions are stateless — one triggers the next via a queue or event. Fine for simple 'A → B → C' pipelines where each step is independent.
>
> Durable Functions shines with patterns like:
>
> Function chaining with state — execute A, wait for result, use it to call B, then C, all with in-memory state that persists.
>
> Fan-out/fan-in — kick off 100 parallel tasks, wait for all to complete, aggregate results. Regular Functions would need external state store to track completion.
>
> Human interaction — start a workflow, send approval email, pause until human clicks approve, then continue. Regular Functions can't 'wait' for hours or days.
>
> Long-running orchestrations — workflows that need to run for hours, days, or weeks with state preserved.
>
> Timers — schedule a step to run 3 days later without external scheduler.
>
> AWS equivalent is Step Functions with its JSON-based state machine language. Durable Functions writes the orchestrator in code (C#, Python, JS) — feels more natural to developers. Different style, same category.
>
> Interview signal: candidates who default to Durable Functions for everything are over-engineering. Reach for it when the workflow genuinely needs orchestration or state. Simple chain? Regular Functions with queue triggers is fine."
