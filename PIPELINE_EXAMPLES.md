# Примеры работы CI/CD пайплайна

Практические примеры использования пайплайна в различных сценариях.

## 📊 Визуализация пайплайна

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            VALIDATE STAGE                                │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  validate:dockerfiles                                           │    │
│  │  ✓ Проверка Dockerfile.backend                                 │    │
│  │  ✓ Проверка Dockerfile.frontend                                │    │
│  │  ✓ Проверка Dockerfile.proxy                                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                             BUILD STAGE                                  │
│                        (Параллельное выполнение)                        │
│                                                                           │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌──────────────────┐│
│  │  build:backend      │  │  build:frontend     │  │  build:proxy     ││
│  │                     │  │                     │  │                  ││
│  │  ├─ dependencies    │  │  ├─ dependencies    │  │  ├─ nginx:alpine││
│  │  ├─ builder         │  │  ├─ dev-deps        │  │  └─ copy config ││
│  │  └─ final image     │  │  ├─ builder         │  │                  ││
│  │                     │  │  └─ production      │  │  3 min          ││
│  │  15 min             │  │                     │  │                  ││
│  │                     │  │  12 min             │  │                  ││
│  └─────────────────────┘  └─────────────────────┘  └──────────────────┘│
│           │                         │                        │           │
│           └─────────────────────────┼────────────────────────┘           │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                             TEST STAGE                                   │
│                        (Параллельное выполнение)                        │
│                                                                           │
│  ┌─────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │  test:backend               │  │  test:frontend                  │  │
│  │                             │  │                                 │  │
│  │  ├─ mvn test                │  │  ├─ npm ci                     │  │
│  │  ├─ JUnit reports           │  │  ├─ npm test                   │  │
│  │  └─ Coverage: 75%           │  │  └─ Coverage: 80%              │  │
│  │                             │  │                                 │  │
│  │  5 min                      │  │  3 min                          │  │
│  └─────────────────────────────┘  └─────────────────────────────────┘  │
│           │                                  │                           │
│           └──────────────────────────────────┘                           │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                             SCAN STAGE                                   │
│                        (Параллельное выполнение)                        │
│                                                                           │
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────────┐   │
│  │  scan:backend     │  │  scan:frontend    │  │  scan:proxy      │   │
│  │                   │  │                   │  │                  │   │
│  │  Trivy Scanner    │  │  Trivy Scanner    │  │  Trivy Scanner   │   │
│  │  ✓ 0 CRITICAL     │  │  ✓ 0 CRITICAL     │  │  ✓ 0 CRITICAL    │   │
│  │  ⚠ 2 HIGH         │  │  ✓ 0 HIGH         │  │  ✓ 0 HIGH        │   │
│  │                   │  │                   │  │                  │   │
│  │  2 min            │  │  2 min            │  │  1 min           │   │
│  └───────────────────┘  └───────────────────┘  └──────────────────┘   │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                          DEPLOY-DEV STAGE                                │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  deploy:dev                                                     │    │
│  │                                                                 │    │
│  │  ├─ Pull images from registry                                  │    │
│  │  ├─ Update docker-compose.yml                                  │    │
│  │  ├─ Deploy to dev.example.com                                  │    │
│  │  └─ Health check: ✓ OK                                         │    │
│  │                                                                 │    │
│  │  Environment: https://dev.example.com                          │    │
│  │  3 min                                                          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                       DEPLOY-STAGING STAGE                               │
│                          (Manual trigger)                                │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  deploy:staging                                                 │    │
│  │  🖱️  [Click to deploy]                                          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                        DEPLOY-PROD STAGE                                 │
│                     (Manual trigger, tags only)                          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  deploy:production                                              │    │
│  │  🖱️  [Click to deploy]                                          │    │
│  │  ⚠️  Requires tag or main branch                                │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🎬 Сценарий 1: Обычная разработка (Feature Branch)

### Ситуация
Разработчик работает над новой фичей в backend.

### Действия

```bash
# 1. Создаем feature branch
git checkout -b feature/add-user-profile

# 2. Вносим изменения только в backend
vim spring-backend/src/main/java/backend/hobbiebackend/web/UserController.java

# 3. Коммитим
git add spring-backend/
git commit -m "Add user profile endpoint"
git push origin feature/add-user-profile
```

### Что происходит в пайплайне

