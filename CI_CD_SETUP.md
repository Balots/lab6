# CI/CD Pipeline Setup Guide

Руководство по настройке и использованию CI/CD пайплайна для автоматической сборки и публикации Docker образов.

## 📋 Содержание

1. [Обзор пайплайна](#обзор-пайплайна)
2. [Архитектура](#архитектура)
3. [Быстрый старт](#быстрый-старт)
4. [Детальная настройка](#детальная-настройка)
5. [Использование модулей](#использование-модулей)
6. [Переменные окружения](#переменные-окружения)
7. [Оптимизация сборки](#оптимизация-сборки)
8. [Примеры использования](#примеры-использования)
9. [FAQ](#faq)

---

## 🎯 Обзор пайплайна

Наш CI/CD пайплайн автоматически:

- ✅ Собирает Docker образы при изменении кода
- ✅ Оптимизирует сборку с помощью multi-stage builds и кэширования
- ✅ Запускает тесты (unit, integration)
- ✅ Сканирует образы на уязвимости
- ✅ Публикует образы в Container Registry
- ✅ Деплоит приложение в различные окружения (dev, staging, prod)

### Ключевые особенности

- **Модульность**: CI/CD компоненты вынесены в отдельные файлы для переиспользования
- **Параметризация**: Все настройки управляются через переменные GitLab
- **Оптимизация**: Умное кэширование зависимостей и Docker слоев
- **Безопасность**: Сканирование образов с Trivy
- **Условное выполнение**: Jobs запускаются только при изменении соответствующих файлов

---

## 🏗️ Архитектура

### Структура проекта

```
.
├── .gitlab-ci.yml                 # Главный пайплайн
├── .gitlab/
│   └── ci/
│       ├── variables.yml          # Глобальные переменные
│       ├── stages.yml             # Определение стадий
│       ├── rules.yml              # Условия выполнения jobs
│       ├── cache.yml              # Конфигурация кэширования
│       ├── docker-build.yml       # Шаблон сборки Docker
│       └── docker-scan.yml        # Шаблон сканирования
├── Dockerfile.backend             # Multi-stage Dockerfile для Java/Spring Boot
├── Dockerfile.frontend            # Multi-stage Dockerfile для React
├── Dockerfile.proxy               # Dockerfile для Nginx
├── spring-backend/                # Backend приложение
├── react-frontend/                # Frontend приложение
└── nginx/                         # Nginx конфигурация
```

### Стадии пайплайна

```
validate → build → test → scan → deploy-dev → deploy-staging → deploy-prod
```

1. **validate**: Проверка Dockerfiles (hadolint)
2. **build**: Сборка Docker образов для backend, frontend, proxy
3. **test**: Запуск unit и integration тестов
4. **scan**: Сканирование образов на уязвимости (Trivy)
5. **deploy-dev**: Автоматический деплой в dev окружение
6. **deploy-staging**: Ручной деплой в staging
7. **deploy-prod**: Ручной деплой в production

### Условия запуска

Jobs выполняются только при изменении соответствующих файлов:

| Job | Триггеры |
|-----|----------|
| `build:backend` | Изменения в `spring-backend/**`, `Dockerfile.backend`, `backend.env` |
| `build:frontend` | Изменения в `react-frontend/**`, `Dockerfile.frontend`, `frontend.env` |
| `build:proxy` | Изменения в `nginx/**`, `Dockerfile.proxy` |

---

## 🚀 Быстрый старт

### Шаг 1: Установка GitLab Runner

См. детальное руководство: [GITLAB_RUNNER_SETUP.md](GITLAB_RUNNER_SETUP.md)

**Быстрая установка (Linux):**

```bash
# Установка
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Установка Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker gitlab-runner

# Регистрация
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --registration-token "YOUR_TOKEN" \
  --executor "docker" \
  --docker-image alpine:latest \
  --tag-list "docker" \
  --docker-privileged="true" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"
```

### Шаг 2: Настройка переменных в GitLab

Перейдите в **Settings → CI/CD → Variables** и добавьте:

#### Обязательные переменные

| Ключ | Значение | Защищенная | Описание |
|------|----------|-----------|----------|
| `CI_REGISTRY_USER` | `your-gitlab-username` | Нет | Username для GitLab Container Registry |
| `CI_REGISTRY_PASSWORD` | `your-access-token` | **Да** | Personal Access Token с scope `write_registry` |

#### Опциональные переменные (с значениями по умолчанию)

| Ключ | Значение по умолчанию | Описание |
|------|---------------------|----------|
| `BACKEND_IMAGE_NAME` | `backend` | Имя образа backend |
| `FRONTEND_IMAGE_NAME` | `frontend` | Имя образа frontend |
| `PROXY_IMAGE_NAME` | `proxy` | Имя образа proxy |
| `FRONTEND_BUILD_TARGET` | `production` | Target для multi-stage build frontend |

### Шаг 3: Включение Container Registry

1. **Settings → General → Visibility, project features, permissions**
2. Включите **Container Registry**

### Шаг 4: Первый запуск

```bash
# Сделайте изменение в коде
echo "# Test change" >> spring-backend/README.md

# Закоммитьте и запушьте
git add .
git commit -m "Test CI/CD pipeline"
git push

# Откройте CI/CD → Pipelines в GitLab UI
```

✅ Пайплайн должен запуститься автоматически!

---

## ⚙️ Детальная настройка

### Настройка Container Registry

#### Создание Personal Access Token

1. **Profile → Preferences → Access Tokens**
2. Name: `CI_CD_Pipeline`
3. Scopes: `read_registry`, `write_registry`
4. Скопируйте сгенерированный токен

#### Тестирование доступа

```bash
# Локальный тест
echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER registry.gitlab.com

# Проверка
docker pull registry.gitlab.com/your-username/your-project/backend:latest
```

### Настройка multi-stage builds

#### Backend (Spring Boot)

[Dockerfile.backend](Dockerfile.backend) использует 3 стадии:

1. **dependencies**: Кэширование Maven зависимостей
2. **builder**: Сборка приложения
3. **final**: Production образ

**Кэш работает так:**
- При изменении `pom.xml` → пересобирается `dependencies`
- При изменении `.java` файлов → используется кэш `dependencies`, пересобирается только `builder`

#### Frontend (React)

[Dockerfile.frontend](Dockerfile.frontend) использует 5 стадий:

1. **dependencies**: Production npm зависимости
2. **dev-dependencies**: Dev npm зависимости
3. **builder**: Сборка production build
4. **development**: Development образ для dev окружения
5. **production**: Production образ с nginx

**Выбор target:**

```yaml
# В .gitlab-ci.yml или переменных GitLab
FRONTEND_BUILD_TARGET: "production"  # Для production
FRONTEND_BUILD_TARGET: "development" # Для development
```

### Оптимизация кэширования

#### Локальный кэш Runner

В `.gitlab/ci/cache.yml` настроено кэширование:

```yaml
.cache:maven:
  cache:
    key:
      files:
        - spring-backend/pom.xml  # Кэш инвалидируется при изменении pom.xml
    paths:
      - spring-backend/.m2/repository

.cache:npm:
  cache:
    key:
      files:
        - react-frontend/package-lock.json  # Кэш инвалидируется при изменении
    paths:
      - react-frontend/node_modules
```

#### Docker Layer Cache

В `.gitlab/ci/docker-build.yml`:

```yaml
docker build \
  --cache-from ${IMAGE_LATEST} \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  ...
```

Это позволяет переиспользовать слои из предыдущих сборок.

---

## 🧩 Использование модулей

### Переиспользование в других проектах

Модули в `.gitlab/ci/` универсальны и могут использоваться в любом проекте.

#### Вариант 1: Копирование модулей

```bash
# В новом проекте
mkdir -p .gitlab/ci
cp /path/to/this/project/.gitlab/ci/*.yml .gitlab/ci/
```

#### Вариант 2: Подключение через include (remote)

Создайте публичный репозиторий с модулями, затем:

```yaml
# В .gitlab-ci.yml нового проекта
include:
  - remote: 'https://gitlab.com/your-username/ci-cd-modules/-/raw/main/.gitlab/ci/docker-build.yml'
  - remote: 'https://gitlab.com/your-username/ci-cd-modules/-/raw/main/.gitlab/ci/docker-scan.yml'
```

#### Вариант 3: Подключение через project include

```yaml
include:
  - project: 'your-username/ci-cd-modules'
    ref: main
    file:
      - '.gitlab/ci/docker-build.yml'
      - '.gitlab/ci/docker-scan.yml'
```

### Создание custom job

```yaml
# В .gitlab-ci.yml вашего проекта
build:my-custom-service:
  stage: build
  extends:
    - .docker_build_template  # Используем шаблон
  variables:
    IMAGE_NAME: "my-service"
    DOCKERFILE_PATH: "services/my-service/Dockerfile"
    BUILD_CONTEXT: "services/my-service"
  rules:
    - changes:
        - services/my-service/**/*
```

---

## 🔐 Переменные окружения

### Переменные в .gitlab/ci/variables.yml

```yaml
variables:
  # Registry
  CI_REGISTRY: "registry.gitlab.com"
  CI_REGISTRY_IMAGE: "${CI_REGISTRY}/${CI_PROJECT_PATH}"

  # Образы
  BACKEND_IMAGE_NAME: "backend"
  FRONTEND_IMAGE_NAME: "frontend"
  PROXY_IMAGE_NAME: "proxy"

  # Build настройки
  DOCKER_BUILDKIT: "1"
  MAVEN_OPTS: "-Dmaven.repo.local=${CI_PROJECT_DIR}/spring-backend/.m2/repository"
  NODE_ENV: "production"
```

### Переопределение в GitLab UI

Все переменные можно переопределить в **Settings → CI/CD → Variables**.

### Использование в jobs

```yaml
build:backend:
  variables:
    IMAGE_NAME: ${BACKEND_IMAGE_NAME}  # Из variables.yml
    CUSTOM_VAR: "custom-value"          # Custom переменная
```

### Секретные переменные

**Никогда не храните в коде:**
- Пароли
- API ключи
- Токены
- Credentials

Используйте только **Settings → CI/CD → Variables** с флагом **Protected**.

---

## ⚡ Оптимизация сборки

### 1. Параллельное выполнение

Jobs в одной стадии выполняются параллельно:

```
build:backend ───┐
build:frontend ──┼─→ test stage
build:proxy ─────┘
```

### 2. Условное выполнение (rules)

```yaml
.rules:backend_changes:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - spring-backend/**/*
        - Dockerfile.backend
```

**Результат**: Backend собирается только при изменении backend файлов.

### 3. Artifacts между stages

```yaml
build:backend:
  artifacts:
    reports:
      dotenv: build.env  # Экспортируем переменные
  script:
    - echo "BACKEND_IMAGE=${IMAGE_TAGGED}" >> build.env

scan:backend:
  needs:
    - job: build:backend
      artifacts: true  # Получаем BACKEND_IMAGE
  script:
    - trivy image ${BACKEND_IMAGE}
```

### 4. Распределенное кэширование (опционально)

Для больших команд можно настроить S3 кэш:

```toml
# В config.toml GitLab Runner
[[runners]]
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      BucketName = "gitlab-runner-cache"
```

---

## 📚 Примеры использования

### Пример 1: Изменение только backend

```bash
# Изменяем backend
vim spring-backend/src/main/java/backend/hobbiebackend/web/HomeController.java

git add spring-backend/
git commit -m "Update HomeController"
git push
```

**Что произойдет:**
- ✅ `validate:dockerfiles` - выполнится
- ✅ `build:backend` - выполнится
- ❌ `build:frontend` - пропустится (no changes)
- ❌ `build:proxy` - пропустится (no changes)
- ✅ `test:backend` - выполнится
- ✅ `scan:backend` - выполнится

### Пример 2: Изменение зависимостей

```bash
# Обновляем зависимости
vim spring-backend/pom.xml  # Добавляем новую dependency

git add spring-backend/pom.xml
git commit -m "Add new dependency"
git push
```

**Что произойдет:**
- Кэш Maven инвалидируется (key based on pom.xml)
- Этап `dependencies` в Dockerfile пересобирается
- Зависимости скачиваются заново
- Новый кэш сохраняется для следующей сборки

### Пример 3: Ручной деплой в production

```bash
# Создаем тег для release
git tag v1.0.0
git push origin v1.0.0
```

**Что произойдет:**
- Pipeline запускается для тега
- Все образы собираются с тегом `v1.0.0`
- Job `deploy:production` становится доступным (manual)
- В GitLab UI: нажимаете кнопку для деплоя в production

### Пример 4: Сборка для dev окружения

```bash
# В GitLab UI: Settings → CI/CD → Variables
# Добавляем переменную:
FRONTEND_BUILD_TARGET = "development"

# Запускаем pipeline вручную
```

**Результат:**
- Frontend собирается с target `development` (с npm start, не nginx)
- Полезно для отладки в контейнере

---

## 🛠️ Настройка деплоя

### Docker Compose на удаленном сервере

Измените job `deploy:dev` в `.gitlab-ci.yml`:

```yaml
deploy:dev:
  stage: deploy-dev
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
  script:
    - scp docker-compose.yml $DEPLOY_USER@$DEPLOY_HOST:/app/
    - |
      ssh $DEPLOY_USER@$DEPLOY_HOST << EOF
        cd /app
        echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
        docker-compose pull
        docker-compose up -d
        docker logout $CI_REGISTRY
      EOF
  environment:
    name: development
    url: http://$DEPLOY_HOST
```

**Добавьте переменные:**
- `SSH_PRIVATE_KEY` - приватный SSH ключ
- `DEPLOY_USER` - username на сервере
- `DEPLOY_HOST` - IP или hostname сервера

### Kubernetes (kubectl)

```yaml
deploy:k8s:
  stage: deploy-dev
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials admin --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=admin
    - kubectl config use-context default
    - kubectl set image deployment/backend backend=$BACKEND_IMAGE -n $DEPLOY_NAMESPACE
    - kubectl set image deployment/frontend frontend=$FRONTEND_IMAGE -n $DEPLOY_NAMESPACE
    - kubectl rollout status deployment/backend -n $DEPLOY_NAMESPACE
```

### Helm

```yaml
deploy:helm:
  stage: deploy-dev
  image: alpine/helm:latest
  script:
    - helm upgrade --install myapp ./helm-chart \
        --set backend.image=$BACKEND_IMAGE \
        --set frontend.image=$FRONTEND_IMAGE \
        --namespace $DEPLOY_NAMESPACE
```

---

## ❓ FAQ

### Q: Как пропустить CI для коммита?

A: Добавьте `[ci skip]` или `[skip ci]` в commit message:
```bash
git commit -m "Update README [ci skip]"
```

### Q: Как запустить только определенный job?

A: Используйте переменную при ручном запуске пайплайна:
```yaml
build:backend:
  rules:
    - if: '$RUN_BACKEND == "true"'
```

Затем в UI запускайте с переменной `RUN_BACKEND=true`.

### Q: Как очистить кэш?

A:
1. **В GitLab UI**: CI/CD → Pipelines → Clear runner caches
2. **На Runner сервере**:
```bash
sudo gitlab-runner cache-clear
```

### Q: Почему сборка медленная?

A:
1. Проверьте, что кэш работает: смотрите логи `Checking cache for...`
2. Проверьте, что Docker layer cache работает: `CACHED` в логах
3. Увеличьте ресурсы Runner: CPU, RAM в `config.toml`
4. Используйте более быстрый storage driver: `overlay2`

### Q: Как сканировать только critical уязвимости?

A: Измените переменную:
```yaml
# В .gitlab-ci.yml или GitLab Variables
TRIVY_SEVERITY: "CRITICAL"
```

### Q: Как добавить уведомления в Slack/Discord?

A: Добавьте в конце `.gitlab-ci.yml`:
```yaml
notify:success:
  stage: .post
  script:
    - 'curl -X POST -H "Content-Type: application/json" -d "{\"text\":\"Pipeline $CI_PIPELINE_ID succeeded\"}" $SLACK_WEBHOOK'
  when: on_success

notify:failure:
  stage: .post
  script:
    - 'curl -X POST -H "Content-Type: application/json" -d "{\"text\":\"Pipeline $CI_PIPELINE_ID failed\"}" $SLACK_WEBHOOK'
  when: on_failure
```

### Q: Можно ли использовать без Docker?

A: Да, но не рекомендуется. Измените executor на `shell` в `config.toml` и адаптируйте jobs для использования локального окружения.

---

## 📖 Дополнительные ресурсы

- [GITLAB_RUNNER_SETUP.md](GITLAB_RUNNER_SETUP.md) - Детальная установка GitLab Runner
- [.gitlab-ci.yml](.gitlab-ci.yml) - Главный пайплайн
- [.gitlab/ci/](.gitlab/ci/) - Модульные компоненты
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Docker Multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)

---

## 🤝 Поддержка

Если у вас возникли проблемы:

1. Проверьте логи в **CI/CD → Pipelines → Failed job**
2. Проверьте статус Runner: **Settings → CI/CD → Runners**
3. Проверьте переменные: **Settings → CI/CD → Variables**
4. Посмотрите [GITLAB_RUNNER_SETUP.md - Troubleshooting](GITLAB_RUNNER_SETUP.md#troubleshooting)

---

**Happy CI/CD! 🚀**
