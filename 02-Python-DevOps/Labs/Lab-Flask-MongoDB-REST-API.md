---
title: Lab — Flask + MongoDB REST API
aliases:
  - Flask-MongoDB-API
type: lab
module: python
status: "✅ Completed & Working"
interview-relevance: "🔴 High — REST API design, request lifecycle, DB operations"
tags:
  - python
  - flask
  - mongodb
  - rest-api
  - day1
  - golden-thread
  - lab
file: "`~/aakash/devops/taskmanager/app.py`"
up: "[[Python-DevOps-MOC]]"
prev: "[[Lab-Python-DevOps-Scripts]]"
related:
  - "[[Project-Flask-MongoDB-REST-API]]"
  - "[[Lab-Taskmanager-Dockerization]]"
origin: DevOps-Mastery/02-Python-DevOps/Labs/Flask-MongoDB-API.md
---

# Lab — Flask + MongoDB REST API

<!-- nav -->
↑ [[Python-DevOps-MOC|Python for DevOps]] · ← [[Lab-Python-DevOps-Scripts|Lab — Python DevOps Scripts]]
<!-- /nav -->

## Architecture

```
Client (curl/browser)  →  Flask (app.py :5000)  →  MongoDB (Docker :27017)
```

Three layers:
- **Client** — sends HTTP requests
- **Flask** — receives, processes, talks to MongoDB
- **MongoDB** — stores and returns data

---

## How Flask Routing Works

```python
@app.route("/tasks", methods=["POST"])
def create_task():
    ...
```

- `@app.route` is a decorator — it maps `METHOD + PATH → function`
- Flask receives an HTTP request, looks at the method and path, calls the matching function
- `<id>` in a route captures a URL segment as a variable:

```python
@app.route("/tasks/<id>", methods=["PUT"])
def update_task(id):   # id = "69d7b48d..." from the URL
```

---

## Request → Flask → MongoDB Flow

### POST /tasks — Create

```
curl POST /tasks {"title": "Learn Docker"}
        │
        ▼
Flask: request.get_json() → {"title": "Learn Docker"}
        │
        ▼
Validate: "title" must exist → 400 if missing
        │
        ▼
Build document: {"title": "Learn Docker", "done": False}
        │
        ▼
tasks.insert_one(document) → MongoDB stores on disk, returns ObjectId
        │
        ▼
Convert ObjectId → string, jsonify → HTTP 201 response
```

### GET /tasks — Read All

```python
for task in tasks.find():
    task["_id"] = str(task["_id"])
    result.append(task)
return jsonify(result), 200
```

- `tasks.find()` returns a cursor — lazy iterator over all documents
- Must convert `_id` from ObjectId → string (ObjectId is not JSON serializable)

### PUT /tasks/<id> — Update

```python
tasks.update_one(
    {"_id": ObjectId(id)},   # filter — find this document
    {"$set": data}            # update — only change these fields
)
```

- `ObjectId(id)` converts string back to MongoDB ObjectId for querying
- `$set` only updates the fields you pass — other fields stay untouched
- Without `$set` you'd replace the entire document

### DELETE /tasks/<id> — Delete

```python
tasks.delete_one({"_id": ObjectId(id)})
```

- Filter by `_id`, delete that document

---

## Key Flask Objects

| Object | What It Is |
|--------|-----------|
| `request` | Global object with everything about the incoming HTTP request |
| `request.get_json()` | Parses JSON body → Python dict |
| `request.method` | GET, POST, PUT, DELETE |
| `request.args` | URL query parameters (`?key=value`) |
| `jsonify()` | Converts Python dict → JSON response with correct Content-Type |

---

## MongoDB Concepts

| Concept | SQL Equivalent | Example |
|---------|---------------|---------|
| Database | Database | `taskmanager` |
| Collection | Table | `tasks` |
| Document | Row | `{"title": "Learn Docker", "done": false}` |
| `_id` | Primary Key | Auto-generated ObjectId |

**Common pymongo operations:**
```python
tasks.insert_one(doc)              # insert one document
tasks.find()                       # get all documents
tasks.find_one({"_id": ObjectId})  # get one document
tasks.update_one(filter, update)   # update one document
tasks.delete_one(filter)           # delete one document
```

---

## Environment Variable Config — 12-Factor App

```python
MONGO_URI = os.getenv("MONGO_URI", "mongodb://admin:password@localhost:27017/")
DB_NAME   = os.getenv("DB_NAME", "taskmanager")
```

**Why this matters:**
- In Docker, MongoDB won't be at `localhost` — it'll be at a container hostname
- In Kubernetes, it'll be at a service DNS name
- With env vars, you change config without changing code
- This is **12-factor app principle #3** — store config in the environment

**How to pass env vars at runtime:**
```bash
# Local
MONGO_URI="mongodb://admin:password@mongo:27017/" python3 app.py

# Docker (Day 2)
docker run -e MONGO_URI="mongodb://admin:password@mongo:27017/" taskmanager

# Kubernetes (Day 3)
env:
  - name: MONGO_URI
    value: "mongodb://admin:password@mongo:27017/"
```

---

## Running the App

```bash
cd ~/aakash/devops/taskmanager
source venv/bin/activate

# Start MongoDB first
docker run -d --name mongodb -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  mongo:6

# Run Flask
python3 app.py
```

**Test endpoints:**
```bash
# Health
curl http://192.168.1.18:5000/health

# Create
curl -X POST http://192.168.1.18:5000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Docker"}'

# Get all
curl http://192.168.1.18:5000/tasks

# Update (replace ID)
curl -X PUT http://192.168.1.18:5000/tasks/<id> \
  -H "Content-Type: application/json" \
  -d '{"done": true}'

# Delete
curl -X DELETE http://192.168.1.18:5000/tasks/<id>
```

---

## Interview — Ready to Speak

**Q: "Walk me through what happens when a POST request hits your Flask API."**

> "Flask receives the HTTP request and matches the method and path to a route using the `@app.route` decorator. Inside the handler, `request.get_json()` parses the JSON body into a Python dict. After validation, I build the document and call `insert_one()` via pymongo, which sends it to MongoDB over port 27017. MongoDB stores it on disk and returns an auto-generated ObjectId. I convert that ObjectId to a string — because ObjectId isn't JSON serializable — and return it in the response with `jsonify()` and a 201 status code."

**Q: "Why use environment variables for database config?"**

> "Because config changes between environments — local, Docker, Kubernetes, production. Hardcoding `localhost` works locally but breaks in Docker where the DB is at a container hostname. With `os.getenv()`, I pass the correct URI at runtime without changing code. This is the 12-factor app pattern — separate config from code."

---

## Related notes
- [[2.1-JSON-YAML-Parsing]]
- [[2.2-Python-Requests]]
- [[2.3-Python-Subprocess]]

<!-- related -->
---
**Related:** [[Project-Flask-MongoDB-REST-API|Project — Flask MongoDB REST API]] · [[Lab-Taskmanager-Dockerization|Lab — Dockerizing the Taskmanager App]]
<!-- /related -->