```
✓ validate:dockerfiles       [30s]  - Выполнился (нет changes rule)
✓ build:backend              [15m]  - Выполнился (есть изменения в spring-backend/)
✗ build:frontend             [-]    - Пропущен (нет изменений в react-frontend/)
✗ build:proxy                [-]    - Пропущен (нет изменений в nginx/)
✓ test:backend               [5m]   - Выполнился (есть изменения в spring-backend/)
✗ test:frontend              [-]    - Пропущен
✓ scan:backend               [2m]   - Выполнился
✗ deploy:dev                 [-]    - Пропущен (не main ветка)
```

**Общее время**: ~22 минуты
**Экономия**: ~15 минут (не собирали frontend и proxy)

### Созданные образы

```
registry.gitlab.com/your-username/project/backend:feature-add-user-profile
registry.gitlab.com/your-username/project/backend:feature-add-user-profile-a1b2c3d
```

---

## 🎬 Сценарий 2: Merge в main (Production Release)

### Ситуация
Мержим готовую фичу в main ветку.

### Действия

```bash
# 1. Создаем Merge Request в GitLab UI
# 2. После аппрува мержим в main
# 3. Pipeline запускается автоматически
```

### Что происходит в пайплайне

```
✓ validate:dockerfiles       [30s]
✓ build:backend              [15m]
✗ build:frontend             [-]    - Пропущен (нет изменений)
✗ build:proxy                [-]    - Пропущен (нет изменений)
✓ test:backend               [5m]
✓ scan:backend               [2m]
✓ deploy:dev                 [3m]   - Автоматически деплоит в dev
🖱️  deploy:staging            [-]    - Ждет ручного запуска
🖱️  deploy:production         [-]    - Ждет ручного запуска
```

### Созданные образы

```
registry.gitlab.com/your-username/project/backend:main
registry.gitlab.com/your-username/project/backend:main-a1b2c3d
registry.gitlab.com/your-username/project/backend:latest  ← Обновлен!
```

### Deploy в dev окружение

```
✓ Образы подтянуты: backend:latest
✓ Контейнеры перезапущены
✓ Health check passed
🌐 Доступно: https://dev.example.com
```

---

## 🎬 Сценарий 3: Изменение зависимостей

### Ситуация
Обновляем версию Spring Boot и добавляем новую библиотеку.

### Действия

```bash
# 1. Обновляем pom.xml
vim spring-backend/pom.xml

# Изменяем:
# <version>2.4.2</version> → <version>2.7.0</version>
# Добавляем новую dependency

# 2. Коммитим
git add spring-backend/pom.xml
git commit -m "Update Spring Boot to 2.7.0, add Redis support"
git push
```

### Что происходит в Dockerfile

```dockerfile
# Этап 1: dependencies - ПЕРЕСОБИРАЕТСЯ
FROM maven:3.8.6-eclipse-temurin-11 AS dependencies
COPY spring-backend/pom.xml .
RUN mvn dependency:go-offline -B  ← Скачивает ВСЕ зависимости заново

# Этап 2: builder - использует новый dependencies layer
FROM dependencies AS builder
COPY spring-backend/src ./src
RUN mvn clean package -DskipTests -o  ← Использует уже скачанные зависимости

# Этап 3: final - использует новый JAR
FROM eclipse-temurin:11-jre-alpine
COPY --from=builder /app/target/*.jar /app/app.jar
```

### Timeline

```
Stage 1: dependencies
├─ Cache miss (pom.xml changed)
├─ Downloading dependencies... [10 min]
└─ New layer cached ✓

Stage 2: builder
├─ Using cached dependencies ✓
├─ Copying source code
└─ Building JAR [5 min]

Stage 3: final
└─ Creating production image [1 min]

Total: 16 min
```

### Следующая сборка (без изменения pom.xml)

```
Stage 1: dependencies
└─ Cache hit! ✓ [0 min]  ← Используется кэш!

Stage 2: builder
├─ Using cached dependencies ✓
└─ Building JAR [5 min]

Stage 3: final
└─ Creating production image [1 min]

Total: 6 min (экономия 10 минут!)
```

---

## 🎬 Сценарий 4: Hotfix в production

### Ситуация
Критический баг в production, нужен срочный фикс.

### Действия

```bash
# 1. Создаем hotfix ветку от последнего релиза
git checkout v1.2.0
git checkout -b hotfix/critical-auth-bug

# 2. Фиксим баг
vim spring-backend/src/main/java/backend/hobbiebackend/security/JwtFilter.java

# 3. Коммитим и пушим
git add .
git commit -m "Fix: JWT token validation vulnerability"
git push origin hotfix/critical-auth-bug

# 4. Создаем MR в main
# 5. После мержа создаем новый тег
git checkout main
git pull
git tag v1.2.1
git push origin v1.2.1
```

