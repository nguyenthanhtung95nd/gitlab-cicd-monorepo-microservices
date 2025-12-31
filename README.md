# GitLab CI/CD Learning Projects

Personal learning repository documenting my journey through GitLab CI/CD. Contains hands-on examples progressing from basic pipeline concepts to production-ready multi-stage deployments and monorepo microservices architecture.

## Repository Structure

```
gitlab-cicd-projects/
├── nodeapp-cicd/                 # Fundamentals & Progressive Learning
│   ├── pipeline-code/            # Step-by-step pipeline examples (1-17)
│   ├── app/                      # Sample Node.js application
│   ├── FINAL-.gitlab-ci.yml      # Production-ready pipeline
│   └── README.md                 # Detailed notes on each concept
│
└── monorepo-microservice-cicd/   # Advanced: Monorepo Architecture
    ├── frontend/                 # Frontend microservice
    ├── products/                 # Products API microservice
    ├── shopping-cart/            # Shopping cart microservice
    ├── .gitlab-ci.yml            # Optimized monorepo pipeline
    └── README.md                 # Monorepo CI/CD patterns
```

---

## Learning Path

### Module 1: nodeapp-cicd - CI/CD Fundamentals

Progressive examples covering core GitLab CI/CD concepts:

| # | Topic | Key Concepts |
|---|-------|--------------|
| 1 | Jobs | Basic job structure, `script`, `before_script`, `after_script` |
| 2 | Stages | Stage ordering, parallel execution |
| 3 | Needs | Job dependencies (DAG) |
| 4 | Script | OS commands, shell execution |
| 5 | Bash | Running external shell scripts |
| 6 | Only/Except | Conditional job execution |
| 7 | Workflow Rules | Pipeline-level control |
| 8 | Variables | Global, job-level, predefined variables |
| 9 | Runners | Tags, executors, runner types |
| 10 | Unit Tests | Testing with artifacts & JUnit reports |
| 11 | Build Image | Docker build & push to registry |
| 12 | Deploy | SSH deployment to servers |
| 13 | Docker Compose | Multi-container deployments |
| 14 | Versioning | Dynamic versioning with dotenv |
| 15 | Cache | Dependency caching strategies |
| 16 | SAST | Security scanning templates |
| 17 | Multi-Stage | Dev → Staging → Production pipeline |

### Module 2: monorepo-microservice-cicd - Advanced Patterns

Real-world monorepo CI/CD implementation:

| Topic | Description |
|-------|-------------|
| `only:changes` | Selective build/deploy based on changed files |
| `extends` | DRY job templates with hidden jobs |
| Docker Networking | Shared network for microservice communication |
| Environment Variables | Dynamic configuration per service |

---

## Key Concepts Covered

### Pipeline Fundamentals

```yaml
# Basic pipeline structure
stages:
  - test
  - build
  - deploy

job_name:
  stage: test
  script:
    - echo "Running tests..."
```

### GitLab Runners

| Type | Scope | Use Case |
|------|-------|----------|
| Instance Runners | All projects | Shared infrastructure |
| Group Runners | Group projects | Team-specific runners |
| Project Runners | Single project | Isolated/specialized needs |

| Executor | Description |
|----------|-------------|
| Shell | Commands on host OS |
| Docker | Isolated container per job |
| Kubernetes | Pods for each job |
| Docker Machine | Auto-scaling Docker hosts |

### Artifacts vs Cache

| Feature | Artifacts | Cache |
|---------|-----------|-------|
| Storage | GitLab server | Runner machine |
| Purpose | Build outputs, test results | Dependencies (npm, pip) |
| Reliability | Guaranteed | Optimization only |
| Scope | Within pipeline | Between pipelines |

### Change Detection (Monorepo)

```yaml
build_frontend:
  only:
    changes:
      - "frontend/**/*"    # Only runs if frontend/ changes
```

### Job Templates

```yaml
.template:          # Hidden job (not executed)
  script:
    - echo "Shared logic"

actual_job:
  extends: .template  # Inherits from template
```

---

## Technologies Used

| Category | Technologies |
|----------|-------------|
| CI/CD | GitLab CI/CD |
| Containerization | Docker, Docker Compose |
| Runtime | Node.js |
| Infrastructure | AWS EC2 |
| Registry | GitLab Container Registry |

---

## Quick Start

### Prerequisites

- GitLab account with CI/CD enabled
- Docker installed (for local testing)
- GitLab Runner (optional, for self-hosted)

### Clone & Explore

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/gitlab-cicd-projects.git
cd gitlab-cicd-projects

# Explore fundamentals
cd nodeapp-cicd
cat README.md

# Explore monorepo patterns
cd ../monorepo-microservice-cicd
cat README.md
```

### Test Locally with GitLab Runner

```bash
# Install gitlab-runner locally
gitlab-runner exec docker job_name
```

---

## Pipeline Visualizations

### Basic Pipeline Flow

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│  TEST   │────▶│  BUILD  │────▶│ DEPLOY  │
│ stage   │     │  stage  │     │  stage  │
└─────────┘     └─────────┘     └─────────┘
     │               │               │
  ┌──┴──┐        ┌──┴──┐        ┌──┴──┐
  │job 1│        │job 3│        │job 4│
  │job 2│        └─────┘        └─────┘
  └─────┘
  parallel
```

### Multi-Stage Deployment

```
┌─────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────┐   ┌───────┐   ┌─────┐   ┌─────┐   ┌──────┐  │
│  │ Test │──▶│ Build │──▶│ Dev │──▶│Stag │──▶│ Prod │  │
│  └──────┘   └───────┘   └─────┘   └─────┘   └──────┘  │
│                            │         │      (manual)   │
│                         auto      auto                 │
└─────────────────────────────────────────────────────────┘
```

### Monorepo Selective Builds

```
┌─────────────────────────────────────────────────────────┐
│                    MONOREPO                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────────────┐   │
│  │ frontend/ │  │ products/ │  │  shopping-cart/   │   │
│  │  changed? │  │  changed? │  │     changed?      │   │
│  │     ↓     │  │     ↓     │  │        ↓          │   │
│  │  Build ✓  │  │  Skip ✗   │  │     Skip ✗        │   │
│  │ Deploy ✓  │  │           │  │                   │   │
│  └───────────┘  └───────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Best Practices Learned

1. **Cache is optimization, not guarantee** - Jobs should work without cache
2. **Use artifacts for build outputs** - Cache for dependencies only
3. **Mask sensitive variables** - Never expose secrets in logs
4. **Use job templates (`.job_name`)** - DRY principle
5. **Use `needs` for DAG** - Speed up pipeline execution
6. **Define environments** - Track deployments in GitLab UI
7. **Use `when: manual` for production** - Prevent accidental deployments
8. **Use `only:changes` in monorepos** - Avoid unnecessary builds

---

## Resources

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Predefined Variables Reference](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)
- [GitLab Runner Documentation](https://docs.gitlab.com/runner/)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## License

This project is for educational purposes. Feel free to use and modify for your own learning.

---

## Author

Learning notes by **Tung Nguyen**
