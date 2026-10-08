---
title: DevOps Interview Prep — Batch 1 (28 Questions)
aliases:
  - DevOps Interview Prep — Batch 1 (28 Questions)Untitled
type: interview
module: interview
tags:
  - interview
  - devops
  - spoken-answers
up: "[[Interview-Prep-MOC]]"
next: "[[Concept-Bank-Batch-2-105-QA]]"
related:
  - "[[4.9-Docker-Interview-QA]]"
  - "[[4.4-Dockerfile-Reference]]"
  - "[[4.7-Docker-Compose]]"
  - "[[7.4-Jenkins-CICD-Pipeline]]"
origin: OnlyQuestions/DevOps Interview Prep — Batch 1 (28 Questions)Untitled.md
---

# DevOps Interview Prep — Batch 1 (28 Questions)

<!-- nav -->
↑ [[Interview-Prep-MOC|Interview Prep]] · → [[Concept-Bank-Batch-2-105-QA|DevOps Interview Prep — Batch 2 (Concept Bank, ~105 Q&A)]]
<!-- /nav -->

**How to use this:** These are spoken-ready answers, written the way you'd actually say them out loud — not textbook dumps. Read each one 2-3 times, then say it in your own words. Don't memorize verbatim.

**One flag before you start:** Questions #7, #14, #15, #26, #27 assume a Java + Tomcat + Maven + WAR/JAR pipeline. That's not your actual stack — your flagship projects are Go (distroless containers on k3s/EKS) plus JS7/Keycloak (JVM-based, but not Tomcat-hosted web apps). I've given you the **honest bridge answer** for each of these — don't claim Spring Boot or Tomcat production experience you don't have. If an interviewer digs in with follow-ups, a confident "that's not my project, here's what I've actually done" beats a shaky fabricated answer every time. If you _have_ worked with Tomcat/Maven/Spring Boot somewhere I don't know about, tell me and I'll rewrite these properly.

---

## 1. Tell me about yourself.

"I'm a Cloud and Automation Engineer at Zehntech Technologies in Indore, with about 3 years of experience. I started in production support — troubleshooting JS7 JobScheduler automation workflows and managing Keycloak/OIDC for authentication — which gave me strong Linux and incident-diagnosis fundamentals. Over the past year I've moved deep into DevOps: I containerize and deploy apps on Kubernetes, both k3s and EKS, using Helm. I built a full GitOps pipeline — GitHub Actions builds and pushes the image, ArgoCD auto-syncs it to the cluster, with self-healing proven live. I also stood up an EKS cluster on Fargate with the complete IRSA chain — OIDC provider, least-privilege IAM role, STS AssumeRoleWithWebIdentity — so pods get scoped AWS access without static keys. And I use Terraform for provisioning infra like VPC, ALB, and ECS. Right now I'm looking to move fully into a DevOps/Cloud role where I own CI/CD and infrastructure end-to-end."

---

## 2. Explain your CI/CD pipeline project end-to-end.

"I built a full GitOps loop for a Go web app. On every push to main, GitHub Actions kicks off: it compiles the Go binary in a multi-stage Docker build, with the final image based on distroless for a minimal attack surface. The image gets tagged and pushed to Docker Hub. The pipeline then bumps that image tag in the Helm chart's `values.yaml` and commits it back to the repo — that commit is the only trigger ArgoCD cares about. ArgoCD is watching that repo; it detects the diff and auto-syncs, rolling the new version out to the cluster — I've run this against both k3s locally and EKS. Liveness and readiness probes gate traffic until the pod is actually healthy. The part I'm proudest of: I proved self-heal live — manually deleted a deployment, and ArgoCD reconciled it back to match Git within seconds, no human intervention. I also had to add a `paths-ignore` rule so the pipeline's own values.yaml bump commit doesn't re-trigger itself in an infinite loop."

---

## 3. Why do you push code to the main branch?

"For my personal project, it's a single-developer setup, so I push directly to main and that's what triggers the pipeline — there's no team to gate against. But that's not how I'd run it on a real team. Professionally, the right pattern is: feature branch → PR → review → merge to main. Main should represent the one branch that's always deployable, so it's the only branch CI/CD should build and deploy from. You don't want every feature branch auto-deploying to production."

---

## 4. If you push code to another branch, will the Jenkins pipeline trigger?

"Depends entirely on how the job's configured. If it's a standard pipeline job scoped to `main` via the webhook + branch specifier, pushing to any other branch won't trigger anything — the trigger just doesn't match. If it's a multibranch pipeline and that branch has a Jenkinsfile, Jenkins will discover it and run build/test stages on it — but the deploy stage is normally gated with something like `when { branch 'main' }`, so even then it won't deploy. The trigger firing and the deploy stage running are two separate things."

