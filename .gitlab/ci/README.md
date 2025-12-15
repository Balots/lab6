# CI/CD Modules - Универсальные компоненты для GitLab CI/CD

Этот каталог содержит переиспользуемые модули для построения CI/CD пайплайнов.

## 📁 Структура

```
.gitlab/ci/
├── cache.yml           # Конфигурации кэширования
├── docker-build.yml    # Шаблон сборки Docker образов
├── docker-scan.yml     # Шаблон сканирования безопасности
├── rules.yml           # Условия выполнения jobs
├── stages.yml          # Определение стадий пайплайна
└── variables.yml       # Глобальные переменные
```

## 🔧 Модули

### 1. variables.yml

**Назначение**: Глобальные переменные для всего пайплайна

**Что внутри**:
- Настройки Docker Registry
- Имена образов
- Пути к Dockerfile
- Настройки Maven, npm
- Параметры безопасности

**Использование**:
```yaml
include:
  - local: '.gitlab/ci/variables.yml'

# Переменные доступны во всех jobs
my-job:
  script:
    - echo $BACKEND_IMAGE_NAME
```

---

### 2. stages.yml

**Назначение**: Определение стадий пайплайна

**Стадии**:
```
validate → build → test → scan → deploy-dev → deploy-staging → deploy-prod
```

**Использование**:
```yaml
include:
  - local: '.gitlab/ci/stages.yml'

my-job:
  stage: build  # Используем определенную стадию
```

---

### 3. rules.yml

**Назначение**: Переиспользуемые правила для условного выполнения jobs

**Доступные правила**:

| Правило | Описание | Когда выполняется |
|---------|----------|-------------------|
| `.rules:backend_changes` | Изменения backend | При изменении `spring-backend/**`, `Dockerfile.backend` |
| `.rules:frontend_changes` | Изменения frontend | При изменении `react-frontend/**`, `Dockerfile.frontend` |
| `.rules:proxy_changes` | Изменения proxy | При изменении `nginx/**`, `Dockerfile.proxy` |
| `.rules:maven_deps_changes` | Изменения Maven зависимостей | При изменении `pom.xml` |
| `.rules:npm_deps_changes` | Изменения npm зависимостей | При изменении `package.json`, `package-lock.json` |
| `.rules:main_branch` | Только на main ветке | На `master`/`main` ветке |
| `.rules:on_tags` | Только на тегах | При создании тега |
| `.rules:merge_requests` | На MR | При создании/обновлении MR |
| `.rules:manual_production` | Ручной запуск для prod | Manual на `main` или тегах |

**Использование**:
```yaml
include:
  - local: '.gitlab/ci/rules.yml'

build:backend:
  extends:
    - .rules:backend_changes  # Job запустится только при изменении backend
  script:
    - docker build -f Dockerfile.backend .
```

---

### 4. cache.yml

**Назначение**: Конфигурации кэширования для различных технологий

**Доступные кэши**:

| Шаблон | Описание | Кэшируемые пути |
|--------|----------|-----------------|
| `.cache:maven` | Maven зависимости | `.m2/repository` |
| `.cache:npm` | npm зависимости | `node_modules`, `.npm` |
| `.cache:docker` | Docker слои | `docker-cache/` |
| `.cache:trivy` | База данных Trivy | `.trivycache/` |

**Key strategy**: Кэш инвалидируется при изменении `pom.xml` или `package-lock.json`

**Использование**:
```yaml
include:
  - local: '.gitlab/ci/cache.yml'

test:backend:
  extends:
    - .cache:maven  # Используем Maven кэш
  script:
    - mvn test
```

---

### 5. docker-build.yml

**Назначение**: Универсальный шаблон для сборки и публикации Docker образов

**Возможности**:
- ✅ Поддержка multi-stage builds
- ✅ Кэширование Docker слоев
- ✅ Автоматическое тегирование (branch, sha, latest)
- ✅ Передача build аргументов
- ✅ Публикация в Container Registry

**Параметры**:

| Переменная | Описание | Обязательная | Пример |
|------------|----------|--------------|--------|
| `IMAGE_NAME` | Имя образа | Да | `backend` |
| `DOCKERFILE_PATH` | Путь к Dockerfile | Нет (default: `Dockerfile`) | `Dockerfile.backend` |
| `BUILD_CONTEXT` | Build context | Нет (default: `.`) | `.` |
| `BUILD_TARGET` | Target для multi-stage | Нет | `production` |
| `BUILD_ARGS` | Build аргументы | Нет | `NODE_ENV=production` |

