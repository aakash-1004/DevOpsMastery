---
title: "PHASE 7 — Jenkinsfile: Writing, Debugging, and Correction"
type: project
module: cicd
status: completed
tags:
  - jenkins
  - kubeadm
  - kubernetes
  - rbac
  - sonarqube
  - nexus
  - trivy
  - prometheus
  - grafana
  - blackbox-exporter
  - cicd
  - devops
  - notes
  - project
order: 13
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-06-RBAC-Jenkins-to-Kubernetes-Access]]"
next: "[[Boardgame-08-Pipeline-Runs-and-Real-Debugging]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 7 — Jenkinsfile: Writing, Debugging, and Correction

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-06-RBAC-Jenkins-to-Kubernetes-Access|PHASE 6 — RBAC: Jenkins → Kubernetes Deployment Access]] · → [[Boardgame-08-Pipeline-Runs-and-Real-Debugging|PHASE 8 — Pipeline Runs and Real Debugging]]
<!-- /nav -->

## Initial Jenkinsfile written by hand (from the video), with bugs caught in review

Bugs found and fixed:

1. `scripts{}` → `script{}` — appeared 3 times, not a valid Jenkins DSL step name
2. `docker built` → `docker build` — not a valid Docker subcommand
3. `mvn deply` → `mvn deploy` — not a valid Maven lifecycle phase
4. `credentialId` → `credentialsId` — wrong parameter name on waitForQualityGate
5. `abortPipeline: false` → `true` — a non-enforcing quality gate defeats its purpose
6. Both Trivy scans wrote to the same filename (`trivy-fs-report.html`) — the image scan silently overwrote the filesystem scan's report; renamed second to `trivy-image-report.html`
7. `:latest` tag → `${BUILD_NUMBER}` — for traceability between deployed image and the Jenkins run that produced it
8. Stage name typo: "Dcoker Image Scan" → "Docker Image Scan"

## deployment-service.yaml — 3 bugs caught before it could fail silently

1. Wrong image entirely: `adijaiswal/boardshack:latest` → `aakash0908/boardgame:BUILD_TAG_PLACEHOLDER` (pointed at tutorial creator's image, not ours)
2. Missing `namespace: webapps` on both Deployment and Service — would silently bypass all RBAC work by deploying to `default` namespace
3. `type: LoadBalancer` — kept deliberately; EXTERNAL-IP stays `<pending>` forever on bare-metal kubeadm (no cloud controller manager to provision a real LB) — accessible via auto-assigned NodePort instead

## BUILD_TAG_PLACEHOLDER mechanism

The Jenkinsfile's deploy stage runs:

```groovy
sh "sed -i 's/BUILD_TAG_PLACEHOLDER/${BUILD_NUMBER}/g' deployment-service.yaml"
```

This rewrites the placeholder in the YAML to the actual current build number (e.g., `aakash0908/boardgame:2`) at deploy time, matching the exact image this specific pipeline run just built and pushed. The sed operates on the Jenkins workspace copy only — doesn't commit back to Git, so each run starts fresh from the placeholder.

## Final working Jenkinsfile (13 stages + post block)

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/aakash-1004/boardgame-jenkins-cicd-pipeline.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=boardgame -Dsonar.projectKey=boardgame \
                           -Dsonar.java.binaries=. \
                           -Dsonar.exclusions=**/trivy-*-report.html '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t aakash0908/boardgame:${BUILD_NUMBER} ."
                    }
                }
            }
        }
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html aakash0908/boardgame:${BUILD_NUMBER}"
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push aakash0908/boardgame:${BUILD_NUMBER}"
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh "sed -i 's/BUILD_TAG_PLACEHOLDER/${BUILD_NUMBER}/g' deployment-service.yaml"
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.33.140:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.33.140:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """
                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'aakashrao1004@gmail.com',
                    from: 'aakashrao0908@gmail.com',
                    replyTo: 'aakashrao0908@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-*-report.html'
                )
            }
        }
    }
}
```

## sonar.exclusions — a bug traced to a single missing character

First Quality Gate failure: 79.62% code duplication. Investigation via SonarQube's file-level breakdown showed all actual Java source at 0.0% duplication — the false positive came from Trivy's own HTML scan reports (templated, repetitive boilerplate) sitting in the analysis scope. Added `-Dsonar.exclusions=**/trivy-*-report.html` to fix this. First attempt silently failed because the pattern was truncated to `.htm` (missing final `l`) — confirmed by reading the scanner's own echoed config line in the console log: `Excluded sources: **/trivy-*-report.htm`, `0 files ignored`.

## Pipeline-from-SCM conversion

Changed job configuration from "Pipeline script" (Jenkinsfile typed in Jenkins UI) to "Pipeline script from SCM":

- SCM: Git
- Repository URL: `https://github.com/aakash-1004/boardgame-jenkins-cicd-pipeline.git`
- Credentials: `git-cred`
- Branch: `*/main`
- Script Path: `Jenkinsfile`

Why this matters: the Jenkinsfile now lives in Git, version-controlled alongside the application code — has real commit history, can be code-reviewed via PRs, survives Jenkins server rebuilds.

## Email notification debugging

1. Tested via Jenkins System → Extended E-mail Notification → "Test configuration" button — test email arrived successfully to `aakashrao1004@gmail.com`
2. Pipeline emails never arrived — diagnosed via Jenkins Script Console test, which revealed: SMTP port was set to 25, while "Use SSL" was checked — port 25 doesn't support SSL, AND AWS blocks outbound port 25 by default on all EC2 instances as an anti-spam measure
3. Fixed by changing SMTP port to 465 — emails started arriving immediately
