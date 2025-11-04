# Развертывание проекта Skillbox-App

## Инфраструктура

- **GitLab CI**: `ci.avtostrada.kz`
- **Репозиторий**: `ci.avtostrada.kz:root/service`
- **Docker Registry**: Docker Hub (`maxpack2030/skillbox-app`)
- **Целевой сервер**: 'skillbox.avtostrada.kz'

## Компоненты

### 1. GitLab Runner
- Тип: Docker executor
- Runner ID: `2irHybG9z`
- Теги: `docker`
- Статус: Active

### 2. Pipeline структура
```
test → build → deploy
```

#### Stage 1: Test
- Запускает Go тесты в Docker контейнере
- Использует multi-stage build (target: test)
- Падает при ошибках в тестах

#### Stage 2: Build
- Собирает Docker образ приложения
- Тегирует как `maxpack2030/skillbox-app:uat`
- Пушит в Docker Hub
- Требует успешного прохождения тестов

#### Stage 3: Deploy
- Запускает Ansible playbook через Alpine контейнер
- Деплоит на целевой сервер через SSH
- Использует systemd для управления сервисом
- Требует успешной сборки

### 3. Секреты

Все секреты хранятся в GitLab CI/CD Variables (Settings → CI/CD → Variables):

- `DOCKER_HUB_USER` - username для Docker Hub
- `DOCKER_HUB_TOKEN` - access token (masked)
- `ANSIBLE_PRIVATE_KEY` - SSH ключ для деплоя (masked)
- `APP_SERVER_IP` - IP адрес целевого сервера

### 4. Ansible деплой

Playbook: `playbooks/deploy_service.yml`

Что делает:
1. Устанавливает Docker на целевом сервере
2. Создает systemd unit файл из template
3. Systemd автоматически:
   - Останавливает старый контейнер
   - Удаляет старый контейнер
   - Тянет новый образ
   - Запускает новый контейнер

Template: `templates/service.service.j2`

Сервис доступен на порту 80 целевого сервера.

## Структура проекта
```
.
├── .gitlab-ci.yml              # Конфигурация CI/CD
├── playbooks/
│   └── deploy_service.yml      # Ansible playbook для деплоя
├── templates/
│   └── service.service.j2      # Systemd unit template
└── skillbox-app/
    ├── Dockerfile              # Multi-stage Docker build
    ├── cmd/server/
    │   ├── app.go
    │   └── app_test.go         # Go тесты
    └── ansible/
        └── inventory/
            └── hosts.yml
```

## Запуск деплоя

Деплой запускается автоматически при push в ветку `uat`:

git push origin uat


## Проверка результата

После успешного деплоя:

# SSH на сервер
ssh ubuntu@34.186.24.52

# Проверка статуса сервиса
sudo systemctl status skillbox-app

# Проверка контейнера
docker ps | grep skillbox-app

# Проверка приложения
curl http://localhost:80


## Мониторинг

- Pipeline логи: GitLab CI/CD → Pipelines
- Логи сервиса: `sudo journalctl -u skillbox-app -f`
- Логи контейнера: `docker logs skillbox-app`
