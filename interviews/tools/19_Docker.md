# Docker

> **Теги:** #interviews #tools #docker #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Image vs Container

### Image (Образ)

**Image** — read-only шаблон для создания контейнеров.

```dockerfile
# Dockerfile — инструкция для создания image
FROM openjdk:17-jdk-slim
COPY target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Характеристики:**
- Read-only
- Состоит из слоёв
- Хранится в Docker Registry (Docker Hub)

### Container (Контейнер)

**Container** — запущенный экземпляр image.

```bash
# Запуск контейнера из image
docker run -p 8080:8080 my-app:1.0
```

**Характеристики:**
- Read-write слой поверх image
- Изолированное окружение
- Может быть остановлен, запущен, удалён

### Сравнение

| Характеристика | Image | Container |
|----------------|-------|-----------|
| Чтение/запись | Read-only | Read-write |
| Состояние | Неизменяемый | Может меняться |
| Запуск | Нет | Да |
| Слои | Только image слои | Image слои + RW слой |

---

## 🔹 Слои и кэш

### Docker слои

Каждая инструкция в Dockerfile создаёт новый слой.

```dockerfile
FROM openjdk:17-jdk-slim       # Layer 1
WORKDIR /app                   # Layer 2
COPY pom.xml .                 # Layer 3
RUN mvn dependency:go-offline # Layer 4
COPY src ./src                 # Layer 5
RUN mvn package                # Layer 6
COPY target/app.jar app.jar    # Layer 7
ENTRYPOINT ["java", "-jar", "app.jar"] # Layer 8
```

### Кэширование слоёв

Docker кэширует слои. Если слой не изменился — используется кэш.

```dockerfile
# ПЛОХО — каждый раз пересобирается
COPY . .
RUN mvn package

# ХОРОШО — кэшируется если pom.xml не изменился
COPY pom.xml .
RUN mvn dependency:go-offline  # кэшируется
COPY src ./src                # пересобирается только если src изменился
RUN mvn package
```

### Layer ordering (порядок слоёв)

Правило: часто меняющиеся инструкции — в конце.

```dockerfile
# ХОРОШО
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline  # редко меняется
COPY src ./src                 # часто меняется
RUN mvn package                # часто меняется
```

> [!tip] Порядок важен
- Сначала редко меняющиеся файлы (pom.xml, requirements.txt)
- Потом часто меняющиеся (src, код)
- В конце — ENTRYPOINT/CMD

---

## 🔹 COPY vs ADD

### COPY

```dockerfile
COPY src /app/src
COPY file.txt /app/
```

**Характеристики:**
- Копирует файлы/директории из host в container
- Простой и предсказуемый
- Рекомендуется для обычного копирования

### ADD

```dockerfile
ADD src /app/src
ADD archive.tar.gz /app/      # автоматически распаковывает
ADD http://example.com/file /app/  # может скачивать по URL
```

**Характеристики:**
- Копирует файлы/директории
- Автоматически распаковывает tar/zip
- Может скачивать по URL
- Рекомендуется только для распаковки/URL

### Сравнение

| Характеристика | COPY | ADD |
|----------------|------|-----|
| Копирование | ✅ | ✅ |
| Распаковка tar/zip | ❌ | ✅ |
| Скачивание по URL | ❌ | ✅ |
| Рекомендация | ✅ Для обычного копирования | ✅ Для распаковки/URL |

> [!tip] Используйте COPY по умолчанию, ADD только для распаковки/URL

---

## 🔹 Volume

**Volume** — хранилище данных, независимое от контейнера.

### Типы volumes

#### 1. Named volume

```bash
# Создание volume
docker volume create my-data

# Использование
docker run -v my-data:/app/data my-app
```

#### 2. Bind mount

```bash
# Монтирование директории host в container
docker run -v /host/path:/container/path my-app
```

#### 3. Anonymous volume

```bash
# Автоматически создаётся, удаляется с контейнером
docker run -v /container/path my-app
```

### Примеры

```dockerfile
# Dockerfile
VOLUME /app/data  # объявляет volume (но не создаёт)
```

```bash
# Запуск с volume
docker run -v my-data:/app/data my-app

# Запуск с bind mount
docker run -v $(pwd)/data:/app/data my-app
```

### Когда использовать

| Тип | Когда использовать |
|-----|-------------------|
| **Named volume** | Постоянное хранение данных |
| **Bind mount** | Разработка (hot reload), конфиги |
| **Anonymous volume** | Временные данные |

---

## 🔹 Dockerfile best practices

### 1. Использовать .dockerignore

```dockerignore
# .dockerignore
target/
*.log
.git/
node_modules/
```

### 2. Минимизировать слои

```dockerfile
# ПЛОХО — много слоёв
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# ХОРОШО — один слой
RUN apt-get update && \
    apt-get install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Использовать multi-stage builds

```dockerfile
# Build stage
FROM maven:3.8-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Runtime stage
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Преимущества:**
- Меньший размер final image
- Без build инструментов в runtime

### 4. Не запускать от root

```dockerfile
# ПЛОХО
USER root

# ХОРОШО
RUN adduser -u 1000 appuser
USER appuser
```

### 5. Использовать конкретные теги

```dockerfile
# ПЛОХО — может измениться
FROM openjdk:17

# ХОРОШО — конкретная версия
FROM openjdk:17-jdk-slim
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Image** — read-only шаблон, **Container** — запущенный экземпляр
> - **Слои:** каждая инструкция создаёт слой, кэшируется если не изменился
> - **Layer ordering:** редко меняющиеся — в начале, часто меняющиеся — в конце
> - **COPY** — обычное копирование, **ADD** — распаковка/URL
> - **Volume:** Named (постоянное), Bind mount (разработка), Anonymous (временное)
> - **Best practices:** .dockerignore, минимизация слоёв, multi-stage builds, не root

```
Image vs Container:
Image → read-only, шаблон
Container → read-write, запущенный экземпляр

Слои и кэш:
Каждая инструкция → новый слой
Кэш → если слой не изменился
Layer ordering → редко меняющиеся в начале

COPY vs ADD:
COPY → обычное копирование
ADD → распаковка tar/zip, скачивание по URL

Volume:
Named volume → постоянное хранение
Bind mount → разработка, конфиги
Anonymous → временные данные

Best practices:
.dockerignore → исключить ненужное
Минимизация слоёв → && в одной RUN
Multi-stage builds → меньше размер
Не root → безопасность
```
