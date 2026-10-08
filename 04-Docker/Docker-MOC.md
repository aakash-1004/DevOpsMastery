---
title: Docker — Map of Content
type: moc
module: docker
tags:
  - moc
  - docker
up: "[[Home]]"
related:
  - "[[Revision-04-Docker]]"
---

# Docker — Map of Content

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

Architecture → images/containers → Dockerfile → networking → storage → Compose → interview Q&A, plus the Taskmanager and fullstack projects.

**Related modules:** [[Kubernetes-MOC|Kubernetes]] · [[CICD-MOC|CI/CD]] · [[AWS-MOC|AWS]]

## Concepts
- [[4.1-Docker-Architecture|Docker Architecture]] — *concept* · interview: 🔴 High — architecture is asked in almost every Docker interview
  - Covers: The Problem Docker Solves; Containers vs Virtual Machines; How Docker Achieves Isolation; Docker Architecture — The Three Components; Docker Engine vs Docker Daemon; The Docker Object Hierarchy; Image Layers — How Docker Storage Works
- [[4.2-Docker-Core-Concepts|Docker Core Concepts]] — *concept* · interview: 🔴 High — Docker is tested in every DevOps interview
  - Covers: What Problem Docker Solves; Core Concepts; How Images Work — Layers; Dockerfile — Taskmanager; .dockerignore; Docker Networking; Essential Docker Commands
- [[4.3-Docker-Images-and-Containers|Docker Images & Containers]] — *concept* · interview: 🔴 High — daily operational knowledge
  - Covers: Images vs Containers — The Mental Model; Docker Login; Image Management; Container Lifecycle; Running Containers — `docker run`; Accessing Running Containers; Container Management Commands
- [[4.4-Dockerfile-Reference|Dockerfile — Complete Reference]] — *reference* · interview: 🔴 High — Dockerfile knowledge is tested in every Docker interview
  - Covers: What Is a Dockerfile?; Build Context — What Docker Can See; Dockerfile Instructions — Complete Reference; Complete Dockerfile Example — Python Flask App; Multi-Stage Build — Java Example; Interview — Ready to Speak
- [[4.5-Docker-Networking|Docker Networking Deep Dive]] — *concept* · interview: 🔴 High — networking is a frequent interview topic
  - Covers: Why Docker Networking Exists; How Port Mapping Works; Docker Network Types; Bridge Network — The Default; Host Network; None Network; Overlay Network — Multi-Host Networking
- [[4.6-Docker-Storage|Docker Storage — Volumes & Bind Mounts]] — *concept* · interview: 🔴 High — data persistence is a critical production concern
  - Covers: The Core Problem — Container Ephemerality; How Container Storage Works; Three Storage Options; Named Volume vs Bind Mount — When to Use Which; Volumes in Docker Compose; Multi-Stage Build Impact on Storage; Interview — Ready to Speak
- [[4.7-Docker-Compose|Docker Compose]] — *concept* · interview: 🔴 High — used daily in real DevOps work
  - Covers: What Is Docker Compose?; Taskmanager docker-compose.yml; Breaking Down Each Field; Volumes — Why They Matter; Essential Compose Commands; `-d` Flag — Detached Mode; Common Gotcha — depends_on
- [[4.8-Docker-Compose-Deep-Dive|Docker Compose — Deep Dive]] — *concept* · interview: 🔴 High — Docker Compose is used in almost every real project
  - Covers: What Problem Does Docker Compose Solve?; Docker Compose File Structure; Services — The Heart of Compose; Every Service Key Explained; Top-Level Volumes and Networks; Complete Example — Taskmanager Stack; Essential Compose Commands

## Interview
- [[4.9-Docker-Interview-QA|Docker — Interview Q&A]] — *interview* · interview: 🔴 Must review before every interview
  - Covers: Core Concepts; Images and Containers; Networking; Storage; Dockerfile; Docker Compose; Debugging

## Labs & projects
- [[Lab-Taskmanager-Dockerization|Lab — Dockerizing the Taskmanager App]] — *lab*
  - Covers: What Was Built; Project Structure; Dockerfile; docker-compose.yml; Running the App; Key Lessons from This Lab; Bugs / Gotchas Hit
- [[Project-Fullstack-Docker-Express-Flask|Project — Fullstack Docker (Express + Flask + MongoDB Atlas)]] — *project* · interview: 🔴 High — multi-container Docker, networking, Docker Hub, full-stack architecture
  - Covers: What We Built; Architecture; Key Concepts Learned; Project Structure; Commands Reference; .gitignore for This Project; Bugs Encountered

[[Home]] · [[Interview-Prep-MOC]]

<!-- related -->
---
**Related:** [[Revision-04-Docker|Revision — Docker]]
<!-- /related -->
