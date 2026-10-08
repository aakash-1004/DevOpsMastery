---
title: Amplify — Fullstack Platform
type: reference
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
order: 18
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-17-EKS]]"
next: "[[AWS-19-Interview-Master-QA]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# Amplify — Fullstack Platform

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-17-EKS|EKS — Elastic Kubernetes Service]] · → [[AWS-19-Interview-Master-QA|Interview Master Q&A]]
<!-- /nav -->

## Concept

**AWS Amplify** is a platform that simplifies building, deploying, and hosting full-stack web and mobile apps.

**Combines:**

- Frontend hosting (React, Vue, Angular, static sites)
- Backend services (auth, API, database, storage)
- CI/CD (auto-deploy from Git)
- Custom domains and SSL

## What Amplify Handles

**Frontend:**

- HTML, CSS, JavaScript
- Frameworks: React, Angular, Vue.js, Next.js
- Static site hosting with global CDN

**Backend:**

- Python, Node.js, Java, PHP
- Frameworks: Express, Django, Spring Boot
- Databases: MySQL, MongoDB, PostgreSQL

**Key features:**

- Connect to backend services (Cognito for auth, AppSync for GraphQL API, S3 for storage)
- Automatic deployments from GitHub/GitLab/CodeCommit
- Custom domains with free SSL via ACM
- Preview environments (deploy PR to test URL)

## What You Can Build

```
1. Static Websites
   ├── Marketing sites, blogs, landing pages
   └── Deploy from GitHub → auto-deploys on every push

2. Single Page Apps (SPAs)
   ├── React, Vue, Angular apps
   ├── With client-side routing
   └── Global CDN, fast load times

3. Full-Stack Apps
   ├── Frontend + backend + database
   ├── User authentication via Cognito
   ├── GraphQL API via AppSync
   └── File storage via S3

4. Mobile App Backends
   ├── iOS/Android SDKs
   └── Same backend serves web + mobile
```

## Practical Workflow

**Deploy static site from GitHub:**

```
1. Push site code to GitHub repo
2. Amplify Console → Connect to GitHub
3. Select repo and branch (e.g., main)
4. Amplify auto-detects build settings (React, Vue, etc.)
5. Deploy → live at https://main.xxxxxxx.amplifyapp.com
6. Custom domain: point Route 53 to Amplify
7. SSL: automatic via ACM
8. Auto-deploy: every push to main → auto-deploy
```

## Amplify vs Other Options

```
Amplify vs S3 + CloudFront (static site)
    ├── Amplify: easier — GitHub integration, auto-deploy, one dashboard
    ├── S3+CF: more granular control, cheaper at scale
    └── Amplify: better for developer productivity, S3+CF for cost-optimized

Amplify vs Vercel/Netlify
    ├── Amplify: better AWS ecosystem integration
    ├── Vercel: better Next.js support, faster deploys
    └── Netlify: mature JAMstack platform

Amplify vs EKS/ECS (full-stack)
    ├── Amplify: PaaS — simpler, less control
    ├── EKS/ECS: IaaS-style — full control, more complexity
    └── Amplify: for developer-focused teams, EKS/ECS for production scale
```

## Memory Hints

- **Amplify** = "AWS's Vercel/Netlify" (fullstack PaaS)
- **Fullstack platform** = "one dashboard for frontend + backend"
- **Auto-deploy** = "git push → live in minutes"
