---
title: "Deployment (Nginx, PM2, Gunicorn, systemd) — Map of Content"
type: moc
module: deployment
tags:
  - moc
  - deployment
up: "[[Home]]"
related:
  - "[[Revision-11-Deployment-Nginx-Gunicorn-systemd]]"
---

# Deployment (Nginx, PM2, Gunicorn, systemd) — Map of Content

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

Frontend/backend concepts, Nginx, PM2/Gunicorn/systemd, deployment debugging, framework deployment guide.

**Related modules:** [[Linux-Bash-MOC|Linux & Bash]] · [[Docker-MOC|Docker]] · [[AWS-MOC|AWS]]

## Concepts
- [[10.1-Frontend-Backend-Web-Concepts|Frontend, Backend & Web Concepts]] — *concept* · interview: 🔴 High — foundational knowledge tested in every DevOps/Cloud role
  - Covers: The Core Distinction; Language vs Framework vs Runtime; Frontend Deep Dive; Backend Deep Dive; Full-Stack Frameworks; The API: How Frontend and Backend Talk; Database Layer
- [[10.2-Nginx-Core-Concepts|Nginx Core Concepts]] — *concept* · interview: 🔴 High — Nginx comes up in every deployment interview
  - Covers: What Problem Does Nginx Solve?; What Is Nginx?; Two Modes: Web Server vs Reverse Proxy; The Full Request Flow in Production; Why Not Expose Your App Directly?; What Nginx Does — The Full List; Install & Core Commands
- [[10.3-Process-Management-PM2-Gunicorn-systemd|Process Management — PM2, Gunicorn, systemd]] — *concept* · interview: 🔴 High — every backend deployment interview covers this
  - Covers: What Is a Process?; The Problem: Apps Die Without Process Management; Three Tools, Three Use Cases; PM2 — The Node.js Standard; Gunicorn — Python's Production WSGI Server; systemd — The Production Standard; Environment Variables — The Right Way
- [[10.4-Deployment-Debugging|Deployment Debugging]] — *concept* · interview: 🔴 High — debugging broken deployments separates junior from senior engineers
  - Covers: Why Debugging Skill Matters More Than Setup Skill; The Mental Model: Layers; HTTP Error Codes — What They Tell You; The 5-Step Debug Sequence; Nginx Debugging Commands; App Process Debugging; Common Deployment Mistakes and Fixes
- [[10.6-Apache-vs-Nginx|Apache vs Nginx]] — *concept* · interview: 🟡 Medium — understanding both shows depth, but Nginx is what you'll use
  - Covers: The Big Picture — Why Two Web Servers Exist; Forward Proxy vs Reverse Proxy — The Foundation; Apache and Nginx — Architecture Differences; Key Differences Side by Side; .htaccess — Apache's Superpower and Nginx's Missing Feature; When to Use Each; Setting Up Apache
- [[10.7-Framework-Deployment-Guide|Framework Deployment Guide]] — *reference* · interview: 🔴 High — "walk me through deploying X" is a standard interview question
  - Covers: The Universal Mental Model; Framework Type Determines Everything; Deployment Step-by-Step; Traditional vs Docker Deployment; Quick Reference — Strategy by Framework; Interview — Ready to Speak

## Labs & projects
- [[Project-Flask-MongoDB-REST-API|Project — Flask MongoDB REST API]] — *project* · interview: 🔴 High — REST APIs, MongoDB, environment variables, error handling
  - Covers: What We Built; Key Concepts Learned; Project Structure; Commands Reference; Bugs Encountered; Real-World Context; Interview Answer

[[Home]] · [[Interview-Prep-MOC]]

<!-- related -->
---
**Related:** [[Revision-11-Deployment-Nginx-Gunicorn-systemd|Revision — Deployment: Nginx, Gunicorn, systemd]]
<!-- /related -->
