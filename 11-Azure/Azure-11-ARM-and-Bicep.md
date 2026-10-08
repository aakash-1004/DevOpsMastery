---
title: "ARM & Bicep"
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
order: 11
up: "[[Azure-MOC]]"
prev: "[[Azure-10-Azure-Functions]]"
next: "[[Azure-12-Azure-DNS]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# ARM & Bicep

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-10-Azure-Functions|Azure Functions]] · → [[Azure-12-Azure-DNS|Azure DNS]]
<!-- /nav -->

## Concept

Azure Resource Manager (ARM) is the deployment and management layer for all Azure resources. ARM templates are JSON files declaring desired infrastructure. Bicep is a domain-specific language that transpiles to ARM — same power, way better ergonomics.

## AWS Equivalent

**ARM ≈ AWS CloudFormation (raw JSON/YAML) but ARM is chattier**
**Bicep ≈ AWS CDK or CloudFormation with cleaner syntax**

## Key Differences from AWS

**1. ARM Templates Are Verbose**
Even a simple VM in ARM JSON is 100+ lines with heavy nesting. CloudFormation YAML is more readable. Bicep fixes this.

**2. Bicep Is First-Party**
Microsoft ships Bicep as the recommended IaC language for Azure. Not a community tool — official product with tooling, extensions, and documentation. AWS's equivalent is CDK (multi-language SDKs generating CloudFormation).

**3. Terraform Is the Practical Cross-Cloud Choice**
For anyone working across clouds or with existing Terraform investment, use Terraform. Both Bicep and Terraform are legitimate on Azure. Bicep for Azure-only shops, Terraform for multi-cloud or existing HCL codebases.

## Bicep vs ARM JSON — Same Resource

**ARM JSON (the old way):**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "stdevopsaakash01",
      "location": "eastus",
      "sku": { "name": "Standard_LRS" },
      "kind": "StorageV2",
      "properties": {
        "accessTier": "Hot"
      }
    }
  ]
}
```

**Bicep (the new way, same result):**
```bicep
resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stdevopsaakash01'
  location: 'eastus'
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}
```

Bicep is dramatically cleaner. Type-checked. IntelliSense in VS Code. Compiles to ARM JSON at deploy time — Azure only executes ARM under the hood, Bicep is a syntactic layer.

## Real Configuration

```bash
# Install Bicep CLI (bundled with Azure CLI)
az bicep install
az bicep version

# Author main.bicep, then deploy
az deployment group create \
  --resource-group rg-iac-lab \
  --template-file main.bicep \
  --parameters environment=dev location=eastus

# Preview changes (like terraform plan)
az deployment group what-if \
  --resource-group rg-iac-lab \
  --template-file main.bicep \
  --parameters environment=dev location=eastus

# Convert existing ARM JSON to Bicep
az bicep decompile --file template.json
# Produces template.bicep — great for learning from Microsoft's quickstart samples
```

## Bicep vs Terraform — Real Talk

| | Bicep | Terraform |
|---|---|---|
| Scope | Azure only | Multi-cloud |
| State | Managed by Azure (ARM) | State file (you manage) |
| Language | DSL, Azure-native | HCL, universal |
| Learning curve | Small if you know ARM | Steeper but transferable |
| Ecosystem | Growing, Azure-focused | Massive, mature |
| Best for | Azure-only shops | Multi-cloud, larger teams |

**My recommendation for you specifically:** learn both, prefer Terraform for portfolio. Terraform skills transfer to AWS, GCP, and third-party providers (GitHub, Datadog, Cloudflare). Bicep is Azure-locked. But knowing Bicep matters for Azure interviews because you might inherit a Bicep codebase.

## Production Use Case

**Multi-environment infrastructure with Bicep modules:**

```
infra/
├── main.bicep                    # entry point
├── modules/
│   ├── networking.bicep          # VNet, subnets, NSGs
│   ├── compute.bicep             # VMSS, App Gateway
│   ├── data.bicep                # SQL, Cosmos
│   └── monitoring.bicep          # Log Analytics, Application Insights
└── environments/
    ├── dev.bicepparam            # dev-specific parameters
    ├── staging.bicepparam
    └── prod.bicepparam
```

Deploy dev: `az deployment group create --parameters environments/dev.bicepparam`
Deploy prod: `az deployment group create --parameters environments/prod.bicepparam`

Same templates, environment-specific parameters. Analogous to Terraform's tfvars.

## Console Path

- Portal → **Deployments** (under resource group) — history of ARM/Bicep deployments
- **Templates** → export any existing resource as ARM JSON (great for reverse-engineering)
- **Deploy a custom template** (search) → paste ARM/Bicep, deploy from portal

## Memory Hint

- **ARM = raw JSON, verbose, don't write it by hand**
- **Bicep = clean DSL, transpiles to ARM, first-party**
- **Terraform = multi-cloud alternative, use if you're not Azure-only**
- Bicep decompile from ARM JSON is a killer learning technique

## Interview Q&A — ARM & Bicep

**Q: When would you choose Bicep over Terraform for Azure infrastructure?**

> "Depends on team context and multi-cloud aspirations.
>
> Bicep for: Azure-only shops with no plans to go multi-cloud. Teams who value tighter integration with Azure — new preview services get Bicep support immediately, Terraform sometimes lags by weeks. Teams that want state management to be Azure's problem, not theirs.
>
> Terraform for: multi-cloud teams. Teams with existing HCL codebases. Teams valuing a larger ecosystem and community modules. Teams that want cross-provider composition — provisioning Azure resources AND GitHub secrets AND Datadog monitors from one apply.
>
> My default recommendation is Terraform for most modern teams, because 'Azure-only forever' is rare in enterprise reality, and Terraform skills carry across clouds. But if a team is genuinely Azure-committed and values the first-party experience, Bicep is a fine choice — dramatically better than raw ARM.
>
> One anti-pattern: raw ARM JSON. Nobody should be hand-writing that in 2026. If you inherit an ARM codebase, plan a Bicep migration — `az bicep decompile` handles most of the conversion automatically.
>
> Interview signal: candidates who dogmatically pick one without acknowledging context are junior. Senior answer is 'here's how I'd decide.'"
