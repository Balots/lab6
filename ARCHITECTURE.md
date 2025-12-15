# Архитектура CI/CD Pipeline

Визуальное представление архитектуры системы непрерывной интеграции и доставки.

## 🏗️ Общая архитектура

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DEVELOPER WORKSTATION                          │
│                                                                           │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐            │
│  │   Backend    │     │   Frontend   │     │     Nginx    │            │
│  │  (Java 11)   │     │   (React)    │     │    (Proxy)   │            │
│  └──────────────┘     └──────────────┘     └──────────────┘            │
│         │                     │                     │                    │
│         └─────────────────────┴─────────────────────┘                    │
│                               │                                          │
│                          git commit                                      │
│                          git push                                        │
└─────────────────────────────────┼──────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                            GITLAB SERVER                                 │
│                         (gitlab.com / self-hosted)                       │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Git Repository                                                 │    │
│  │  ├── .gitlab-ci.yml (main pipeline)                            │    │
│  │  ├── .gitlab/ci/ (modules)                                     │    │
│  │  ├── Dockerfile.backend                                        │    │
│  │  ├── Dockerfile.frontend                                       │    │
│  │  └── Dockerfile.proxy                                          │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                  │                                       │
│                          Pipeline Trigger                                │
│                                  │                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Pipeline Orchestrator                                          │    │
│  │  ├── Parse .gitlab-ci.yml                                      │    │
│  │  ├── Evaluate rules (changes detection)                        │    │
│  │  ├── Prepare jobs queue                                        │    │
│  │  └── Assign jobs to runners                                    │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                  │                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Container Registry                                             │    │
│  │  registry.gitlab.com/username/project/                         │    │
│  │  ├── backend:latest                                            │    │
│  │  ├── backend:main-a1b2c3d                                      │    │
│  │  ├── frontend:latest                                           │    │
│  │  └── proxy:latest                                              │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────┼──────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                          GITLAB RUNNER SERVER                            │
│                      (On-premise / Cloud / Docker)                       │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  GitLab Runner Process                                          │    │
│  │  ├── Polling for jobs (every 3s)                               │    │
│  │  ├── Job queue management                                      │    │
│  │  └── Docker executor                                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                  │                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Docker Engine (DinD - Docker-in-Docker)                       │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐ │    │
│  │  │  Job Container: build:backend                             │ │    │
│  │  │  Image: docker:24-dind                                    │ │    │
│  │  │  ├── docker build -f Dockerfile.backend                   │ │    │
│  │  │  ├── docker push registry.gitlab.com/...                  │ │    │
│  │  │  └── Cache: /var/run/docker.sock                          │ │    │
│  │  └──────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐ │    │
│  │  │  Job Container: test:backend                              │ │    │
│  │  │  Image: maven:3.8.6-eclipse-temurin-11                    │ │    │
│  │  │  ├── mvn test                                             │ │    │
│  │  │  ├── Generate JUnit reports                               │ │    │
│  │  │  └── Cache: .m2/repository                                │ │    │
│  │  └──────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐ │    │
│  │  │  Job Container: scan:backend                              │ │    │
│  │  │  Image: aquasec/trivy:latest                              │ │    │
│  │  │  ├── trivy image backend:latest                           │ │    │
│  │  │  ├── Generate security reports                            │ │    │
│  │  │  └── Cache: .trivycache/                                  │ │    │
│  │  └──────────────────────────────────────────────────────────┘ │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                  │                                       │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Local Cache Storage                                            │    │
│  │  ├── maven-pom.xml-abc123/                                     │    │
│  │  ├── npm-package-lock-def456/                                  │    │
│  │  ├── docker-cache/                                             │    │
│  │  └── .trivycache/                                              │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────┼──────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                           DEPLOYMENT TARGETS                             │
│                                                                           │
│  ┌───────────────────────┐  ┌──────────────────────┐  ┌──────────────┐│
│  │   DEV Environment     │  │  STAGING Environment │  │ PRODUCTION   ││
│  │   (Automatic)         │  │  (Manual)            │  │ (Manual)     ││
│  │                       │  │                      │  │              ││
│  │  Docker Compose       │  │  Kubernetes          │  │ Kubernetes   ││
│  │  or Kubernetes        │  │  ├── Namespace:      │  │ ├── NS: prod ││
│  │                       │  │  │   staging         │  │ └── HA setup ││
│  │  dev.example.com      │  │  staging.example.com │  │ example.com  ││
│  └───────────────────────┘  └──────────────────────┘  └──────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Детальный flow пайплайна