**Теги, которые создаются**:
- `${IMAGE_NAME}:${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHORT_SHA}` - уникальный тег
- `${IMAGE_NAME}:${CI_COMMIT_REF_SLUG}` - тег ветки
- `${IMAGE_NAME}:latest` - только для main ветки

**Использование**:
```yaml
include:
  - local: '.gitlab/ci/docker-build.yml'

build:my-service:
  extends:
    - .docker_build_template
  variables:
    IMAGE_NAME: "my-service"
    DOCKERFILE_PATH: "services/my-service/Dockerfile"
    BUILD_CONTEXT: "services/my-service"
    BUILD_TARGET: "production"
    BUILD_ARGS: "APP_VERSION=${CI_COMMIT_TAG}"
```

**Пример с multiple build args**:
```yaml
build:backend:
  extends:
    - .docker_build_template
  variables:
    IMAGE_NAME: "backend"
    BUILD_ARGS: "MAVEN_OPTS=-Xmx512m BUILD_NUMBER=${CI_PIPELINE_ID}"
```

---

### 6. docker-scan.yml

**Назначение**: Сканирование Docker образов на уязвимости с помощью Trivy

**Возможности**:
- ✅ Сканирование на CRITICAL, HIGH уязвимости
- ✅ JSON отчет для GitLab Security Dashboard
- ✅ Кэширование базы данных Trivy
- ✅ Настраиваемые уровни severity

**Параметры**:

| Переменная | Описание | По умолчанию |
|------------|----------|--------------|
| `IMAGE_NAME` | Имя образа для сканирования | - |
| `SEVERITY_LEVELS` | Уровни severity | `CRITICAL,HIGH` |

**Использование**:
```yaml
include:
  - local: '.gitlab/ci/docker-scan.yml'

scan:backend:
  extends:
    - .docker_scan_template
  variables:
    IMAGE_NAME: "backend"
    SEVERITY_LEVELS: "CRITICAL,HIGH,MEDIUM"  # Можно настроить
  needs:
    - job: build:backend
```

**Артефакты**:
- `trivy-report-${IMAGE_NAME}.json` - доступен 30 дней
- Интегрируется с GitLab Security Dashboard

---

## 🚀 Быстрый старт

### Минимальный .gitlab-ci.yml

```yaml
include:
  - local: '.gitlab/ci/variables.yml'
  - local: '.gitlab/ci/stages.yml'
  - local: '.gitlab/ci/rules.yml'
  - local: '.gitlab/ci/cache.yml'
  - local: '.gitlab/ci/docker-build.yml'

build:my-app:
  stage: build
  extends:
    - .docker_build_template
  variables:
    IMAGE_NAME: "my-app"
    DOCKERFILE_PATH: "Dockerfile"
```

### Полноценный пайплайн

```yaml
include:
  - local: '.gitlab/ci/variables.yml'
  - local: '.gitlab/ci/stages.yml'
  - local: '.gitlab/ci/rules.yml'
  - local: '.gitlab/ci/cache.yml'
  - local: '.gitlab/ci/docker-build.yml'
  - local: '.gitlab/ci/docker-scan.yml'

# BUILD STAGE
build:app:
  stage: build
  extends:
    - .docker_build_template
    - .cache:npm
    - .rules:frontend_changes  # Условное выполнение
  variables:
    IMAGE_NAME: "app"
    DOCKERFILE_PATH: "Dockerfile"
    BUILD_TARGET: "production"
  artifacts:
    reports:
      dotenv: build.env
  script:
    - !reference [.docker_build_template, script]
    - echo "APP_IMAGE=${IMAGE_TAGGED}" >> build.env

# TEST STAGE
test:app:
  stage: test
  image: node:18-alpine
  extends:
    - .cache:npm
    - .rules:frontend_changes
  script:
    - npm ci
    - npm test

# SCAN STAGE
scan:app:
  stage: scan
  extends:
    - .docker_scan_template
    - .rules:frontend_changes
  variables:
    IMAGE_NAME: "app"
  needs:
    - job: build:app
      artifacts: true

# DEPLOY STAGE
deploy:dev:
  stage: deploy-dev
  extends:
    - .rules:main_branch
  script:
    - echo "Deploying ${APP_IMAGE}"
    - kubectl set image deployment/app app=${APP_IMAGE}
  environment:
    name: development
    url: https://dev.example.com
  needs:
    - job: build:app
      artifacts: true
```

