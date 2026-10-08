---
title: PHASE 2 — Jenkins Server Setup
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
order: 8
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-01F-Security-Auditing]]"
next: "[[Boardgame-03-Nexus-Server-Setup]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 2 — Jenkins Server Setup

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-01F-Security-Auditing|PHASE 1F — Security Auditing]] · → [[Boardgame-03-Nexus-Server-Setup|PHASE 3 — Nexus Server Setup]]
<!-- /nav -->

## Jenkins instance was already provisioned earlier (m7i-flex.large)

Started it after overnight stop:

```bash
aws ec2 start-instances --region ap-south-1 --instance-ids i-0c120d94e908a53f2
```

## Full setup script (final corrected version, incorporating all bugs fixed during the session)

```bash
#!/bin/bash
set -e

# 0. Clean stale repo registrations from any prior attempts
sudo rm -f /etc/apt/sources.list.d/jenkins.list
sudo rm -f /usr/share/keyrings/jenkins-keyring.asc

# 1. Update system
sudo apt-get update
sudo apt-get upgrade -y

# 2. Remove docker.io if present (avoid conflict with official Docker repo)
sudo systemctl stop docker 2>/dev/null || true
sudo apt-get purge -y docker.io docker-doc docker-compose podman-docker containerd runc 2>/dev/null || true
sudo apt-get autoremove -y --purge

# 3. Install Docker from official repo (not docker.io)
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

# 4. Install Java 21 (Jenkins current LTS requires 21, NOT 17)
sudo apt install -y fontconfig openjdk-21-jre

# 5. Add Jenkins repo key (2026 key — rotated Dec 2025) and repo
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# 6. Install Jenkins
sudo apt-get update
sudo apt-get install -y jenkins

# 7. Grant jenkins user Docker access (persistent, scoped — NOT chmod 666)
sudo usermod -aG docker jenkins

# 8. Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl restart jenkins

# 9. Install Trivy
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy

echo "✅ Jenkins + Docker (official) + Java 21 + Trivy setup complete."
echo "Jenkins initial admin password:"
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Bugs hit and fixed during Jenkins setup

1. **Stale Jenkins GPG key (jenkins.io-2023.key):** Jenkins rotated their signing key in Dec 2025. The old key URL from tutorials fails with `NO_PUBKEY 7198F4B714ABFC68`. Fixed by using the 2026 key URL.

2. **Java 17 vs Java 21 requirement:** Jenkins installed fine with Java 17, apt reported success, but the service immediately crash-looped (`status=1/FAILURE`, CPU: 79ms — barely ran at all). Root cause only visible in journalctl — Jenkins now requires Java 21 minimum. Silently fatal, no clear error in `systemctl status` output alone.

3. **Docker socket permissions:** tutorials commonly use `chmod 666 /var/run/docker.sock` which is world-writable (any user on the box gets Docker = effectively root access), and doesn't survive Docker daemon restarts. Proper fix: `usermod -aG docker jenkins` — scoped to just the jenkins user, persistent across restarts.

4. **Docker.io vs Docker official repo:** `docker.io` is Ubuntu's repackaged, often-outdated build. The official repo provides `docker-ce` (current), plus `docker-buildx-plugin` and `docker-compose-plugin` which `docker.io` doesn't include at all.

5. **Stale repo file from prior failed attempt:** a leftover `jenkins.list` (with the old broken key reference) from an earlier attempt caused `apt-get update` to fail at step 1, before the script's own key fix at step 5 ever got a chance to run. Fixed by adding a cleanup step (step 0) that removes any stale repo/key files before starting.

## Install kubectl on Jenkins server

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

Why needed: the `withKubeConfig` step in the Jenkinsfile needs the actual `kubectl` binary present on the Jenkins server — the plugin just injects credentials, it doesn't bundle kubectl itself.

## Jenkins initial configuration (via browser)

Accessed at `http://<jenkins-public-ip>:8080`

- Logged in with initial admin password from `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
- Installed suggested plugins
- Created admin user

## Jenkins Global Tool Configuration (Manage Jenkins → Tools)

- JDK: name `jdk17`, auto-install
- Maven: name `maven3`, auto-install
- SonarQube Scanner: name `sonar-scanner`, auto-install
- Docker: name `docker`, Install automatically UNCHECKED, Installation root: `/usr` (points at the system's already-installed docker-ce, not a stale auto-downloaded binary)

Why Docker tool path matters: with "Install automatically" checked, Jenkins auto-downloads its own ancient Docker CLI from a legacy URL (`get.docker.com/builds/...`, API v1.29) — incompatible with the modern daemon (needs API 1.40+). Setting Installation root to `/usr` makes Jenkins use `/usr/bin/docker` (the real, modern docker-ce binary) instead.

## Jenkins Credentials (Manage Jenkins → Credentials → Global)

Created these credentials (all Kind: Username with password, except where noted):

- `git-cred` — GitHub username (aakash-1004) + Personal Access Token (scope: repo)
- `docker-cred` — Docker Hub username (aakash0908) + Docker Hub Access Token
- `sonar-token` — Kind: Secret text — SonarQube token (generated from SonarQube UI → Administration → Security → Users → Tokens)
- `k8s-cred` — Kind: Secret text — Kubernetes ServiceAccount token (from the RBAC secret we create later)

## Jenkins System Configuration (Manage Jenkins → System)

SonarQube servers:

- Name: `sonar`
- URL: `http://<sonarqube-ip>:9000`
- Authentication token: `sonar-token` credential

Extended E-mail Notification:

- SMTP server: `smtp.gmail.com`
- SMTP port: `465` (NOT 25 — port 25 doesn't support SSL, and AWS blocks outbound port 25 by default)
- Use SSL: checked
- SMTP Authentication: username `aakashrao0908@gmail.com`, password: Google App Password (not the real Gmail account password)

## Jenkins Managed Files (Manage Jenkins → Managed files)

Added: Global Maven Settings (`global-settings`) Contains a `<servers>` block with Nexus credentials, whose `<id>` matches the `<distributionManagement>` repository IDs in `pom.xml`.