### 1. Code Push → Pipeline Trigger

```
Developer
    │
    ├─ git add spring-backend/src/main/java/...
    ├─ git commit -m "Add new feature"
    └─ git push origin feature/new-feature
         │
         ↓
    GitLab Server
         │
         ├─ Receive push event
         ├─ Parse .gitlab-ci.yml
         ├─ Evaluate rules (check file changes)
         └─ Create pipeline
              │
              ├─ Job: validate:dockerfiles (queued)
              ├─ Job: build:backend (queued)
              ├─ Job: build:frontend (skipped - no changes)
              └─ Job: build:proxy (skipped - no changes)
```

### 2. Job Execution на Runner

```
GitLab Runner
    │
    ├─ Poll for jobs (every 3 seconds)
    │
    ├─ Receive job: build:backend
    │
    ├─ Prepare environment
    │   ├─ Pull image: docker:24-dind
    │   ├─ Mount volumes: /var/run/docker.sock
    │   └─ Setup cache paths
    │
    ├─ Execute before_script
    │   ├─ docker info
    │   └─ docker login registry.gitlab.com
    │
    ├─ Execute script
    │   ├─ Check cache: maven-pom.xml-abc123 ✓ Found
    │   ├─ docker build (using BuildKit)
    │   │   │
    │   │   ├─ Stage 1: dependencies
    │   │   │   ├─ FROM maven:3.8.6-eclipse-temurin-11
    │   │   │   ├─ COPY pom.xml → Cache HIT ✓
    │   │   │   └─ RUN mvn dependency:go-offline → Cache HIT ✓
    │   │   │
    │   │   ├─ Stage 2: builder
    │   │   │   ├─ FROM dependencies
    │   │   │   ├─ COPY src → Changed files
    │   │   │   └─ RUN mvn package → Execute (5 min)
    │   │   │
    │   │   └─ Stage 3: final
    │   │       ├─ FROM eclipse-temurin:11-jre-alpine
    │   │       └─ COPY --from=builder /app/target/*.jar
    │   │
    │   ├─ docker tag backend:latest
    │   ├─ docker tag backend:feature-new-feature
    │   └─ docker push registry.gitlab.com/.../backend
    │
    ├─ Execute after_script
    │   └─ docker logout
    │
    └─ Upload artifacts & cache
        ├─ Save cache: maven-pom.xml-abc123
        └─ Upload artifacts: build.env
```

---

## 🗂️ Модульная структура

```
.gitlab-ci.yml (Main Pipeline)
    │
    ├─ include:
    │   ├─ .gitlab/ci/variables.yml
    │   │   └─ Defines:
    │   │       ├─ CI_REGISTRY
    │   │       ├─ IMAGE_NAMES
    │   │       ├─ BUILD_TARGETS
    │   │       └─ CACHE_CONFIGS
    │   │
    │   ├─ .gitlab/ci/stages.yml
    │   │   └─ Defines:
    │   │       ├─ validate
    │   │       ├─ build
    │   │       ├─ test
    │   │       ├─ scan
    │   │       └─ deploy-*
    │   │
    │   ├─ .gitlab/ci/rules.yml
    │   │   └─ Defines:
    │   │       ├─ .rules:backend_changes
    │   │       ├─ .rules:frontend_changes
    │   │       ├─ .rules:main_branch
    │   │       └─ .rules:manual_production
    │   │
    │   ├─ .gitlab/ci/cache.yml
    │   │   └─ Defines:
    │   │       ├─ .cache:maven
    │   │       ├─ .cache:npm
    │   │       ├─ .cache:docker
    │   │       └─ .cache:trivy
    │   │
    │   ├─ .gitlab/ci/docker-build.yml
    │   │   └─ Template: .docker_build_template
    │   │       ├─ before_script: docker login
    │   │       ├─ script: build & push
    │   │       └─ after_script: docker logout
    │   │
    │   └─ .gitlab/ci/docker-scan.yml
    │       └─ Template: .docker_scan_template
    │           ├─ script: trivy scan
    │           └─ artifacts: security reports
    │
    └─ Jobs:
        ├─ build:backend (extends .docker_build_template + .cache:maven)
        ├─ build:frontend (extends .docker_build_template + .cache:npm)
        ├─ test:backend (extends .cache:maven)
        ├─ scan:backend (extends .docker_scan_template)
        └─ deploy:dev (depends on: build:* jobs)
```

