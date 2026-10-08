---
title: Lab — Python DevOps Scripts
aliases:
  - Python-DevOps-Scripts
type: lab
module: python
status: ✅ Completed
tags:
  - python
  - devops
  - lab
  - day1
files: "`~/aakash/devops/test_subprocess.py`, `test_yaml.py`, `test_requests.py`"
up: "[[Python-DevOps-MOC]]"
prev: "[[2.3-Python-Subprocess]]"
next: "[[Lab-Flask-MongoDB-REST-API]]"
related:
  - "[[2.1-JSON-YAML-Parsing]]"
  - "[[2.2-Python-Requests]]"
origin: DevOps-Mastery/02-Python-DevOps/Labs/Python-DevOps-Scripts.md
---

# Lab — Python DevOps Scripts

<!-- nav -->
↑ [[Python-DevOps-MOC|Python for DevOps]] · ← [[2.3-Python-Subprocess|Python — subprocess]] · → [[Lab-Flask-MongoDB-REST-API|Lab — Flask + MongoDB REST API]]
<!-- /nav -->

## Setup

```bash
cd ~/aakash/devops
python3 -m venv venv
source venv/bin/activate   # run this every new terminal session
pip install pyyaml requests
```

**Note:** venv needs to be activated every new terminal. You'll know it's active when prompt shows `(venv)`.

---

## Script 1 — Memory Usage via subprocess

```python
import subprocess

def get_memory_usage():
    result = subprocess.run(["free", "-m"], capture_output=True, text=True, check=True)
    fields = result.stdout.strip().split("\n")[1].split()
    total = int(fields[1])
    used = int(fields[2])
    return round(used / total * 100)

print(f"Memory usage: {get_memory_usage()}%")
```

**Output:** `Memory usage: 46%`

---

## Script 2 — YAML Manifest Parsing

```python
import yaml

manifest = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  image: nginx:latest
"""

parsed = yaml.safe_load(manifest)
print(f"Deploying {parsed['metadata']['name']} with {parsed['spec']['replicas']} replicas")
print(f"Image: {parsed['spec']['image']}")
```

**Output:**
```
Deploying nginx with 3 replicas
Image: nginx:latest
```

---

## Script 3 — API Request & JSON Navigation

```python
import requests

response = requests.get("https://httpbin.org/json")
data = response.json()

slides = data["slideshow"]["slides"]
for slide in slides:
    print(slide["title"])
```

**Output:**
```
Wake up to WonderWidgets!
Overview
```

---

## Key Lessons from This Lab

- Never name files the same as standard library modules (`subprocess.py` breaks `import subprocess`)
- `subprocess.run()` returns output as bytes by default — always add `text=True`
- `yaml.safe_load()` not `yaml.load()` — security reason
- Always set `timeout` in requests calls
- venv is the correct way to manage Python packages on Ubuntu 23.04+

---

## boto3 — Deferred to AWS Week

`boto3` (AWS SDK for Python) will be covered in Week 06 when AWS credentials are set up. Pattern is the same as `requests` — call API, parse JSON response.

Preview:
```python
import boto3
s3 = boto3.client("s3")
buckets = s3.list_buckets()["Buckets"]
for bucket in buckets:
    print(bucket["Name"])
```

---

## Related notes
- [[2.3-Python-Subprocess]]
- [[2.1-JSON-YAML-Parsing]]
- [[2.2-Python-Requests]]

<!-- related -->
---
**Related:** [[2.1-JSON-YAML-Parsing|JSON & YAML Parsing]] · [[2.2-Python-Requests|Python — requests]]
<!-- /related -->
