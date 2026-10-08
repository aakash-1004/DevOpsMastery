---
title: PHASE 3 — Nexus Server Setup
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
order: 9
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-02-Jenkins-Server-Setup]]"
next: "[[Boardgame-04-SonarQube-and-Postgres-Server-Setup]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 3 — Nexus Server Setup

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-02-Jenkins-Server-Setup|PHASE 2 — Jenkins Server Setup]] · → [[Boardgame-04-SonarQube-and-Postgres-Server-Setup|PHASE 4 — SonarQube + Postgres Server Setup]]
<!-- /nav -->

## Launched instance

```bash
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-0aa761682283b4cc8 \
  --instance-type c7i-flex.large \
  --key-name myWebServer \
  --security-group-ids sg-09c50e5f8249e552e \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nexus-server}]' \
  --query "Instances[0].InstanceId" --output text
# Result: i-0ca0c755453e7ce6e
```

## Setup script

```bash
#!/bin/bash
set -e

sudo apt-get update
sudo apt-get upgrade -y

# Install Docker (official repo)
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER

# Run Nexus with persistent volume
sudo docker run -d -p 8081:8081 --name nexus --restart unless-stopped -v nexus-data:/nexus-data sonatype/nexus3
```

Why `-v nexus-data:/nexus-data`: persistent volume — without it, all uploaded artifacts and Nexus config vanish if the container is ever recreated.

Why `--restart unless-stopped`: auto-restarts the container after host reboots — without it, EC2 stop/start would bring the host back but Nexus itself would stay down.

## Get initial admin password

```bash
sudo docker exec nexus cat /nexus-data/admin.password
```

Takes 1-2 minutes after first container start before this file exists (Nexus is slow to initialize).

## Nexus initial configuration (via browser at :8081)

- Logged in with admin + the initial password
- Set new admin password during setup wizard
- Created maven-releases and maven-snapshots repositories (if not already present by default)