---

## 📦 Multi-stage Dockerfile Architecture

### Backend (Spring Boot)

```
┌─────────────────────────────────────────────────────────────────┐
│ Dockerfile.backend                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Stage 1: dependencies (Cacheable)                              │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ FROM maven:3.8.6-eclipse-temurin-11                     │    │
│  │ COPY pom.xml, mvnw, .mvn/                              │    │
│  │ RUN mvn dependency:go-offline                           │    │
│  │ ↓                                                       │    │
│  │ Cached when: pom.xml unchanged                         │    │
│  │ Size: ~200MB (dependencies)                            │    │
│  └────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ↓                                                        │
│  Stage 2: builder                                               │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ FROM dependencies                                       │    │
│  │ COPY src/                                              │    │
│  │ RUN mvn clean package -DskipTests -o                   │    │
│  │ ↓                                                       │    │
│  │ Cached when: src/ unchanged                            │    │
│  │ Size: ~500MB (with compiled classes)                   │    │
│  └────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ↓                                                        │
│  Stage 3: final (Production)                                    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ FROM eclipse-temurin:11-jre-alpine                      │    │
│  │ COPY --from=builder /app/target/*.jar /app/app.jar    │    │
│  │ ENTRYPOINT ["java", "-jar", "/app/app.jar"]           │    │
│  │ ↓                                                       │    │
│  │ Final image size: ~180MB                               │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Optimization:                                                   │
│  • Dependencies cached separately (10 min saved)                │
│  • Multi-stage reduces final image by 320MB (64%)               │
│  • Alpine base for smaller footprint                            │
└─────────────────────────────────────────────────────────────────┘
```

### Frontend (React)