---

## 5. What was your role in Jenkins?

"My primary CI/CD stack day-to-day is GitHub Actions plus ArgoCD for GitOps — that's what I run in production for my own projects. On Jenkins, I've built declarative pipelines with the standard stages — checkout, build, test, Docker build and push, deploy — and worked with credentials binding and both static and Kubernetes-based dynamic agents. If the role is Jenkins-heavy, I'm comfortable there; it's just not my default tool by choice."

_Be ready for a follow-up asking which is better — say GitHub Actions + ArgoCD reduces infra you have to maintain (no Jenkins master to patch/scale) and gives you a proper GitOps audit trail, but Jenkins wins on flexibility for complex, non-container pipelines and huge plugin ecosystem._

---

## 6. Why did you choose Git as the Source Code Management tool?

"Git is the industry standard — distributed, so every clone is a full backup with history, and the branching model is cheap and fast, which matters for feature-branch workflows. Beyond that, it's foundational to how I actually deploy things: my whole ArgoCD setup is GitOps, meaning Git _is_ the source of truth for what's running in the cluster. Every infra and app-config change is a commit, so I get a full audit trail and instant rollback by just reverting a commit."

---

## 7. What is Apache Tomcat, and why did you use it?

**Honest bridge:** "My recent projects are Go-based — I compile to a static binary and run it in a distroless container, so I haven't personally deployed on Tomcat. That said, I know what it does: Tomcat is a Java servlet container — it runs Java web apps by unpacking a WAR file into its `webapps` directory and handling the servlet/JSP lifecycle, HTTP request handling, and connection pooling for you. I have run other JVM-based production systems though — JS7 JobScheduler and Keycloak — so I'm comfortable with JVM tuning, heap sizing, and reading Java stack traces from logs, just not specifically through Tomcat."

---

## 8. Explain the difference between a Docker Image and a Docker Container.

"An image is a read-only, immutable template — layers built from a Dockerfile, containing the app code, runtime, and dependencies. A container is a running (or stopped) _instance_ of that image, with its own writable layer on top, plus its own isolated filesystem, network namespace, and process tree. Same relationship as a job definition versus a job execution in JS7 — one template, many independent runs."

---

## 9. Write a simple Dockerfile and explain each instruction.

Use your real multi-stage Go build:

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app .