### Pipeline для тега v1.2.1

```
✓ validate:dockerfiles       [30s]
✓ build:backend              [6m]   - Используется кэш зависимостей
✓ test:backend               [5m]
✓ scan:backend               [2m]
🖱️  deploy:production         [-]    - Ждет подтверждения
```

### Созданные образы для тега

```
registry.gitlab.com/your-username/project/backend:v1.2.1
registry.gitlab.com/your-username/project/backend:v1-2-1-a1b2c3d
registry.gitlab.com/your-username/project/backend:latest  ← Обновлен
```

### Деплой в production

```
1. В GitLab UI открываем Pipeline
2. Нажимаем "Play" на job deploy:production
3. Подтверждаем деплой

🔄 Deployment process:
   ├─ Pulling image: backend:v1.2.1
   ├─ Stopping old containers
   ├─ Starting new containers
   ├─ Running health checks
   └─ ✓ Deployment successful!

🌐 Production: https://example.com
⏱️ Downtime: < 5 seconds (rolling update)
```

---

## 🎬 Сценарий 5: Полная пересборка всех сервисов

### Ситуация
Обновили docker-compose.yml, изменили конфигурацию nginx, обновили frontend и backend.

### Действия

```bash
# Изменяем несколько файлов
vim docker-compose.yml              # Меняем порты
vim nginx/nginx.conf                # Обновляем proxy настройки
vim react-frontend/src/App.js       # Новая фича в UI
vim spring-backend/src/.../HomeController.java  # Новый endpoint

git add .
git commit -m "Major update: new features and infrastructure changes"
git push
```

### Pipeline (все jobs выполняются)

```
Stage: VALIDATE
└─ validate:dockerfiles [30s]

Stage: BUILD (параллельно)
├─ build:backend   [15m]  ✓
├─ build:frontend  [12m]  ✓
└─ build:proxy     [3m]   ✓

Stage: TEST (параллельно)
├─ test:backend    [5m]   ✓
└─ test:frontend   [3m]   ✓

Stage: SCAN (параллельно)
├─ scan:backend    [2m]   ✓
├─ scan:frontend   [2m]   ✓
└─ scan:proxy      [1m]   ✓

Stage: DEPLOY-DEV
└─ deploy:dev      [3m]   ✓

Total pipeline time: ~23 минуты (благодаря параллелизму)
```

**Без параллелизма было бы**: 15+12+3+5+3+2+2+1+3 = 46 минут!

---

## 🎬 Сценарий 6: Работа с Merge Request

### Ситуация
Создан Merge Request для проверки перед мержем.

### Действия в GitLab UI

```
1. Create Merge Request: feature/new-api → main
2. Pipeline запускается автоматически
3. Проверяются только измененные компоненты
```

### Pipeline для MR

```
✓ validate:dockerfiles
✓ build:backend          - Только backend изменен
✓ test:backend
✓ scan:backend
✗ deploy:dev             - Не выполняется (не main branch)

Status: ✓ All checks passed
```

### В интерфейсе MR

```
┌────────────────────────────────────────────────────┐
│ Merge Request: feature/new-api → main              │
├────────────────────────────────────────────────────┤
│                                                     │
│ Pipeline: ✓ Passed                                 │
│ Coverage: 75% backend                              │
│ Security: ⚠️ 2 HIGH vulnerabilities found          │
│                                                     │
│ Checks:                                            │
│ ✓ Dockerfile validation                            │
│ ✓ Backend tests (120/120)                         │
│ ✓ Backend build                                    │
│ ⚠️ Security scan (see details)                     │
│                                                     │
│ [View Pipeline] [Merge] [Close]                   │
└────────────────────────────────────────────────────┘
```

---

## 🎬 Сценарий 7: Оптимизация с использованием кэша

### Ситуация
Измененили только один .java файл, не затронув зависимости.

### Первая сборка (холодный кэш)

```
build:backend [15 min]:
├─ Stage 1: dependencies [10 min]
│  ├─ Downloading Maven dependencies
│  └─ Cache saved: maven-pom.xml-abc123
│
├─ Stage 2: builder [4 min]
│  ├─ Copying source code
│  └─ Running mvn package
│
└─ Stage 3: final [1 min]
   └─ Creating production image
```

### Вторая сборка (теплый кэш)

```bash
# Меняем только один файл
vim spring-backend/src/main/java/backend/hobbiebackend/web/UserController.java
git commit -am "Fix user controller bug"
git push
```

