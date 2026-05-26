# GitHub Actions

> **Теги:** #tools #ci-cd #конспект  

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Что такое GitHub Actions

> [!note] Определение
> **GitHub Actions** — CI/CD платформа встроенная в GitHub: автоматизация build, test, deploy по событиям в репозитории.

```
Repository
    └── .github/workflows/ci.yml
            └── Workflow
                    └── Job(s)
                            └── Step(s)
```

| Понятие | Описание |
|---------|----------|
| **Workflow** | YAML-файл автоматизации |
| **Event (trigger)** | push, pull_request, schedule, ... |
| **Job** | Набор steps на одном runner |
| **Step** | Команда или action |
| **Runner** | VM/контейнер выполнения (GitHub-hosted или self-hosted) |
| **Action** | Переиспользуемый блок (checkout, setup-java, ...) |

---

## 🔹 Базовый workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: gradle

      - name: Build and test
        run: ./gradlew build --no-daemon

      - name: Upload test report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/reports/tests/
```

---

## 🔹 Триггеры (on)

| Trigger | Когда |
|---------|-------|
| `push` | Push в branch |
| `pull_request` | PR opened/sync/reopened |
| `workflow_dispatch` | Ручной запуск |
| `schedule` | Cron `0 2 * * *` |
| `release` | Published release |
| `workflow_call` | Вызов из другого workflow |

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'build.gradle.kts'
    tags:
      - 'v*'
  pull_request:
    types: [opened, synchronize]
```

---

## 🔹 Jobs — параллельность и зависимости

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./gradlew test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./gradlew bootJar

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - run: echo "Deploying..."
```

**Matrix — несколько конфигураций:**

```yaml
strategy:
  matrix:
    java: [17, 21]
    os: [ubuntu-latest, windows-latest]
  fail-fast: false
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-java@v4
    with:
      java-version: ${{ matrix.java }}
```

---

## 🔹 Secrets и переменные

| Тип | Scope | Пример |
|-----|-------|--------|
| **Secrets** | Encrypted | `DB_PASSWORD`, `DOCKER_TOKEN` |
| **Variables** | Plain (repo/org) | `APP_VERSION` |
| **Environment secrets** | Per environment (prod/staging) | deploy keys |

```yaml
steps:
  - name: Deploy
    env:
      DB_URL: ${{ secrets.DATABASE_URL }}
      VERSION: ${{ vars.APP_VERSION }}
    run: ./deploy.sh
```

> [!warning] Secrets в логах
> GitHub маскирует secrets, но **не логируй** их явно (`echo $SECRET`).

---

## 🔹 Кэширование Gradle

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: gradle   # кэш ~/.gradle/caches
```

Или явно:

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
```

---

## 🔹 Docker build + push

```yaml
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## 🔹 Spring Boot CI — типичный pipeline

```
push / PR
    ↓
checkout → setup-java (cache gradle)
    ↓
./gradlew test                    ← unit + integration
    ↓
./gradlew bootJar                 ← artifact
    ↓
docker build & push (main only)
    ↓
deploy to k8s / ECS (environment: production)
```

**С Testcontainers в CI:**

```yaml
services:
  docker:
    image: docker:dind

steps:
  - run: ./gradlew test
    env:
      TESTCONTAINERS_RYUK_DISABLED: false
```

---

## 🔹 Permissions и безопасность

```yaml
permissions:
  contents: read
  packages: write

jobs:
  build:
    permissions:
      contents: read
```

> [!tip] Principle of least privilege
> `GITHUB_TOKEN` — минимальные permissions в workflow.

---

## 🔹 Итог

1. Workflow = trigger + jobs + steps в `.github/workflows/`.
2. `setup-java` + `cache: gradle` — быстрый Java CI.
3. Secrets — через `${{ secrets.NAME }}`, не в коде.
4. `needs` — порядок jobs; `matrix` — multi-Java/OS.
5. `environment: production` — approval gates для deploy.

```
Шпаргалка GitHub Actions:
─────────────────────────────────────────
.github/workflows/*.yml
on: push / pull_request / schedule
jobs.<name>.runs-on: ubuntu-latest
actions/checkout@v4
actions/setup-java@v4 + cache: gradle
./gradlew build
needs: [test]                      → job deps
secrets.NAME / vars.NAME
if: github.ref == 'refs/heads/main'
environment: production
```
