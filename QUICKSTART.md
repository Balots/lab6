# 🚀 Quick Start Guide - CI/CD Pipeline

Минимальная инструкция для быстрого запуска CI/CD пайплайна.

## ⚡ За 5 минут

### 1. Установите GitLab Runner (Ubuntu/Debian)

```bash
# Установка Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Установка Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
```

### 2. Регистрация Runner

Получите токен: **Settings → CI/CD → Runners** в GitLab UI

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --registration-token "PASTE_YOUR_TOKEN_HERE" \
  --executor "docker" \
  --docker-image alpine:latest \
  --tag-list "docker" \
  --docker-privileged="true" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"
```

### 3. Настройте переменные в GitLab

**Settings → CI/CD → Variables** добавьте:

| Ключ | Значение | Защищенная |
|------|----------|-----------|
| `CI_REGISTRY_USER` | `your-gitlab-username` | Нет |
| `CI_REGISTRY_PASSWORD` | `your-personal-access-token` | **Да** |

**Создание Personal Access Token:**
1. Profile → Preferences → Access Tokens
2. Name: `CI_CD_Pipeline`
3. Scopes: `read_registry`, `write_registry`
4. Создать и скопировать токен

### 4. Включите Container Registry

**Settings → General → Visibility** → Включите **Container Registry**

### 5. Запустите первый Pipeline

```bash
# Сделайте любое изменение
echo "# CI/CD Pipeline configured" >> README.md

# Закоммитьте и запушьте
git add README.md
git commit -m "Configure CI/CD pipeline"
git push
```

✅ **Готово!** Откройте **CI/CD → Pipelines** в GitLab UI

---

## 📂 Структура проекта

Все файлы уже созданы и настроены:

```
.
├── .gitlab-ci.yml                 ← Главный пайплайн
├── .gitlab/ci/                    ← Модули
│   ├── variables.yml
│   ├── stages.yml
│   ├── rules.yml
│   ├── cache.yml
│   ├── docker-build.yml
│   ├── docker-scan.yml
│   └── README.md
├── Dockerfile.backend             ← Multi-stage builds
├── Dockerfile.frontend
├── Dockerfile.proxy
├── CI_CD_SETUP.md                 ← Полная документация
├── GITLAB_RUNNER_SETUP.md         ← Детальная установка Runner
├── PIPELINE_EXAMPLES.md           ← Примеры использования
└── QUICKSTART.md                  ← Вы здесь
```

---

## 🎯 Что делает пайплайн

### Автоматически при push:

1. ✅ **Validate** - Проверяет Dockerfiles
2. ✅ **Build** - Собирает Docker образы (только измененные!)
3. ✅ **Test** - Запускает тесты
4. ✅ **Scan** - Сканирует на уязвимости
5. ✅ **Deploy to Dev** - Автоматический деплой (только main ветка)

### Вручную:

6. 🖱️ **Deploy to Staging** - По кнопке
7. 🖱️ **Deploy to Production** - По кнопке (только тэги/main)

---

## 📊 Стадии Pipeline

```
validate → build → test → scan → deploy-dev → deploy-staging → deploy-prod
  30s       15m     5m     2m        3m            manual          manual
```

**Общее время**: ~25 минут для полной сборки всех сервисов

---

## 🔑 Ключевые особенности

### ⚡ Умная сборка

- Собираются **только измененные** сервисы
- При изменении только backend - frontend не собирается
- Экономия времени до **60%**

### 💾 Кэширование

- **Maven зависимости** - обновляются только при изменении `pom.xml`
- **npm зависимости** - обновляются только при изменении `package.json`
- **Docker слои** - переиспользуются между сборками

### 🔒 Безопасность

- Автоматическое сканирование на уязвимости (Trivy)
- Отчеты в GitLab Security Dashboard
- Блокировка деплоя при критичных уязвимостях (опционально)

### 🎨 Модульность

- Все CI/CD компоненты в `.gitlab/ci/`
- Легко переиспользовать в других проектах
- Параметризация через переменные

---

## 💡 Частые вопросы

### Q: Как пропустить CI для коммита?

```bash
git commit -m "Update docs [ci skip]"
```

### Q: Как посмотреть логи сборки?

1. Откройте **CI/CD → Pipelines**
2. Кликните на Pipeline
3. Кликните на нужный Job
4. Смотрите логи в реальном времени

### Q: Почему не собирается frontend?

Проверьте что вы изменили файлы в `react-frontend/` или `Dockerfile.frontend`

### Q: Как очистить кэш?

**CI/CD → Pipelines → Clear runner caches**

### Q: Образы не пушатся в registry?

Проверьте:
1. Container Registry включен в настройках проекта
2. `CI_REGISTRY_PASSWORD` правильный и **Protected**
3. Runner имеет доступ к интернету

---

## 🛠️ Локальное тестирование

### Проверка Dockerfile

```bash
# Backend
docker build -f Dockerfile.backend -t test-backend .

# Frontend
docker build -f Dockerfile.frontend --target production -t test-frontend .

# Proxy
docker build -f Dockerfile.proxy -t test-proxy .
```

### Запуск локально

```bash
docker-compose up -d
```

### Проверка образов

```bash
docker images | grep test-
```

---

## 📚 Дополнительная информация

### Для начинающих
- [CI_CD_SETUP.md](CI_CD_SETUP.md) - Полное руководство
- [PIPELINE_EXAMPLES.md](PIPELINE_EXAMPLES.md) - Примеры сценариев

### Для продвинутых
- [GITLAB_RUNNER_SETUP.md](GITLAB_RUNNER_SETUP.md) - Детальная настройка Runner
- [.gitlab/ci/README.md](.gitlab/ci/README.md) - Документация модулей

### Troubleshooting
- [GITLAB_RUNNER_SETUP.md - Troubleshooting](GITLAB_RUNNER_SETUP.md#troubleshooting)

---

## ✅ Чеклист первого запуска

- [ ] GitLab Runner установлен и запущен
- [ ] Runner зарегистрирован (тег: `docker`)
- [ ] Docker установлен на машине Runner
- [ ] `CI_REGISTRY_USER` добавлен в переменные
- [ ] `CI_REGISTRY_PASSWORD` добавлен (Protected)
- [ ] Container Registry включен
- [ ] Сделан тестовый push
- [ ] Pipeline запустился и прошел успешно
- [ ] Образы появились в Container Registry

---

## 🎉 Готово!

Ваш CI/CD пайплайн настроен и готов к работе!

**Следующие шаги:**
1. Добавьте деплой скрипты для вашего окружения
2. Настройте уведомления (Slack, Discord, Email)
3. Добавьте дополнительные проверки (linting, security scans)
4. Настройте автоматический rollback при ошибках

---

## 🆘 Нужна помощь?

1. Проверьте логи: **CI/CD → Pipelines → Failed Job**
2. Проверьте Runner: **Settings → CI/CD → Runners** (должен быть зеленый)
3. Проверьте переменные: **Settings → CI/CD → Variables**
4. Смотрите документацию в проекте
5. [GitLab CI/CD Docs](https://docs.gitlab.com/ee/ci/)

---

**Happy CI/CD! 🚀**
