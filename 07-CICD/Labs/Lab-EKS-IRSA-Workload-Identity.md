---
title: AWS EKS + IRSA — Secure Workload Identity
type: project
module: cicd
status: "Completed — verified working, ALB provisioned and torn down cleanly"
interview-relevance: "High — IRSA/OIDC trust chains are a strong differentiator, directly maps to Keycloak/OIDC background"
tags:
  - kubernetes
  - aws
  - eks
  - iam
  - oidc
  - irsa
  - project
  - cicd
repo: github.com/aakash-1004/eks-irsa
up: "[[CICD-MOC]]"
prev: "[[Lab-GoWebApp-Full-GitOps-Pipeline]]"
related:
  - "[[5.5-EKS-Fargate-IRSA-ALB-Project]]"
  - "[[AWS-03-IAM]]"
  - "[[Lab-GitOps-Go-App-on-EKS]]"
origin: DevOps-Mastery/07-CICD/Labs/AWS EKS + IRSA — Secure Workload Identity.md
---

# AWS EKS + IRSA — Secure Workload Identity

<!-- nav -->
↑ [[CICD-MOC|CI/CD]] · ← [[Lab-GoWebApp-Full-GitOps-Pipeline|Go Web App — Full Containerization to GitOps CI/CD Pipeline]]
<!-- /nav -->

## Overview

Set up IAM Roles for Service Accounts (IRSA) on the same live EKS cluster used for `gitops-go-app`. Goal: get the AWS Load Balancer Controller to authenticate to AWS with zero static credentials, and prove it by having the controller provision a real ALB.

**Full chain:** IAM OIDC Provider -> IAM Policy -> IAM Role (trust policy scoped to one ServiceAccount) -> Kubernetes ServiceAccount annotation -> Pod -> STS AssumeRoleWithWebIdentity -> temporary AWS credentials.

---

## 1. The Problem IRSA Solves

| Approach                    | How it works                                            | Problem                                                                          |
| --------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Static credentials          | AWS access key/secret mounted as a K8s Secret           | Long-lived, doesn't rotate, a leak = full compromise until manually revoked      |
| Instance profile (EC2-wide) | Every pod on the node shares the node's IAM permissions | Over-permissioned - any pod on that node gets the same access, no isolation      |
| IRSA                        | Pod-level identity via OIDC federation                  | Short-lived (auto-rotated), scoped per-ServiceAccount, no static secret anywhere |

**Why it matters:** IRSA is the AWS-native equivalent of workload identity federation - conceptually identical to how Keycloak/OIDC issues short-lived, verifiable tokens instead of static passwords. Same trust-and-token pattern, different ecosystem.

---

## 2. Step 1 - IAM OIDC Provider

```bash
export cluster_name=gitops-go-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

aws iam list-open-id-connect-providers | grep $oidc_id
# (nothing printed - none existed yet)

eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

|Command|What it does|
|---|---|
|`aws eks describe-cluster ... oidc.issuer`|Every EKS cluster has a built-in OIDC issuer URL, created automatically - this pulls it out|
|`aws iam list-open-id-connect-providers \| grep`|Checks whether IAM already trusts this issuer - checked before assuming, since re-registering isn't harmful but isn't necessary either|
|`eksctl utils associate-iam-oidc-provider`|Registers the cluster's OIDC issuer as a trusted external identity provider inside IAM|

**Why it matters:** without this, IAM has no reason to believe any token coming from the cluster. This is the foundational trust anchor - every other step depends on it.

**Real-world use:** confirmed via console at IAM -> Identity providers - one entry matching the cluster's issuer URL exactly (cross-checked against EKS -> Clusters -> Overview -> OpenID Connect provider URL).

---

## 3. Step 2 - IAM Policy (Pure AWS, No Kubernetes Yet)

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Result: `EntityAlreadyExists` - this policy already existed from an earlier attempt. Fetched its ARN instead of recreating:

```bash
aws iam list-policies --query \
  "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn" --output text
```

**Why it matters:** this step has nothing to do with Kubernetes - it's a plain IAM policy document defining exactly which AWS actions are allowed (`elasticloadbalancing:*`, relevant `ec2:Describe*` calls). Least-privilege: scoped to only what the ALB controller needs, nothing broader.

---

## 4. Step 3 - The Actual IRSA Binding

```bash
eksctl create iamserviceaccount \
  --cluster=gitops-go-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**One command, four things happen:**

|What it creates|Detail|
|---|---|
|IAM Role|`AmazonEKSLoadBalancerControllerRole`|
|Trust policy on that role|"Only ServiceAccount `kube-system/aws-load-balancer-controller`, authenticated via the OIDC provider from Step 1, may assume this role"|
|Attached permissions|The policy from Step 2|
|Kubernetes ServiceAccount|Created and annotated with the role's ARN|

**Verified the binding directly:**

```bash
kubectl get serviceaccount aws-load-balancer-controller -n kube-system -o yaml
```

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/AmazonEKSLoadBalancerControllerRole
```

**Why it matters:** that annotation is the literal handshake. It tells EKS's built-in Pod Identity Webhook to automatically inject AWS credentials into any pod that mounts this ServiceAccount - the pod's code never needs to know this is happening; the AWS SDK just finds the credentials in the expected location.

**Real-world use:** checked the IAM Role's Trust relationships tab in the console - the JSON trust policy shows the `Federated` principal (the OIDC provider ARN) and a `Condition` block matching the exact `system:serviceaccount:kube-system:aws-load-balancer-controller` string. This is the single most concrete artifact for explaining IRSA in an interview - it's not abstract, it's a literal JSON condition you can point to.

---

## 5. Step 4 - Deploy the Controller Using the IRSA Identity

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=gitops-go-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<vpc-id>
```

