---
title: "Revision — CI/CD: GitHub Actions & Jenkins"
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
order: 8
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-07-Terraform]]"
next: "[[Revision-09-GitOps-and-ArgoCD]]"
related:
  - "[[CICD-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — CI/CD: GitHub Actions & Jenkins

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-07-Terraform|Revision — Terraform (IaC)]] · → [[Revision-09-GitOps-and-ArgoCD|Revision — GitOps & ArgoCD]]
<!-- /nav -->

## Concept

CI/CD is the backbone of DevOps. CI (Continuous Integration) automates build + test on every commit. CD (Continuous Delivery/Deployment) automates delivery to environments.

**Why it matters:** Without CI/CD, deployments are manual, risky, and infrequent. With CI/CD, deployments are automatic, tested, and happen multiple times a day.

---

## GitHub Actions

**Trigger on push to main, run tests, build Docker image, push to Docker Hub:**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOCKER_IMAGE: aakash0908/flask-app

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest --tb=short

  build-and-push:
    needs: test                         # Only runs if test passes — pipeline gate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' # Only on main branch, not PRs
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.DOCKER_IMAGE }}:${{ github.sha }},${{ env.DOCKER_IMAGE }}:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            docker pull ${{ env.DOCKER_IMAGE }}:latest
            docker stop flask-app || true
            docker rm flask-app || true
            docker run -d --name flask-app -p 80:5000 \
              --env-file /home/ubuntu/.env \
              ${{ env.DOCKER_IMAGE }}:latest
```

---

## Jenkins

**Declarative Pipeline:**

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "aakash0908/flask-app:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/aakash-1004/flask-mongo-app.git'
            }
        }

        stage('Test') {
            steps {
                sh 'pip install -r requirements.txt'
                sh 'pytest --tb=short'
            }
        }

        stage('Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker build -t $DOCKER_IMAGE .
                        docker push $DOCKER_IMAGE
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} "
                            docker pull ${DOCKER_IMAGE} &&
                            docker stop flask-app || true &&
                            docker rm flask-app || true &&
                            docker run -d --name flask-app -p 80:5000 ${DOCKER_IMAGE}
                        "
                    '''
                }
            }
        }
    }

    post {
        failure { echo "Pipeline FAILED" }
        success { echo "Pipeline SUCCEEDED" }
    }
}
```

---

## Senior Interview Q&A

**Q: How do you handle secrets in a CI/CD pipeline?**

> "Several layers.
>
> For GitHub Actions: GitHub Secrets (Settings → Secrets and variables → Actions). They're encrypted, not visible in logs (shown as `***`), and not passed to forked PRs by default — that last point is security-critical.
>
> For Jenkins: Credentials store (Manage Jenkins → Credentials). Use `withCredentials` block in pipeline. Never echo secrets or set -x in the block.
>
> For both: never hardcode secrets in the pipeline YAML/Jenkinsfile. Those files are in source control. Rotate secrets regularly. Use different secrets per environment.
>
> Best practice I push for: replace static credentials with OIDC-based authentication where possible. GitHub Actions can assume an AWS IAM role via OIDC — no AWS keys stored as GitHub Secrets at all. The GitHub Actions runtime presents a JWT to AWS, AWS validates it and issues short-lived credentials. Zero long-lived secrets."

---

**Q: Your deployment pipeline takes 40 minutes. How do you optimize it?**

> "Audit each stage for parallelism and caching opportunities.
>
> First: parallelize independent stages. Tests, linting, security scanning — run them concurrently. In GitHub Actions that's separate jobs with `needs: []`. In Jenkins it's `parallel {}` blocks. This alone can cut time in half.
>
> Second: cache dependencies. In GitHub Actions: `actions/cache` for pip, npm, Maven. If dependencies haven't changed, restore from cache instead of reinstalling. Dependency install going from 5 minutes to 30 seconds is common.
>
> Third: optimize Docker builds. Multi-stage builds, `.dockerignore` to exclude unnecessary files, and layer ordering so the expensive `pip install` layer is only rebuilt when `requirements.txt` changes.
>
> Fourth: use self-hosted runners for builds. GitHub-hosted runners cold-start every job. A self-hosted runner on a powerful machine eliminates startup time and network latency for pulling Docker images.
>
> Fifth: split test suites. If you have 5000 unit tests, split them into parallel shards. GitHub Actions matrix strategy is perfect for this.
>
> I've taken a 40-minute pipeline to 8 minutes with parallelism + caching alone."

---

**Q: Difference between GitHub Actions and Jenkins — when would you choose each?**

> "GitHub Actions is the right choice for most teams today. It's cloud-native, zero infrastructure to maintain, deep GitHub integration — PRs get automatic status checks, you can gate merges on pipeline green. For teams already on GitHub, the path of least resistance.
>
> Jenkins is the right choice when you need: on-premises builds (sensitive code can't go to cloud runners), complex enterprise integrations (Jira, Artifactory, LDAP), highly customized agents (specific hardware, GPUs, network access), or a mature organization that already has Jenkins infrastructure.
>
> Jenkins overhead is real: you maintain the master node, plugins, security updates, agent provisioning. For a 10-person team, that's a part-time job. For a 1000-person enterprise with specialized needs, it pays for itself.
>
> The modern compromise: GitHub Actions for most pipelines, self-hosted runners on your own infrastructure when you need control, still zero Jenkins maintenance."

<!-- related -->
---
**Related:** [[CICD-MOC|CI/CD]]
<!-- /related -->