```
build:backend [6 min]:
├─ Stage 1: dependencies [0 min] ✓ CACHED
│  └─ Cache hit: maven-pom.xml-abc123
│
├─ Stage 2: builder [5 min]
│  ├─ Cache hit for dependencies layer ✓
│  ├─ Copying source code (only changed files)
│  └─ Running mvn package
│
└─ Stage 3: final [1 min]
   └─ Creating production image

Saved: 9 minutes (60% faster!)
```

### Детали кэширования

```yaml
Cache strategy:
┌─────────────────────────────────────────────────┐
│ Local GitLab Runner cache:                      │
│ ├─ spring-backend/.m2/repository (Maven deps)   │
│ ├─ react-frontend/node_modules (npm deps)       │
│ └─ .trivycache/ (Trivy DB)                      │
│                                                  │
│ Docker BuildKit cache:                          │
│ ├─ Layer: FROM maven:3.8.6                      │
│ ├─ Layer: COPY pom.xml                          │
│ ├─ Layer: RUN mvn dependency:go-offline         │
│ └─ Layer: COPY src & RUN mvn package            │
└─────────────────────────────────────────────────┘
```

---

## 📊 Статистика и метрики

### Типичные времена выполнения

| Job | Холодный кэш | Теплый кэш | Экономия |
|-----|--------------|------------|----------|
| build:backend | 15 min | 6 min | 60% |
| build:frontend | 12 min | 4 min | 67% |
| build:proxy | 3 min | 1 min | 67% |
| test:backend | 5 min | 5 min | 0% |
| test:frontend | 3 min | 3 min | 0% |
| scan:backend | 2 min | 1 min | 50% |
| scan:frontend | 2 min | 1 min | 50% |
| scan:proxy | 1 min | 30 sec | 50% |

### Сценарии с экономией времени

| Сценарий | Без оптимизаций | С оптимизациями | Экономия |
|----------|-----------------|-----------------|----------|
| Изменение только backend | 28 min | 14 min | 50% |
| Изменение только frontend | 25 min | 11 min | 56% |
| Изменение всех сервисов | 46 min | 23 min | 50% |
| Hotfix (мелкое изменение) | 28 min | 10 min | 64% |

---

## 🎯 Best Practices по примерам

### ✅ DO: Atomic commits

```bash
# GOOD: Разделяем изменения по компонентам
git add spring-backend/
git commit -m "Backend: Add user profile API"

git add react-frontend/
git commit -m "Frontend: Add user profile page"

# Результат: 2 пайплайна, каждый собирает только нужное
```

### ❌ DON'T: Смешивание изменений

```bash
# BAD: Все в одном коммите
git add .
git commit -m "Various updates"

# Результат: Пересобираются ВСЕ сервисы, даже если изменения минимальны
```

### ✅ DO: Осмысленные названия веток

```bash
# GOOD
git checkout -b feature/user-authentication
git checkout -b fix/memory-leak-backend
git checkout -b refactor/frontend-components

# Легко понять что изменялось, можно настроить специальные rules
```

### ❌ DON'T: Общие названия

```bash
# BAD
git checkout -b test
git checkout -b fix
git checkout -b new-branch
```

---

## 🔍 Отладка пайплайна

### Пример: Build fails

```
Job: build:backend
Status: ❌ Failed
Duration: 8m 32s

Error:
[ERROR] Failed to execute goal on project hobbie-backend:
Could not resolve dependencies for project backend.hobbiebackend:hobbie-backend:jar:0.0.1-SNAPSHOT

Solution:
1. Проверить доступность Maven Central:
   curl https://repo.maven.apache.org/maven2/

2. Очистить кэш Maven:
   Settings → CI/CD → Clear runner caches

3. Добавить зеркало в pom.xml:
   <mirrors>
     <mirror>
       <id>central-mirror</id>
       <url>https://maven-mirror.example.com</url>
     </mirror>
   </mirrors>
```

### Пример: Docker build timeout

```
Job: build:frontend
Status: ❌ Failed
Error: Timeout after 1 hour

Solution:
1. Проверить размер контекста:
   du -sh react-frontend/

2. Добавить .dockerignore:
   node_modules/
   build/
   .git/
   *.log

3. Увеличить timeout в .gitlab-ci.yml:
   build:frontend:
     timeout: 2h
```

---

## 📝 Заключение

Эти примеры демонстрируют:

1. **Гибкость**: Пайплайн адаптируется под изменения
2. **Эффективность**: Умное кэширование и параллелизм
3. **Безопасность**: Автоматическое сканирование
4. **Контроль**: Ручные подтверждения для критичных окружений

Используйте эти паттерны как основу для вашего CI/CD процесса!
