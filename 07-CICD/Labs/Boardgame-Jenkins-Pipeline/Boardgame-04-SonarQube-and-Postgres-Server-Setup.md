---
title: PHASE 4 — SonarQube + Postgres Server Setup
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
order: 10
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-03-Nexus-Server-Setup]]"
next: "[[Boardgame-05-Source-Code-Private-Repo-Setup]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 4 — SonarQube + Postgres Server Setup

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-03-Nexus-Server-Setup|PHASE 3 — Nexus Server Setup]] · → [[Boardgame-05-Source-Code-Private-Repo-Setup|PHASE 5 — Source Code: Private Repo Setup]]
<!-- /nav -->

## Launched instance

```bash
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-0aa761682283b4cc8 \
  --instance-type m7i-flex.large \
  --key-name myWebServer \
  --security-group-ids sg-09c50e5f8249e552e \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=sonarqube-server}]' \
  --query "Instances[0].InstanceId" --output text
# Result: i-0ecf03e9ec37e1224
```

## Setup script

```bash
#!/bin/bash
set -e

sudo apt-get update
sudo apt-get upgrade -y

# Set required kernel parameter for SonarQube's embedded Elasticsearch
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system

# Install Docker (official repo — same steps as other servers)
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

# Create dedicated Docker network for SonarQube + Postgres
sudo docker network create sonarnet

# Run Postgres (external DB — SonarQube's embedded H2 is unsupported for real use)
sudo docker run -d --name sonarqube_db --network sonarnet --restart unless-stopped \
  -e POSTGRES_USER=sonar \
  -e POSTGRES_PASSWORD=sonar \
  -e POSTGRES_DB=sonarqube \
  -v postgresql_data:/var/lib/postgresql/data \
  postgres:15

# Run SonarQube, pointed at Postgres
sudo docker run -d --name sonarqube --network sonarnet --restart unless-stopped -p 9000:9000 \
  -e SONAR_JDBC_URL=jdbc:postgresql://sonarqube_db:5432/sonarqube \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=sonar \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:lts-community
```

Why vm.max_map_count: SonarQube bundles Elasticsearch, which has a hard documented requirement for this kernel parameter to be >= 262144. Docker containers inherit this from the HOST kernel, not settable inside the container. Without it, SonarQube starts then immediately crashes with an Elasticsearch bootstrap-check failure.

Why external Postgres instead of embedded H2: SonarSource's own docs mark H2 as "for evaluation only" and unsupported for production. Deliberate, documented production-hardening choice, diverging from the tutorial on purpose.

Why docker network create sonarnet: creates an isolated Docker network so SonarQube and Postgres can communicate by container name (`sonarqube_db:5432`) without exposing Postgres to the host network or needing manual IP management.

Why postgres:15 (pinned version, not :latest): avoids a moving-target version that could unexpectedly break compatibility.

## SonarQube initial configuration (via browser at :9000)

- Default login: admin / admin (forced password change on first login)
- Verified it's using Postgres (Administration → System → database info shows PostgreSQL, not H2)
- Generated an authentication token (Administration → Security → Users → Tokens) — this is what gets stored in Jenkins as `sonar-token`