# Stage 2: Run
FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=builder /app/app .
EXPOSE 8080
ENTRYPOINT ["./app"]
```

- `FROM golang:1.22-alpine AS builder` — starting base image for the build stage, named `builder` so I can reference it later.
- `WORKDIR /app` — sets the working directory for every instruction after it, avoids scattering `cd` commands.
- `COPY go.mod go.sum ./` then `RUN go mod download` — copy just the dependency manifest first so Docker caches this layer; dependencies don't re-download unless `go.mod`/`go.sum` change.
- `COPY . .` — now copy the rest of the source.
- `RUN go build -o app .` — compile a static binary, `CGO_ENABLED=0` so it has no C library dependency (needed for distroless).
- `FROM gcr.io/distroless/static-debian12` — second stage, a minimal image with no shell, no package manager, nothing but what's needed to run a static binary.
- `COPY --from=builder /app/app .` — pull only the compiled binary out of the build stage; none of the Go toolchain or source ships in the final image.
- `EXPOSE 8080` — documents the port the app listens on (doesn't actually publish it — that's `-p` at runtime).
- `ENTRYPOINT ["./app"]` — the fixed command that always runs when the container starts.

---

## 10. Is your Dockerfile production-ready? What improvements would you make?

"It's close — distroless already gets me a small image with no shell, which cuts the attack surface a lot. To make it fully production-grade I'd: pin exact versions instead of floating tags (`golang:1.22.4-alpine` not `:latest`), run as the `nonroot` distroless variant so the process isn't root inside the container, add a `.dockerignore` so build context stays small, scan the final image with something like Trivy in CI before it's allowed to push, and add labels for version/commit metadata so I can trace an image back to the exact build."

---

## 11. What is the purpose of WORKDIR?

"It sets the working directory for every instruction that follows — `RUN`, `COPY`, `CMD`, `ENTRYPOINT` — all resolve relative to it. It's cleaner than chaining `RUN cd /app && ...`, because `cd` doesn't persist across separate `RUN` layers anyway; `WORKDIR` does."

---

## 12. What is the purpose of ENTRYPOINT?

"ENTRYPOINT defines the fixed command that always executes when the container starts — it can't be overridden at `docker run` time unless you explicitly pass `--entrypoint`. That's different from CMD, which just supplies default arguments and _can_ be overridden. In my Go app, `ENTRYPOINT ["./app"]` means the container's only job is to run that binary — nothing else."

---

## 13. Why use `COPY app.jar app.jar` instead of `COPY . .`?

"Layer caching and image hygiene. If you `COPY . .`, any change anywhere in the source tree invalidates that layer and forces a re-copy, even for unrelated changes. Copying just the built artifact means the layer only changes when the artifact itself changes, keeps the final image free of source code, build tools, and test files, and reduces what's exposed if the image is ever compromised. It's the exact same principle as my multi-stage Go build — stage 2 only copies the compiled binary out of stage 1, never the source."

---

## 14. What is the difference between Maven and Gradle?

**Honest framing:** not personally driven either in production, but you should know this cold as baseline knowledge:

"Maven uses XML (`pom.xml`) and a fixed, convention-over-configuration build lifecycle — predictable but verbose and slower since it doesn't cache builds well. Gradle uses a Groovy or Kotlin DSL, supports incremental builds and build caching, so repeated builds are noticeably faster, and it's more programmable/flexible — it's also Android's default build tool for that reason."

---

## 15. Have you worked on Spring Boot?

**Honest bridge:** "Not directly — my hands-on projects are Go-based, so I haven't built or deployed a Spring Boot app myself. I do understand the model conceptually: it packages the app and an embedded server (usually Tomcat) into one self-contained executable JAR, so you skip installing a separate app server. That's actually very similar in spirit to what I do with a static Go binary in a distroless container — one self-contained artifact, no external runtime dependency."

---

## 16. How do you manage environment-specific configuration in Docker?

"A few layers depending on context. At the plain Docker level: `--env-file` or `docker-compose` `env_file` per environment. In Kubernetes, which is where I actually operate — ConfigMaps for non-sensitive config, Secrets for sensitive values, injected as env vars or mounted files. And since I deploy through Helm, the real separation happens at the values-file level — `values-dev.yaml`, `values-prod.yaml` — same chart template, different values per environment, so the actual manifests are generated per-environment rather than hand-maintained."

---

## 17. Your deployment succeeded, but the application isn't opening. How would you troubleshoot it?

This is a strong one for you — walk it top-down, this is basically your EKS+ALB troubleshooting muscle:

"I'd work from the pod outward. First, `kubectl get pods` — is it actually Running, or stuck in CrashLoopBackOff/Pending? `kubectl describe pod` and `kubectl logs` tell me if the app itself is even healthy inside the container. If the pod's fine, next is the Service — does the selector's labels actually match the pod's labels, and is the target port correct? Then I check readiness probes — if the probe's failing, the pod never gets added to the Service endpoints even though it's 'running.' From there I go external: is the Ingress or ALB actually pointing at that Service, are the ALB target groups healthy, and finally security groups and NACLs — a lot of 'succeeded but not opening' issues are just a security group blocking the ALB-to-pod path, not the app at all. I'd isolate app-level versus network-level early by doing a quick `kubectl port-forward` straight to the pod — if that works, the app's fine and it's a networking/routing problem outside the pod."

---

## 18. How do you connect to an AWS EC2 instance?

"Standard way is SSH with the key pair: `ssh -i keypair.pem ec2-user@<public-ip>`. If the instance is in a private subnet, I go through a bastion host in the public subnet first. I've also used AWS Systems Manager Session Manager, which doesn't need an open inbound port at all — it goes through the SSM agent over an outbound HTTPS connection, which is the more secure option since there's no port 22 exposed to the internet."

_Small personal detail you can drop if asked how you avoid key issues: "I generate SSH keys via the AWS CLI on Linux rather than downloading a PEM on Windows — Windows-downloaded PEMs get corrupted with extra carriage returns and SSH rejects them."_

---

## 19. What if port 22 is closed?

"Then direct SSH won't work no matter what key I have. Options: use AWS Systems Manager Session Manager, which needs no inbound port at all — just the SSM agent and an outbound connection; or fix the security group to allow 22 from my IP if that's actually intended; or go through a bastion host that already has permitted access; EC2 Instance Connect is another option if it's enabled. In practice, Session Manager is the one I'd reach for first since it avoids opening 22 to begin with."

---

## 20. AWS provides an HTTP endpoint. How would you enable HTTPS?

"Put an Application Load Balancer in front of it, request a certificate through AWS Certificate Manager for the domain, attach that cert to a listener on port 443 on the ALB, and add a listener rule on port 80 that redirects to 443. If it's static/cacheable content instead, I'd put CloudFront in front with the ACM cert there instead."

---

## 21. Does AWS provide SSL certificates? What is AWS Certificate Manager (ACM)?

"Yes — ACM issues and manages free public TLS certificates for use with integrated AWS services like ALB, CloudFront, and API Gateway. It handles domain validation, either via DNS or email, and auto-renews the cert before expiry so you're not manually rotating certs. The catch is ACM certs are locked to those AWS-integrated services — you can't export the private key and use it on, say, a self-managed Nginx box, unless you're using ACM Private CA."

---

## 22. Explain the architecture of your CI/CD pipeline.

"End to end it looks like this: source repo on GitHub → GitHub Actions runner builds and tests the code, does the multi-stage Docker build → pushes the image to Docker Hub → the pipeline bumps the image tag in a separate GitOps repo's Helm values.yaml and commits → ArgoCD, running inside the cluster, watches that GitOps repo and auto-syncs the diff → Kubernetes — k3s or EKS depending on environment — rolls out the new pods → for the EKS side, AWS Load Balancer Controller watches the Ingress resource and provisions/updates an ALB → that ALB is what actually receives user traffic. I deliberately keep the app repo and the GitOps repo separate — app code changes shouldn't be what triggers a deploy directly; only a validated image tag bump should."

---

## 23. Have you worked with Microservices?

"My hands-on project is a single Go service, so I haven't run a multi-service architecture end-to-end myself yet. But I've applied the underlying patterns through Kubernetes — namespace isolation, service-to-service communication through ClusterIP and CoreDNS, independent Helm charts per component, each deployable and scalable on its own. Those are the same building blocks a microservices setup uses; I just haven't had a project with more than one service in it yet."

---

## 24. How do you deploy multiple microservices using Docker Compose?

```yaml
version: '3.8'
services:
  web:
    build: ./web
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      - DB_HOST=db
    networks:
      - app-net

  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=examplepass
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-net

