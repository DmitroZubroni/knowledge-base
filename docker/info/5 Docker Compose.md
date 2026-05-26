# 🎼 Docker Docker Compose и YAML

> **Теги:** #docker #compose #конспект  

> [!abstract] Связи
> [[main]] | [[main Docker]]

---

##  Теория: Зачем нужен Docker Compose

Представьте приложение из нескольких частей:
- Backend (Node.js)
- Frontend (React + Nginx)
- База данных (PostgreSQL)
- Кэш (Redis)

Без Compose нужно запустить **4 команды** с десятками флагов и не забыть про порты, сети, тома, переменные окружения. Compose решает это одним файлом `docker-compose.yml` и одной командой.

```bash
# ❌ Без Compose — вручную:
docker network create app-net
docker volume create pg-data
docker run -d --name postgres --network app-net -v pg-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:16
docker run -d --name redis --network app-net redis:alpine
docker run -d --name backend --network app-net -p 3000:3000 -e DB_HOST=postgres myapp-backend
docker run -d --name frontend --network app-net -p 80:80 myapp-frontend

# ✅ С Compose:
docker compose up -d
```

---

##  Структура docker-compose.yml

```yaml
version: "3.9"           # версия формата (можно опустить в новых версиях)

services:                # список сервисов (контейнеров)
  servicename:           # имя сервиса (произвольное)
    ...                  # настройки сервиса

volumes:                 # именованные тома
  ...

networks:                # сети
  ...
```

---

##  Ключи сервиса: полный справочник

### `image` — использовать готовый образ

```yaml
services:
  db:
    image: postgres:16
  cache:
    image: redis:alpine
  web:
    image: nginx:1.25
```

---

### `build` — собрать образ из Dockerfile

```yaml
services:
  app:
    build: .                         # Dockerfile в текущей папке

  app2:
    build:
      context: ./backend             # папка с Dockerfile
      dockerfile: Dockerfile.prod    # нестандартное имя файла
      args:
        VERSION: 1.0
```

---

### `ports` — проброс портов

```yaml
services:
  nginx:
    image: nginx
    ports:
      - "80:80"           # хост:контейнер
      - "443:443"
      - "8080:80"         # порт 80 контейнера → 8080 хоста
      - "127.0.0.1:80:80" # только localhost
```

---

### `environment` — переменные окружения

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb

  # Или из .env файла:
  db2:
    image: postgres:16
    env_file:
      - .env
      - .env.production
```

---

### `volumes` — монтирование томов

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pg-data:/var/lib/postgresql/data    # именованный том
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # bind mount

  app:
    build: .
    volumes:
      - .:/app                  # код для разработки (live reload)
      - /app/node_modules       # исключить node_modules

volumes:
  pg-data:                      # объявить именованный том
```

---

### `networks` — сети

```yaml
services:
  app:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net             # БД только в бэкенд-сети

  nginx:
    networks:
      - frontend-net            # nginx только в фронтенд-сети

networks:
  frontend-net:
  backend-net:
```

> [!NOTE] Сеть по умолчанию
> Если `networks` не указаны, Compose автоматически создаёт сеть `projectname_default`, и все сервисы в ней. Они могут общаться по именам сервисов.

---

### `depends_on` — порядок запуска

```yaml
services:
  app:
    build: .
    depends_on:
      - db
      - redis

  db:
    image: postgres:16

  redis:
    image: redis:alpine
```

> [!WARNING] depends_on ≠ ожидание готовности
> `depends_on` запускает контейнеры в нужном порядке, но не ждёт, пока PostgreSQL будет готов принимать соединения. Для этого нужен `healthcheck`.

---

### `healthcheck` — проверка готовности

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  app:
    build: .
    depends_on:
      db:
        condition: service_healthy    # ждать до healthy
```

---

### `restart` — политика перезапуска

```yaml
services:
  app:
    restart: no               # никогда не перезапускать (по умолчанию)
  nginx:
    restart: always           # всегда перезапускать
  worker:
    restart: on-failure       # только при ошибке
  db:
    restart: unless-stopped   # всегда, кроме явной остановки
