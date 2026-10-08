---
title: "Revision — Deployment: Nginx, Gunicorn, systemd"
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
order: 11
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-10-Monitoring-Prometheus-and-Grafana]]"
next: "[[Revision-12-System-Design-for-DevOps]]"
related:
  - "[[Deployment-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Deployment: Nginx, Gunicorn, systemd

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-10-Monitoring-Prometheus-and-Grafana|Revision — Monitoring: Prometheus & Grafana]] · → [[Revision-12-System-Design-for-DevOps|Revision — System Design for DevOps]]
<!-- /nav -->

## Concept

Before Kubernetes, and still in many production setups, apps are deployed directly on Linux servers. Understanding traditional deployment is essential for debugging, legacy environments, and when you don't need Kubernetes complexity.

**The stack:**

```
Internet → Nginx (reverse proxy, SSL termination, static files) 
         → Gunicorn (Python WSGI server, manages worker processes)
         → Flask app code
```

**Why not run Flask directly?** Flask's dev server is single-threaded and not production-safe. Gunicorn manages multiple worker processes to handle concurrent requests. Nginx sits in front because it's much better at SSL termination, static file serving, rate limiting, and handling slow clients.

---

## Core Commands

```bash
# Nginx
sudo apt-get install nginx
sudo systemctl start nginx
sudo systemctl enable nginx             # Start on boot
sudo nginx -t                           # Test config syntax
sudo systemctl reload nginx             # Reload config without downtime

# Nginx config location
/etc/nginx/nginx.conf                   # Main config
/etc/nginx/sites-available/myapp        # Per-site config
/etc/nginx/sites-enabled/               # Symlink enabled configs

# Gunicorn
pip install gunicorn
gunicorn --workers 4 --bind 0.0.0.0:5000 app:app          # 4 workers
gunicorn --workers 4 --bind unix:/tmp/gunicorn.sock app:app # Unix socket

# systemd service file: /etc/systemd/system/myapp.service
sudo systemctl daemon-reload            # After creating/editing service file
sudo systemctl enable myapp             # Start on boot
sudo systemctl start myapp
sudo journalctl -u myapp -f             # Follow service logs
```

**Nginx reverse proxy config:**

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /static {
        alias /var/www/myapp/static;    # Serve static files directly (faster)
    }
}
```

**systemd service:**

```ini
[Unit]
Description=Flask App
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/myapp
Environment="PATH=/home/ubuntu/myapp/venv/bin"
EnvironmentFile=/home/ubuntu/myapp/.env
ExecStart=/home/ubuntu/myapp/venv/bin/gunicorn --workers 4 --bind 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## Senior Interview Q&A

**Q: Why do you put Nginx in front of Gunicorn? Why not expose Gunicorn directly?**

> "Four reasons.
>
> First, SSL termination. Nginx handles HTTPS with minimal config. Offloading SSL from Gunicorn means your Python workers only handle HTTP internally — no crypto overhead.
>
> Second, static file serving. Nginx serves static files (JS, CSS, images) in C at wire speed — far faster than Python. Gunicorn workers are expensive (memory, CPU). You don't want them serving a 50KB CSS file that Nginx can handle in microseconds.
>
> Third, slow client protection. A slow HTTP client (mobile on 3G) can tie up a Gunicorn worker for seconds waiting for the request to arrive. Nginx buffers the full request before handing it to Gunicorn, so your workers are never waiting on slow clients.
>
> Fourth, connection concurrency. Nginx handles thousands of simultaneous connections in a single process (event-driven). Gunicorn workers are one-connection-at-a-time per worker. Nginx sits at the edge, Gunicorn handles the compute.
>
> Modern alternative: in cloud environments with a load balancer, SSL is often terminated at the LB. But the static file and slow client arguments still apply."

<!-- related -->
---
**Related:** [[Deployment-MOC|Deployment (Nginx, PM2, Gunicorn, systemd)]]
<!-- /related -->
