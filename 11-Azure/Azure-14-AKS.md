---
title: AKS — Azure Kubernetes Service
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
order: 14
up: "[[Azure-MOC]]"
prev: "[[Azure-13-Azure-CDN]]"
next: "[[Azure-15-Networking-VNets-Subnets-NSGs]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# AKS — Azure Kubernetes Service

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-13-Azure-CDN|Azure CDN]] · → [[Azure-15-Networking-VNets-Subnets-NSGs|Networking — VNets, Subnets, NSGs]]
<!-- /nav -->

**This is the deepest section — Kubernetes is your strongest area, and AKS mastery is a huge multiplier for interviews at MNC roles.**

## Concept

Azure Kubernetes Service is managed Kubernetes. Azure runs the control plane, you manage node pools (or use Virtual Nodes / Fargate-like for serverless).

## AWS Equivalent

**AKS = AWS EKS**

Same category, similar responsibility split (managed control plane, self-managed or managed worker nodes). Different pricing model, different integrations, different defaults.

## Key Differences from AWS

**1. Control Plane Is FREE**

EKS: $73/month per cluster for the control plane. AKS: **free** for standard tier control plane (paid tier for uptime SLA guarantee available but not required).

**Practical impact:** Multi-cluster architecture is cheaper on Azure. Teams that would consolidate on one big EKS cluster to save $73/month can freely run per-environment AKS clusters — one for dev, one for staging, one for prod — with no control-plane cost overhead.

**2. Node Pool Structure**

Both AKS and EKS use "node pools" (AKS terminology) / "managed node groups" (EKS). Very similar. Key AKS features:
- **System node pool** — mandatory, runs system pods (CoreDNS, kube-proxy, metrics-server). Kept separate from user workloads.
- **User node pools** — for your app workloads. Multiple pools with different VM sizes, taints, labels.
- **Spot node pools** — mix spot into an existing cluster easily
- **Virtual Nodes** — serverless Fargate-equivalent via Azure Container Instances

**3. Entra ID Integration Is Native**

`kubectl` authentication via Entra ID with RBAC mapped to Kubernetes RBAC. Users sign in with their corporate identity, get short-lived tokens, cluster access governed by Entra ID group membership.

EKS supports similar via AWS IAM + `aws-auth` ConfigMap, but AKS integration is smoother because Entra ID is a proper identity provider (whereas AWS IAM is more resource-focused).

**4. Azure CNI vs Kubenet**

Two networking modes:
- **Kubenet** — pods get IPs from internal network, translated at node boundary. Fewer IPs consumed, simpler. Default in older AKS versions.
- **Azure CNI** — pods get real VNet IPs. Direct routing, better integration with other Azure services, uses more IPs. Modern default.

For interviews: know both exist, know Azure CNI is the modern choice, understand the IP consumption trade-off (Azure CNI can burn through subnet IPs fast with many pods).

**5. AGIC — Application Gateway Ingress Controller**

Native integration between AKS and Application Gateway. Instead of running an nginx ingress controller inside the cluster, AKS spawns/manages an Application Gateway that acts as the cluster ingress.

Trade-off: more Azure-native, deeper integration with WAF, no ingress controller pods to maintain. But more Azure-locked and Application Gateway has some Kubernetes-specific limitations to watch.

Alternative: Nginx Ingress Controller or Traefik running inside AKS — same as any Kubernetes, more portable.

## Real Configuration

```bash
# Create resource group
az group create --name rg-aks-prod-eastus --location eastus

# Create AKS cluster with:
# - System node pool: 3 nodes across 3 AZs (Standard_D2s_v5)
# - Azure CNI networking
# - Entra ID integration with Azure RBAC for Kubernetes
# - Managed identity for cluster
# - Cluster autoscaler enabled
az aks create \
  --resource-group rg-aks-prod-eastus \
  --name aks-prod-eastus \
  --location eastus \
  --kubernetes-version 1.29 \
  --node-count 3 \
  --node-vm-size Standard_D2s_v5 \
  --zones 1 2 3 \
  --network-plugin azure \
  --enable-managed-identity \
  --enable-aad \
  --enable-azure-rbac \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --generate-ssh-keys \
  --tier standard

# Get credentials (updates ~/.kube/config)
az aks get-credentials \
  --resource-group rg-aks-prod-eastus \
  --name aks-prod-eastus \
  --admin  # or --admin for legacy, prefer without for Entra ID auth

# Verify
kubectl get nodes -o wide

# Add a user node pool (for app workloads, tainted so system pods don't schedule here)
az aks nodepool add \
  --resource-group rg-aks-prod-eastus \
  --cluster-name aks-prod-eastus \
  --name userpool \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20 \
  --labels workload=general \
  --node-taints workload=general:NoSchedule

# Add a spot node pool for batch workloads (up to 90% discount)
az aks nodepool add \
  --resource-group rg-aks-prod-eastus \
  --cluster-name aks-prod-eastus \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-count 2 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 30 \
  --labels workload=batch \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule

# Enable AGIC (App Gateway Ingress Controller)
az aks enable-addons \
  --resource-group rg-aks-prod-eastus \
  --name aks-prod-eastus \
  --addons ingress-appgw \
  --appgw-name agw-aks-prod \
  --appgw-subnet-cidr 10.2.0.0/16
```

