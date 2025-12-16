# Развертывание GitLab CI/CD

## Предварительные требования

1. GitLab проект с этим кодом
2. GitLab Runner (Docker или локальный)
3. Docker Registry (GitLab Container Registry или свой)

## Установка GitLab Runner

### Вариант 1: Docker Runner

```bash
docker run -d --name gitlab-runner --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v gitlab-runner-config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```

Регистрация:
```bash
docker exec -it gitlab-runner gitlab-runner register \
  --url https://gitlab.com/ \
  --executor docker \
  --docker-image docker:24.0.7 \
  --docker-privileged \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

### Вариант 2: Локальная установка

Windows:
```powershell
# Скачать с https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-windows-amd64.exe
gitlab-runner.exe install
gitlab-runner.exe start
gitlab-runner.exe register
```

Linux:
```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner
sudo gitlab-runner register
```

## Настройка проекта

### 1. Создать отдельный репозиторий для CI-модулей

```bash
# Создать новый проект в GitLab, например: your-group/ci-modules
mkdir ci-modules-repo
cd ci-modules-repo
git init
cp ../lab5/ci-modules/multi-docker-build.yml .
git add multi-docker-build.yml
git commit -m "Add CI module"
git remote add origin git@gitlab.com:your-group/ci-modules.git
git push -u origin main
```

### 2. Обновить .gitlab-ci.yml

В основном проекте отредактировать [.gitlab-ci.yml](.gitlab-ci.yml):
```yaml
include:
  - project: 'your-group/ci-modules'  # Укажите путь к вашему репозиторию
    ref: main
    file: 'multi-docker-build.yml'
```

### 3. Настроить переменные в GitLab

Settings → CI/CD → Variables:

- `CI_REGISTRY_USER` - username для Docker Registry
- `CI_REGISTRY_PASSWORD` - password/token для Docker Registry (masked)
- `CI_REGISTRY` - адрес registry (по умолчанию: registry.gitlab.com)

Для GitLab Container Registry можно использовать встроенные токены:
- User: `gitlab-ci-token`
- Password: `$CI_JOB_TOKEN` (уже доступна автоматически)

### 4. Протестировать

Коммит в любую ветку:
```bash
# Изменить файл backend
echo "// test" >> spring-backend/src/main/java/com/example/demo/DemoApplication.java
git add .
git commit -m "Test backend build"
git push

# Проверить в GitLab: CI/CD → Pipelines
```

## Как работает

- **Сборка по изменениям**: каждый сервис собирается только если изменились его файлы
- **Кэширование**: используется `--cache-from` для ускорения сборки
- **Теги**: образы тегируются SHA коммита и `latest`
- **Универсальность**: модуль `.build-service` можно использовать в любом проекте

## Структура файлов

```
lab5/
├── .gitlab-ci.yml              # Основной CI файл проекта
├── spring-backend/             # Исходники → триггерит backend build
├── react-frontend/             # Исходники → триггерит frontend build
├── nginx/                      # Конфиг → триггерит proxy build
└── Dockerfile.{backend,frontend,proxy}

ci-modules/ (отдельный репозиторий)
└── multi-docker-build.yml      # Переиспользуемый модуль
```
