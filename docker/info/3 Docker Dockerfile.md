# 📄 Docker  Dockerfile

> **Теги:** #docker #dockerfile #build #image  
> **Статус:** #изучено

---

##  Теория: Что такое Dockerfile

**Dockerfile** — текстовый файл с инструкциями, по которым Docker собирает образ. Это "рецепт" образа.

```
Разработчик пишет     Docker читает      Результат
  Dockerfile     →   инструкции    →    Image (образ)
```

### Правила файла
- Называется строго `Dockerfile` (с большой буквы, без расширения)
- Хранится в корне проекта
- Каждая инструкция создаёт новый **слой** в образе
- Слои **кэшируются** — Docker пересобирает только изменённые слои

---

##  Все инструкции Dockerfile

### `FROM` — базовый образ

```dockerfile
FROM ubuntu:22.04
FROM node:20-alpine
FROM openjdk:17-slim
FROM scratch          # пустой образ (для компилируемых языков)
```

> [!INFO] Всегда первая строка
> `FROM` обязателен и идёт первым. Задаёт базу, на которой строится ваш образ.

**Суффиксы образов:**

| Суффикс | Описание |
|---|---|
| (без суффикса) | Полный образ на Debian/Ubuntu |
| `-alpine` | Минималистичный Alpine Linux (~5MB) |
| `-slim` | Облегчённый вариант |
| `-bookworm` / `-bullseye` | Версия Debian |

---

### `WORKDIR` — рабочая директория

```dockerfile
WORKDIR /app
```

- Создаёт директорию, если она не существует
- Все последующие команды (`RUN`, `COPY`, `CMD`) выполняются внутри неё
- Лучше, чем `RUN mkdir && cd`

```dockerfile
WORKDIR /app
COPY . .         # копирует в /app
RUN ls           # выводит содержимое /app
```

---

### `COPY` — копировать файлы с хоста в образ

```dockerfile
COPY src dest

COPY app.jar /app/app.jar         # конкретный файл
COPY . .                          # всё из текущей папки в WORKDIR
COPY package*.json ./             # все файлы по паттерну
COPY --chown=node:node . .        # с указанием владельца
```

---

### `ADD` — расширенное копирование

```dockerfile
ADD archive.tar.gz /app/          # автоматически распакует архив
ADD https://example.com/file /app/ # скачает файл из URL
```

> [!NOTE] COPY vs ADD
> Используйте `COPY` для обычных файлов — он проще и понятнее.
> `ADD` только когда нужно распаковать архив или скачать файл по URL.

---

### `RUN` — выполнить команду при сборке образа

```dockerfile
RUN apt-get update && apt-get install -y curl
RUN npm install
RUN go build -o app .
RUN chmod +x /app/start.sh
```

> [!TIP] Объединяйте RUN командой &&
> Каждый `RUN` — это новый слой. Объединяйте связанные команды через `&&` и `\`, чтобы уменьшить размер образа.

```dockerfile
# ❌ Плохо — три слоя вместо одного
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ Хорошо — один слой
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

---

### `CMD` — команда запуска по умолчанию

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
CMD ["java", "-jar", "/app/app.jar"]
CMD ["node", "server.js"]
CMD ["python", "app.py"]
```

> [!NOTE] Особенности CMD
> - Выполняется когда запускается контейнер (`docker run`)
> - **Можно переопределить** при запуске: `docker run image другая_команда`
> - В Dockerfile должна быть **одна** `CMD` (последняя побеждает)

---

### `ENTRYPOINT` — точка входа контейнера

```dockerfile
ENTRYPOINT ["java", "-jar"]
CMD ["/app/app.jar"]          # аргумент по умолчанию для ENTRYPOINT
```

```bash
docker run myapp                    # java -jar /app/app.jar
docker run myapp /app/other.jar     # java -jar /app/other.jar
```

### CMD vs ENTRYPOINT

| | `CMD` | `ENTRYPOINT` |
|---|---|---|
| Можно переопределить | ✅ Да | Только с `--entrypoint` |
| Аргумент по умолчанию | Для ENTRYPOINT | — |
| Типичное использование | Команда по умолчанию | Фиксированная точка входа |

```dockerfile
# Паттерн: ENTRYPOINT + CMD
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0"]     # аргументы по умолчанию

# docker run myapp → python app.py --host 0.0.0.0
# docker run myapp --port 8080 → python app.py --port 8080
```

---

### `ENV` — переменные окружения

```dockerfile
ENV NODE_ENV=production
ENV DB_HOST=localhost DB_PORT=5432
ENV APP_VERSION=1.0.0
```

Доступны внутри контейнера и при сборке. Можно переопределить при запуске:

```bash
docker run -e NODE_ENV=development myapp
```

---

### `ARG` — аргументы сборки

```dockerfile
ARG VERSION=latest
FROM node:${VERSION}