---

## 🎨 Примеры адаптации под разные проекты

### Пример 1: Node.js микросервис

```yaml
build:api:
  extends:
    - .docker_build_template
    - .cache:npm
  variables:
    IMAGE_NAME: "api"
    DOCKERFILE_PATH: "services/api/Dockerfile"
    BUILD_CONTEXT: "services/api"
    BUILD_TARGET: "production"
  rules:
    - changes:
        - services/api/**/*
```

### Пример 2: Python приложение

```yaml
build:ml-service:
  extends:
    - .docker_build_template
  variables:
    IMAGE_NAME: "ml-service"
    DOCKERFILE_PATH: "Dockerfile.python"
    BUILD_ARGS: "PYTHON_VERSION=3.11"
  rules:
    - changes:
        - "*.py"
        - requirements.txt
        - Dockerfile.python
```

### Пример 3: Monorepo с несколькими сервисами

```yaml
# Backend
build:backend:
  extends:
    - .docker_build_template
    - .cache:maven
  variables:
    IMAGE_NAME: "backend"
    DOCKERFILE_PATH: "backend/Dockerfile"
    BUILD_CONTEXT: "backend"
  rules:
    - changes:
        - backend/**/*

# Frontend
build:frontend:
  extends:
    - .docker_build_template
    - .cache:npm
  variables:
    IMAGE_NAME: "frontend"
    DOCKERFILE_PATH: "frontend/Dockerfile"
    BUILD_CONTEXT: "frontend"
    BUILD_TARGET: "production"
  rules:
    - changes:
        - frontend/**/*

# Admin panel
build:admin:
  extends:
    - .docker_build_template
    - .cache:npm
  variables:
    IMAGE_NAME: "admin"
    DOCKERFILE_PATH: "admin/Dockerfile"
    BUILD_CONTEXT: "admin"
  rules:
    - changes:
        - admin/**/*
```

### Пример 4: С динамическими окружениями

```yaml
build:review:
  extends:
    - .docker_build_template
  variables:
    IMAGE_NAME: "app-review-${CI_MERGE_REQUEST_IID}"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

deploy:review:
  stage: deploy-dev
  script:
    - helm upgrade --install app-mr-${CI_MERGE_REQUEST_IID} ./chart
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://mr-${CI_MERGE_REQUEST_IID}.example.com
    on_stop: stop:review
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

stop:review:
  stage: deploy-dev
  script:
    - helm uninstall app-mr-${CI_MERGE_REQUEST_IID}
  when: manual
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
```

---

## 🔄 Переиспользование в других проектах

### Вариант 1: Копирование модулей

```bash
# Скопируйте папку целиком
cp -r /path/to/this/project/.gitlab/ci /path/to/new/project/.gitlab/

# Или создайте Git submodule
cd /path/to/new/project
git submodule add https://gitlab.com/your/ci-modules.git .gitlab/ci
```

### Вариант 2: Remote include

Создайте публичный репозиторий с модулями:

```yaml
# В новом проекте
include:
  - remote: 'https://gitlab.com/your-username/ci-modules/-/raw/main/docker-build.yml'
  - remote: 'https://gitlab.com/your-username/ci-modules/-/raw/main/docker-scan.yml'
```

### Вариант 3: Project include

```yaml
include:
  - project: 'infrastructure/ci-cd-templates'
    ref: v1.0.0  # Можно указать версию
    file:
      - '/docker-build.yml'
      - '/docker-scan.yml'
      - '/rules.yml'
```

---

## 📋 Checklist для адаптации

При использовании в новом проекте:

- [ ] Скопировать/подключить модули
- [ ] Настроить переменные в `variables.yml` или GitLab UI
- [ ] Адаптировать `rules.yml` под структуру проекта
- [ ] Настроить пути кэширования в `cache.yml`
- [ ] Обновить имена образов
- [ ] Настроить теги для GitLab Runner
- [ ] Добавить секретные переменные в GitLab UI
- [ ] Протестировать пайплайн

---

## 🔗 Связанные документы

- [../../../.gitlab-ci.yml](../../../.gitlab-ci.yml) - Главный пайплайн проекта
- [../../../CI_CD_SETUP.md](../../../CI_CD_SETUP.md) - Полное руководство по настройке
- [../../../GITLAB_RUNNER_SETUP.md](../../../GITLAB_RUNNER_SETUP.md) - Установка GitLab Runner
