---
title: Project — Fullstack Docker (Express + Flask + MongoDB Atlas)
aliases:
  - Fullstack-Docker-Project
type: project
module: docker
status: ✅ Completed
interview-relevance: "🔴 High — multi-container Docker, networking, Docker Hub, full-stack architecture"
tags:
  - docker
  - nodejs
  - express
  - flask
  - mongodb
  - docker-compose
  - project
repo: "`github.com/aakash-1004/fullstack-docker`"
docker-hub: "`aakash0908/flask-backend:v1` | `aakash0908/express-frontend:v1`"
up: "[[Docker-MOC]]"
prev: "[[Lab-Taskmanager-Dockerization]]"
related:
  - "[[4.8-Docker-Compose-Deep-Dive]]"
  - "[[5.2-K8s-Deployment-Express-Flask-k3s]]"
  - "[[6.2-AWS-Deployment-EC2-ECS]]"
origin: DevOps-Mastery/04-Docker/Fullstack-Docker-Project.md
---

# Project — Fullstack Docker (Express + Flask + MongoDB Atlas)

<!-- nav -->
↑ [[Docker-MOC|Docker]] · ← [[Lab-Taskmanager-Dockerization|Lab — Dockerizing the Taskmanager App]]
<!-- /nav -->

## What We Built

A two-container application:

- **Frontend** — Express (Node.js) serves an HTML form
- **Backend** — Flask handles form submission and stores data in MongoDB Atlas
- Both containers connected via Docker Compose on a shared network

---

## Architecture

```
User's Browser
      |
      | HTTP (port 3001)
      v
Express Frontend Container
      |
      | HTTP POST (axios) to http://backend:5000
      v
Flask Backend Container
      |
      | PyMongo
      v
MongoDB Atlas (cloud)
```

Key insight: `backend` in the URL `http://backend:5000` is the **Docker service name** from `docker-compose.yml` — Docker's internal DNS resolves it to the container's IP automatically.

---

## Key Concepts Learned

### 1. Why Separate Frontend and Backend Containers?

Each container has one responsibility:

- Frontend container: serve UI, handle user interactions
- Backend container: business logic, database operations

Benefits:

- Scale them independently (more backend instances if API is slow)
- Deploy them separately
- Different teams can own different services
- If one crashes, the other keeps running

---

### 2. EJS — Express Templating

EJS (Embedded JavaScript) is Node.js's equivalent of Flask's Jinja2:

|Flask (Jinja2)|Express (EJS)|
|---|---|
|`{{ variable }}`|`<%= variable %>`|
|`{% if condition %}`|`<% if (condition) { %>`|
|`{% for item in list %}`|`<% list.forEach(item => { %>`|
|`templates/` folder|`views/` folder|

html

```html
<!-- EJS syntax -->
<% if (error) { %>
  <div class="error"><%= error %></div>
<% } %>
```

---

### 3. Axios — Making HTTP Requests from Node.js

Axios is the Node.js equivalent of Python's `requests` library:

javascript

```javascript
// Python requests equivalent
const response = await axios.post(`${FLASK_BACKEND}/submit`, {
    name,
    email,
    message
});
```

|Python requests|Node.js axios|
|---|---|
|`requests.post(url, json=data)`|`axios.post(url, data)`|
|`response.json()`|`response.data`|
|`response.status_code`|`response.status`|

---

### 4. CORS — Why It's Needed Here

CORS (Cross-Origin Resource Sharing) — browser security rule blocking requests between different origins.

In this project Express calls Flask directly from the server (not the browser), so CORS isn't strictly needed. But Flask-CORS is added as best practice for when a browser-based frontend calls a Flask API directly.

python

```python
from flask_cors import CORS
CORS(app)  # allow all origins
```

---

### 5. Docker Networking — How Containers Talk

When Docker Compose creates containers on the same network, they can reach each other using their **service name** as hostname:

yaml

```yaml
services:
  backend:      # ← this name becomes the hostname
    ...
  frontend:
    environment:
      - FLASK_BACKEND=http://backend:5000  # ← uses service name
```

Docker's internal DNS resolves `backend` → container's IP automatically. You never need to know the actual IP.

---

### 6. Docker Compose — Key Concepts

yaml

```yaml
version: '3.8'

services:
  backend:
    build: ./backend        # build from Dockerfile in this folder
    container_name: flask-backend
    ports:
      - "5000:5000"         # host:container
    env_file:
      - ./backend/.env      # load env vars from file
    networks:
      - app-network
    restart: always         # restart if container crashes

  frontend:
    build: ./frontend
    ports:
      - "3001:3000"         # host 3001 → container 3000
    environment:
      - FLASK_BACKEND=http://backend:5000
    depends_on:
      - backend             # start backend first
    networks:
      - app-network

networks:
  app-network:
    driver: bridge          # standard Docker bridge network
```

