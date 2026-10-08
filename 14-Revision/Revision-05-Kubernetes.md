---
title: Revision — Kubernetes
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 5
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-04-Docker]]"
next: "[[Revision-06-AWS-Core-Services]]"
related:
  - "[[Kubernetes-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Kubernetes

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-04-Docker|Revision — Docker]] · → [[Revision-06-AWS-Core-Services|Revision — AWS Core Services]]
<!-- /nav -->

## Concept

Kubernetes is a container orchestration platform. It solves the problem that Docker alone doesn't: running containers at scale across multiple machines with health management, scaling, load balancing, rolling updates, and self-healing.

**The core principle:** You declare desired state. Kubernetes continuously reconciles actual state to match desired state. This is the reconciliation loop.

**Architecture:**

```
Control Plane:
  kube-apiserver      → All communication goes through here (the brain)
  etcd                → Distributed key-value store: source of truth for all cluster state
  kube-scheduler      → Decides which node a pod runs on
  kube-controller-manager → Runs controllers (ReplicaSet, Node, Endpoint, etc.)
  cloud-controller-manager → Cloud-provider specific (AWS, GCP, Azure)

Worker Nodes:
  kubelet             → Agent on each node: ensures containers are running
  kube-proxy          → Implements Service networking (iptables/IPVS rules)
  Container runtime   → containerd (or CRI-O) — actually runs containers
```

**Pod — the atomic unit:** A Pod is one or more containers sharing a network namespace (same IP) and storage. All containers in a Pod communicate via localhost.

---

## Core Commands

```bash
# Cluster info
kubectl cluster-info                  # API server and CoreDNS endpoints
kubectl get nodes                     # List nodes and their status
kubectl top nodes                     # CPU/memory usage per node
kubectl describe node <node>         # Detailed node info (capacity, allocations)

# Pod management
kubectl get pods -n <namespace>      # List pods in namespace
kubectl get pods -A                  # All pods across all namespaces
kubectl describe pod <pod>           # Detailed pod info + Events (crucial for debugging)
kubectl logs <pod>                   # Pod logs
kubectl logs <pod> -c <container>    # Logs from specific container in pod
kubectl logs <pod> --previous        # Logs from crashed previous instance
kubectl exec -it <pod> -- /bin/bash  # Shell into pod
kubectl delete pod <pod>             # Delete pod (Deployment will recreate it)

# Apply/Delete resources
kubectl apply -f deployment.yaml     # Create or update resource from file
kubectl delete -f deployment.yaml    # Delete resources defined in file
kubectl apply -f ./k8s/             # Apply all YAML files in a directory

# Deployments
kubectl get deployments             # List deployments
kubectl rollout status deployment/<name>   # Check rollout progress
kubectl rollout history deployment/<name>  # Show revision history
kubectl rollout undo deployment/<name>     # Rollback to previous version
kubectl set image deployment/<name> <container>=<image:tag>  # Update image

# Scaling
kubectl scale deployment <name> --replicas=5   # Manual scale
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=70  # HPA

# Services
kubectl get svc                      # List services
kubectl port-forward svc/<name> 8080:80  # Forward local port to service

# ConfigMaps & Secrets
kubectl create configmap myconfig --from-file=./config/   # From files
kubectl create secret generic mysecret --from-literal=DB_PASS=secret123
kubectl get secret mysecret -o jsonpath='{.data.DB_PASS}' | base64 -d  # Decode

# RBAC
kubectl get roles,rolebindings -n <ns>        # List RBAC in namespace
kubectl auth can-i get pods --as=dev-user     # Test permissions

# Debugging
kubectl get events --sort-by=.metadata.creationTimestamp  # All events sorted
kubectl run debug --image=busybox --rm -it -- sh          # Throw away debug pod
```

---

## Key YAML Manifests

**Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask
          image: aakash0908/flask-app:1.2
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: flask-config
            - secretRef:
                name: flask-secrets
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 3
```

**Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-svc
  namespace: production
spec:
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 5000
  type: ClusterIP       # ClusterIP | NodePort | LoadBalancer
```

**HPA:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Helm

```bash
helm repo add stable https://charts.helm.sh/stable   # Add a chart repo
helm repo update                                      # Refresh repo index
helm search repo nginx                                # Search for charts

helm install myrelease ./mychart                     # Install from local chart
helm install myrelease ./mychart -f custom-values.yaml  # Override values
helm upgrade myrelease ./mychart -f custom-values.yaml  # Upgrade
helm rollback myrelease 1                            # Rollback to revision 1
helm history myrelease                               # Revision history
helm uninstall myrelease                             # Uninstall (remove all resources)

helm template ./mychart                             # Render YAML without installing (debugging)
helm lint ./mychart                                 # Validate chart syntax
```

**Helm chart structure:**

```
mychart/
├── Chart.yaml           # Chart metadata (name, version, appVersion)
├── values.yaml          # Default values
├── templates/
│   ├── deployment.yaml  # Uses {{ .Values.image.tag }}
│   ├── service.yaml
│   ├── _helpers.tpl     # Template helpers/partials
│   └── NOTES.txt        # Post-install instructions
```

---

## Senior Interview Q&A

**Q: What happens when you `kubectl apply -f deployment.yaml`?**

