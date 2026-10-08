---
title: "PHASE 1A — Kubernetes Cluster: EC2 Provisioning"
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
order: 2
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-00-Planning-and-Infrastructure-Decisions]]"
next: "[[Boardgame-01B-K8s-Node-Setup]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 1A — Kubernetes Cluster: EC2 Provisioning

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-00-Planning-and-Infrastructure-Decisions|PHASE 0 — Planning & Infrastructure Decisions]] · → [[Boardgame-01B-K8s-Node-Setup|PHASE 1B — Kubernetes Cluster: Node Setup (All 3 Nodes)]]
<!-- /nav -->

## Launched 3 instances for the K8s cluster

```bash
# Master node
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-0aa761682283b4cc8 \
  --instance-type c7i-flex.large \
  --key-name myWebServer \
  --security-group-ids sg-09c50e5f8249e552e \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8s-master}]' \
  --query "Instances[0].InstanceId" --output text
# Result: i-04e788d52fb81aef9

# Slave-1
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-0aa761682283b4cc8 \
  --instance-type t3.small \
  --key-name myWebServer \
  --security-group-ids sg-09c50e5f8249e552e \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8s-slave-1}]' \
  --query "Instances[0].InstanceId" --output text
# Result: i-00dd691370aa08e55

# Slave-2
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-0aa761682283b4cc8 \
  --instance-type t3.small \
  --key-name myWebServer \
  --security-group-ids sg-09c50e5f8249e552e \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8s-slave-2}]' \
  --query "Instances[0].InstanceId" --output text
# Result: i-03f2096f48784f669
```

Why 20GB root volumes: default is often 8GB, too small for Docker images + system packages + kubeadm components combined.

## Added self-referencing security group rule for inter-node traffic

```bash
aws ec2 authorize-security-group-ingress \
  --region ap-south-1 \
  --group-id sg-09c50e5f8249e552e \
  --protocol all \
  --source-group sg-09c50e5f8249e552e
```

What this does: allows ALL traffic between any two instances sharing this same security group, on any port/protocol. This covers kubelet (10250), etcd (2379-2380), Calico CNI ports, and any other inter-node communication — without manually enumerating every port individually. Does NOT affect inbound rules from the internet (those remain scoped to specific ports like 22, 80, 3000-10000 as already configured).

## Fetched all instance IPs

```bash
aws ec2 describe-instances --region ap-south-1 \
  --instance-ids i-04e788d52fb81aef9 i-00dd691370aa08e55 i-03f2096f48784f669 \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,State:State.Name,PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress}" \
  --output table
```

Result:

- k8s-master: private 172.31.33.140, public 13.232.63.245
- k8s-slave-1: private 172.31.6.84, public 13.204.67.146
- k8s-slave-2: private 172.31.7.252, public 13.203.77.37

Why both IPs matter: SSH in using public IPs; kubeadm cluster communication uses private IPs (traffic stays inside AWS's internal network, faster and more secure).
