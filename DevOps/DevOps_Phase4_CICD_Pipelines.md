# DevOps Phase 4 — CI/CD Pipelines

> **Learning Goal:** Understand how software moves from a developer's laptop to production automatically, safely, and repeatably — and master the tools that make it happen.

---

## Table of Contents

1. [What is CI/CD?](#1-what-is-cicd)
2. [Why CI/CD Exists](#2-why-cicd-exists)
3. [CI vs CD — The Difference](#3-ci-vs-cd--the-difference)
4. [CI/CD Pipeline Anatomy](#4-cicd-pipeline-anatomy)
5. [Pipeline Stages Deep Dive](#5-pipeline-stages-deep-dive)
6. [Artifacts](#6-artifacts)
7. [Jenkins — Complete Guide](#7-jenkins--complete-guide)
8. [GitHub Actions — Complete Guide](#8-github-actions--complete-guide)
9. [GitLab CI — Complete Guide](#9-gitlab-ci--complete-guide)
10. [Rollbacks — Strategies & Implementation](#10-rollbacks--strategies--implementation)
11. [Advanced Pipeline Patterns](#11-advanced-pipeline-patterns)
12. [Security in CI/CD (DevSecOps Basics)](#12-security-in-cicd-devsecops-basics)
13. [Interview Mastery](#13-interview-mastery)

---

## 1. What is CI/CD?

### Beginner Explanation

Imagine a team of 10 developers all writing code for the same app. Without a system, they'd finish their work, email their code to someone, that person would try to combine it all, fix conflicts, test it manually, package it up, and upload it to the server. This takes days, breaks constantly, and is terrifying every release.

**CI/CD automates that entire pipeline** — from code commit to live production — in minutes.

| Term | Stands For | What It Means |
|------|-----------|---------------|
| CI | Continuous Integration | Every code change is automatically built and tested |
| CD | Continuous Delivery | Every passing build is automatically packaged and ready to deploy |
| CD | Continuous Deployment | Every passing build is automatically deployed to production (no human needed) |

### Technical Explanation

CI/CD is a **software delivery practice** that automates the lifecycle from source code commit to production deployment using a series of scripted stages (a "pipeline") executed by a CI/CD server.

Each pipeline run is triggered by a **VCS event** (push, PR, tag) and executes deterministic steps — compile, test, scan, package, push artifact, deploy — producing a **traceable, reproducible delivery**.

### The Problem It Solves

```
WITHOUT CI/CD:
Developer → manually test locally → manually build → manually upload → manual test on server
  ↓
"Works on my machine" syndrome
Merge conflicts discovered late
Deployment takes hours
Rollback is manual and risky
No audit trail

WITH CI/CD:
Developer pushes code → automated pipeline in 5 minutes
  ↓
Conflicts caught immediately
Build verified on clean environment
Deployment is one-click or automatic
Rollback is instant
Every run is logged
```

---

## 2. Why CI/CD Exists

### The Pre-CI/CD World (2005–2010)

```
Timeline without CI/CD:
Week 1-3: Developers code in isolation
Week 4: "Integration week" — combine everyone's code
         → Compilation errors from conflicts
         → Tests fail in ways nobody understands
         → 80% of the sprint is debugging integration
Week 5: Manual testing by QA team
Week 6: "Release day" — manually deploy, pray it works
Week 7: Emergency hotfixes because something broke
```

This was called **"integration hell"** — the longer you wait to integrate, the worse it gets.

### The CI/CD Solution (2010–present)

```
With CI/CD:
Every commit → pipeline runs in 5–15 minutes
  → If tests pass: green build, safe to merge
  → If tests fail: developer fixes immediately
  
Release is not an event — it's a side effect of passing tests
```

### Business Value

| Metric | Without CI/CD | With CI/CD |
|--------|--------------|------------|
| Release frequency | Monthly/quarterly | Daily/hourly |
| Lead time (idea → production) | Weeks | Hours |
| Mean time to recovery (MTTR) | Hours/days | Minutes |
| Change failure rate | 15–45% | 0–15% |

*(Source: DORA metrics — DevOps Research and Assessment)*

---

## 3. CI vs CD — The Difference

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   CODE                CONTINUOUS INTEGRATION                       │
│   COMMIT              ─────────────────────                        │
│      │                Build → Unit Tests → Integration Tests       │
│      ▼                → Code Coverage → Static Analysis            │
│   ┌──────┐                                                         │
│   │  CI  │──────────► Green Build (Artifact Created)              │
│   └──────┘                     │                                   │
│                                ▼                                   │
│                   CONTINUOUS DELIVERY                              │
│                   ──────────────────                               │
│                   Package → Deploy to Staging                      │
│                   → Acceptance Tests → Performance Tests           │
│                   → Manual Approval Gate ────────► Deploy         │
│                                                   to Prod         │
│                                                                     │
│                   CONTINUOUS DEPLOYMENT                            │
│                   ─────────────────────                            │
│                   Same as above but NO manual approval gate        │
│                   Automatically deploys to production              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Distinction

| | Continuous Integration | Continuous Delivery | Continuous Deployment |
|-|----------------------|--------------------|-----------------------|
| Automation | Build + Test | + Staging deploy | + Production deploy |
| Human involvement | Developer fixes failures | Approves production push | Zero (fully automated) |
| Risk level | Low | Medium | High (needs mature testing) |
| Who uses it | Everyone | Most mature teams | Netflix, Amazon, Google |

---

## 4. CI/CD Pipeline Anatomy

### Full Pipeline Diagram

```mermaid
graph LR
    A[Developer Push] --> B[Source Control\nGit Trigger]
    B --> C[Pipeline Starts]
    
    C --> D[Stage 1:\nCode Checkout]
    D --> E[Stage 2:\nDependency Install]
    E --> F[Stage 3:\nBuild/Compile]
    F --> G{Build\nSuccess?}
    G -- No --> H[Notify Developer\nFAIL]
    G -- Yes --> I[Stage 4:\nUnit Tests]
    I --> J{Tests\nPass?}
    J -- No --> H
    J -- Yes --> K[Stage 5:\nCode Quality\nSAST Scan]
    K --> L[Stage 6:\nBuild Artifact\nDocker Image]
    L --> M[Stage 7:\nPush to Registry\nArtifact Store]
    M --> N[Stage 8:\nDeploy to Staging]
    N --> O[Stage 9:\nIntegration Tests\nSmoke Tests]
    O --> P{All\nTests Pass?}
    P -- No --> H
    P -- Yes --> Q{Manual\nApproval?}
    Q -- No Approval Needed --> R[Stage 10:\nDeploy to Production]
    Q -- Approved --> R
    R --> S[Stage 11:\nPost-Deploy Tests\nMonitoring]
    S --> T[SUCCESS ✅]
```

### Pipeline as Code vs GUI

**Pipeline as Code (recommended):**
- Defined in a file in your repo (`.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`)
- Versioned alongside application code
- Reviewable via pull requests
- Reproducible

**GUI-based pipelines:**
- Configured via web interface
- Not version controlled
- Harder to reproduce, audit, or review

---

## 5. Pipeline Stages Deep Dive

### Stage 1: Source Checkout

The CI server clones the exact commit that triggered the pipeline.

```bash
# What happens internally
git clone https://github.com/org/repo.git
git checkout <commit-sha>
```

**Why SHA, not branch name?** Branch names are mutable pointers. The SHA guarantees you build the exact code that was committed, not what's on the branch tip at build time.

---

### Stage 2: Dependency Installation

```bash
# Node.js
npm ci                    # NOT npm install — ci is deterministic from lockfile

# Python
pip install -r requirements.txt

# Java/Maven
mvn dependency:resolve

# Go
go mod download
```

**`npm ci` vs `npm install`:** `npm install` can update `package-lock.json`. `npm ci` strictly follows the lockfile — critical for reproducible builds.

---

### Stage 3: Build / Compile

```bash
# Java
mvn clean package -DskipTests

# Go
go build -o ./bin/app ./cmd/main.go

# Docker
docker build -t myapp:$GIT_SHA .

# Frontend
npm run build
```

**Build in CI, not local.** The CI environment is clean and standardized. Local builds can have "works on my machine" problems (different OS, cached files, environment variables).

---

### Stage 4: Unit Tests

```bash
# Run tests with coverage report
pytest --cov=app --cov-report=xml tests/unit/

# JUnit
mvn test

# Jest (Node.js)
npm test -- --coverage
```

**What makes a good unit test stage:**
- Fast (< 2 minutes)
- Isolated (no database, no network)
- High coverage (70%+ meaningful coverage)
- Fails fast on first failure (fail-fast mode)

---

### Stage 5: Code Quality & Security Scanning

```bash
# Linting
eslint src/
pylint app/
golangci-lint run

# Code coverage threshold
# Fail if coverage drops below 70%

# SAST (Static Application Security Testing)
# Finds vulnerabilities in your code before it runs
bandit -r app/           # Python security scanner
semgrep --config=auto .  # Multi-language

# Dependency vulnerability scanning
npm audit
safety check             # Python
trivy fs .               # Multi-language
```

---

### Stage 6: Build Artifact

An **artifact** is the deployable unit produced by the build. The artifact is built once and promoted through environments — never rebuilt.

```bash
# Docker image (most common in modern CI/CD)
docker build -t registry.example.com/myapp:$GIT_SHA .
docker build -t registry.example.com/myapp:latest .

# JAR file (Java)
mvn package   # Produces target/myapp-1.0.0.jar

# Binary (Go)
go build -o myapp-linux-amd64

# Zip/tar (Lambda, etc.)
zip -r function.zip app/ requirements.txt
```

**Immutable artifacts:** Once built, an artifact is never modified. If a bug is found, a new artifact is built from new code. This ensures what you test is exactly what you deploy.

---

### Stage 7: Push to Artifact Registry

```bash
# Docker
docker push registry.example.com/myapp:$GIT_SHA

# AWS ECR
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:$GIT_SHA

# Nexus / JFrog Artifactory
mvn deploy
```

**Why push before deploy?** The registry is the single source of truth. Any environment (staging, prod, region) pulls from the same registry, ensuring identical artifacts.

---

### Stage 8: Deploy to Staging

```bash
# Kubernetes
kubectl set image deployment/myapp myapp=registry.example.com/myapp:$GIT_SHA

# Docker Compose (simpler environments)
docker-compose up -d

# Helm (Kubernetes package manager)
helm upgrade myapp ./chart --set image.tag=$GIT_SHA

# Terraform (infra)
terraform apply -var="image_tag=$GIT_SHA"
```

---

### Stage 9: Integration & Smoke Tests

```bash
# Wait for deployment to be ready
kubectl rollout status deployment/myapp

# Smoke test: verify the app is alive
curl -f https://staging.example.com/health

# Integration tests against real staging environment
pytest tests/integration/ --base-url=https://staging.example.com

# API contract tests
newman run postman_collection.json --environment staging.json
```

**Smoke test:** A minimal set of tests that verify the app is alive and critical paths work. Runs in < 1 minute. If smoke tests fail, the deployment is rolled back immediately.

---

### Stage 10: Production Deploy

```bash
# With manual approval gate (Continuous Delivery)
# Human clicks "Approve" in the CI/CD UI

# Kubernetes rolling update (zero-downtime)
kubectl set image deployment/myapp myapp=registry.example.com/myapp:$GIT_SHA
kubectl rollout status deployment/myapp --timeout=5m

# Blue/Green deploy
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# AWS CodeDeploy / Elastic Beanstalk
aws deploy create-deployment \
  --application-name myapp \
  --deployment-group-name production \
  --revision revisionType=GitHub,gitHubLocation={repository=org/repo,commitId=$GIT_SHA}
```

---

### Stage 11: Post-Deploy Verification

```bash
# Run smoke tests in production
curl -f https://api.production.com/health

# Check error rate in monitoring (wait 5 minutes and check)
# Alert if error rate spikes

# Automated rollback trigger
if [ "$(curl -s -o /dev/null -w "%{http_code}" https://api.production.com/health)" != "200" ]; then
    kubectl rollout undo deployment/myapp
    echo "ROLLBACK TRIGGERED"
    exit 1
fi
```

---

## 6. Artifacts

### What is an Artifact?

An artifact is any file produced by a build step that is:
1. **Stored** in an artifact repository
2. **Versioned** (tagged with git SHA, version number, or timestamp)
3. **Immutable** — never changed after creation
4. **Promoted** through environments (never rebuilt per environment)

### Types of Artifacts

| Type | Examples | Storage |
|------|---------|---------|
| Docker image | `myapp:abc1234` | Docker Registry (ECR, GCR, Docker Hub, Harbor) |
| JAR/WAR | `myapp-1.0.0.jar` | Nexus, JFrog Artifactory, Maven Central |
| npm package | `mylib-2.1.0.tgz` | npm registry, Verdaccio |
| Zip/tar | `function.zip` | S3, GCS |
| Helm chart | `myapp-1.2.0.tgz` | ChartMuseum, OCI registry |
| Binary | `myapp-linux-amd64` | GitHub Releases, S3 |
| Test reports | `coverage.xml`, `junit.xml` | CI system, S3 |

### Artifact Versioning Strategy

```
# Strategy 1: Git SHA (most precise, recommended for Docker)
myapp:a3f2c1b

# Strategy 2: Semantic version (for libraries/packages)
mylib:1.4.2

# Strategy 3: Timestamp + SHA (for traceability)
myapp:20240115-a3f2c1b

# Strategy 4: Build number
myapp:build-1234

# NEVER: "latest" as the only tag in production
# "latest" is mutable — you lose traceability
```

### Artifact Promotion

```
Build Once → Test with same artifact → Deploy same artifact to prod

DEV BUILD                STAGING                   PRODUCTION
  ┌──────┐               ┌──────┐                   ┌──────┐
  │Build │               │Pull  │                   │Pull  │
  │Image │──── push ────►│Same  │── manual/auto ───►│Same  │
  │:sha  │               │Image │                   │Image │
  └──────┘               └──────┘                   └──────┘

This guarantees: what you tested IS what you deployed.
If you rebuild for prod, environment differences can creep in.
```

---

## 7. Jenkins — Complete Guide

### What is Jenkins?

Jenkins is an **open-source automation server** written in Java, used to build, test, and deploy software. It is the oldest and most widely used CI/CD tool — the "Swiss Army knife" of CI/CD.

**Founded:** 2011 (forked from Hudson)  
**Language:** Java  
**Plugins:** 1,800+ plugins  
**Model:** Self-hosted (you manage the server)

### Jenkins Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    JENKINS MASTER                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Job/Pipeline│  │ Build Queue │  │  Plugin     │    │
│  │  Scheduler   │  │             │  │  Manager    │    │
│  └──────┬───────┘  └─────────────┘  └─────────────┘    │
│         │                                               │
│         │ Delegates work to agents                      │
└─────────┼───────────────────────────────────────────────┘
          │
    ┌─────┴──────────────────────────────────┐
    │                                        │
    ▼                                        ▼
┌──────────────────┐                ┌──────────────────┐
│  JENKINS AGENT 1 │                │  JENKINS AGENT 2 │
│  (Linux, Java)   │                │  (Windows, .NET) │
│                  │                │                  │
│  - Runs builds   │                │  - Runs builds   │
│  - Executes jobs │                │  - Executes jobs │
└──────────────────┘                └──────────────────┘
```

**Master:** Orchestrates — schedules jobs, distributes to agents, shows UI  
**Agent:** Executes — runs the actual build commands on its machine

### Jenkinsfile (Declarative Pipeline)

The Jenkinsfile is a text file that defines your pipeline as code. Stored in root of your repository.

```groovy
// Declarative Pipeline (recommended)
pipeline {
    // Where to run — any available agent
    agent any

    // Environment variables available to all stages
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME      = 'myapp'
        GIT_SHA         = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    // Run pipeline on these triggers
    triggers {
        pollSCM('H/5 * * * *')   // Poll SCM every 5 minutes
        // OR: webhook triggers (preferred)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm    // Checkout the triggering commit
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    // Publish test results in Jenkins UI
                    junit 'coverage/junit.xml'
                    publishHTML(target: [
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_SHA} .
                    docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
                               ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push to Registry') {
            steps {
                // Use stored credential — never hardcode passwords
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} \
                            -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_SHA}
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh """
                    kubectl set image deployment/myapp \
                        myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
                        --namespace staging
                    kubectl rollout status deployment/myapp --namespace staging --timeout=3m
                """
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl -f https://staging.example.com/health'
            }
        }

        stage('Deploy to Production') {
            // Only run on main branch
            when {
                branch 'main'
            }
            // Require manual approval
            input {
                message "Deploy to Production?"
                ok "Deploy"
                submitter "admin,release-manager"
            }
            steps {
                sh """
                    kubectl set image deployment/myapp \
                        myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
                        --namespace production
                    kubectl rollout status deployment/myapp --namespace production --timeout=5m
                """
            }
        }
    }

    // Run these after pipeline regardless of result
    post {
        success {
            slackSend channel: '#deployments',
                      message: "✅ ${IMAGE_NAME}:${GIT_SHA} deployed to production"
        }
        failure {
            slackSend channel: '#deployments',
                      message: "❌ Pipeline FAILED for ${IMAGE_NAME}:${GIT_SHA}"
            // Automatically roll back on failure
            sh 'kubectl rollout undo deployment/myapp --namespace production || true'
        }
        always {
            cleanWs()    // Clean workspace after every run
        }
    }
}
```

### Scripted Pipeline (older style)

```groovy
// Scripted Pipeline — full Groovy, more flexible but verbose
node {
    def gitSha = ''
    
    stage('Checkout') {
        checkout scm
        gitSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
    
    stage('Build') {
        try {
            sh 'mvn clean package'
        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            throw e
        }
    }
    
    stage('Archive') {
        archiveArtifacts artifacts: 'target/*.jar'
    }
}
```

### Jenkins Key Concepts

| Concept | Description |
|---------|-------------|
| Job | A single build task (freestyle, pipeline, multibranch) |
| Build | One execution of a job |
| Workspace | Directory on agent where job runs |
| Artifact | File saved after a build |
| Credential | Stored secret (password, SSH key, token) |
| Plugin | Extension that adds capability |
| Multibranch Pipeline | Automatically creates pipelines for every branch |
| Blue Ocean | Modern UI plugin for Jenkins |

### Jenkins Credentials (Never Hardcode Secrets)

```groovy
// ✅ CORRECT — use stored credentials
withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
    sh 'git clone https://x-access-token:$GITHUB_TOKEN@github.com/org/repo'
}

// ✅ SSH key credential
sshagent(credentials: ['deploy-ssh-key']) {
    sh 'ssh deploy@server "cd /app && git pull"'
}

// ❌ WRONG — hardcoded secret
sh 'git clone https://mytoken123@github.com/org/repo'   // NEVER DO THIS
```

### Jenkins Pros and Cons

| Pros | Cons |
|------|------|
| Extremely flexible (1800+ plugins) | Complex to set up and maintain |
| Self-hosted (full control) | Requires dedicated infrastructure |
| Mature, battle-tested | Plugin compatibility issues |
| Any language, any platform | Groovy learning curve |
| Active community | UI is dated (without Blue Ocean) |
| Free and open source | Can become a "snowflake" server |

---

## 8. GitHub Actions — Complete Guide

### What is GitHub Actions?

GitHub Actions is a **cloud-native CI/CD platform** built directly into GitHub. No separate server needed — GitHub runs your pipelines on their infrastructure.

**Released:** 2019  
**Model:** SaaS — GitHub manages infrastructure  
**Pricing:** Free for public repos; free minutes for private (2,000 min/month on free tier)

### Core Concepts

```
WORKFLOW          = The entire automation (lives in .github/workflows/*.yml)
JOB               = A group of steps that run on the same runner
STEP              = A single command or action
ACTION            = A reusable unit (like a plugin)
RUNNER            = The VM that executes your job
EVENT             = What triggers the workflow (push, PR, schedule, etc.)
```

### GitHub Actions Architecture

```
.github/
  workflows/
    ci.yml              ← Workflow definition
    deploy.yml
    cron-cleanup.yml

TRIGGER (Event)
    │
    ▼
WORKFLOW (ci.yml)
    │
    ├── JOB: build (runs on ubuntu-latest)
    │       │
    │       ├── Step: Checkout code
    │       ├── Step: Install Node.js
    │       ├── Step: npm ci
    │       ├── Step: npm test
    │       └── Step: Upload coverage
    │
    └── JOB: deploy (runs on ubuntu-latest, needs: build)
            │
            ├── Step: Download build artifacts
            ├── Step: Build Docker image
            ├── Step: Push to registry
            └── Step: Deploy to Kubernetes
```

### Complete GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml

name: CI/CD Pipeline

# ─── TRIGGERS ───────────────────────────────────────────────
on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
  schedule:
    - cron: '0 2 * * 1'   # Every Monday at 2am UTC
  workflow_dispatch:        # Manual trigger button in GitHub UI

# ─── GLOBAL ENVIRONMENT VARIABLES ───────────────────────────
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

# ─── JOBS ────────────────────────────────────────────────────
jobs:

  # ─── JOB 1: Test ─────────────────────────────────────────
  test:
    name: Build & Test
    runs-on: ubuntu-latest    # GitHub-hosted runner

    # Matrix: run tests on multiple Node versions simultaneously
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      # Step 1: Checkout code at the triggering commit
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up Node.js (with caching)
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'              # Cache node_modules between runs

      # Step 3: Install dependencies (from lockfile)
      - name: Install dependencies
        run: npm ci

      # Step 4: Lint
      - name: Lint
        run: npm run lint

      # Step 5: Run tests with coverage
      - name: Run tests
        run: npm test -- --coverage

      # Step 6: Upload coverage to Codecov
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

      # Step 7: Upload test artifacts
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()    # Upload even if tests fail
        with:
          name: test-results-node${{ matrix.node-version }}
          path: coverage/
          retention-days: 30

  # ─── JOB 2: Security Scan ────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test             # Only run after test passes

    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'    # Fail the job if vulnerabilities found

  # ─── JOB 3: Build & Push Docker Image ────────────────────
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    
    # Only build from main or develop, not from all PRs
    if: github.event_name == 'push'

    # Outputs available to downstream jobs
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.push.outputs.digest }}

    permissions:
      contents: read
      packages: write     # Required to push to ghcr.io

    steps:
      - uses: actions/checkout@v4

      # Set up Docker Buildx (for multi-platform builds)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Login to GitHub Container Registry
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}    # Auto-provided by GitHub

      # Generate image tags and labels
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}

      # Build and push (with layer caching from registry)
      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
          platforms: linux/amd64,linux/arm64   # Multi-arch build

  # ─── JOB 4: Deploy to Staging ────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
      url: https://staging.example.com    # Shows in GitHub UI

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Kubernetes (staging)
        uses: azure/k8s-deploy@v4
        with:
          namespace: staging
          manifests: k8s/staging/
          images: ${{ needs.build.outputs.image-tag }}

      - name: Run smoke tests
        run: |
          sleep 30   # Wait for pods to start
          curl -f https://staging.example.com/health

  # ─── JOB 5: Deploy to Production (with approval) ─────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    
    # Only deploy from main branch
    if: github.ref == 'refs/heads/main'
    
    environment:
      name: production          # Environment with required reviewers configured
      url: https://example.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Kubernetes (production)
        uses: azure/k8s-deploy@v4
        with:
          namespace: production
          manifests: k8s/production/
          images: ${{ needs.build.outputs.image-tag }}

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n production --timeout=5m
          curl -f https://example.com/health
```

### GitHub Actions Key Concepts

#### Events (Triggers)

```yaml
on:
  push:                   # Any push
  pull_request:           # PR opened, synchronized, reopened
  release:
    types: [published]    # When release is published
  schedule:
    - cron: '0 6 * * *'  # Daily at 6am
  workflow_dispatch:      # Manual trigger
  workflow_call:          # Called by another workflow
  repository_dispatch:    # Triggered via API
```

#### Contexts and Expressions

```yaml
# Access GitHub context
${{ github.sha }}           # Full commit SHA
${{ github.ref }}           # refs/heads/main
${{ github.event_name }}    # push, pull_request, etc.
${{ github.actor }}         # Username who triggered
${{ github.repository }}    # org/repo-name
${{ github.run_number }}    # Sequential build number

# Access secrets
${{ secrets.MY_SECRET }}    # Never printed in logs

# Access job outputs
${{ needs.build.outputs.image-tag }}

# Conditional expressions
if: github.ref == 'refs/heads/main'
if: failure()
if: success() && github.event_name == 'push'
```

#### Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      KUBE_CONFIG:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ inputs.environment }}
        run: echo "Deploying to ${{ inputs.environment }}"

---
# .github/workflows/ci.yml — using the reusable workflow
jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

#### Self-hosted Runners

When GitHub-hosted runners aren't enough (need specific hardware, VPC access, etc.):

```bash
# Install self-hosted runner on your server
# 1. Go to GitHub repo → Settings → Actions → Runners → New self-hosted runner
# 2. Follow the generated instructions:

mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
./config.sh --url https://github.com/org/repo --token YOUR_TOKEN
./run.sh
```

```yaml
# Use self-hosted runner in workflow
jobs:
  build:
    runs-on: self-hosted    # Or: [self-hosted, linux, x64]
```

### GitHub Actions Pros and Cons

| Pros | Cons |
|------|------|
| Zero infrastructure to manage | Vendor lock-in to GitHub |
| Native GitHub integration | Limited free minutes for private repos |
| Huge Actions marketplace | Debugging is harder (remote execution) |
| Matrix builds built-in | Secret management less flexible than Jenkins |
| Free for public repos | Expensive at scale |
| OIDC support for cloud auth | Some 3rd-party actions can be risky |

---

## 9. GitLab CI — Complete Guide

### What is GitLab CI?

GitLab CI is the **built-in CI/CD system in GitLab** (both gitlab.com and self-hosted GitLab). Defined by a `.gitlab-ci.yml` file at the root of the repository.

**Key Differentiator:** GitLab is an all-in-one DevOps platform (SCM + CI/CD + container registry + security scanning + monitoring) — you don't need to integrate multiple tools.

### GitLab CI Architecture

```
GITLAB SERVER
    │
    │ Reads .gitlab-ci.yml
    ▼
GITLAB CI/CD (Coordinator)
    │
    │ Distributes jobs to runners
    ├──────────────────────────────────┐
    │                                  │
    ▼                                  ▼
GITLAB RUNNER 1                  GITLAB RUNNER 2
(Shell executor)                 (Docker executor)
- Runs directly on host          - Each job gets a fresh Docker container
- Fast, but less isolated        - Isolated, reproducible
```

**Executors:**
- `shell` — runs on the host machine (fast, less isolated)
- `docker` — each job in a fresh container (recommended)
- `kubernetes` — each job as a Kubernetes pod (best for scale)
- `virtualbox` / `parallels` — full VM per job (most isolated)

### Complete `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml

# Default image for all jobs (overridable per job)
default:
  image: node:20-alpine
  
  # Cache between jobs in the same pipeline run
  cache:
    paths:
      - node_modules/
    
  # Always retry failed jobs once (handles transient failures)
  retry:
    max: 1
    when: runner_system_failure

# ─── VARIABLES ──────────────────────────────────────────────
variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  FF_USE_FASTZIP: "true"   # Speed up artifact compression

# ─── STAGES ─────────────────────────────────────────────────
# Jobs in the same stage run in parallel
# Stages run sequentially
stages:
  - validate
  - test
  - build
  - security
  - staging
  - production

# ─── TEMPLATES (reusable YAML anchors) ──────────────────────
.deploy_template: &deploy_definition
  image: bitnami/kubectl:latest
  before_script:
    - kubectl config use-context ${KUBE_CONTEXT}
  script:
    - kubectl set image deployment/myapp myapp=${IMAGE_TAG} -n ${NAMESPACE}
    - kubectl rollout status deployment/myapp -n ${NAMESPACE} --timeout=5m

# ─── STAGE: VALIDATE ────────────────────────────────────────
lint:
  stage: validate
  script:
    - npm ci
    - npm run lint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# ─── STAGE: TEST ────────────────────────────────────────────
unit-tests:
  stage: test
  script:
    - npm ci
    - npm test -- --coverage --ci
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'   # Regex to extract coverage %
  artifacts:
    when: always
    reports:
      junit: coverage/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 1 week

integration-tests:
  stage: test
  image: docker:24
  services:
    - docker:24-dind    # Docker-in-Docker
    - postgres:15       # Side-car database for tests
  variables:
    DATABASE_URL: "postgresql://postgres:password@postgres:5432/testdb"
    POSTGRES_PASSWORD: password
  script:
    - npm ci
    - npm run test:integration

# ─── STAGE: BUILD ────────────────────────────────────────────
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    # Also tag as latest for the branch
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  only:
    - main
    - develop

# ─── STAGE: SECURITY ─────────────────────────────────────────
container-scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE_TAG
  allow_failure: false
  needs:
    - build-image

# Built-in GitLab security scanning (Ultimate tier)
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

# ─── STAGE: STAGING ──────────────────────────────────────────
deploy-staging:
  <<: *deploy_definition    # Use YAML anchor template
  stage: staging
  variables:
    NAMESPACE: staging
    KUBE_CONTEXT: my-cluster/staging
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop-staging   # Clean up on branch deletion
  needs:
    - build-image
    - container-scan
  only:
    - main
    - develop

stop-staging:
  stage: staging
  script:
    - kubectl delete namespace staging-$CI_COMMIT_REF_SLUG || true
  environment:
    name: staging
    action: stop
  when: manual

smoke-test-staging:
  stage: staging
  needs:
    - deploy-staging
  script:
    - curl -f https://staging.example.com/health

# ─── STAGE: PRODUCTION ───────────────────────────────────────
deploy-production:
  <<: *deploy_definition
  stage: production
  variables:
    NAMESPACE: production
    KUBE_CONTEXT: my-cluster/production
  environment:
    name: production
    url: https://example.com
  when: manual        # Requires human to click "Play" button
  only:
    - main
  needs:
    - smoke-test-staging
```

### GitLab CI Predefined Variables

```yaml
$CI_COMMIT_SHA           # Full git commit SHA
$CI_COMMIT_SHORT_SHA     # 8-char short SHA
$CI_COMMIT_BRANCH        # Branch name
$CI_COMMIT_TAG           # Tag name (if a tag triggered the pipeline)
$CI_PIPELINE_ID          # Unique pipeline ID
$CI_JOB_ID               # Unique job ID
$CI_REGISTRY             # GitLab container registry URL
$CI_REGISTRY_IMAGE       # Registry image URL for this repo
$CI_REGISTRY_USER        # Auto-generated registry username
$CI_REGISTRY_PASSWORD    # Auto-generated registry password
$CI_MERGE_REQUEST_IID    # MR number (only in MR pipelines)
$GITLAB_USER_LOGIN       # Username who triggered pipeline
```

### GitLab CI Special Features

**Environments & Deployments:**
```yaml
# GitLab tracks every deployment in the Environments UI
environment:
  name: production
  url: https://example.com
  kubernetes:
    namespace: production
```

**DAG (Directed Acyclic Graph) — `needs`:**
```yaml
# Instead of waiting for an entire stage, wait only for specific jobs
# This runs deploy-staging as soon as build-image completes,
# even if other test jobs are still running
deploy-staging:
  needs:
    - job: build-image
      artifacts: true
```

**Merge Request Pipelines:**
```yaml
# Run a different pipeline on MRs vs main branch merges
unit-tests:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - when: never
```

### GitLab CI Pros and Cons

| Pros | Cons |
|------|------|
| All-in-one platform (SCM + CI + Registry) | Self-hosting is resource-heavy |
| Built-in container registry | Free tier has CI minute limits |
| Auto DevOps (zero-config pipelines) | More complex YAML than GitHub Actions |
| Built-in security scanning | Smaller Actions marketplace |
| Powerful DAG scheduling | |
| Excellent Kubernetes integration | |

---

## 10. Rollbacks — Strategies & Implementation

### Why Rollbacks Matter

A rollback is the ability to **quickly revert a deployment to a known-good state** when something goes wrong. Without rollbacks, a bad deploy means hours of emergency work while users are impacted.

**The golden rule:** You should be able to roll back a production deployment in under 2 minutes.

### Rollback Strategy 1: Git Revert + Redeploy

```bash
# Bad commit was abc123
# Create a new commit that reverses it
git revert abc123 --no-edit
git push origin main
# Pipeline triggers, redeploys the reverted code
```

**Pros:** Clean history, auditable  
**Cons:** Takes as long as a normal deployment (5–15 minutes)

---

### Rollback Strategy 2: Kubernetes Rollout Undo (Fast)

```bash
# See rollout history
kubectl rollout history deployment/myapp

# Undo to previous version (FAST — just updates the image ref)
kubectl rollout undo deployment/myapp

# Undo to a specific revision
kubectl rollout undo deployment/myapp --to-revision=3

# Monitor the rollback
kubectl rollout status deployment/myapp
```

```
How kubectl rollout undo works:
  
Current state: Deployment running image v3
kubectl rollout undo
  ↓
Kubernetes starts new pods with image v2
  ↓
As new pods become Ready, old v3 pods are terminated
  ↓
Rollback complete — entire process ~30 seconds
```

---

### Rollback Strategy 3: Blue-Green Deployment

```
BLUE-GREEN:

PRODUCTION TRAFFIC
       │
       ▼
  ┌─────────┐
  │  LOAD   │
  │BALANCER │
  └────┬────┘
       │
  ┌────┴────┐
  │         │
  ▼         ▼
BLUE       GREEN
(v2, live) (v3, new)
           
After testing GREEN:
  → Flip load balancer to GREEN
  → If GREEN fails: flip back to BLUE instantly (< 1 second)
  → No rebuilding needed — BLUE is still running
```

```bash
# Kubernetes blue-green via service selector

# Current production service points to "blue" pods
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'

# Deploy new "green" version
kubectl apply -f deployment-green.yaml
kubectl rollout status deployment/myapp-green

# Run tests against green (via staging URL or internal address)
# If tests pass: switch traffic
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# If something goes wrong: instant rollback
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

### Rollback Strategy 4: Canary Deployment

```
CANARY DEPLOYMENT:

ALL TRAFFIC
    │
    ▼
LOAD BALANCER
    │
    ├──── 90% ────► STABLE (v2) — 9 pods
    │
    └──── 10% ────► CANARY (v3) — 1 pod

Monitor error rates, latency, business metrics.
If metrics are good: gradually shift more traffic to canary.
If metrics degrade: route all traffic back to stable.
```

```yaml
# Kubernetes canary (simple weight-based via 2 deployments)
# Stable: 9 replicas
# Canary: 1 replica
# Service selects BOTH via shared label: app=myapp
# Traffic ratio ≈ 9:1 (depends on pod count, not native %)

# For true % routing, use Istio or NGINX Ingress:
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp-stable
          weight: 90
        - destination:
            host: myapp-canary
          weight: 10
```

---

### Rollback Decision Matrix

| Scenario | Best Rollback Strategy | Time to Rollback |
|----------|----------------------|-----------------|
| Bad code pushed, caught in <1 min | `kubectl rollout undo` | 30 seconds |
| Bad code in production for 30 min | Blue-green flip | < 1 second |
| Gradual degradation | Canary: reduce canary weight to 0 | < 1 minute |
| Bad database migration | DB restore + git revert | Minutes to hours |
| Wrong config pushed | Restore ConfigMap + rollout restart | 1–2 minutes |

### Database Rollbacks — The Hard Problem

Code rollbacks are easy. Database rollbacks are hard.

```
PROBLEM: You deploy v3 which adds a column "email_verified".
         v3 sets email_verified=true for new users.
         You roll back to v2.
         v2 doesn't know about email_verified.
         → No data loss, but v2 might crash if it queries that column.

SOLUTIONS:
1. Backward-compatible migrations (preferred):
   - Add columns as nullable (not NOT NULL)
   - Never drop columns until old code is gone
   - Use "expand-contract" pattern

2. Schema version in config: 
   - v2 code expects schema v2
   - v3 code expects schema v3
   - Keep v2-compatible schema until v2 is fully retired

3. Separate migration from code deployment:
   - Run migration first (additive only)
   - Deploy new code
   - If rollback: code rolls back, old schema is still compatible
```

---

## 11. Advanced Pipeline Patterns

### Parallel Execution

```yaml
# GitHub Actions — parallel jobs
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps: [...]

  test-integration:
    runs-on: ubuntu-latest
    steps: [...]

  lint:
    runs-on: ubuntu-latest
    steps: [...]

  # All three run simultaneously — saves time
  # Total time = max(unit, integration, lint) instead of sum
```

### Fan-out / Fan-in

```
FAN-OUT:                              FAN-IN:
Build ──► Test Unit  ─────────────────► Deploy
     ├──► Test Integration ──────────►
     ├──► Test E2E    ───────────────►
     └──► Security Scan ─────────────►

All test jobs run in parallel.
Deploy only runs after ALL pass.
```

### Dynamic Pipelines (GitLab)

```yaml
# Generate pipeline YAML dynamically based on changed files
generate-pipeline:
  stage: .pre
  script:
    - python scripts/generate_pipeline.py > generated-pipeline.yml
  artifacts:
    paths:
      - generated-pipeline.yml

trigger-generated:
  trigger:
    include:
      - artifact: generated-pipeline.yml
        job: generate-pipeline
```

### Ephemeral Review Environments (Per PR)

```yaml
# Deploy a temporary environment for every PR
deploy-review:
  stage: deploy
  script:
    - helm upgrade --install review-$CI_MERGE_REQUEST_IID ./chart \
        --set ingress.host=review-$CI_MERGE_REQUEST_IID.example.com \
        --namespace review-$CI_MERGE_REQUEST_IID \
        --create-namespace
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://review-$CI_MERGE_REQUEST_IID.example.com
    on_stop: stop-review
  only:
    - merge_requests

stop-review:
  script:
    - helm uninstall review-$CI_MERGE_REQUEST_IID -n review-$CI_MERGE_REQUEST_IID
    - kubectl delete namespace review-$CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
  when: manual
```

### Pipeline Caching Strategies

```yaml
# GitHub Actions — cache node_modules
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# The key includes a hash of package-lock.json
# Cache is invalidated when lockfile changes
# Saves ~60 seconds per job on Node.js projects
```

---

## 12. Security in CI/CD (DevSecOps Basics)

### The DevSecOps Shift

```
TRADITIONAL:
Dev → Build → Test → Security Team Reviews → Deploy
                     (security is at the end, bottleneck)

DEVSECOPS (Shift Left):
Dev → Security checks run AT COMMIT TIME → Build → Deploy
     (fast feedback, no bottleneck, problems found early)
```

### Security Checks in the Pipeline

```
STAGE 1 — PRE-COMMIT (developer's machine)
  Secret detection (git-secrets, pre-commit hooks)
  
STAGE 2 — CI/CD PIPELINE
  SAST    — Static Application Security Testing
            Scans source code for vulnerabilities
            Tools: Semgrep, Bandit, SonarQube, Checkmarx
            
  SCA     — Software Composition Analysis
            Finds CVEs in your dependencies
            Tools: Trivy, Snyk, npm audit, Safety
            
  DAST    — Dynamic Application Security Testing
            Attacks the running app to find vulnerabilities
            Tools: OWASP ZAP, Burp Suite
            
  Secret Scanning
            Finds hardcoded API keys, passwords in code
            Tools: GitLeaks, truffleHog, GitHub secret scanning
            
  Container Scanning
            Finds CVEs in Docker base images and installed packages
            Tools: Trivy, Clair, Grype
```

### Example: Secret Scanning in GitHub Actions

```yaml
- name: Detect secrets
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

### Managing Secrets in CI/CD

```
❌ WRONG WAYS:
- Hardcode in .yml files
- Store in git repo
- Pass as plain env vars in logs

✅ CORRECT WAYS:
Jenkins:   Credentials Manager
GitHub:    Repository/Organization Secrets
GitLab:    CI/CD Variables (masked + protected)
External:  HashiCorp Vault, AWS Secrets Manager

Example — GitHub Actions using Vault:
- name: Import secrets from Vault
  uses: hashicorp/vault-action@v2
  with:
    url: https://vault.example.com
    method: approle
    roleId: ${{ secrets.VAULT_ROLE_ID }}
    secretId: ${{ secrets.VAULT_SECRET_ID }}
    secrets: |
      secret/data/prod/database password | DB_PASSWORD ;
      secret/data/prod/api key | API_KEY
```

### OIDC — Keyless Authentication to Cloud

The modern way to authenticate CI/CD to cloud without storing long-lived credentials:

```yaml
# GitHub Actions → AWS (no stored AWS keys needed)
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1
    # GitHub generates a short-lived JWT, AWS verifies it
    # No ACCESS_KEY_ID or SECRET_ACCESS_KEY stored anywhere
```

---

## 13. Interview Mastery

---

### Beginner Interview Questions

---

**Q1: What is CI/CD?**

**Perfect Answer:**

"CI/CD stands for Continuous Integration and Continuous Delivery (or Deployment). 

Continuous Integration means every developer's code changes are automatically built and tested when they push to the repository. This catches bugs immediately instead of letting them compound.

Continuous Delivery extends that by automatically packaging and deploying the application to a staging environment after tests pass. It means software is always in a releasable state.

Continuous Deployment goes one step further — it automatically deploys to production without any human approval, as long as all tests pass.

The business value is clear: companies that adopt CI/CD deploy 200 times more frequently with 24x faster recovery from failures, according to DORA research."

**How to answer confidently:** Draw the code-to-production flow on a whiteboard. Distinguish the three terms. Always include business impact — interviewers want to know you understand the "why".

---

**Q2: What is the difference between Continuous Delivery and Continuous Deployment?**

**Perfect Answer:**

"Both are extensions of CI, but they differ in the final step:

Continuous Delivery: The pipeline automatically deploys to staging, runs tests, and then STOPS — waiting for a human to approve the production deployment. The code is always *ready* to go to production.

Continuous Deployment: There is no human approval gate. If all tests pass, code automatically goes to production. This requires extremely high test confidence.

Most companies practice Continuous Delivery because even mature teams want a human reviewing what goes to production. Companies like Netflix and Amazon practice Continuous Deployment because they have extensive automated test suites and monitor deployments with automated rollback."

---

**Q3: What is a pipeline stage?**

**Perfect Answer:**

"A pipeline stage is a logical grouping of steps that represents one phase of the delivery process. Common stages are: Build, Test, Package, Deploy-Staging, Acceptance-Test, Deploy-Production.

Stages run sequentially by default — the next stage only starts if the previous one succeeded. Within a stage, jobs can run in parallel.

The benefit of stages is fail-fast behavior: if the build fails, we don't waste time running tests. If unit tests fail, we don't waste time building a Docker image. This gives fast feedback with minimal cost."

---

### Intermediate Interview Questions

---

**Q4: How do you handle secrets in a CI/CD pipeline?**

**Perfect Answer:**

"Secrets should never be hardcoded in pipeline YAML files or stored in version control — that's a critical security vulnerability.

The right approaches are:

1. **CI/CD native secrets** — GitHub Secrets, GitLab CI/CD Variables, Jenkins Credentials Manager. These are encrypted, never exposed in logs, and injected as environment variables at runtime.

2. **External secrets managers** — HashiCorp Vault, AWS Secrets Manager, Azure Key Vault. The pipeline authenticates to the secrets manager and fetches secrets at runtime. This centralizes secret rotation.

3. **OIDC (keyless auth)** — For cloud deployments, use OpenID Connect so the CI/CD platform generates a short-lived token that your cloud trusts. No long-lived credentials stored at all.

In practice, I prefer OIDC for cloud auth (no secrets to rotate) and CI/CD native secrets for everything else, with HashiCorp Vault for large teams that need audit trails and dynamic secrets."

**Hidden interviewer expectation:** They want to hear that you understand the risk of secrets in logs and have thought about rotation and audit trails.

---

**Q5: What is an artifact in CI/CD and why should you build once and promote?**

**Perfect Answer:**

"An artifact is the deployable unit produced by a build — typically a Docker image, JAR, binary, or zip file. It's stored in an artifact registry and versioned (usually with the git SHA).

The principle of 'build once, promote' means you build the artifact once in CI, then deploy that exact same artifact to staging, then promote it to production. You never rebuild for each environment.

Why? Because every build, even from the same code, can produce slightly different results (different dependency versions resolved, different build machine state, different timestamps). If you rebuild for production, you've introduced a gap between what you tested and what you deployed.

With build-once-promote, you have a guarantee: the bytes running in production are identical to what passed all your tests."

---

**Q6: Explain a Jenkins Declarative Pipeline. What are the main sections?**

**Perfect Answer:**

"A Declarative Pipeline is a structured way to define a Jenkins pipeline as code in a `Jenkinsfile`. Its main sections are:

- **`pipeline {}`** — the root block
- **`agent`** — where to run (any agent, specific label, Docker image)
- **`environment`** — global environment variables
- **`stages`** — contains the ordered list of stages
- **`stage('Name') {}`** — a named stage containing steps
- **`steps {}`** — the actual commands (sh, bat, script)
- **`post {}`** — actions to run after the pipeline (success, failure, always)

The power is in `post`: on failure you can send Slack notifications and trigger rollbacks; on success you notify the team; `always` cleans up the workspace.

I prefer Declarative over Scripted because it's easier to read, has built-in syntax validation, and the `when` directive makes conditional stages clean."

---

### Advanced Interview Questions

---

**Q7: How would you design a CI/CD pipeline for a microservices architecture with 50 services?**

**Perfect Answer:**

"With 50 services, the main challenges are:
1. Not running every pipeline for every commit
2. Fast enough to give feedback quickly
3. Ensuring cross-service compatibility

My approach:

**Change detection:** Only trigger pipelines for services whose code actually changed. In a monorepo, use tools like `nx affected` or path filters in GitHub Actions to detect which services changed.

**Shared pipeline templates:** Define a reusable pipeline (GitHub Actions `workflow_call` or GitLab CI includes) that all services inherit. Services override only what's different.

**Dependency graph:** If Service A depends on Service B, when Service B changes, automatically trigger Service A's integration tests too.

**Service mesh + contract testing:** Use Pact or Spring Cloud Contract to test service contracts in isolation. This catches compatibility issues without deploying everything.

**Environment strategy:** Use ephemeral per-PR environments for each service. After merge, deploy to a shared integration environment, then promote to production.

**Artifact management:** Each service builds its own Docker image, tagged with git SHA. A release manifest records which version of each service is deployed per environment."

---

**Q8: A deployment to production is causing elevated error rates. Walk me through how you respond.**

**Perfect Answer (incident response format):**

"This is an incident — I follow a structured response:

**1. Detect (< 2 minutes):**
Monitoring alert fires (Prometheus → PagerDuty). I check the dashboard immediately: what's the error rate, which endpoints are affected, what changed recently.

**2. Communicate (immediate):**
Post in the incident Slack channel: 'Investigating elevated error rate in production, started ~14:23 UTC, likely related to the 14:15 deploy.'

**3. Decide: Roll back or fix forward? (< 5 minutes):**
If the cause is known and the fix is a config change → fix forward.
If the cause is unclear or complex → roll back immediately.
I always prefer rolling back quickly over investigating while users are impacted.

**4. Roll back (< 2 minutes):**
`kubectl rollout undo deployment/myapp -n production`
`kubectl rollout status deployment/myapp`
Check that error rate returns to baseline.

**5. Communicate recovery:**
'14:31 UTC — Rollback complete. Error rate back to normal. Root cause investigation ongoing.'

**6. Post-mortem:**
After the incident, write a blameless post-mortem: timeline, root cause, contributing factors, action items. The goal is to prevent recurrence, not assign blame.

**Key principle I follow:** Rollback fast, restore service first, investigate after. A 5-minute investigation while users are impacted is almost never worth it."

---

### Scenario-Based Questions

---

**Q9: A developer says "CI/CD is slowing us down — our pipelines take 45 minutes." How do you address this?**

**Perfect Answer:**

"Forty-five minutes is too long — developers stop waiting for CI and start merging anyway, defeating the purpose. I'd approach this systematically:

**Measure first:** Profile where time is spent. Usually it's one of: slow dependency installation, no caching, large Docker builds, slow test suite, or over-sequential stages.

**Quick wins (first week):**
- Add dependency caching (`npm ci` with cached `node_modules`)
- Parallelize jobs that can run simultaneously
- Use Docker layer caching for image builds

**Medium-term:**
- Run only unit tests on every commit; integration tests on merge to main
- Use test sharding to split test suite across parallel runners
- Optimize Dockerfile layer order (rarely-changing layers first)

**Architecture changes:**
- In a monorepo: only run pipelines for changed services
- Use incremental builds (Gradle, Bazel, Nx)

**Target:** < 10 minutes for the critical path (build + unit tests). Longer tests can run in parallel or on a separate schedule.

I've seen 45-minute pipelines reduced to 8 minutes by adding caching and parallelization alone — without changing any tests."

---

**Q10: How do you ensure zero-downtime deployments?**

**Perfect Answer:**

"Zero-downtime deployment means users never see an error or service interruption during a deploy. There are several mechanisms:

**1. Kubernetes Rolling Update (default):**
Kubernetes gradually replaces old pods with new ones. At any point, both old and new pods are running. Traffic is only sent to Ready pods.
```
maxSurge: 1        # Allow 1 extra pod during rollout
maxUnavailable: 0  # Never take a pod down until the new one is ready
```

**2. Readiness probes:**
The new pod only receives traffic after Kubernetes confirms it's ready:
```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```
Without this, Kubernetes can send traffic to a pod that's starting up and not ready.

**3. Graceful shutdown:**
When Kubernetes terminates an old pod, it sends SIGTERM. The app should:
- Stop accepting new requests
- Finish in-flight requests
- Close DB connections
- Exit cleanly

**4. Database compatibility:**
New code must be backward-compatible with the current schema. Use expand-contract migrations: add the new column first, deploy the new code, then remove the old column later.

**5. Blue-green for critical deployments:**
For very high-risk changes, use blue-green so rollback is a load balancer flip (< 1 second), not a pod rotation (30 seconds)."

---

**Q11: FAANG-style: Design the CI/CD system for a service that handles 1 million deploys per day (like Amazon's deployment system).**

**Perfect Answer:**

"At that scale, the CI/CD system is itself a critical production system. Key design decisions:

**Decentralized pipelines:** Each service team owns their pipeline definition. No centralized team is a bottleneck. Common libraries (security scans, standard deployment templates) are shared via internal registries.

**Asynchronous orchestration:** Pipelines are event-driven. A push event publishes to a queue; pipeline workers consume and execute. This handles burst traffic without blocking.

**Incremental, modular testing:** Full test suites don't run on every commit. Fast unit tests run first. Integration tests run only for changed components. E2E tests run on a schedule or before major releases.

**Progressive delivery at scale:** Every deploy is a canary by default. 1% → 10% → 50% → 100% traffic shift, automated by monitoring signals. The system watches error rate, latency, and business metrics. If any metric degrades, it automatically halts and rolls back.

**Deployment rings:** Deploy to a small internal region first, monitor for 5 minutes, then expand to progressively larger rings (10%, 25%, 100% of regions).

**Centralized observability:** Every deployment emits events to a central system. Any engineer can answer: what version is running where? When was it deployed? What's the deploy health?

**Automated rollback:** The system, not humans, triggers rollbacks. Humans are too slow for 1M deploys/day.

This is roughly how Amazon's Apollo deployment system and Netflix's Spinnaker work — progressive delivery with automated safety gates."

---

### Production Debugging Questions

---

**Q12: Your pipeline is intermittently failing on the Docker build step with "no space left on device". How do you debug and fix this?**

**Perfect Answer:**

"'No space left on device' on the Docker build step is a classic CI/CD infrastructure issue.

**Diagnose:**
```bash
df -h           # Check disk usage on the runner
docker system df  # Check Docker's disk usage
docker images   # Look for accumulated old images
docker ps -a    # Look for stopped containers not cleaned up
```

**Root cause:** CI runners accumulate Docker images and layers over time. If cleanup doesn't happen between jobs, the disk fills up.

**Immediate fix:**
```bash
docker system prune -f           # Remove stopped containers, dangling images
docker image prune -a --filter "until=24h" -f   # Remove images older than 24h
```

**Permanent fixes:**
1. Add cleanup step at end of every pipeline:
```yaml
post:
  always:
    sh 'docker system prune -f'
```

2. Use `--no-cache` builds occasionally to start fresh.

3. Provision larger disk for runners or use ephemeral runners (each job gets a fresh machine — GitHub-hosted runners do this by default).

4. Use multi-stage builds to reduce final image size, which also reduces layer cache size.

5. Monitor disk usage and alert before it hits 100%."

---

### Key Terms to Drop in Interviews

When discussing CI/CD, naturally use these terms to signal expertise:

| Term | When to Use |
|------|------------|
| **DORA metrics** | When discussing CI/CD business value |
| **Shift-left security** | When discussing security in pipelines |
| **Immutable artifacts** | When discussing build-once-promote |
| **Idempotent pipelines** | Pipelines that can safely re-run |
| **Pipeline as code** | Pipelines defined in version-controlled files |
| **Progressive delivery** | Canary, blue-green, feature flags |
| **Ephemeral environments** | Temporary per-PR deploy environments |
| **OIDC / keyless auth** | Cloud auth without stored credentials |
| **Fail-fast** | Stop pipeline early on first failure |
| **Mean Time to Recovery** | How fast you can fix a broken production |

---

[⬇️ Download This File](#)

---

*Phase 4 Complete. Ready for Phase 5 — Docker.*