networks:
  app-net:

volumes:
  db-data:
```

"Each service gets its own build context/image, they share a network so they can reach each other by service name (`db` resolves via Compose's built-in DNS), and I use named volumes for anything that needs to persist — like the database — beyond a container's lifecycle."

---

## 25. What is `depends_on` in Docker Compose?

"It controls startup _order_ — Compose will start `db` before `web`. But it only waits for the `db` container to _start_, not for Postgres inside it to actually be ready to accept connections. That gap has bitten people before — the fix is adding a `healthcheck` to the dependency and using `depends_on: condition: service_healthy`, or handling retries in the app itself."

---

## 26. Which dependencies are defined in your pom.xml?

**Honest bridge:** "That question assumes a Maven-based Java project — my actual projects are Go, so there's no pom.xml involved. If you're asking generically what a typical Spring Boot pom.xml would carry: things like `spring-boot-starter-web` for REST endpoints, `spring-boot-starter-data-jpa` plus a DB driver for persistence, and test starters like `spring-boot-starter-test`. But I'd rather be upfront that this isn't my project's stack than guess at specifics I didn't actually configure."

---

## 27. What type of Java application did you deploy (WAR or JAR)?

**Honest bridge:** "I haven't deployed a Java WAR or JAR — my deployment artifact is a compiled Go binary shipped in a distroless container. Conceptually though: a WAR gets deployed _into_ an external servlet container like Tomcat, while a JAR — especially a Spring Boot 'fat jar' — is self-contained with an embedded server and runs directly with `java -jar`. Modern practice leans toward executable JARs specifically because it's simpler to containerize — one artifact, no external app server dependency — which is the same reason I like distroless: one binary, nothing else in the image."

---

## 28. Do you have any questions for the interviewer?

Always have 3-4 ready — tailored toward your Senior DevOps/Tech Lead trajectory, not generic:

- "What does the current CI/CD and infra setup look like, and where's the biggest pain point the team wants solved?"
- "How is on-call/incident response structured for the infra team?"
- "How much ownership does this role have over cloud cost decisions versus just infrastructure design?"
- "What does progression from this role to a Senior/Lead position typically look like on this team?"

---

_Send me the next batch whenever you're ready — same format, and flag me if any of the honest-bridge answers above actually don't apply (i.e., if you do have real Tomcat/Maven/Spring Boot experience I don't know about) and I'll rewrite those properly._

<!-- related -->
---
**Related:** [[4.9-Docker-Interview-QA|Docker — Interview Q&A]] · [[4.4-Dockerfile-Reference|Dockerfile — Complete Reference]] · [[4.7-Docker-Compose|Docker Compose]] · [[7.4-Jenkins-CICD-Pipeline|Jenkins CI/CD — Automated Deployment Pipeline]]
<!-- /related -->