ARG BUILD_DATE
RUN echo "Built on: $BUILD_DATE"
```

```bash
docker build --build-arg VERSION=20 --build-arg BUILD_DATE=$(date) .
```

> [!NOTE] ARG vs ENV
> `ARG` — только на время сборки (не попадает в контейнер).  
> `ENV` — доступна и при сборке, и в работающем контейнере.

---

### `EXPOSE` — объявить порт

```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 8080/tcp
EXPOSE 5353/udp
```

> [!NOTE] EXPOSE — это документация, не публикация
> `EXPOSE` не открывает порт автоматически. Нужен флаг `-p` при `docker run` или `ports:` в Compose.

---

### `VOLUME` — объявить точку монтирования

```dockerfile
VOLUME ["/var/lib/mysql"]
VOLUME /data /logs
```

Создаёт анонимный том при запуске контейнера (если не указан явно).

---

### `USER` — переключить пользователя

```dockerfile
RUN useradd -m appuser
USER appuser
```

> [!TIP] Безопасность
> Не запускайте приложения от `root`. Создайте отдельного пользователя через `RUN useradd` и переключитесь на него через `USER`.

---

### `LABEL` — метаданные образа

```dockerfile
LABEL maintainer="your@email.com"
LABEL version="1.0"
LABEL description="My awesome app"
```

---

### `.dockerignore` — исключить файлы из контекста

Создаётся рядом с `Dockerfile`:

```
node_modules/
.git/
.env
*.log
__pycache__/
.DS_Store
dist/
build/
```

> [!TIP] Всегда создавайте .dockerignore
> Без него Docker отправит в контекст сборки всю папку, включая `node_modules`. Это замедляет сборку.

---

##  Многоэтапная сборка (Multi-stage build)

Позволяет использовать один Dockerfile для сборки и создания финального лёгкого образа.

```dockerfile
# ─── Этап 1: сборка ───────────────────────────────
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# ─── Этап 2: финальный образ ──────────────────────
FROM alpine:3.18
WORKDIR /app
COPY --from=builder /app/myapp .   # берём только бинарник из этапа 1
EXPOSE 8080
CMD ["./myapp"]
```

**Результат:**
- Этап сборки: `~800MB` (golang + инструменты)
- Финальный образ: `~10MB` (alpine + бинарник)

---

##  Примеры Dockerfile по языкам

### Node.js

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

### Java (Spring Boot)

```dockerfile
FROM openjdk:17-slim
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Python

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### Nginx со своим HTML

```dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
EXPOSE 80
```

### Go (многоэтапная сборка)

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o main .

FROM alpine:3.18
RUN adduser -D appuser
USER appuser
COPY --from=builder /app/main /app/main
EXPOSE 8080
CMD ["/app/main"]
```

---

##  Схема: порядок инструкций и кэширование

```
┌──────────────────────────────────────────┐
│              Dockerfile                  │
│                                          │
│  FROM node:20        ← меняется редко   │
│  WORKDIR /app        ← меняется редко   │
│  COPY package.json . ← меняется редко   │
│  RUN npm install     ← КЭШИРУЕТСЯ       │
│  COPY . .            ← меняется часто   │  ← Ставьте это ПОЗЖЕ
│  CMD ["node", "app"] ← меняется редко   │
└──────────────────────────────────────────┘
```

> [!TIP] Оптимизация кэша
> Располагайте инструкции от редко меняемых к часто. Файлы зависимостей (`package.json`, `requirements.txt`, `pom.xml`) копируйте **до** `RUN install`, а код — **после**.

---

##  Шпаргалка инструкций

| Инструкция | Когда выполняется | Назначение |
|---|---|---|
| `FROM` | Сборка | Базовый образ |
| `WORKDIR` | Сборка | Рабочая директория |
| `COPY` | Сборка | Скопировать файлы |
| `ADD` | Сборка | Копировать + распаковать |
| `RUN` | Сборка | Выполнить команду |
| `ENV` | Сборка + запуск | Переменная окружения |
| `ARG` | Только сборка | Аргумент сборки |
| `EXPOSE` | Документация | Объявить порт |
| `VOLUME` | Запуск | Точка монтирования |
| `USER` | Сборка + запуск | Переключить пользователя |
| `LABEL` | Метаданные | Метаданные образа |
| `CMD` | Запуск | Команда по умолчанию |
| `ENTRYPOINT` | Запуск | Точка входа |

