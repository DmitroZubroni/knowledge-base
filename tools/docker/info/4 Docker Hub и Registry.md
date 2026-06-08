# Docker  Docker Hub и Registry

> **Теги:** #docker #registry #конспект  

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

##  Теория: Что такое Registry

**Docker Registry** — хранилище Docker образов. Как GitHub, только для образов.

```
Разработчик              Registry              Сервер
    │                       │                     │
    ├── docker build ──────►│                     │
    │   (собрать образ)     │                     │
    │                       │                     │
    ├── docker push ───────►│                     │
    │   (загрузить)         │◄── docker pull ─────┤
    │                       │    (скачать)        │
    └───────────────────────┘                     │
                                            docker run
                                          (запустить конт.)
```

---

## 🌐 Docker Hub

**Docker Hub** (`hub.docker.com`) — официальный публичный registry от Docker.

### Что там есть

```
Docker Hub
├── Official Images        ← проверены Docker Inc.
│   ├── nginx, postgres, redis, node, python…
│   └── Синтаксис: just nginx (без префикса)
│
├── Verified Publishers    ← от проверенных компаний
│   └── microsoft/dotnet, elastic/elasticsearch…
│
└── Community Images       ← от обычных пользователей
    └── Синтаксис: username/imagename
```

### Формат имени образа

```
[registry/][username/]image[:tag]

nginx                      # официальный, latest
nginx:1.25                 # официальный, версия 1.25
nginx:alpine               # официальный, alpine-вариант
john/myapp                 # пользовательский
john/myapp:2.0             # пользовательский с тегом
ghcr.io/john/myapp:latest  # другой registry (GitHub)
```

---

## 🔑 Аутентификация

```bash
docker login                           # войти в Docker Hub
docker login -u username               # с именем пользователя
docker login -u user -p pass           # ⚠️ небезопасно (пароль в истории)
docker login ghcr.io                   # войти в GitHub Container Registry
docker logout                          # выйти
```

---

## 📥 Скачивание образов (pull)

```bash
docker pull nginx                      # последняя версия (latest)
docker pull nginx:1.25                 # конкретная версия
docker pull nginx:1.25-alpine          # версия + вариант
docker pull postgres:16                # PostgreSQL 16
docker pull ubuntu:22.04               # Ubuntu 22.04
docker pull python:3.11-slim           # Python 3.11 slim
```

### Популярные официальные образы

| Образ | Теги | Описание |
|---|---|---|
| `nginx` | `latest`, `alpine`, `1.25` | Веб-сервер |
| `postgres` | `16`, `15-alpine` | PostgreSQL |
| `mysql` | `8.0`, `8-debian` | MySQL |
| `redis` | `latest`, `alpine`, `7` | Redis кэш |
| `mongo` | `latest`, `6` | MongoDB |
| `node` | `20`, `20-alpine`, `lts` | Node.js |
| `python` | `3.11`, `3.11-slim`, `3.11-alpine` | Python |
| `openjdk` | `17`, `17-slim`, `21-alpine` | Java JDK |
| `golang` | `1.21`, `1.21-alpine` | Go |
| `ubuntu` | `22.04`, `20.04` | Ubuntu |
| `alpine` | `3.18`, `latest` | Alpine Linux |
| `nginx` | `alpine` | Nginx на Alpine |

---

## 📤 Публикация образа (push)

### Шаг 1: Создать образ

```bash
docker build -t myapp .
```

### Шаг 2: Присвоить правильное имя (тег)

Имя должно быть в формате `username/imagename:tag`:

```bash
docker tag myapp username/myapp:1.0
docker tag myapp username/myapp:latest

# Или сразу при сборке
docker build -t username/myapp:1.0 .
```

### Шаг 3: Войти в Docker Hub

```bash
docker login
```

### Шаг 4: Загрузить образ

```bash
docker push username/myapp:1.0
docker push username/myapp:latest
```

### Полный пример

```bash
# 1. Сборка
docker build -t phrases .

# 2. Тегирование
docker tag phrases john/phrases:1.0
docker tag phrases john/phrases:latest

# 3. Логин
docker login

# 4. Пуш
docker push john/phrases:1.0
docker push john/phrases:latest

# 5. Проверка: зайти на hub.docker.com → ваш профиль → репозитории
```

---

## 🏷️ Теги (Tags)

Тег — это версия образа. Если не указан — используется `latest`.

```
nginx:latest    → последняя стабильная версия
nginx:1.25      → точная версия
nginx:alpine    → вариант на Alpine
nginx:1.25-alpine → версия + вариант
```

> [!WARNING] latest — не всегда безопасно
> В продакшене всегда указывайте конкретную версию (`nginx:1.25.3`), а не `latest` — следующий pull может сломать совместимость.

### Стратегия тегирования

```bash
docker tag myapp user/myapp:1.0.0    # семантическая версия
docker tag myapp user/myapp:1.0      # минорная версия
docker tag myapp user/myapp:1        # мажорная версия
docker tag myapp user/myapp:latest   # всегда последняя
```

---

##  Private Registry

Иногда нужно своё хранилище (код не должен быть публичным):

### Docker Hub Private Repos
- Бесплатный аккаунт: 1 приватный репозиторий
- Платный: неограниченно

### Self-hosted Registry

```bash
# Запустить свой registry локально
docker run -d -p 5000:5000 --name registry registry:2

# Загрузить образ в свой registry
docker tag myapp localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# Скачать из своего registry
docker pull localhost:5000/myapp:1.0
```

### Популярные облачные Registry

| Registry | URL | Описание |
|---|---|---|
| Docker Hub | `hub.docker.com` | Официальный, самый популярный |
| GitHub Registry | `ghcr.io` | Интеграция с GitHub Actions |
| AWS ECR | `*.ecr.amazonaws.com` | Для AWS |
| Google GCR | `gcr.io` | Для Google Cloud |
| Azure ACR | `*.azurecr.io` | Для Azure |
| GitLab Registry | `registry.gitlab.com` | Интеграция с GitLab CI |

---

##  Поиск образов

### Через CLI

```bash
docker search nginx                    # найти образы nginx
docker search --limit 5 postgres       # ограничить результаты
docker search --filter is-official=true nginx  # только официальные
docker search --filter stars=100 node  # с рейтингом 100+
```

### Вывод `docker search`

```
NAME                DESCRIPTION                     STARS  OFFICIAL
nginx               Official build of Nginx         18000  [OK]
unit                Official build of NGINX Unit    40     [OK]
bitnami/nginx       Bitnami nginx Docker Image      160
```

---

##  Управление репозиторием на Docker Hub

### Через сайт hub.docker.com

| Раздел | Возможности |
|---|---|
| **Repositories** | Создать, удалить, настроить публичность |
| **Tags** | Просмотр всех тегов, скачать конкретный |
| **Overview** | README для образа (Markdown) |
| **Settings** | Webhook, автосборка |
| **Collaborators** | Совместный доступ |

---

##  Автоматизация: GitHub Actions → Docker Hub

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: username/myapp:latest
```

---

##  Схема: полный цикл работы с образами

```
[Локальная разработка]
         │
    docker build
         │
         ▼
    [Local Image]
         │
    docker tag
         │
         ▼
    [Tagged Image] ──── docker push ────► [Docker Hub]
                                               │
                                          docker pull
                                               │
                                               ▼
                                       [Production Server]
                                               │
                                          docker run
                                               │
                                               ▼
                                         [Container 🚀]
```

