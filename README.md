# Momo Store — Docker

Контейнеризация приложения Momo Store (Go backend + Vue.js frontend).

## Быстрый старт

```bash
# Сборка и запуск (фронтенд: http://localhost, API: http://localhost:8081)
docker compose up --build -d

# Проверка
curl http://localhost:8081/health
curl http://localhost/momo-store/
```

Остановка:

```bash
docker compose down
```

## Профили и окружения

| Режим | Команда | Описание |
|-------|---------|----------|
| По умолчанию | `docker compose up --build` | Прямой доступ к сервисам, hardened-настройки |
| Разработка | `docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build` | Смягчённые ограничения безопасности |
| Продакшен + LB | `docker compose -f docker-compose.yml -f docker-compose.prod.yml up --build -d --scale backend=3 --scale frontend=2` | Балансировка через nginx, горизонтальное масштабирование |

## Порты

| Сервис | Порт |
|--------|------|
| Frontend | 80 |
| Backend API | 8081 |

## Конфигурация

### Переменные окружения (docker compose)

| Переменная | По умолчанию | Назначение |
|------------|--------------|------------|
| `VUE_APP_API_URL` | `http://localhost:8081` | URL API для сборки фронтенда |
| `DOCKER_USER` | `local` | Префикс тега образа |
| `IMAGE_TAG` | `latest` | Тег образа |

### Build-аргументы

**Frontend** (`frontend/Dockerfile`):

- `VUE_APP_API_URL` — базовый URL API (встраивается при `npm run build`).

### Docker Secrets

Секреты не хранятся в образах. Файл `secrets/api_secret.txt` монтируется через Docker Secrets и доступен контейнеру backend по пути `/run/secrets/api_secret` (переменная `API_SECRET_FILE`).

Для локального запуска используется placeholder-файл. В продакшене замените его на реальный секрет:

```bash
echo "your-secret-value" > secrets/api_secret.txt
```

## Оптимизация образов

Оба образа собираются **multi-stage**:

| Образ | Builder | Runtime | Оценка размера* |
|-------|---------|---------|-----------------|
| Backend | `golang:1.17-alpine` | `alpine:3.19` | ~30 MB |
| Frontend | `node:16-alpine` | `nginxinc/nginx-unprivileged:1.25-alpine` | ~77 MB |

\* Размер зависит от версий базовых образов. Проверка:

```bash
docker compose build
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep docker-project
```

**Принятые меры:**

- Лёгкие alpine-образы вместо полноразмерных дистрибутивов
- Кэширование зависимостей (`go mod download` / `npm ci` до копирования исходников)
- `-ldflags="-s -w"` и `-trimpath` для уменьшения Go-бинарника
- `.dockerignore` для исключения лишних файлов из контекста сборки
- Dev-зависимости не попадают в финальный образ фронтенда

## Безопасность

- Контейнеры работают от непривилегированных пользователей (`app` / `nginx`)
- `read_only: true` для файловой системы (кроме dev-профиля)
- `cap_drop: [ALL]`, `no-new-privileges: true`
- Лимиты CPU и памяти в `docker-compose.yml`
- Изолированные внутренние сети (`frontend-net`, `backend-net`)
- Секреты через Docker Secrets, не в образах
- Сканирование уязвимостей Trivy в CI и локально:

```bash
docker compose build
trivy image local/docker-project-backend:latest
trivy image local/docker-project-frontend:latest
```

## CI/CD

Workflow `.github/workflows/deploy.yaml`:

1. Сборка и публикация образов в Docker Hub
2. Сборка через Docker Compose
3. Сканирование образов Trivy (CRITICAL/HIGH)

## Структура файлов

```
backend/Dockerfile          — multi-stage сборка Go API
frontend/Dockerfile         — multi-stage сборка Vue + nginx
docker-compose.yml          — основная оркестрация
docker-compose.dev.yml      — профиль разработки
docker-compose.prod.yml     — балансировка и масштабирование
nginx/                      — конфиги load balancer
secrets/api_secret.txt      — placeholder для Docker Secrets
```