> "Full lifecycle:
>
> 1. kubectl serializes the YAML and sends a PUT/PATCH HTTP request to the kube-apiserver.
> 2. kube-apiserver authenticates (your kubeconfig credentials), authorizes (RBAC check), then validates the schema.
> 3. The API server writes the desired state to etcd.
> 4. The Deployment controller (inside kube-controller-manager) is watching for Deployment changes via the watch API. It detects the new/updated Deployment.
> 5. The Deployment controller creates or updates a ReplicaSet to match the desired replica count.
> 6. The ReplicaSet controller creates Pod objects.
> 7. The scheduler watches for unscheduled pods (pods with no `nodeName`). It runs its scheduling algorithm: filters nodes that meet resource requests, then ranks remaining nodes. It assigns a node by setting `nodeName` on the Pod.
> 8. The kubelet on the selected node is watching for pods assigned to its node. It sees the new pod, calls containerd to pull the image and start the container.
> 9. kubelet reports back container status to the API server.
> 10. kube-proxy on all nodes detects service endpoint changes and updates iptables rules accordingly.
>
> All of this happens in seconds. It's a beautiful system of independent controllers all watching etcd and reacting."

---

**Q: Pod is in CrashLoopBackOff. Walk me through your diagnosis.**

> "CrashLoopBackOff means the container started, ran, crashed, and Kubernetes has restarted it enough times that it's now adding exponential backoff delay before trying again.
>
> Step 1: `kubectl describe pod <pod> -n <namespace>`. Look at the Events section — this is gold. You'll often see the exact error: OOMKilled, image pull error, readiness probe failed, config mount error.
>
> Step 2: `kubectl logs <pod> --previous`. The `--previous` flag gets logs from the crashed instance, not the current (potentially-starting) one. This shows the actual crash reason.
>
> Common causes and fixes:
>
> - **OOMKilled** → container hit its memory limit, kernel killed it. Fix: increase memory limit or fix memory leak.
> - **Missing env var** → app fails to start because required config isn't there. Check your ConfigMap/Secret references.
> - **Command not found** → entrypoint binary doesn't exist. Check your Dockerfile CMD/ENTRYPOINT.
> - **Permission denied** → app can't read a mounted secret or config file. Check file permissions, check if running as non-root.
> - **Readiness probe failure** → probe path doesn't exist or app isn't ready in time. Adjust initialDelaySeconds.
>
> If logs don't help: `kubectl exec -it <pod> --image-override busybox -- sh` to get into the container environment and inspect manually."

---

**Q: Explain the difference between liveness and readiness probes.**

> "They solve different problems.
>
> **Readiness probe:** Is this pod ready to receive traffic? If readiness fails, Kubernetes removes the pod's endpoint from the Service — it stops sending it new requests, but doesn't kill the container. This is used for: graceful startup (don't send traffic until the app has loaded its cache), graceful degradation (temporarily remove a pod from rotation if its backend is down, add it back when the backend recovers).
>
> **Liveness probe:** Is this pod alive? If liveness fails repeatedly, Kubernetes kills and restarts the container. This handles deadlocks, infinite loops — situations where the process is running but effectively stuck and will never recover.
>
> Common mistake I've seen: people set liveness probes too aggressively. If `initialDelaySeconds` is too low and your app takes 20 seconds to start, liveness will kill it before it ever becomes healthy, creating a death loop.
>
> **Startup probe** (the third one, often forgotten): Is this pod still starting? It's used for slow-starting apps to prevent liveness from killing them during initialization. Once startup probe succeeds, liveness and readiness take over."

---

**Q: How does Kubernetes networking work? How does a request reach a pod?**

> "Let's trace a request from outside the cluster to a pod.
>
> External request → LoadBalancer (cloud LB, e.g., AWS NLB) → NodePort on any cluster node → ClusterIP Service → Pod IP.
>
> ClusterIP is a virtual IP. It doesn't exist on any interface. kube-proxy on every node writes iptables DNAT rules: 'traffic to ClusterIP:port, NAT to one of these pod IPs'. The kernel intercepts the packet before routing and rewrites the destination to a real pod IP. The reply goes back NAT'd to the original source.
>
> Pod-to-pod networking: every pod gets a unique IP from the pod CIDR. The CNI plugin (Flannel, Calico, Cilium) ensures pods on different nodes can reach each other — typically via overlay networking (VXLAN encapsulation) or BGP route advertisement.
>
> Calico uses BGP: each node advertises 'I have pods in subnet X' to other nodes. No encapsulation overhead — packets route natively. Flannel uses VXLAN: encapsulates pod packets in UDP datagrams to cross node boundaries.
>
> This is why every pod needs a unique IP cluster-wide: because there's no NAT between pods, and routing requires unique addresses."

---

**Q: How does Helm differ from just `kubectl apply`?**

> "kubectl apply gives you CRUD on Kubernetes resources. Helm gives you release management.
>
> With kubectl apply: you have a bunch of YAML files. There's no concept of grouping, versioning, or rollback. If you `kubectl delete` a deployment, it's gone. There's no 'which version was deployed and what did it look like' history.
>
> Helm treats a set of Kubernetes resources as a single release. It stores release history in Secrets in the cluster. `helm rollback myrelease 1` doesn't just undo one resource — it rolls back the entire set of resources to the exact state of revision 1. That includes Deployments, Services, ConfigMaps, RBAC — everything in the chart.
>
> Helm also handles templating: the same chart deploys to dev with 1 replica and `debug` image tag, and to production with 10 replicas and `1.5.2` image tag — just different values files.
>
> When NOT to use Helm: for one-off debugging or simple single-resource deploys. The overhead isn't worth it. For anything production and multi-resource, Helm is the right tool."

<!-- related -->
---
**Related:** [[Kubernetes-MOC|Kubernetes]]
<!-- /related -->