|Flag|Why|
|---|---|
|`serviceAccount.create=false`|Tells the Helm chart NOT to make its own ServiceAccount - use the IRSA-bound one from Step 3 instead|
|`region` / `vpcId`|Controller needs to know which VPC to operate in to create ALBs/target groups correctly|

**Verification:**

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
# 2/2 Running, 0 restarts
```

**Why "0 restarts" matters:** a broken trust policy or a missing IAM permission shows up immediately as `CrashLoopBackOff` with an AWS `AccessDenied` error in the pod logs. A clean `Running` state on first deploy is itself the proof that Steps 1-3 were wired correctly - no separate "did it work" check was needed at this stage.

---

## 6. Step 5 - Proving It: A Real ALB, Not Just a Running Pod

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: eks-sample-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitops-go-app
                port:
                  number: 80
```

|Detail|Why|
|---|---|
|`ingressClassName: alb`|Routes to the AWS Load Balancer Controller specifically, not the existing `ingress-nginx` - two ingress controllers coexisting in the same cluster, each owning its own class|
|`scheme: internet-facing`|Without this, ALB defaults to `internal` (VPC-only) - would create a working ALB with no public reachability|
|Points at `gitops-go-app` Service|See debugging note below - not the original plan|

**Result:**

```bash
kubectl get ingress eks-sample-ingress
# ADDRESS: k8s-default-ekssampl-....ap-south-1.elb.amazonaws.com
curl http://k8s-default-ekssampl-....elb.amazonaws.com
# GitOps Go App v3 - built by Aakash Rao
```

A pod, authenticated via IRSA with zero static credentials, called the AWS API and provisioned a real, internet-facing ALB that correctly routed live traffic.

---

## Real Debugging Log (chronological)

1. **First attempt used a standalone nginx sample app** (`eks-sample-linux-deployment`, 3 replicas) - got stuck `Pending` indefinitely.
2. **Diagnosed via `kubectl describe pod`:**

    ```
    0/2 nodes are available: 2 Too many pods.
    ```

    Not a CPU/memory issue - this is an EC2 instance-type limit: max pods per node is capped by available IP addresses per network interface, not resource requests. `t3.small` nodes were already near that ceiling from ingress-nginx, all of ArgoCD (7 pods), cert-manager (3 pods), the new ALB controller (2 pods), and `gitops-go-app` (2 pods).
3. **Decision point:** resize/add nodes just to run a throwaway test app, or find another way to prove the same thing? Chose the latter - deleted the stuck deployment/service, rewrote the Ingress to point at the already-running `gitops-go-app` Service instead. Same proof (IRSA -> ALB Controller -> real ALB), zero new scheduling pressure.
4. **New ALB came up correctly**, but `curl` from bumblebee worked before the browser did - ALB target-group health checks take a short warm-up window after target registration before marking targets healthy, even though the pods were already running. Not a config bug - just needed 1-2 minutes.
5. **Cleaned up:** `kubectl delete -f ingress.yaml` - confirmed the ALB Controller automatically deallocates the ALB, listener, and target group in AWS. No manual EC2 console cleanup needed.

---

## Interview-Ready Spoken Answer

"I set up IRSA on EKS for the AWS Load Balancer Controller. The chain is: an IAM OIDC provider that trusts the cluster's token issuer, an IAM policy scoped to exactly what the controller needs, and an IAM role whose trust policy only allows one specific Kubernetes ServiceAccount to assume it. `eksctl create iamserviceaccount` does the role creation and the ServiceAccount binding in one step. Once the controller pod uses that ServiceAccount, it gets temporary AWS credentials through STS AssumeRoleWithWebIdentity - no static keys anywhere. I verified it wasn't just running but actually working by creating an Ingress with `ingressClassName: alb` and confirming the controller provisioned a real, internet-facing ALB that routed live traffic. I also hit a real scheduling limit along the way - my test pods got stuck Pending because t3.small nodes have a hard cap on pods per node tied to available IPs, not CPU or memory - so instead of resizing the cluster for a throwaway test, I pointed the Ingress at a Service that was already running and got the same proof with no extra infrastructure."

---

## Related notes

- [[5.5-EKS-Fargate-IRSA-ALB-Project|IRSA and OIDC Trust Chains]]
- [[5.5-EKS-Fargate-IRSA-ALB-Project|AWS Load Balancer Controller]]
- [[AWS-03-IAM|IAM Policies and Least Privilege]]
- STS AssumeRoleWithWebIdentity
- Kubernetes - Pod Scheduling Limits
- EKS - Cloud Controller Manager

<!-- related -->
---
**Related:** [[5.5-EKS-Fargate-IRSA-ALB-Project|EKS on Fargate + IRSA + ALB Ingress Controller — End to End]] · [[AWS-03-IAM|IAM — Identity & Access Management]] · [[Lab-GitOps-Go-App-on-EKS|GitOps Go App — End-to-End Pipeline on AWS EKS]]
<!-- /related -->
