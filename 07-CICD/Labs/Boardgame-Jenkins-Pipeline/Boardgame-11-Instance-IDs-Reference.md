---
title: Instance IDs — Full Reference
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
order: 17
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-10-Repo-Polish]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# Instance IDs — Full Reference

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-10-Repo-Polish|PHASE 10 — Repo Polish]]
<!-- /nav -->

|Server|Instance ID|Type|
|---|---|---|
|Jenkins|i-0c120d94e908a53f2|m7i-flex.large|
|Nexus|i-0ca0c755453e7ce6e|c7i-flex.large|
|SonarQube|i-0ecf03e9ec37e1224|m7i-flex.large|
|K8s master|i-04e788d52fb81aef9|c7i-flex.large|
|K8s slave-1|i-00dd691370aa08e55|t3.small|
|K8s slave-2|i-03f2096f48784f669|t3.small|
|Monitoring|i-0964449c2cec9ab33|c7i-flex.large|

All in ap-south-1, all using security group sg-09c50e5f8249e552e (myWebServer), all using key pair myWebServer.

## Stop all instances (to pause billing)

```bash
aws ec2 stop-instances --region ap-south-1 --instance-ids \
  i-0c120d94e908a53f2 i-0ca0c755453e7ce6e i-0ecf03e9ec37e1224 \
  i-04e788d52fb81aef9 i-00dd691370aa08e55 i-03f2096f48784f669 \
  i-0964449c2cec9ab33
```

Note on resume: public IPs will change on next start (no Elastic IPs allocated). Private IPs persist. Prometheus config (prometheus.yml) references public IPs for Blackbox/Jenkins targets and will need updating after restart. The Jenkinsfile's serverUrl uses the master's private IP (172.31.33.140) which is safe across stop/start.