```

---

### `command` и `entrypoint` — переопределить команду

```yaml
services:
  app:
    image: python:3.11
    command: python app.py --debug    # переопределить CMD
    entrypoint: /app/start.sh         # переопределить ENTRYPOINT
```

---

### `expose` — открыть порт внутри сети (не наружу)

```yaml
services:
  db:
    image: postgres
    expose:
      - "5432"                 # доступен другим сервисам, но не хосту
```

---

### `container_name` — имя контейнера

```yaml
services:
  db:
    image: postgres:16
    container_name: my-postgres    # имя вместо автоматического
```

---

### `profiles` — запускать сервисы по условию

```yaml
services:
  app:
    build: .

  debug-tools:
    image: curlimages/curl
    profiles: ["debug"]           # запускать только с профилем debug
```

```bash
docker compose --profile debug up
```

---

##  Команды Docker Compose

```bash
docker compose up              # запустить все сервисы
docker compose up -d           # в фоне (detach)
docker compose up --build      # пересобрать образы перед запуском
docker compose up app db       # запустить только указанные сервисы

docker compose down            # остановить и удалить контейнеры и сети
docker compose down -v         # + удалить тома
docker compose down --rmi all  # + удалить образы

docker compose start           # запустить остановленные сервисы
docker compose stop            # остановить (без удаления)
docker compose restart         # перезапустить
docker compose restart app     # перезапустить конкретный сервис

docker compose ps              # статус сервисов
docker compose logs            # логи всех сервисов
docker compose logs -f app     # следить за логами сервиса
docker compose exec app bash   # войти в контейнер
docker compose run app sh      # запустить одноразовый контейнер

docker compose build           # пересобрать образы
docker compose pull            # обновить образы из registry
docker compose config          # проверить и вывести итоговый конфиг
```

---

##  Полный рабочий пример: Web App + PostgreSQL + Redis

```yaml
# docker-compose.yml
version: "3.9"

services:

  # ── База данных ──────────────────────────────────
  db:
    image: postgres:16-alpine
    container_name: app-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-appuser}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-appdb}
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"              # убрать в продакшене
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-appuser}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backend-net

  # ── Кэш ─────────────────────────────────────────
  cache:
    image: redis:7-alpine
    container_name: app-redis
    restart: unless-stopped
    ports:
      - "6379:6379"              # убрать в продакшене
    networks:
      - backend-net

  # ── Бэкенд ──────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: app-backend
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: ${DB_USER:-appuser}
      DB_PASSWORD: ${DB_PASSWORD:-secret}
      DB_NAME: ${DB_NAME:-appdb}
      REDIS_HOST: cache
      REDIS_PORT: 6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - backend-net
      - frontend-net

  # ── Фронтенд / Nginx ─────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: app-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./frontend/dist:/usr/share/nginx/html:ro
      - ./ssl:/etc/ssl:ro
    depends_on:
      - backend
    networks:
      - frontend-net

volumes:
  pg-data:
    driver: local

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
```

---

##  Пример .env файла

```env
# .env
DB_USER=appuser
DB_PASSWORD=supersecret
DB_NAME=appdb
NODE_ENV=production
```

> [!WARNING] .env в .gitignore!
> Никогда не коммитьте `.env` с паролями в репозиторий.

```bash
# .gitignore
.env
.env.local
.env.production
```

---

##  Пример для разработки (с live reload)

```yaml
# docker-compose.dev.yml
version: "3.9"

services:
  app:
    build:
      context: .
      target: development       # стадия сборки в multi-stage Dockerfile
    volumes:
      - .:/app                  # монтировать код для live reload
      - /app/node_modules       # не перезаписывать node_modules
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"
    command: npm run dev        # команда с hot-reload
```

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

---

##  Схема: взаимодействие сервисов

```
        Internet
            │
       ┌────▼────┐
       │  nginx  │ :80, :443
       └────┬────┘
  frontend-net│
       ┌────▼────┐
       │ backend │ :3000
       └────┬────┘
  backend-net│
      ┌──────┴──────┐
┌─────▼─────┐  ┌────▼────┐
│ postgres  │  │  redis  │
│   :5432   │  │  :6379  │
└─────┬─────┘  └─────────┘
      │
   [Volume]
  pg-data
```

