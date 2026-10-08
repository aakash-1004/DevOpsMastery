---
title: Lab — CI/CD Pipeline for Taskmanager
aliases:
  - Taskmanager-CICD-Lab
type: lab
module: cicd
status: ✅ Completed
tags:
  - cicd
  - github-actions
  - docker
  - lab
  - day6
  - golden-thread
repo: "https://github.com/aakash-1004/taskmanager"
docker-hub: "https://hub.docker.com/r/aakash0908/taskmanager"
up: "[[CICD-MOC]]"
prev: "[[7.4.2-Jenkins-Declarative-Pipeline-Anatomy]]"
next: "[[Lab-GitOps-Go-App-on-EKS]]"
related:
  - "[[7.1-GitHub-Actions]]"
origin: DevOps-Mastery/07-CICD/Labs/Taskmanager-CICD-Lab.md
---

# Lab — CI/CD Pipeline for Taskmanager

<!-- nav -->
↑ [[CICD-MOC|CI/CD]] · ← [[7.4.2-Jenkins-Declarative-Pipeline-Anatomy|Jenkins — Declarative Pipeline Anatomy]] · → [[Lab-GitOps-Go-App-on-EKS|GitOps Go App — End-to-End Pipeline on AWS EKS]]
<!-- /nav -->

## What Was Built

A complete CI/CD pipeline using GitHub Actions:
- Automated test run on every push to main
- Docker image built and pushed to Docker Hub only if tests pass
- Pipeline gate proven — broken test stops the build
- Image available publicly at `aakash0908/taskmanager:latest`

---

## Files Created

```
taskmanager/
├── .github/
│   └── workflows/
│       └── ci.yml       ← pipeline definition
└── test_app.py          ← pytest tests
```

---

## ci.yml — Full Pipeline

```yaml
name: CI — Build and Push Docker Image

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.12'

    - name: Install dependencies
      run: pip install -r requirements.txt

    - name: Run tests
      run: pytest test_app.py -v

  build-and-push:
    runs-on: ubuntu-latest
    needs: test
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/taskmanager:latest
```

---

## test_app.py

```python
import pytest
from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_health(client):
    response = client.get('/health')
    assert response.status_code == 200
    assert response.json['status'] == 'ok'
```

---

## GitHub Secrets Added

| Secret Name | Value |
|-------------|-------|
| `DOCKERHUB_USERNAME` | `aakash0908` |
| `DOCKERHUB_TOKEN` | Docker Hub access token (Read & Write) |

---

## Pipeline Runs

| Run | Trigger | Result | Duration |
|-----|---------|--------|---------|
| #1 | Initial push | ✅ Success | 24s |
| #2 | Added test step | ✅ Success | 42s |
| #3 | Intentional test failure | ❌ test failed, build skipped | 17s |
| #4 | Reverted fix | ✅ Success | ~40s |

---

## Pipeline Gate Demo

**Broke test intentionally:**
```python
assert response.json['status'] == 'broken'  # wrong
```

**Result:**
```
test job      → ❌ Failed (12s)
build-and-push → ⬛ Skipped (never ran)
```

Docker Hub image was NOT updated — bad code was blocked.

**Key lesson:** `needs: test` is the gate. Without it, build runs regardless of test result.

---

## How the Runner Works

1. GitHub spins up a fresh Ubuntu VM on their infrastructure
2. Runner clones your repo
3. Executes each step in order
4. VM is destroyed after job completes
5. No state carries over between runs

This is why CI is reliable — every run starts from a clean slate.

---

## Verify Image on Docker Hub

```bash
docker pull aakash0908/taskmanager:latest
docker run -p 5000:5000 aakash0908/taskmanager:latest
```

---

## Bugs / Gotchas Hit

| Issue | Cause | Fix |
|-------|-------|-----|
| Push rejected for ci.yml | PAT missing `workflow` scope | Regenerate token with `workflow` + `repo` scopes |
| Divergent branches on pull | Different commits on local vs remote | `git pull origin main --no-rebase` |
| Untracked file conflict | `.dockerignore` existed locally + remotely | `rm .dockerignore` then pull |
| Staged but not committed | `git add` without `git commit` | Always commit before push |
| Docker token Read-only | Wrong permission on access token | Regenerate with Read & Write |

---

## What's Missing (Coming Later)

- **CD step** — auto-deploy to K8s after image push (ArgoCD)
- **Multi-environment** — separate pipelines for dev/staging/prod
- **Image tagging** — tag with git SHA instead of `latest`
- **Notifications** — Slack alert on pipeline failure

---

## Related notes
- [[7.1-GitHub-Actions]]
- [[4.2-Docker-Core-Concepts]]
- [[3.1-Git-Workflow]]
- [[5.1-Kubernetes-Core-Concepts]]

<!-- related -->
---
**Related:** [[7.1-GitHub-Actions|GitHub Actions — CI/CD]]
<!-- /related -->
