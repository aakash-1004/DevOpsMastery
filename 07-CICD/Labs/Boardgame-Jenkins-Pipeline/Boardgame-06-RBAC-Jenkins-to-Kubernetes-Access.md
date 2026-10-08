---
title: "PHASE 6 — RBAC: Jenkins → Kubernetes Deployment Access"
type: project
module: cicd
status: completed
tags:
  - jenkins
  - kubeadm
  - kubernetes
  - rbac
  - sonarqube
  - nexus
  - trivy
  - prometheus
  - grafana
  - blackbox-exporter
  - cicd
  - devops
  - notes
  - project
order: 12
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-05-Source-Code-Private-Repo-Setup]]"
next: "[[Boardgame-07-Jenkinsfile-Writing-Debugging-Correction]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 6 — RBAC: Jenkins → Kubernetes Deployment Access

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-05-Source-Code-Private-Repo-Setup|PHASE 5 — Source Code: Private Repo Setup]] · → [[Boardgame-07-Jenkinsfile-Writing-Debugging-Correction|PHASE 7 — Jenkinsfile: Writing, Debugging, and Correction]]
<!-- /nav -->

All run on the K8s master.

## Create namespace

```bash
kubectl create namespace webapps
```

## Create ServiceAccount

```bash
cat > serviceaccount.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
EOF
kubectl apply -f serviceaccount.yaml
```

## Create Role (namespace-scoped, not ClusterRole)

```bash
cat > role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - pods
      - deployments
      - services
      - replicasets
      - configmaps
      - secrets
      - ingresses
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
EOF
kubectl apply -f role.yaml
```

Why Role (namespace-scoped) not ClusterRole: least-privilege — Jenkins only needs to manage resources inside `webapps`, not the entire cluster.

## Create RoleBinding

```bash
cat > rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: webapps
EOF
kubectl apply -f rolebinding.yaml
```

## Create token Secret (required since K8s 1.24)

```bash
cat > secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  namespace: webapps
  annotations:
    kubernetes.io/service-account.name: jenkins
EOF
kubectl apply -f secret.yaml
```

Why this must be explicitly created: Kubernetes used to auto-generate a token Secret for every ServiceAccount. This behavior was removed in 1.24 as a security hardening measure — now you must explicitly opt in.

## Retrieve token and add to Jenkins

```bash
kubectl get secret mysecretname -n webapps -o jsonpath='{.data.token}' | base64 --decode
```

Copied this JWT-style token string into Jenkins as credential `k8s-cred` (Kind: Secret text).

## Verified network connectivity (Jenkins → K8s API)

From the Jenkins EC2 box:

```bash
curl -k https://172.31.33.140:6443
```

Result: `403 Forbidden` from `system:anonymous` — confirms network path works (a real timeout would mean security group blocking), and RBAC correctly rejects unauthenticated access. The 403 is expected, not an error.