|Field|Purpose|
|---|---|
|`build`|Path to folder containing Dockerfile|
|`ports`|Map host port to container port|
|`env_file`|Load environment variables from file|
|`environment`|Set individual environment variables|
|`depends_on`|Start order (doesn't wait for healthy, just started)|
|`networks`|Which network to join|
|`restart: always`|Auto-restart on crash or reboot|

---

### 7. Dockerfile — Frontend vs Backend

**Python (Flask) Dockerfile:**

dockerfile

```dockerfile
FROM python:3.12-slim       # small Python base image
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

**Node.js (Express) Dockerfile:**

dockerfile

```dockerfile
FROM node:20-alpine         # small Node.js base image
WORKDIR /app
COPY package*.json ./       # copy package files first (layer caching)
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

Key pattern: **copy dependency files first, install, then copy source code.** Why: Docker caches layers. If source code changes but `package.json` didn't, the `npm install` layer is reused — much faster rebuilds.

---

### 8. Port Conflict — What We Hit

Grafana was already using port 3000 (from Kubernetes port-forward). Docker tried to bind port 3000 on the host — failed.

Fix: map to a different host port in docker-compose.yml:

yaml

```yaml
ports:
  - "3001:3000"  # host 3001 → container 3000
```

The container still runs on 3000 internally. Only the host mapping changed.

---

### 9. Pushing to Docker Hub

bash

```bash
# Login
docker login

# Tag — format: username/repository:tag
docker tag fullstack-docker-backend aakash0908/flask-backend:v1
docker tag fullstack-docker-frontend aakash0908/express-frontend:v1

# Push
docker push aakash0908/flask-backend:v1
docker push aakash0908/express-frontend:v1
```

**Pushed vs Mounted in push output:**

- `Pushed` — your custom layers (app code, installed packages)
- `Mounted from library/python` — base image layers already on Docker Hub, just referenced not re-uploaded

---

### 10. npm Commands Explained

|Command|What It Does|
|---|---|
|`npm init -y`|Creates `package.json` with defaults, skipping all questions|
|`npm install express ejs axios`|Installs packages + adds to `package.json` dependencies|
|`npm install --production`|Installs only production dependencies, skips devDependencies|

---

## Project Structure

```
fullstack-docker/
├── docker-compose.yml
├── .gitignore
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env            ← never in Git
└── frontend/
    ├── app.js
    ├── package.json
    ├── package-lock.json
    ├── Dockerfile
    └── views/
        └── index.ejs
```

---

## Commands Reference

bash

```bash
# Build and start all containers
docker compose up --build

# Start without rebuilding
docker compose up

# Run in background (detached)
docker compose up -d

# Stop all containers
docker compose down

# View running containers
docker ps

# View logs of specific service
docker compose logs frontend
docker compose logs backend

# Rebuild single service
docker compose up --build backend
```

---

## .gitignore for This Project

```
# Python
backend/.env
backend/__pycache__/
backend/venv/

# Node
frontend/node_modules/

# Editor
.vscode/
```

---

## Bugs Encountered

|Bug|Cause|Fix|
|---|---|---|
|Port 3000 already in use|Grafana port-forward on 3000|Changed docker-compose ports to `3001:3000`|
|`version` attribute warning|Obsolete in newer Docker Compose|Remove `version:` line from docker-compose.yml|

---

## Real-World Context

This pattern — multiple containers on a shared Docker network, each with a single responsibility — is the foundation of microservices architecture. In production:

- Each service gets its own Docker image
- Docker Compose is used in development/staging
- Kubernetes replaces Docker Compose in production (same concept, more features)
- Images are stored in a registry (Docker Hub, AWS ECR, GCR)

---

## Interview Answers

**Q: How do containers in Docker Compose communicate with each other?**

> "Containers on the same Docker Compose network can reach each other using the service name as a hostname. Docker has an internal DNS that resolves service names to container IPs automatically. So if my backend service is named 'backend', my frontend can call [http://backend:5000](http://backend:5000) without knowing the actual IP."

**Q: What's the difference between ports and expose in Docker?**

> "EXPOSE in a Dockerfile is documentation — it tells you which port the container listens on but doesn't actually publish it. ports in docker-compose.yml actually maps a host port to a container port, making it accessible from outside. Format is host:container — so 3001:3000 means port 3001 on your machine maps to port 3000 inside the container."

**Q: Why do you copy package.json before copying source code in a Node Dockerfile?**

> "Docker builds images in layers and caches each layer. If I copy everything first then run npm install, every code change invalidates the npm install cache and reinstalls all packages. By copying only package.json first, running npm install, then copying source code — the npm install layer is only invalidated when package.json changes, not on every code change. Builds are much faster."

**Q: What does depends_on do in Docker Compose?**

> "depends_on controls start order — it ensures the backend container starts before the frontend. But it only waits for the container to start, not for the application inside to be ready. If you need to wait for the app to be healthy, you need healthcheck combined with depends_on condition: service_healthy."

---

## Links

- [[4.2-Docker-Core-Concepts]]
- [[4.7-Docker-Compose]]
- [[10.1-Frontend-Backend-Web-Concepts]]
- [[10.7-Framework-Deployment-Guide]]

<!-- related -->
---
**Related:** [[4.8-Docker-Compose-Deep-Dive|Docker Compose — Deep Dive]] · [[5.2-K8s-Deployment-Express-Flask-k3s|Kubernetes Deployment — Express + Flask on k3s]] · [[6.2-AWS-Deployment-EC2-ECS|AWS Deployment — EC2 & ECS]]
<!-- /related -->
