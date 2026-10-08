---
title: Project — Flask MongoDB REST API
aliases:
  - Flask-MongoDB-Project
type: project
module: deployment
status: ✅ Completed
interview-relevance: "🔴 High — REST APIs, MongoDB, environment variables, error handling"
tags:
  - python
  - flask
  - mongodb
  - rest-api
  - deployment
  - project
repo: "`github.com/aakash-1004/flask-mongo-app`"
up: "[[Deployment-MOC]]"
prev: "[[10.7-Framework-Deployment-Guide]]"
related:
  - "[[Lab-Flask-MongoDB-REST-API]]"
origin: DevOps-Mastery/10-Deployment/Flask-MongoDB-Project.md
---

# Project — Flask MongoDB REST API

<!-- nav -->
↑ [[Deployment-MOC|Deployment (Nginx, PM2, Gunicorn, systemd)]] · ← [[10.7-Framework-Deployment-Guide|Framework Deployment Guide]]
<!-- /nav -->

## What We Built

A Flask web application with two core features:

- A `/api` route that reads data from a JSON file and returns it as a JSON response
- An HTML form that submits data to MongoDB Atlas, with success redirect and error handling on the same page

---

## Key Concepts Learned

### 1. Flask Routing

Flask maps URLs to Python functions using the `@app.route()` decorator.

```python
@app.route("/api")
def api_data():
    # runs when someone visits /api
```

Every route has:

- A **path** (`/api`, `/submit`, `/`)
- An optional **method** (`GET` by default, or `POST`, etc.)
- A **function** that runs and returns a response

---

### 2. Reading a File and Returning JSON

```python
import json
from flask import jsonify

@app.route("/api")
def api_data():
    with open("data.json", "r") as f:
        data = json.load(f)
    return jsonify(data), 200
```

|Part|What It Does|
|---|---|
|`open("data.json", "r")`|Opens file in read mode|
|`json.load(f)`|Parses JSON file into Python list/dict|
|`jsonify(data)`|Converts Python object to HTTP JSON response|
|`200`|HTTP status code — means OK/success|

---

### 3. Handling Form Submissions (POST)

```python
@app.route("/submit", methods=["POST"])
def submit():
    name = request.form.get("name", "").strip()
```

- `methods=["POST"]` — this route only accepts POST requests
- `request.form.get("name")` — reads form field named "name"
- `.strip()` — removes leading/trailing whitespace
- Default `""` — prevents KeyError if field is missing

---

### 4. MongoDB with PyMongo

```python
from pymongo import MongoClient

client = MongoClient(os.getenv("MONGO_URI"))
db = client["users_db"]
collection = db["users"]

collection.insert_one({"name": name, "email": email})
```

|Term|Meaning|
|---|---|
|`MongoClient`|Connection to MongoDB server|
|`db`|A database inside the cluster|
|`collection`|A collection inside the database (like a table)|
|`insert_one()`|Inserts a single document|

---

### 5. Environment Variables with python-dotenv

Never hardcode credentials in code. Use `.env` file + `python-dotenv`:

```python
from dotenv import load_dotenv
load_dotenv()

uri = os.getenv("MONGO_URI")
```

`.env` file (never commit to Git):

```
MONGO_URI=mongodb+srv://user:password@cluster.mongodb.net/
MONGO_DB=users_db
MONGO_COLLECTION=users
```

`.gitignore` must include:

```
.env
venv/
__pycache__/
```

---

### 6. Error Handling — Same Page vs Redirect

The assignment required:

- **Success** → redirect to another page
- **Error** → show on the SAME page, no redirect

```python
# Success — redirect
return redirect(url_for("success"))

# Error — render same template with error variable
return render_template("index.html", error="All fields are required."), 400
```

In the HTML template (Jinja2):

```html
{% if error %}
<div class="error">{{ error }}</div>
{% endif %}
```

**Why remove `required` from HTML inputs?** Browser's built-in `required` attribute catches empty fields BEFORE the request reaches Flask. For the assignment, we needed Flask's error handling to trigger — so we removed `required` to let the request through.

---

### 7. Flask Templates — Jinja2

Flask uses Jinja2 for templating — HTML with dynamic content:

|Syntax|Purpose|
|---|---|
|`{{ variable }}`|Output a variable|
|`{% if condition %}`|Conditional block|
|`{% for item in list %}`|Loop|
|`{% extends "base.html" %}`|Template inheritance|

---

### 8. HTTP Status Codes Used

|Code|Meaning|When Used|
|---|---|---|
|200|OK|Successful GET request|
|201|Created|Successful POST that created data|
|400|Bad Request|Validation failed (missing fields)|
|500|Server Error|Database error, unexpected crash|

---

## Project Structure

```
flask-mongo-app/
├── app.py              ← Flask application, all routes
├── data.json           ← Static data for /api route
├── .env                ← Credentials (never in Git)
├── .gitignore
├── requirements.txt
└── templates/
    ├── index.html      ← Form page
    └── success.html    ← Success page
```

---

## Commands Reference

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install flask pymongo python-dotenv

# Run app
python app.py

# Test API route
curl http://localhost:5000/api

# Test form submission
curl -X POST http://localhost:5000/submit \
  -d "name=Aakash&email=a@a.com&message=hello"
```

---

## Bugs Encountered

|Bug|Cause|Fix|
|---|---|---|
|Browser catches empty field before Flask|HTML `required` attribute|Removed `required` from inputs|
|`TemplateNotFound: success.html`|Accidentally deleted wrong file|Recreated success.html in templates/|
|`.env` not copied with `cp *`|`*` doesn't copy hidden files|Use `cp -r source/. destination/`|

---

## Real-World Context

This pattern — Flask API + MongoDB + environment variables — is used in:

- Microservices that handle specific data operations
- ML model serving APIs (Flask is dominant in ML/AI backends)
- Internal tools and dashboards

In production, you'd add authentication (JWT tokens), input sanitization, rate limiting, and proper logging.

---

## Interview Answer

**Q: How do you handle errors in a Flask API?**

> "I use try/except blocks around database operations to catch PyMongoError and other exceptions. For validation errors I check fields before hitting the database and return a 400 with an error message. For unexpected errors I return 500. In web forms, errors render back to the same template with an error variable so the user doesn't lose context."

**Q: Why should you never commit a .env file?**

> "The .env file contains credentials like database URIs, API keys, and secret keys. Committing it to Git — even a private repo — is a security risk because repo access can be compromised, and Git history persists even after deletion. The correct approach is .gitignore the .env, store credentials only on the server, and use a secrets manager like AWS Secrets Manager or Kubernetes Secrets in production."

---

## Links

- [[10.7-Framework-Deployment-Guide]]
- [[10.1-Frontend-Backend-Web-Concepts]]
- [[10.2-Nginx-Core-Concepts]]
- [[10.3-Process-Management-PM2-Gunicorn-systemd]]

<!-- related -->
---
**Related:** [[Lab-Flask-MongoDB-REST-API|Lab — Flask + MongoDB REST API]]
<!-- /related -->