```
┌─────────────────────────────────────────────────────────────────┐
│ Dockerfile.frontend                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Stage 1: dependencies (Production deps)                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ FROM node:18-alpine                                     │    │
│  │ COPY package*.json                                     │    │
│  │ RUN npm ci --only=production                           │    │
│  │ Cached when: package-lock.json unchanged               │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Stage 2: dev-dependencies (All deps)                           │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ FROM node:18-alpine                                     │    │
│  │ COPY package*.json                                     │    │
│  │ RUN npm ci                                             │    │
│  │ Cached when: package-lock.json unchanged               │    │
│  └────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ↓                                                        │
│  Stage 3: builder (Build artifacts)                            │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ FROM dev-dependencies                                   │    │
│  │ COPY src/, public/                                     │    │
│  │ RUN npm run build                                      │    │
│  │ ↓ Produces: build/ directory                           │    │
│  └────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ├───────────────────────────────────────┐               │
│         ↓                                       ↓               │
│  Stage 4: development           Stage 5: production             │
│  ┌──────────────────────┐      ┌──────────────────────┐       │
│  │ FROM node:18-alpine   │      │ FROM nginx:alpine     │       │
│  │ COPY node_modules/    │      │ COPY build/ → /html/ │       │
│  │ COPY src/             │      │ COPY nginx.conf       │       │
│  │ CMD ["npm", "start"]  │      │ CMD ["nginx", "-g"... │       │
│  │ Size: ~500MB          │      │ Size: ~50MB           │       │
│  │ For: Development      │      │ For: Production       │       │
│  └──────────────────────┘      └──────────────────────┘       │
│                                                                   │
│  Build argument controls which stage is final:                  │
│  --target=development  OR  --target=production                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Cache Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                        CACHE HIERARCHY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Level 1: GitLab Runner Local Cache                             │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ Location: /cache or S3 bucket                          │    │
│  │                                                         │    │
│  │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │    │
│  │ │ Maven Cache  │  │  npm Cache   │  │ Trivy Cache  │ │    │
│  │ │ Key: pom.xml │  │ Key: pkg.json│  │ Key: static  │ │    │
│  │ │ .m2/repo/    │  │ node_modules/│  │ .trivycache/ │ │    │
│  │ │ ~200MB       │  │ ~150MB       │  │ ~50MB        │ │    │
│  │ └──────────────┘  └──────────────┘  └──────────────┘ │    │
│  │                                                         │    │
│  │ Invalidation: When key files change                   │    │
│  │ Retention: Until manual clear or disk full            │    │
│  └────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ↓                                   │
│  Level 2: Docker BuildKit Layer Cache                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ Location: Docker daemon / BuildKit cache               │    │
│  │                                                         │    │
│  │ Backend layers:                                        │    │
│  │ ├─ maven:3.8.6-eclipse-temurin-11  ✓ CACHED          │    │
│  │ ├─ COPY pom.xml                     ✓ CACHED          │    │
│  │ ├─ RUN mvn dependency:go-offline    ✓ CACHED          │    │
│  │ ├─ COPY src/                        ✗ Changed          │    │
│  │ └─ RUN mvn package                  ⟳ Rebuilding       │    │
│  │                                                         │    │
│  │ Frontend layers:                                       │    │
│  │ ├─ node:18-alpine                   ✓ CACHED          │    │
│  │ ├─ COPY package*.json               ✓ CACHED          │    │
│  │ ├─ RUN npm ci                       ✓ CACHED          │    │
│  │ ├─ COPY src/                        ✗ Changed          │    │
│  │ └─ RUN npm run build                ⟳ Rebuilding       │    │
│  │                                                         │    │
│  │ Invalidation: Automatically by Docker layer hash      │    │
│  │ Retention: Based on docker system prune settings      │    │
│  └────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ↓                                   │
│  Level 3: Container Registry Cache                             │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ Location: registry.gitlab.com                          │    │
│  │                                                         │    │
│  │ Used via: --cache-from registry.gitlab.com/.../image   │    │
│  │                                                         │    │
│  │ Allows: Pulling previous layers from registry         │    │
│  │ Benefit: Shared cache between different runners       │    │
│  │ Drawback: Network overhead for large images           │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Cache Hit Ratio (typical):                                     │
│  • Cold build: 0% cached  → 15-20 minutes                      │
│  • Warm build (deps unchanged): 80% cached → 5-8 minutes       │
│  • Hot build (only code change): 90% cached → 3-5 minutes      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Rules Engine - Conditional Execution

```
┌─────────────────────────────────────────────────────────────────┐
│                    RULES EVALUATION ENGINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  On Push Event                                                   │
│  ├─ Extract commit info                                         │
│  │   ├─ Branch: feature/new-api                                │
│  │   ├─ Commit: a1b2c3d                                        │
│  │   └─ Changed files: spring-backend/src/.../*.java            │
│  │                                                              │
│  └─ Evaluate rules for each job                                │
│                                                                   │
│  build:backend                                                  │
│  ├─ Rule 1: if MR event + changes in spring-backend/**         │
│  │   ├─ MR event? ✗ (push to branch)                          │
│  │   └─ Skip this rule                                         │
│  ├─ Rule 2: if main branch + changes in spring-backend/**      │
│  │   ├─ Main branch? ✗ (feature branch)                       │
│  │   └─ Skip this rule                                         │
│  ├─ Rule 3: if tag                                             │
│  │   ├─ Tag? ✗                                                 │
│  │   └─ Skip this rule                                         │
│  └─ Rule 4: when manual (allow_failure: true)                  │
│      └─ Result: JOB CAN RUN MANUALLY ✓                         │
│                                                                   │
│  build:frontend                                                 │
│  ├─ Rule 1: changes in react-frontend/**                       │
│  │   ├─ Changed files match? ✗                                 │
│  │   └─ SKIP JOB                                               │
│  └─ Result: JOB SKIPPED ✗                                      │
│                                                                   │
│  deploy:dev                                                     │
│  ├─ Rule 1: if main branch                                     │
│  │   ├─ Main branch? ✗ (feature branch)                       │
│  │   └─ SKIP JOB                                               │
│  └─ Result: JOB SKIPPED ✗                                      │
│                                                                   │
│  Final Pipeline:                                                │
│  ✓ validate:dockerfiles                                        │
│  🖱️ build:backend (manual, allowed to run)                      │
│  ✗ build:frontend (skipped, no changes)                        │
│  ✗ build:proxy (skipped, no changes)                           │
│  ✗ deploy:dev (skipped, not main branch)                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 Resource Utilization

```
GitLab Runner Server Specifications:
├─ CPU: 4 cores (minimum)
├─ RAM: 8 GB (minimum)
├─ Disk: 100 GB SSD (recommended)
└─ Network: 100 Mbps (for pulling/pushing images)

Resource Usage per Job:
┌──────────────────────┬─────────┬─────────┬──────────┬───────────┐
│ Job                  │ CPU     │ RAM     │ Disk I/O │ Network   │
├──────────────────────┼─────────┼─────────┼──────────┼───────────┤
│ build:backend        │ 200-300%│ 2-3 GB  │ High     │ 500 MB    │
│ build:frontend       │ 150-250%│ 1-2 GB  │ Medium   │ 300 MB    │
│ test:backend         │ 100-150%│ 1-2 GB  │ Low      │ 50 MB     │
│ test:frontend        │ 80-120% │ 500 MB  │ Low      │ 50 MB     │
│ scan:backend         │ 50-100% │ 500 MB  │ Medium   │ 200 MB    │
└──────────────────────┴─────────┴─────────┴──────────┴───────────┘

Concurrent Jobs (configured: 4):
├─ Allows 4 jobs to run simultaneously
├─ Recommended: 1 concurrent job per 2 CPU cores
└─ Can be increased based on server capacity
```

---

## 🔒 Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       SECURITY LAYERS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Layer 1: Access Control                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • GitLab authentication & authorization                 │    │
│  │ • Protected branches (main, production)                │    │
│  │ • Protected variables (CI_REGISTRY_PASSWORD)           │    │
│  │ • Runner registration tokens                           │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Layer 2: Build Isolation                                       │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • Each job runs in isolated Docker container           │    │
│  │ • Privileged mode only for Docker-in-Docker             │    │
│  │ • Ephemeral containers (destroyed after job)            │    │
│  │ • Network isolation between jobs                       │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Layer 3: Secrets Management                                    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • Variables masked in logs                             │    │
│  │ • Protected variables for sensitive data               │    │
│  │ • No secrets in Dockerfiles or code                    │    │
│  │ • Vault integration (optional)                         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Layer 4: Image Scanning (Trivy)                               │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • CVE database scanning                                │    │
│  │ • CRITICAL & HIGH severity alerts                      │    │
│  │ • Integrated with GitLab Security Dashboard            │    │
│  │ • Can block deployment on vulnerabilities              │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Layer 5: Container Registry                                    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • TLS encryption in transit                            │    │
│  │ • Access tokens with limited scope                     │    │
│  │ • Image signing (optional, with Notary)                │    │
│  │ • Cleanup policies for old images                      │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📈 Scaling Strategy

```
Small Project (1-5 developers):
├─ 1 GitLab Runner
├─ 2 concurrent jobs
└─ Local cache

Medium Project (5-20 developers):
├─ 2-3 GitLab Runners
├─ 4-8 concurrent jobs
├─ Shared cache (S3 or NFS)
└─ Multiple tagged runners (build, test, deploy)

Large Project (20+ developers):
├─ 5+ GitLab Runners (autoscaling)
├─ 10-20 concurrent jobs
├─ Distributed cache (S3, Redis)
├─ Kubernetes executor
└─ Dedicated runners per team/project
```

---

## 🎓 Заключение

Эта архитектура обеспечивает:

- ✅ **Производительность**: Кэширование, параллелизм, умная сборка
- ✅ **Гибкость**: Модульная структура, легко адаптируется
- ✅ **Безопасность**: Многоуровневая защита, сканирование
- ✅ **Масштабируемость**: От малых до больших проектов
- ✅ **Надежность**: Изоляция jobs, retry механизмы

**Документация проекта:**
- [QUICKSTART.md](QUICKSTART.md) - Быстрый старт
- [CI_CD_SETUP.md](CI_CD_SETUP.md) - Полная настройка
- [GITLAB_RUNNER_SETUP.md](GITLAB_RUNNER_SETUP.md) - Установка Runner
- [PIPELINE_EXAMPLES.md](PIPELINE_EXAMPLES.md) - Примеры использования
