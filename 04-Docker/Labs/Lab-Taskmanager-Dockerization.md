---
title: Lab — Dockerizing the Taskmanager App
aliases:
  - Taskmanager-Docker
type: lab
module: docker
status: ✅ Completed
tags:
  - docker
  - flask
  - mongodb
  - lab
  - day2
  - golden-thread
repo: "https://github.com/aakash-1004/taskmanager"
up: "[[Docker-MOC]]"
prev: "[[4.9-Docker-Interview-QA]]"
next: "[[Project-Fullstack-Docker-Express-Flask]]"
related:
  - "[[Lab-Flask-MongoDB-REST-API]]"
  - "[[4.2-Docker-Core-Concepts]]"
  - "[[4.7-Docker-Compose]]"
  - "[[5.3-Kubernetes-Manifests-Lab]]"
  - "[[Lab-EC2-Deployment]]"
origin: DevOps-Mastery/04-Docker/Taskmanager-Docker.md
---

# Lab — Dockerizing the Taskmanager App

<!-- nav -->
↑ [[Docker-MOC|Docker]] · ← [[4.9-Docker-Interview-QA|Docker — Interview Q&A]] · → [[Project-Fullstack-Docker-Express-Flask|Project — Fullstack Docker (Express + Flask + MongoDB Atlas)]]
<!-- /nav -->

## What Was Built

Took the Flask + MongoDB app from Day 1 and fully containerized it:
- Dockerfile for Flask app
- Docker Compose for running Flask + MongoDB together
- Named volume for MongoDB data persistence
- Custom Docker network for container communication
- `.dockerignore` to keep image clean

---

## Project Structure

```
taskmanager/
├── app.py
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── requirements.txt
└── venv/          ← ignored by .dockerignore
```

---

## Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python3", "app.py"]
```

---

## docker-compose.yml

```yaml
services:
  mongodb:
    image: mongo:6
    container_name: mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodb_data:/data/db
    networks:
      - taskmanager-net

  taskmanager:
    build: .
    container_name: taskmanager
    ports:
      - "5000:5000"
    environment:
      MONGO_URI: mongodb://admin:password@mongodb:27017/
      DB_NAME: taskmanager
    depends_on:
      - mongodb
    networks:
      - taskmanager-net

volumes:
  mongodb_data:

networks:
  taskmanager-net:
    driver: bridge
```

---

## Running the App

```bash
# Start everything
docker compose up -d

# Verify
docker ps
curl http://localhost:5000/health

# View logs
docker compose logs -f taskmanager

# Stop everything
docker compose down

# Stop and delete data
docker compose down -v
```

---

## Key Lessons from This Lab

**Layer caching demo:**
- First build: ~20 seconds (downloads base images, installs deps)
- Rebuild unchanged: ~1.3 seconds (all layers cached)
- Change app.py, rebuild: ~2 seconds (only last COPY layer rebuilds)

**Container hostname:**
- Flask connects to MongoDB via `mongodb:27017` not `localhost:27017`
- Container name = hostname on Docker network
- This is why `MONGO_URI` is an env var — hostname changes per environment

**Volume persistence:**
- `docker compose down` → containers gone, data survives in volume
- `docker compose down -v` → containers AND data gone
- Volume location: `/var/lib/docker/volumes/taskmanager_mongodb_data/_data`

**`.dockerignore` gotcha:**
- Must exclude `.env` — secrets must never be baked into images
- Exclude `venv/` — deps are installed fresh during build via requirements.txt

---

## Bugs / Gotchas Hit

| Issue | Cause | Fix |
|-------|-------|-----|
| `DOCKERFILE` not found | Linux is case-sensitive, saved as uppercase | `mv DOCKERFILE Dockerfile` |
| `docker build` error | Missing `.` at end of command | `docker build -t name .` |
| Git push failed on new machine | Git identity not configured | `git config --global user.name/email` |
| venv creation failed | `python3.12-venv` not installed | `sudo apt install python3.12-venv` |

---

## Test Commands

```bash
# Health check
curl http://localhost:5000/health

# Create task
curl -X POST http://localhost:5000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Running in Docker"}'

# Get all tasks
curl http://localhost:5000/tasks

# Update task (replace ID)
curl -X PUT http://localhost:5000/tasks/<id> \
  -H "Content-Type: application/json" \
  -d '{"done": true}'

# Delete task
curl -X DELETE http://localhost:5000/tasks/<id>
```

---

## Related notes
- [[4.2-Docker-Core-Concepts]]
- [[4.7-Docker-Compose]]
- [[Lab-Flask-MongoDB-REST-API]]

<!-- related -->
---
**Related:** [[Lab-Flask-MongoDB-REST-API|Lab — Flask + MongoDB REST API]] · [[4.2-Docker-Core-Concepts|Docker Core Concepts]] · [[4.7-Docker-Compose|Docker Compose]] · [[5.3-Kubernetes-Manifests-Lab|Lab — Deploying Taskmanager to Kubernetes]] · [[Lab-EC2-Deployment|Lab — Deploying Taskmanager to AWS EC2]]
<!-- /related -->
