# GitLab CI/CD - Personal Notes

My personal notes and hands-on examples while learning GitLab CI/CD from a course. This repository contains practical pipeline configurations progressing from basic jobs to production-ready multi-stage deployments.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Pipeline Examples](#pipeline-examples)
- [GitLab Runner](#gitlab-runner)
- [Advanced Topics](#advanced-topics)
- [Quick Reference](#quick-reference)

---

## Core Concepts

### 1. Jobs

Jobs are the fundamental building blocks of a `.gitlab-ci.yml` file.

**Rules:**
- Must contain at least the `script` clause
- Cannot use reserved keywords as job names: `image`, `services`, `stages`, `types`, `before_script`, `after_script`, `variables`, `cache`, `include`, `true`, `false`, `nil`

```yaml
# Basic job structure
run_tests:
  before_script:
    - echo "Preparing..."      # Runs before main script
  script:
    - echo "Running tests..."  # Main commands (required)
  after_script:
    - echo "Cleanup..."        # Runs after script (even on failure)
```

**File:** [1-jobs.gitlab-ci.yml](pipeline-code/1-jobs.gitlab-ci.yml)

---

### 2. Stages

Stages group jobs and define execution order.

```yaml
stages:
  - test      # All test jobs run first (in parallel)
  - build     # Build jobs run after ALL test jobs pass
  - deploy    # Deploy jobs run after ALL build jobs pass
```

**Key Points:**
- Jobs in the same stage run in parallel
- Pipeline proceeds to next stage only if ALL jobs in current stage succeed
- If any job fails, the pipeline stops

```yaml
stages:
  - test
  - build
  - deploy

run_unit_tests:
  stage: test
  script:
    - echo "Running unit tests..."

run_lint_tests:
  stage: test          # Runs parallel with run_unit_tests
  script:
    - echo "Running lint tests..."

build_image:
  stage: build         # Waits for ALL test jobs to pass
  script:
    - echo "Building..."
```

**File:** [2-stages.gitlab-ci.yml](pipeline-code/2-stages.gitlab-ci.yml)

---

### 3. Job Dependencies with `needs`

Create dependencies between jobs (DAG - Directed Acyclic Graph).

```yaml
push_image:
  stage: build
  needs:
    - build_image    # Waits specifically for build_image to complete
  script:
    - echo "Pushing..."
```

**Use Case:** `push_image` depends on `build_image` completing first, even within the same stage.

**File:** [3-needs.gitlab-ci.yml](pipeline-code/3-needs.gitlab-ci.yml)

---

### 4. Script Commands

Execute OS commands (Linux/shell commands).

```yaml
run_unit_tests:
  before_script:
    - pwd              # Print working directory
    - ls               # List files
    - mkdir test-data  # Create directory
  script:
    - echo "Running tests..."
  after_script:
    - rm -r test-data  # Cleanup
```

**File:** [4-script.gitlab-ci.yml](pipeline-code/4-script.gitlab-ci.yml)

---

### 5. Running Shell Scripts

Execute external bash scripts within jobs.

```yaml
run_unit_tests:
  before_script:
    - chmod +x ./prepare-tests.sh  # Make script executable
    - ./prepare-tests.sh           # Run the script
  script:
    - echo "Running tests..."
```

**File:** [5-bash.gitlab-ci.yml](pipeline-code/5-bash.gitlab-ci.yml)

---

### 6. Conditional Execution with `only` / `except`

Control when jobs run based on branches, tags, or other conditions.

```yaml
run_unit_tests:
  only:
    - main         # Only runs on 'main' branch
  script:
    - echo "Running tests..."

deploy_image:
  except:
    - develop      # Runs on all branches EXCEPT 'develop'
  script:
    - echo "Deploying..."
```

**File:** [6-only.gitlab-ci.yml](pipeline-code/6-only.gitlab-ci.yml)

---

### 7. Workflow Rules

Global pipeline control - determines if the entire pipeline should run.

```yaml
workflow:
  rules:
    # Don't run pipeline for non-main branches (unless it's a merge request)
    - if: $CI_COMMIT_BRANCH != "main" && $CI_PIPELINE_SOURCE != "merge_request_event"
      when: never
    - when: always   # Otherwise, run the pipeline
```

**Key Points:**
- `workflow:rules` controls the entire pipeline
- `rules` can also be used at job level (replacement for `only`/`except`)

**File:** [7-workflowrules.gitlab-ci.yml](pipeline-code/7-workflowrules.gitlab-ci.yml)

---

### 8. Variables

Define reusable values at global or job level.

```yaml
# Global variables - available to ALL jobs
variables:
  image_repository: docker.io/my-docker-id/myapp
  image_tag: v1.0

build_image:
  script:
    - echo "Building $image_repository:$image_tag"
```

**Variable Scopes:**
| Scope | Availability |
|-------|--------------|
| Global (top-level) | All jobs |
| Job-level | Only that specific job |
| Predefined | Built-in GitLab variables (e.g., `$CI_COMMIT_BRANCH`) |

**Common Predefined Variables:**
- `$CI_COMMIT_BRANCH` - Current branch name
- `$CI_PIPELINE_SOURCE` - What triggered the pipeline
- `$CI_REGISTRY` - Container registry URL
- `$CI_REGISTRY_USER` - Registry username
- `$CI_REGISTRY_PASSWORD` - Registry password
- `$CI_REGISTRY_IMAGE` - Registry image path

**File:** [8-variables.gitlab-ci.yml](pipeline-code/8-variables.gitlab-ci.yml)

---

## GitLab Runner

### Runner Types & Scope

| Scope | Description |
|-------|-------------|
| Instance runners | Available to all projects in GitLab instance |
| Group runners | Available to all projects in a group |
| Project runners | Associated with specific project(s) |

### Executor Types

| Executor | Description | Use Case |
|----------|-------------|----------|
| **Shell** | Commands run on the server's OS shell | Simple tasks, requires manual tool installation |
| **Docker** | Commands run inside containers | Isolated, reproducible environments |
| **Kubernetes** | Creates pods for each job | Auto-scaling, HA setups |
| **Docker Machine** | Auto-scaling Docker hosts | GitLab-hosted runners use this |

### Using Tags to Select Runners

```yaml
run_unit_tests:
  tags:
    - ec2       # Run on runner tagged 'ec2'
    - docker    # AND tagged 'docker'
    - remote    # AND tagged 'remote'
  image: node:23-alpine3.19
  script:
    - npm test
```

### Execution Flow

```
┌─────────────┐     ┌────────────┐     ┌───────────────┐
│   GitLab    │────▶│   Runner   │────▶│   Executor    │
│  Instance   │◀────│            │◀────│ (Docker/Shell)│
└─────────────┘     └────────────┘     └───────────────┘
     │                    │                    │
     │  1. Request jobs   │  2. Compile job    │
     │  5. Update status  │  3. Clone/download │
     │                    │  4. Execute & return│
```

**File:** [9-runners.gitlab-ci.yml](pipeline-code/9-runners.gitlab-ci.yml)

---

## Pipeline Examples

### Unit Testing with Artifacts

```yaml
run_unit_tests:
  image: node:23-alpine3.19
  tags:
    - ec2
    - docker
    - remote
  before_script:
    - cd app
    - npm install
  script:
    - npm run test
  artifacts:
    when: always           # Save even if tests fail
    reports:
      junit: app/junit.xml # JUnit report for GitLab UI
```

**File:** [10-unit-tests.gitlab-ci.yml](pipeline-code/10-unit-tests.gitlab-ci.yml)

---

### Building & Pushing Docker Images

```yaml
variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE/microservice/$MICRO_SERVICE_NAME
  IMAGE_TAG: "1.0"

build_image:
  tags:
    - ec2
    - shell
    - remote
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .

push_image:
  needs:
    - build_image
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker push $IMAGE_NAME:$IMAGE_TAG
```

**File:** [11-build-image.gitlab-ci.yml](pipeline-code/11-build-image.gitlab-ci.yml)

---

### Deploying to a Server

```yaml
variables:
  DEV_SERVER_HOST: 35.180.46.122
  DEV_ENDPOINT: http://ec2-35-180-46-122.eu-west-3.compute.amazonaws.com:3000

deploy_to_dev:
  stage: deploy
  before_script:
    - chmod 400 $SSH_PRIVATE_KEY
  script:
    - ssh -o StrictHostKeyChecking=no -i $SSH_PRIVATE_KEY ubuntu@$DEV_SERVER_HOST "
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY &&
        docker run -d -p 3000:3000 $IMAGE_NAME:$IMAGE_TAG"
  environment:
    name: development
    url: $DEV_ENDPOINT     # Clickable link in GitLab UI
```

**File:** [12-deploy.gitlab-ci.yml](pipeline-code/12-deploy.gitlab-ci.yml)

---

### Deployment with Docker Compose

```yaml
deploy_to_dev:
  script:
    # Copy docker-compose.yaml to server
    - scp -o StrictHostKeyChecking=no -i $SSH_PRIVATE_KEY ./docker-compose.yaml ubuntu@$DEV_SERVER_HOST:/home/ubuntu

    # SSH and run docker-compose
    - ssh -o StrictHostKeyChecking=no -i $SSH_PRIVATE_KEY ubuntu@$DEV_SERVER_HOST "
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY &&
        export DC_IMAGE_NAME=$IMAGE_NAME &&
        export DC_IMAGE_TAG=$IMAGE_TAG &&
        docker-compose down &&
        docker-compose up -d"
```

**docker-compose.yaml:**
```yaml
version: "3.9"
services:
  app:
    image: ${DC_IMAGE_NAME}:${DC_IMAGE_TAG}
    ports:
      - ${DC_APP_PORT}:3000
```

**File:** [13-docker-compose.gitlab-ci.yml](pipeline-code/13-docker-compose.gitlab-ci.yml)

---

## Advanced Topics

### Dynamic Versioning

Generate version from `package.json` + pipeline ID.

**Method 1: Using artifacts file**
```yaml
build_image:
  before_script:
    - export PACKAGE_JSON_VERSION=$(cat app/package.json | jq -r .version)
    - export VERSION=$PACKAGE_JSON_VERSION.$CI_PIPELINE_IID
    - echo $VERSION > version-file.txt
  script:
    - docker build -t $IMAGE_NAME:$VERSION .
  artifacts:
    paths:
      - version-file.txt

push_image:
  needs:
    - build_image
  before_script:
    - export VERSION=$(cat version-file.txt)   # Read from artifact
  script:
    - docker push $IMAGE_NAME:$VERSION
```

**File:** [13-versioning.gitlab-ci.yml](pipeline-code/13-versioning.gitlab-ci.yml)

**Method 2: Using dotenv reports**
```yaml
build_image:
  before_script:
    - export VERSION=$PACKAGE_JSON_VERSION.$CI_PIPELINE_IID
    - echo "VERSION=$VERSION" >> build.env     # Write as KEY=VALUE
  artifacts:
    reports:
      dotenv: build.env                        # Auto-exported to dependent jobs

push_image:
  needs:
    - build_image
  script:
    - docker push $IMAGE_NAME:$VERSION         # $VERSION available automatically!
```

**File:** [14-versioning-dotenv.gitlab-ci.yml](pipeline-code/14-versioning-dotenv.gitlab-ci.yml)

---

### Caching Dependencies

Speed up pipelines by caching downloaded dependencies.

```yaml
run_unit_tests:
  cache:
    key: $CI_COMMIT_REF_NAME        # Cache per branch
    paths:
      - app/node_modules/           # What to cache
  before_script:
    - cd app
    - npm install                   # Uses cache if available
  script:
    - npm test

run_lint_checks:
  cache:
    key: $CI_COMMIT_REF_NAME
    paths:
      - app/node_modules/
    policy: pull                    # Only download, never upload
  script:
    - echo "Running lint"
```

**Cache Policies:**
| Policy | Description |
|--------|-------------|
| `pull-push` | Download at start, upload at end (default) |
| `pull` | Only download, never upload |
| `push` | Only upload, never download |

**Cache vs Artifacts:**
| Feature | Cache | Artifacts |
|---------|-------|-----------|
| Storage | Runner machine | GitLab server |
| Purpose | Dependencies (npm, pip, etc.) | Build outputs, test results |
| Reliability | Not guaranteed | Guaranteed |
| Scope | Between pipelines | Within pipeline |

**File:** [15-cache.gitlab-ci.yml](pipeline-code/15-cache.gitlab-ci.yml)

---

### Security Scanning (SAST)

Include GitLab's built-in security scanning templates.

```yaml
stages:
  - test

sast:
  stage: test

include:
  - template: Jobs/SAST.gitlab-ci.yml
```

**File:** [16-sast.gitlab-ci.yml](pipeline-code/16-sast.gitlab-ci.yml)

---

### Multi-Stage Pipeline with Job Templates

Production-ready pipeline with Dev → Staging → Production deployments.

```yaml
stages:
  - test
  - build
  - deploy_dev
  - deploy_staging
  - deploy_prod

# Hidden job template (starts with dot)
.deploy:
  tags:
    - ec2
    - shell
    - remote
  variables:
    SSH_KEY: ""
    SERVER_HOST: ""
    DEPLOY_ENV: ""
    APP_PORT: ""
    ENDPOINT: ""
  before_script:
    - chmod 400 $SSH_KEY
    - export VERSION=$(cat version-file.txt)
  script:
    - scp -i $SSH_KEY ./docker-compose.yaml ubuntu@$SERVER_HOST:/home/ubuntu
    - ssh -i $SSH_KEY ubuntu@$SERVER_HOST "
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY &&
        export COMPOSE_PROJECT_NAME=$DEPLOY_ENV &&
        export DC_IMAGE_NAME=$IMAGE_NAME &&
        export DC_IMAGE_TAG=$VERSION &&
        export DC_APP_PORT=$APP_PORT &&
        docker-compose down &&
        docker-compose up -d"
  environment:
    name: $DEPLOY_ENV
    url: $ENDPOINT

# Extend template for each environment
deploy_to_dev:
  extends: .deploy
  stage: deploy_dev
  variables:
    SSH_KEY: $SSH_PRIVATE_KEY
    SERVER_HOST: $DEV_SERVER_HOST
    DEPLOY_ENV: development
    APP_PORT: 3000
    ENDPOINT: $DEV_ENDPOINT

deploy_to_staging:
  extends: .deploy
  stage: deploy_staging
  needs:
    - build_image
    - run_functional_tests     # Wait for functional tests
  variables:
    DEPLOY_ENV: staging
    APP_PORT: 4000

deploy_to_prod:
  extends: .deploy
  stage: deploy_prod
  needs:
    - build_image
    - run_performance_tests    # Wait for performance tests
  variables:
    DEPLOY_ENV: production
    APP_PORT: 5000
  when: manual                 # Requires manual approval
```

**File:** [17-multi-stage.gitlab-ci.yml](pipeline-code/17-multi-stage.gitlab-ci.yml)

---

## Quick Reference

### Essential Keywords

| Keyword | Purpose |
|---------|---------|
| `stages` | Define execution order |
| `script` | Commands to execute (required) |
| `before_script` | Commands before main script |
| `after_script` | Commands after main script |
| `image` | Docker image for the job |
| `tags` | Select specific runners |
| `only/except` | Conditional job execution |
| `rules` | Advanced conditional logic |
| `needs` | Job dependencies (DAG) |
| `artifacts` | Files to pass between jobs |
| `cache` | Dependencies to cache |
| `variables` | Define variables |
| `environment` | Deployment environment info |
| `extends` | Inherit from template job |
| `when: manual` | Require manual trigger |
| `include` | Import external YAML |

### Pipeline Files Structure

```
pipeline-code/
├── 1-jobs.gitlab-ci.yml           # Basic job structure
├── 2-stages.gitlab-ci.yml         # Stages and ordering
├── 3-needs.gitlab-ci.yml          # Job dependencies
├── 4-script.gitlab-ci.yml         # Script commands
├── 5-bash.gitlab-ci.yml           # Running shell scripts
├── 6-only.gitlab-ci.yml           # Conditional execution
├── 7-workflowrules.gitlab-ci.yml  # Workflow rules
├── 8-variables.gitlab-ci.yml      # Variables
├── 9-runners.gitlab-ci.yml        # Runner tags and images
├── 10-unit-tests.gitlab-ci.yml    # Unit testing with artifacts
├── 11-build-image.gitlab-ci.yml   # Building Docker images
├── 12-deploy.gitlab-ci.yml        # Server deployment
├── 13-docker-compose.gitlab-ci.yml # Docker Compose deployment
├── 13-versioning.gitlab-ci.yml    # Dynamic versioning
├── 14-versioning-dotenv.gitlab-ci.yml # Versioning with dotenv
├── 15-cache.gitlab-ci.yml         # Caching dependencies
├── 16-sast.gitlab-ci.yml          # Security scanning
└── 17-multi-stage.gitlab-ci.yml   # Complete multi-stage pipeline
```

### Best Practices

1. **Cache is an optimization, not a guarantee** - Jobs should work without cache
2. **Use artifacts for build outputs** - Cache for dependencies
3. **Mask sensitive variables** - Never expose secrets in logs
4. **Use job templates (`.job_name`)** - DRY principle for similar jobs
5. **Use `needs` for DAG** - Speed up pipeline by removing unnecessary waits
6. **Define environments** - Track deployments in GitLab UI
7. **Use `when: manual` for production** - Prevent accidental deployments

---

## Final Pipeline

See [FINAL-.gitlab-ci.yml](FINAL-.gitlab-ci.yml) for a complete production-ready pipeline combining all concepts.

---

## Resources

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Predefined Variables Reference](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)
- [GitLab Runner Documentation](https://docs.gitlab.com/runner/)