## Workload Identity — Managed Identity for Pods

The Azure answer to AWS IRSA (IAM Roles for Service Accounts) is **Workload Identity**. Pods get Entra ID identities via a Kubernetes ServiceAccount → federated identity credential.

Flow:
1. Enable Workload Identity on the cluster (or during create with `--enable-workload-identity`)
2. Create a Managed Identity in Azure
3. Grant it Azure permissions (RBAC role assignments)
4. Create a Kubernetes ServiceAccount annotated with the identity's client ID
5. Configure a federated credential linking the identity to the ServiceAccount
6. Pods using that ServiceAccount get Azure tokens automatically — no secrets

```yaml
# Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: "xxx-xxx-xxx-xxx"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: myapp
        image: myacr.azurecr.io/myapp:v1
```

Pod code uses Azure SDK, SDK reads the projected token, calls Azure APIs. No client_secret, no managed identity endpoint, cleanest pattern available.

## Production Use Case

**Multi-tenant AKS with proper zone and workload isolation:**

```
AKS Cluster: aks-prod-eastus (Kubernetes 1.29, Standard tier, uptime SLA)
├── Networking: Azure CNI, dedicated VNet subnet (10.2.0.0/16)
├── Entra ID auth + Azure RBAC for Kubernetes RBAC
│
├── Node pools:
│   ├── systempool (3× Standard_D2s_v5, zones 1/2/3, cluster autoscale 3-5)
│   │   └── Only kube-system workloads
│   ├── generalpool (3× Standard_D4s_v5, zones 1/2/3, autoscale 3-20)
│   │   └── Most app workloads
│   ├── memorypool (2× Standard_E4s_v5, zones 1/2/3, autoscale 2-10)
│   │   └── Redis, ElasticSearch, other memory-hungry
│   └── spotpool (0-30× D4s_v5 Spot, autoscale 0-30)
│       └── Batch jobs, cost-tolerant workloads
│
├── Addons:
│   ├── AGIC → Application Gateway with WAF
│   ├── Azure Monitor for containers (Container Insights)
│   ├── Azure Key Vault CSI provider (mount secrets as volumes)
│   ├── Workload Identity for pod-level Entra IDs
│   └── Cluster Autoscaler + Vertical Pod Autoscaler
│
├── Ingress: AGIC → Application Gateway (WAF policy attached)
├── Cert-manager for automated TLS
├── ArgoCD for GitOps deployments
└── Prometheus + Grafana via kube-prometheus-stack Helm chart
```

## Console Path

- Portal → **Kubernetes services** (search)
- Cluster detail:
  - **Overview** — status, node count, K8s version, endpoint
  - **Node pools** — manage pools, scaling, upgrades
  - **Configuration** → **Cluster configuration** — cluster autoscaler, workload identity, etc.
  - **Networking** — CNI info, ingress, service mesh
  - **Security** → **Defender for Cloud** for runtime security
  - **Monitoring** → Container Insights, Live metrics, Logs
  - **Kubernetes resources** → Namespaces, Workloads, Services (in-portal view, similar to K8s dashboard)

## Memory Hint

- **AKS control plane = FREE** (vs EKS $73/month) — enables multi-cluster designs
- **Node pools** → System (required, kube-system) + User (your apps) + optional Spot
- **Azure CNI** = real VNet IPs for pods (modern default)
- **Entra ID + Azure RBAC for Kubernetes** = corporate identity → cluster access
- **Workload Identity** = IRSA equivalent, pod-level Entra IDs
- **AGIC** = Application Gateway as native ingress, avoids nginx-controller pods

## Interview Q&A — AKS

**Q: You've worked with EKS. Walk me through migrating to AKS — what changes, what surprises?**

