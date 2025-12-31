# GitLab CI/CD - Monorepo Microservices Pipeline

Personal notes on building CI/CD pipelines for **monorepo microservice architectures** using GitLab. This project demonstrates how to efficiently manage multiple services in a single repository with optimized build and deployment pipelines.

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Key Concepts](#key-concepts)
  - [Monorepo Strategy](#monorepo-strategy)
  - [Change Detection with `only:changes`](#change-detection-with-onlychanges)
  - [Job Templates with `extends`](#job-templates-with-extends)
  - [Docker Networking for Microservices](#docker-networking-for-microservices)
- [Pipeline Architecture](#pipeline-architecture)
- [Configuration Breakdown](#configuration-breakdown)
- [How It Works](#how-it-works)
- [Best Practices](#best-practices)
- [Quick Reference](#quick-reference)

---

## Overview

This repository contains **3 microservices** managed in a single monorepo:

| Service | Port | Description |
|---------|------|-------------|
| `frontend` | 3000 | User-facing web application |
| `products` | 3001 | Products API service |
| `shopping-cart` | 3002 | Shopping cart API service |

**Key Challenge:** In a monorepo, we don't want to rebuild and redeploy ALL services when only ONE service changes.

**Solution:** Use GitLab's `only:changes` feature to detect which service has changed and only trigger jobs for that specific service.

---

## Project Structure

```
monorepo-microservice-cicd/
├── .gitlab-ci.yml          # Main pipeline configuration
├── docker-compose.yaml     # Deployment configuration
├── frontend/               # Frontend microservice
│   ├── Dockerfile
│   ├── server.js
│   ├── index.html
│   └── package.json
├── products/               # Products microservice
│   ├── Dockerfile
│   ├── server.js
│   └── package.json
└── shopping-cart/          # Shopping cart microservice
    ├── Dockerfile
    ├── server.js
    └── package.json
```

---

## Key Concepts

### Monorepo Strategy

A **monorepo** (monolithic repository) stores multiple projects/services in a single repository.

**Advantages:**
- Unified code review and versioning
- Easier cross-service refactoring
- Simplified dependency management
- Single CI/CD configuration

**Challenges:**
- Longer build times if not optimized
- Risk of triggering unnecessary deployments
- Complex dependency graphs

**Our Solution:**
```
┌─────────────────────────────────────────────────────────────┐
│                     Monorepo                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  frontend/  │  │  products/  │  │  shopping-cart/     │ │
│  │  ─────────  │  │  ─────────  │  │  ─────────────────  │ │
│  │  Changes?   │  │  Changes?   │  │  Changes?           │ │
│  │     ↓       │  │     ↓       │  │     ↓               │ │
│  │  Build ✓    │  │  Skip ✗    │  │  Skip ✗             │ │
│  │  Deploy ✓   │  │            │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

### Change Detection with `only:changes`

The `only:changes` keyword tells GitLab to **only run a job if specific files have changed**.

```yaml
build_frontend:
  extends: .build
  variables:
    MICRO_SERVICE: frontend
    SERVICE_VERSION: "1.3"
  only:
    changes:
      - "frontend/**/*"    # Only run if ANY file in frontend/ changes
```

**How it works:**

| Scenario | Files Changed | Jobs Triggered |
|----------|---------------|----------------|
| Frontend update | `frontend/server.js` | `build_frontend`, `deploy_frontend` |
| Products update | `products/Dockerfile` | `build_products`, `deploy_products` |
| Multiple services | `frontend/*`, `products/*` | All frontend + products jobs |
| Root file change | `docker-compose.yaml` | No service jobs (no match) |

**Pattern Syntax:**
```yaml
only:
  changes:
    - "frontend/**/*"      # All files in frontend/ and subdirectories
    - "frontend/*"         # Only direct children of frontend/
    - "*.md"               # All markdown files in root
    - "**/*.js"            # All JS files anywhere in repo
```

---

### Job Templates with `extends`

Use **hidden jobs** (prefixed with `.`) as templates to avoid code duplication.

```yaml
# Template (hidden job - not executed directly)
.build:
  stage: build
  tags:
    - ec2
    - remote
    - shell
  variables:
    MICRO_SERVICE: ""       # To be overridden
    SERVICE_VERSION: ""     # To be overridden
  before_script:
    - cd $MICRO_SERVICE
    - export IMAGE_NAME=$CI_REGISTRY_IMAGE/microservice/$MICRO_SERVICE
    - export IMAGE_TAG=$SERVICE_VERSION
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG

# Actual jobs extending the template
build_frontend:
  extends: .build           # Inherit everything from .build
  variables:
    MICRO_SERVICE: frontend
    SERVICE_VERSION: "1.3"
  only:
    changes:
      - "frontend/**/*"

build_products:
  extends: .build
  variables:
    MICRO_SERVICE: products
    SERVICE_VERSION: "1.8"
  only:
    changes:
      - "products/**/*"
```

**Benefits:**
- DRY (Don't Repeat Yourself) principle
- Easier maintenance - update template once, all jobs inherit changes
- Consistent job configuration across services

---

### Docker Networking for Microservices

Microservices need to communicate with each other. We use a **shared Docker network**.

```yaml
# In the deploy script
docker network create micro_service || true   # Create if not exists
```

**docker-compose.yaml:**
```yaml
version: "3.9"
services:
  app:
    image: ${DC_IMAGE_NAME}:${DC_IMAGE_TAG}
    ports:
      - ${DC_APP_PORT}:${DC_APP_PORT}
    networks:
      - micro_service         # Join the shared network

networks:
  micro_service:
    external:
      name: micro_service     # Use existing external network
```

**Network Architecture:**
```
┌──────────────────────────────────────────────────────────┐
│                Docker Network: micro_service             │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │  frontend   │  │  products   │  │  shopping-cart  │  │
│  │  :3000      │──│  :3001      │──│  :3002          │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│         │                │                  │           │
└─────────┼────────────────┼──────────────────┼───────────┘
          │                │                  │
    ┌─────▼────────────────▼──────────────────▼─────┐
    │              Host Machine                     │
    │        Ports: 3000, 3001, 3002 exposed        │
    └───────────────────────────────────────────────┘
```

**Why `|| true`?**
```bash
docker network create micro_service || true
```
- First deployment: Creates the network
- Subsequent deployments: Command fails (network exists), but `|| true` prevents job failure

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         WORKFLOW RULES                              │
│   Only runs on: main branch OR merge requests                       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          BUILD STAGE                                │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐   │
│  │ build_frontend  │ │ build_products  │ │ build_shopping_cart │   │
│  │ only: frontend/ │ │ only: products/ │ │ only: shopping-cart/│   │
│  │       **/*      │ │       **/*      │ │       **/*          │   │
│  └────────┬────────┘ └────────┬────────┘ └──────────┬──────────┘   │
│           │                   │                     │              │
└───────────┼───────────────────┼─────────────────────┼──────────────┘
            │                   │                     │
            ▼                   ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DEPLOY STAGE                                │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐   │
│  │ deploy_frontend │ │ deploy_products │ │ deploy_shopping-cart│   │
│  │ only: frontend/ │ │ only: products/ │ │ only: shopping-cart/│   │
│  │       **/*      │ │       **/*      │ │       **/*          │   │
│  │ port: 3000      │ │ port: 3001      │ │ port: 3002          │   │
│  └─────────────────┘ └─────────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuration Breakdown

### Workflow Rules

```yaml
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH != "main" && $CI_PIPELINE_SOURCE != "merge_request_event"
      when: never       # Don't run pipeline
    - when: always      # Otherwise, run
```

**Translation:**
- Skip pipeline if: NOT on `main` AND NOT a merge request
- Run pipeline if: On `main` OR is a merge request

### Global Variables

```yaml
variables:
  DEPLOYMENT_SERVER_HOST: "3.68.72.56"
  APP_ENDPOINT: ec2-3-68-72-56.eu-central-1.compute.amazonaws.com
```

Available to ALL jobs in the pipeline.

### Build Template

```yaml
.build:
  stage: build
  tags:
    - ec2
    - remote
    - shell
  variables:
    MICRO_SERVICE: ""
    SERVICE_VERSION: ""
  before_script:
    - cd $MICRO_SERVICE
    - export IMAGE_NAME=$CI_REGISTRY_IMAGE/microservice/$MICRO_SERVICE
    - export IMAGE_TAG=$SERVICE_VERSION
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG
```

**Image Naming Convention:**
```
registry.gitlab.com/<namespace>/<project>/microservice/<service-name>:<version>
```

Example: `registry.gitlab.com/myuser/myproject/microservice/frontend:1.3`

### Deploy Template

```yaml
.deploy:
  stage: deploy
  tags:
    - ec2
    - remote
    - shell
  variables:
    MICRO_SERVICE: ""
    SERVICE_VERSION: ""
    APP_PORT: ""
  before_script:
    - chmod 400 $SSH_PRIVATE_KEY
    - export IMAGE_NAME=$CI_REGISTRY_IMAGE/microservice/$MICRO_SERVICE
    - export IMAGE_TAG=$SERVICE_VERSION
  script:
    # Copy docker-compose to server
    - scp -o StrictHostKeyChecking=no -i $SSH_PRIVATE_KEY ./docker-compose.yaml ubuntu@$DEPLOYMENT_SERVER_HOST:/home/ubuntu/

    # SSH into server and deploy
    - ssh -o StrictHostKeyChecking=no -i $SSH_PRIVATE_KEY ubuntu@$DEPLOYMENT_SERVER_HOST "
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY &&
        export COMPOSE_PROJECT_NAME=$MICRO_SERVICE &&
        export DC_IMAGE_NAME=$IMAGE_NAME &&
        export DC_IMAGE_TAG=$IMAGE_TAG &&
        export DC_APP_PORT=$APP_PORT &&
        docker network create micro_service || true &&
        docker-compose down &&
        docker-compose up -d"
  environment:
    name: development
    url: $APP_ENDPOINT
```

**Key Variables:**
| Variable | Purpose |
|----------|---------|
| `COMPOSE_PROJECT_NAME` | Unique project name per service (prevents container name conflicts) |
| `DC_IMAGE_NAME` | Image name passed to docker-compose |
| `DC_IMAGE_TAG` | Image tag passed to docker-compose |
| `DC_APP_PORT` | Port mapping passed to docker-compose |

---

## How It Works

### Scenario 1: Update Frontend Only

1. Developer modifies `frontend/server.js`
2. GitLab detects changes in `frontend/**/*`
3. Only `build_frontend` and `deploy_frontend` jobs run
4. Other services remain untouched

```
Pipeline:
  ├── build_frontend ✓
  ├── build_products (skipped - no changes)
  ├── build_shopping_cart (skipped - no changes)
  ├── deploy_frontend ✓
  ├── deploy_products (skipped - no changes)
  └── deploy_shopping-cart (skipped - no changes)
```

### Scenario 2: Update Multiple Services

1. Developer modifies `frontend/index.html` and `products/server.js`
2. GitLab detects changes in both directories
3. Jobs for both services run in parallel

```
Pipeline:
  ├── build_frontend ✓
  ├── build_products ✓
  ├── build_shopping_cart (skipped)
  ├── deploy_frontend ✓
  ├── deploy_products ✓
  └── deploy_shopping-cart (skipped)
```

### Scenario 3: New Merge Request

1. Developer creates MR from feature branch
2. Pipeline runs (due to `merge_request_event`)
3. Only changed services are built/deployed

---

## Best Practices

### 1. Keep Services Independent

Each service should have its own:
- Dockerfile
- Package dependencies
- Version number

```
frontend/
├── Dockerfile      # Service-specific build
├── package.json    # Service-specific deps
└── server.js       # Service code
```

### 2. Use Semantic Versioning

Update `SERVICE_VERSION` when making changes:

```yaml
build_frontend:
  variables:
    SERVICE_VERSION: "1.3"    # Update on changes
```

Version format: `MAJOR.MINOR` or `MAJOR.MINOR.PATCH`

### 3. Unique Project Names per Service

Using `COMPOSE_PROJECT_NAME` prevents container conflicts:

```bash
export COMPOSE_PROJECT_NAME=$MICRO_SERVICE
```

This creates unique containers:
- `frontend_app_1`
- `products_app_1`
- `shopping-cart_app_1`

### 4. Shared Network for Inter-Service Communication

All services join the same Docker network:

```yaml
networks:
  micro_service:
    external:
      name: micro_service
```

Services can communicate via service name: `http://products:3001`

### 5. Handle Network Creation Gracefully

```bash
docker network create micro_service || true
```

- First run: Creates network
- Subsequent runs: Fails silently, continues

### 6. Use Hidden Jobs for Templates

Prefix with `.` to prevent direct execution:

```yaml
.build:       # Template - won't run
  script: ...

build_frontend:   # Actual job - extends template
  extends: .build
```

---

## Quick Reference

### Keywords Used

| Keyword | Purpose | Example |
|---------|---------|---------|
| `workflow:rules` | Control pipeline execution | Skip non-main branches |
| `stages` | Define job execution order | build → deploy |
| `extends` | Inherit from template job | `extends: .build` |
| `variables` | Define reusable values | `MICRO_SERVICE: frontend` |
| `only:changes` | Run job only if files changed | `- "frontend/**/*"` |
| `before_script` | Commands before main script | `cd $MICRO_SERVICE` |
| `script` | Main commands to execute | `docker build ...` |
| `tags` | Select specific runners | `ec2, shell, remote` |
| `environment` | Deployment environment info | `name: development` |

### Predefined Variables Used

| Variable | Description |
|----------|-------------|
| `$CI_REGISTRY` | GitLab Container Registry URL |
| `$CI_REGISTRY_USER` | Registry username |
| `$CI_REGISTRY_PASSWORD` | Registry password |
| `$CI_REGISTRY_IMAGE` | Registry image path for project |
| `$CI_COMMIT_BRANCH` | Branch name |
| `$CI_PIPELINE_SOURCE` | What triggered the pipeline |

### File Variables (Stored in GitLab CI/CD Settings)

| Variable | Type | Description |
|----------|------|-------------|
| `SSH_PRIVATE_KEY` | File | Private key for SSH to server |

---

## Pipeline Visualization

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MONOREPO PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Commit to main                                                         │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────┐                                                    │
│  │ Detect Changes  │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│     ┌─────┴─────┬────────────────┐                                     │
│     ▼           ▼                ▼                                      │
│ frontend/   products/     shopping-cart/                                │
│ changed?    changed?      changed?                                      │
│     │           │                │                                      │
│     ▼           ▼                ▼                                      │
│ ┌───────┐   ┌───────┐      ┌───────────┐                               │
│ │ Build │   │ Build │      │   Build   │                               │
│ │ Image │   │ Image │      │   Image   │                               │
│ └───┬───┘   └───┬───┘      └─────┬─────┘                               │
│     │           │                │                                      │
│     ▼           ▼                ▼                                      │
│ ┌───────┐   ┌───────┐      ┌───────────┐                               │
│ │Deploy │   │Deploy │      │  Deploy   │                               │
│ │:3000  │   │:3001  │      │  :3002    │                               │
│ └───────┘   └───────┘      └───────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Resources

- [GitLab CI/CD `only:changes`](https://docs.gitlab.com/ee/ci/yaml/#onlychangesexceptchanges)
- [Job templates with `extends`](https://docs.gitlab.com/ee/ci/yaml/#extends)
- [Workflow rules](https://docs.gitlab.com/ee/ci/yaml/#workflow)
- [Docker Networking](https://docs.docker.com/network/)
- [Microservices with Docker Compose](https://docs.docker.com/compose/)

---

## Summary

| Concept | What We Learned |
|---------|-----------------|
| `only:changes` | Build/deploy only services that changed |
| `extends` | DRY templates for similar jobs |
| Hidden jobs (`.name`) | Template definitions that don't run directly |
| Docker networking | Shared network for microservice communication |
| `COMPOSE_PROJECT_NAME` | Unique naming to prevent container conflicts |
| Workflow rules | Control when entire pipeline runs |
