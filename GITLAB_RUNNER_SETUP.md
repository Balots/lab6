# Установка и настройка GitLab Runner

Этот документ содержит подробные инструкции по установке и настройке GitLab Runner для работы с CI/CD пайплайном.

## Содержание

1. [Что такое GitLab Runner](#что-такое-gitlab-runner)
2. [Типы установки](#типы-установки)
3. [Установка на Linux](#установка-на-linux)
4. [Установка на Windows](#установка-на-windows)
5. [Установка через Docker](#установка-через-docker)
6. [Регистрация Runner](#регистрация-runner)
7. [Настройка Docker Executor](#настройка-docker-executor)
8. [Проверка работы](#проверка-работы)
9. [Troubleshooting](#troubleshooting)

---

## Что такое GitLab Runner

GitLab Runner — это агент, который выполняет задачи CI/CD пайплайна. Он получает задания от GitLab сервера и выполняет их в изолированных средах (контейнерах, виртуальных машинах или на хост-системе).

**Основные типы Executor:**
- **Docker** - выполнение в Docker контейнерах (рекомендуется)
- **Shell** - выполнение на хост-системе
- **Kubernetes** - выполнение в Kubernetes кластере
- **Docker Machine** - динамическое создание Docker хостов

---

## Типы установки

### Рекомендации по выбору:

| Сценарий | Рекомендуемый метод |
|----------|---------------------|
| Production окружение | Linux + systemd service |
| Development/Testing | Docker |
| Windows разработка | Windows service |
| Kubernetes кластер | Helm chart |

---

## Установка на Linux

### Ubuntu/Debian

```bash
# 1. Добавляем официальный репозиторий GitLab
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# 2. Устанавливаем GitLab Runner
sudo apt-get install gitlab-runner

# 3. Проверяем статус
sudo gitlab-runner status

# 4. Проверяем версию
gitlab-runner --version
```

### CentOS/RHEL/Fedora

```bash
# 1. Добавляем официальный репозиторий GitLab
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash

# 2. Устанавливаем GitLab Runner
sudo yum install gitlab-runner

# или для dnf:
sudo dnf install gitlab-runner

# 3. Проверяем статус
sudo gitlab-runner status
```

### Установка Docker (требуется для Docker executor)

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker gitlab-runner
sudo systemctl enable docker
sudo systemctl start docker

# CentOS/RHEL
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker gitlab-runner
```

---

## Установка на Windows

### Через официальный установщик

1. Скачайте установщик с [официального сайта](https://docs.gitlab.com/runner/install/windows.html)

2. Создайте директорию для Runner:
```powershell
New-Item -Path "C:\GitLab-Runner" -ItemType Directory
cd C:\GitLab-Runner
```

3. Скачайте бинарный файл:
```powershell
Invoke-WebRequest -Uri "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-windows-amd64.exe" -OutFile "gitlab-runner.exe"
```

4. Установите как Windows службу:
```powershell
.\gitlab-runner.exe install
.\gitlab-runner.exe start
```

5. Проверьте статус:
```powershell
.\gitlab-runner.exe status
```

### Установка Docker Desktop для Windows

1. Скачайте и установите [Docker Desktop](https://www.docker.com/products/docker-desktop)
2. Убедитесь, что включен Linux containers mode
3. В настройках Docker Desktop включите "Expose daemon on tcp://localhost:2375"

---

## Установка через Docker

Это самый простой способ для быстрого старта и тестирования.

### Docker Compose (Рекомендуется)

Создайте `docker-compose-runner.yml`:

```yaml
version: '3.8'

services:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab-runner-config:/etc/gitlab-runner
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock

volumes:
  gitlab-runner-config:
```

Запустите:
```bash
docker-compose -f docker-compose-runner.yml up -d
```

### Обычный Docker

```bash
docker run -d \
  --name gitlab-runner \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v gitlab-runner-config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```

### Проверка:

```bash
docker ps | grep gitlab-runner
docker logs gitlab-runner
```

---

## Регистрация Runner

После установки необходимо зарегистрировать Runner в GitLab.

### Получение регистрационного токена

1. Откройте ваш проект в GitLab
2. Перейдите в **Settings** → **CI/CD**
3. Раскройте секцию **Runners**
4. Скопируйте **Registration token**

### Интерактивная регистрация

```bash
# Для Linux/Mac
sudo gitlab-runner register

# Для Docker
docker exec -it gitlab-runner gitlab-runner register

# Для Windows (PowerShell с правами администратора)
cd C:\GitLab-Runner
.\gitlab-runner.exe register
```

### Параметры регистрации

Вам будут заданы следующие вопросы:

```
Enter the GitLab instance URL:
https://gitlab.com/

Enter the registration token:
[Вставьте ваш токен]

Enter a description for the runner:
docker-runner-1

Enter tags for the runner (comma-separated):
docker,build,linux

Enter optional maintenance note:
[можно оставить пустым]

Enter an executor:
docker

Enter the default Docker image:
alpine:latest
```

### Автоматическая регистрация (неинтерактивная)

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --registration-token "YOUR_REGISTRATION_TOKEN" \
  --executor "docker" \
  --docker-image alpine:latest \
  --description "docker-runner" \
  --tag-list "docker,build,linux" \
  --run-untagged="true" \
  --locked="false" \
  --docker-privileged="true" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/cache"
```

**Важные параметры:**

- `--docker-privileged="true"` - необходимо для Docker-in-Docker (сборка образов)
- `--docker-volumes "/var/run/docker.sock:/var/run/docker.sock"` - доступ к Docker daemon хоста
- `--run-untagged="true"` - Runner будет выполнять задачи без тегов
- `--tag-list "docker"` - теги должны совпадать с указанными в `.gitlab-ci.yml`

---

## Настройка Docker Executor

После регистрации необходимо настроить конфигурацию Runner.

### Расположение конфигурационного файла

- **Linux**: `/etc/gitlab-runner/config.toml`
- **Windows**: `C:\GitLab-Runner\config.toml`
- **Docker**: внутри контейнера `/etc/gitlab-runner/config.toml`

### Оптимальная конфигурация для нашего проекта

Отредактируйте `config.toml`:

```toml
concurrent = 4  # Количество одновременных задач
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner-1"
  url = "https://gitlab.com/"
  token = "YOUR_RUNNER_TOKEN"
  executor = "docker"

  [runners.custom_build_dir]

  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]

  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true  # Для Docker-in-Docker
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = [
      "/var/run/docker.sock:/var/run/docker.sock",
      "/cache"
    ]
    shm_size = 0

    # Ограничения ресурсов (опционально)
    cpus = "2"
    memory = "4g"
    memory_swap = "4g"

    # Политика pull образов
    pull_policy = ["if-not-present"]

    # Настройка сети
    network_mode = "bridge"
```

### Применение изменений

```bash
# Linux
sudo gitlab-runner restart

# Docker
docker restart gitlab-runner

# Windows
.\gitlab-runner.exe restart
```

---

## Проверка работы

### 1. Проверка статуса Runner

```bash
# Linux
sudo gitlab-runner verify

# Docker
docker exec -it gitlab-runner gitlab-runner verify

# Windows
.\gitlab-runner.exe verify
```

Ожидаемый вывод:
```
Verifying runner... is alive
```

### 2. Проверка в GitLab UI

1. Перейдите в **Settings** → **CI/CD** → **Runners**
2. Вы должны увидеть зеленый индикатор рядом с вашим Runner
3. Статус должен быть "online"

### 3. Тестовый пайплайн

Создайте простой тестовый файл `.gitlab-ci.yml`:

```yaml
test:
  stage: test
  image: alpine:latest
  script:
    - echo "Hello from GitLab Runner!"
    - docker --version
  tags:
    - docker
```

Закоммитьте и запушьте - пайплайн должен успешно выполниться.

---

## Настройка для нашего проекта

После базовой установки необходимо настроить для работы с нашим пайплайном:

### 1. Установите необходимые права

```bash
# Linux
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
```

### 2. Настройте CI/CD переменные в GitLab

Перейдите в **Settings** → **CI/CD** → **Variables** и добавьте:

| Ключ | Значение | Тип | Защищённая |
|------|----------|-----|------------|
| `CI_REGISTRY` | `registry.gitlab.com` | Variable | Нет |
| `CI_REGISTRY_USER` | `your-username` | Variable | Нет |
| `CI_REGISTRY_PASSWORD` | `your-token` | Variable | Да |
| `DOCKER_AUTH_CONFIG` | См. ниже | File | Да |

**Для `DOCKER_AUTH_CONFIG`** создайте файл:

```json
{
  "auths": {
    "registry.gitlab.com": {
      "auth": "base64-encoded-username:token"
    }
  }
}
```

Чтобы получить `base64-encoded-username:token`:
```bash
echo -n "username:token" | base64
```

### 3. Включите Container Registry

В настройках проекта: **Settings** → **General** → **Visibility, project features, permissions** → включите **Container Registry**.

---

## Troubleshooting

### Проблема: Runner не появляется в списке

**Решение:**
```bash
# Проверьте логи
sudo gitlab-runner --debug run

# Проверьте токен
cat /etc/gitlab-runner/config.toml

# Переrегистрируйте Runner
sudo gitlab-runner unregister --all-runners
sudo gitlab-runner register
```

### Проблема: Docker permission denied

**Решение:**
```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner

# Проверка
sudo -u gitlab-runner docker ps
```

### Проблема: Cannot connect to Docker daemon

**Решение:**
```bash
# Проверьте статус Docker
sudo systemctl status docker
sudo systemctl start docker

# Проверьте socket
ls -la /var/run/docker.sock

# Дайте права
sudo chmod 666 /var/run/docker.sock
```

### Проблема: Build fails with "image not found"

**Решение:**
- Проверьте, что Docker Hub доступен
- Используйте локальное зеркало
- В `config.toml` добавьте:
```toml
[runners.docker]
  pull_policy = ["if-not-present", "always"]
```

### Проблема: Out of disk space

**Решение:**
```bash
# Очистка Docker
docker system prune -a --volumes -f

# Автоматическая очистка в config.toml
[runners.docker]
  volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]

# Добавьте cron job для очистки
0 2 * * * docker system prune -f
```

### Проблема: Runner offline после перезагрузки

**Решение:**
```bash
# Включите автозапуск
sudo systemctl enable gitlab-runner

# Для Docker
docker update --restart=always gitlab-runner
```

---

## Расширенная настройка

### Несколько Runner на одном хосте

```bash
# Регистрируйте с разными именами и тегами
sudo gitlab-runner register --name "docker-build" --tag-list "docker,build"
sudo gitlab-runner register --name "docker-deploy" --tag-list "docker,deploy"
```

### Использование кэша S3

В `config.toml`:
```toml
[[runners]]
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AccessKey = "your-access-key"
      SecretKey = "your-secret-key"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "us-east-1"
```

### Мониторинг

```bash
# Prometheus metrics
# В config.toml:
listen_address = ":9252"

# Затем можно собирать метрики
curl http://localhost:9252/metrics
```

---

## Следующие шаги

После успешной установки GitLab Runner:

1. Прочитайте [CI_CD_SETUP.md](CI_CD_SETUP.md) для настройки пайплайна
2. Ознакомьтесь с `.gitlab-ci.yml` в корне проекта
3. Настройте переменные окружения в GitLab
4. Выполните первый тестовый запуск пайплайна

---

## Полезные ссылки

- [Официальная документация GitLab Runner](https://docs.gitlab.com/runner/)
- [Docker executor документация](https://docs.gitlab.com/runner/executors/docker.html)
- [GitLab CI/CD документация](https://docs.gitlab.com/ee/ci/)
- [Troubleshooting Guide](https://docs.gitlab.com/runner/faq/)