> "The Kubernetes layer is identical — kubectl, YAMLs, Helm charts, operators all work the same. That's the whole point of Kubernetes as a standard. What changes is everything around it: identity, networking, ingress, secrets, monitoring.
>
> Identity: EKS uses IAM + IRSA (IAM Roles for Service Accounts) for pod-level auth. AKS uses Entra ID + Workload Identity. Same pattern, different plumbing. You rewrite your IRSA setup as Workload Identity — annotate ServiceAccounts with Entra ID client IDs, configure federated credentials.
>
> Networking: EKS defaults to AWS VPC CNI, giving pods real VPC IPs. AKS defaults to Azure CNI, giving pods real VNet IPs. Similar model — plan subnet IP capacity accordingly. Kubenet is the older simpler alternative in AKS; nobody uses it for production anymore.
>
> Ingress: EKS commonly uses AWS Load Balancer Controller creating ALBs. AKS has AGIC — Application Gateway Ingress Controller — same idea, App Gateway is the ingress. Or run standard nginx-ingress if you want portability.
>
> Secrets: EKS with External Secrets Operator pulling from Secrets Manager. AKS has native CSI driver for Key Vault — annotate a SecretProviderClass, secrets get mounted as volumes. Simpler than the ESO pattern.
>
> Monitoring: EKS integrates with CloudWatch Container Insights. AKS integrates with Azure Monitor Container Insights. Same category, different UIs.
>
> Cost surprise: AKS control plane is free (Standard tier). EKS is $73/month per cluster. This changes the calculus for multi-cluster architectures — teams that consolidated on EKS for cost can spread out on AKS.
>
> Practical migration path: rebuild the platform layer (identity, ingress, monitoring, secrets) natively for Azure. Application manifests port over largely unchanged. Budget 2-4 weeks for platform, then app-by-app cutover."

**Q: Design a cost-optimized AKS cluster for a workload with steady baseline and unpredictable spikes.**

> "Three-tier node pool strategy plus autoscaling.
>
> Tier one: system pool. Three Standard_D2s_v5 nodes across three AZs. Only kube-system pods here. Cluster autoscaler between 3 and 5 for headroom. Small and always-on.
>
> Tier two: general pool. Standard_D4s_v5 for baseline app capacity. Autoscale 3 to 15 based on demand. This carries the steady traffic. Consider Azure Reserved Instances or Savings Plans for these — 1-year commit gets you 40-50% off, worth it because these nodes run 24/7.
>
> Tier three: spot pool. Standard_D4s_v5 spot instances, autoscale from 0 to 30. This absorbs spikes cheaply — up to 90% discount. Tainted so only workloads that tolerate eviction schedule here.
>
> Workload placement:
> - Critical low-latency services → general pool (nodeSelector or preferred affinity)
> - Batch processing, background workers, ML training → spot pool with tolerations
> - System services → system pool (auto-scheduled by taint)
>
> HPA on each deployment based on CPU or custom metrics. VPA for right-sizing pod requests over time.
>
> Cluster autoscaler runs cluster-wide, scales pools independently. When spikes hit, spot pool grows fast (cheap capacity). If spot evictions happen, HPA can push some pods back to general pool temporarily.
>
> Cost model: baseline general pool is Reserved (steady discount), spot pool for elastic capacity (deep discount), system pool is fixed overhead. Combined, you might see 50-60% savings vs running all on-demand.
>
> Monitoring: Azure Cost Analysis tagged per node pool. Track spot eviction rate — if it's too disruptive, tune eviction policy or diversify VM sizes in the spot pool."

**Q: A pod in AKS can't reach an Azure SQL Database. Walk me through debugging.**

> "Systematic, network-outward.
>
> First layer, pod itself: `kubectl exec -it <pod> -- /bin/sh` then try DNS resolution — `nslookup sql-prod-aakash.database.windows.net`. If DNS fails, CoreDNS or Azure DNS integration issue. If DNS returns the SQL server public IP, that's fine but we probably want private endpoint.
>
> Second layer, network connectivity: from the pod, `nc -zv <sql-server-fqdn> 1433` or curl equivalent. Times out? Firewall or NSG blocking. Connection refused? Server not listening. No response at all? Check both directions of NSG rules on the subnet where nodes live.
>
> Third layer, Azure SQL firewall: SQL Server has its own firewall independent of NSGs. Check if the pod's egress IP is allowed. If AKS uses default outbound, all pods share the cluster's outbound public IP — allowlist that. Better pattern: SQL private endpoint in the AKS subnet, DNS resolves to the private IP, firewall becomes moot.
>
> Fourth layer, auth: pod tries to connect using what credentials? If it's a connection string with SQL auth, verify the username/password against the SQL server directly with sqlcmd. If it's Managed Identity via Workload Identity, verify the service principal exists in the SQL server as an Entra ID user with correct permissions. `SELECT * FROM sys.database_principals` in the target database will confirm.
>
> Fifth layer, application: is the driver / connection string correct? Sometimes it's not the network at all — it's the app trying wrong port, wrong TLS setting, wrong AAD authentication mode.
>
> One trap specific to AKS: if you're using Azure CNI with restricted outbound (e.g., forced tunneling through Azure Firewall), the SQL FQDN might need to be explicitly allowed. Azure Firewall logs will show whether the connection attempted and what happened.
>
> The mental model is the same as any Kubernetes + external DB debug — but Azure adds specific layers (SQL firewall, Entra ID auth, private endpoints) worth knowing individually. Interview signal: walking through this systematically shows you know Azure networking, not just K8s."
