#  Docker — Команды

> **Теги:** #docker #commands #конспект  

> [!abstract] Связи
> [[main]] | [[main Docker]]

---

##  Структура команд Docker

```
docker  [управление]  [команда]  [флаги]  [аргументы]
docker     container    run       -d -p    nginx
```

> [!NOTE] Два синтаксиса
> Старый: `docker run`, `docker ps`, `docker images`  
> Новый: `docker container run`, `docker container ls`, `docker image ls`  
> Оба работают. Старый используется чаще.

---

##  Работа с контейнерами

### `docker run` — создать и запустить контейнер

```bash
docker run nginx                     # запустить контейнер из образа nginx
docker run ubuntu echo "hello"       # запустить с командой
docker run -it ubuntu bash           # интерактивный режим + псевдотерминал
docker run -d nginx                  # detach-режим (в фоне)
docker run -d -p 8080:80 nginx       # с проброской порта хост:контейнер
docker run -d --name my-nginx nginx  # с именем контейнера
docker run --rm nginx                # удалить контейнер после остановки
docker run -e MY_VAR=value nginx     # передать переменную окружения
docker run -v /host/path:/container/path nginx  # монтировать папку
docker run --network my-net nginx    # подключить к сети
```

### Флаги `docker run`

| Флаг | Полное название | Описание |
|---|---|---|
| `-d` | `--detach` | Запустить в фоне |
| `-it` | `--interactive --tty` | Интерактивный режим с терминалом |
| `-p` | `--publish` | Пробросить порт `хост:контейнер` |
| `--name` | | Задать имя контейнера |
| `--rm` | | Удалить контейнер после остановки |
| `-e` | `--env` | Переменная окружения |
| `-v` | `--volume` | Примонтировать том или папку |
| `--network` | | Подключить к сети |
| `--restart` | | Политика перезапуска (`always`, `unless-stopped`, `on-failure`) |

---

### Управление запущенными контейнерами

```bash
docker ps                    # список работающих контейнеров
docker ps -a                 # все контейнеры (включая остановленные)
docker ps -q                 # только ID контейнеров

docker stop <id/name>        # остановить контейнер (SIGTERM → SIGKILL)
docker start <id/name>       # запустить остановленный контейнер
docker restart <id/name>     # перезапустить контейнер
docker pause <id/name>       # приостановить (заморозить)
docker unpause <id/name>     # возобновить

docker rm <id/name>          # удалить остановленный контейнер
docker rm -f <id/name>       # принудительно удалить работающий
docker rm $(docker ps -aq)   # удалить ВСЕ остановленные контейнеры
```

---

### Взаимодействие с контейнером

```bash
docker exec -it <id/name> bash       # войти в работающий контейнер
docker exec -it <id/name> sh         # если bash нет, попробовать sh
docker exec <id/name> ls /app        # выполнить команду без входа

docker attach <id/name>              # присоединиться к контейнеру
# Ctrl+P → Ctrl+Q                    # отсоединиться без остановки
# exit или Ctrl+D                    # выйти (контейнер остановится)

docker logs <id/name>                # логи контейнера
docker logs -f <id/name>             # следить за логами
docker logs --tail 50 <id/name>      # последние 50 строк
docker logs --since 1h <id/name>     # логи за последний час
```

---

### Информация о контейнере

```bash
docker inspect <id/name>             # полная информация (JSON)
docker inspect <id> | grep IPAddress # IP-адрес контейнера
docker stats                         # потребление ресурсов (live)
docker stats <id/name>               # для конкретного контейнера
docker top <id/name>                 # процессы внутри контейнера
docker port <id/name>                # проброшенные порты
docker diff <id/name>                # изменения файловой системы
```

---

##  Работа с образами (Images)

```bash
docker images                        # список всех образов
docker image ls                      # то же самое (новый синтаксис)
docker images -a                     # включая промежуточные слои
docker images -q                     # только ID

docker pull nginx                    # скачать последний nginx
docker pull nginx:1.25               # конкретная версия (тег)
docker pull ubuntu:22.04             # Ubuntu 22.04

docker rmi nginx                     # удалить образ
docker rmi -f nginx                  # принудительно удалить
docker rmi $(docker images -q)       # удалить ВСЕ образы
docker image prune                   # удалить неиспользуемые образы
docker image prune -a                # удалить все неиспользуемые (включая без контейнеров)

docker inspect <image>               # информация об образе
docker history <image>               # слои образа
```

---

##  Сборка образа

```bash
docker build .                       # собрать из Dockerfile в текущей папке
docker build -t myapp .              # с именем образа
docker build -t myapp:1.0 .          # с именем и тегом (версией)
docker build -t myapp:latest .       # latest = последняя версия
docker build -f MyDockerfile .       # указать нестандартный Dockerfile
docker build --no-cache .            # сборка без кэша
docker build --build-arg VERSION=1 . # передать аргумент сборки

# Пример полной команды
docker build -t username/myapp:1.0 .
```

---

##  Работа с томами (Volumes)

```bash
docker volume create my-data         # создать том
docker volume ls                     # список томов
docker volume inspect my-data        # информация о томе
docker volume rm my-data             # удалить том
docker volume prune                  # удалить все неиспользуемые тома

# Использование тома при запуске
docker run -v my-data:/var/lib/postgresql postgres  # именованный том
docker run -v /host/dir:/container/dir nginx         # bind mount (папка с хоста)
docker run --mount source=my-data,target=/data nginx # новый синтаксис
```

### Разница: Volume vs Bind Mount

| | Named Volume | Bind Mount |
|---|---|---|
| Синтаксис | `-v my-vol:/app/data` | `-v /host/path:/app/data` |
| Управление | Docker управляет | Вы управляете |
| Расположение | `/var/lib/docker/volumes/` | Любая папка |
| Применение | Данные приложений | Разработка, конфиги |

---

##  Работа с сетью

```bash
docker network ls                              # список сетей
docker network create my-net                   # создать сеть
docker network create --driver bridge my-net   # bridge (по умолчанию)
docker network inspect my-net                  # информация о сети
docker network rm my-net                       # удалить сеть
docker network prune                           # удалить неиспользуемые

# Подключение контейнеров к сети
docker run --network my-net nginx
docker network connect my-net <container>      # подключить работающий контейнер
docker network disconnect my-net <container>   # отключить
```

### Типы сетей

| Тип | Описание |
|---|---|
| `bridge` | Виртуальная сеть на хосте (по умолчанию) |
| `host` | Использовать сеть хоста напрямую |
| `none` | Нет сети (полная изоляция) |
| `overlay` | Для Docker Swarm (между машинами) |

---

##  Работа с портами

```
docker run -p <хост_порт>:<контейнер_порт> image
docker run -p <хост_IP>:<хост_порт>:<контейнер_порт> image
```

```bash
docker run -p 8080:80 nginx        # хост:8080 → контейнер:80
docker run -p 3000:3000 node-app   # один порт
docker run -P nginx                # авто назначить порты для EXPOSED
docker port <container>            # показать проброшенные порты
```

---

##  Копирование файлов

```bash
docker cp file.txt <container>:/app/         # хост → контейнер
docker cp <container>:/app/logs.txt ./        # контейнер → хост
docker cp <container>:/app ./backup/          # директория целиком
```

---

##  Очистка системы

```bash
docker system prune           # удалить всё неиспользуемое
docker system prune -a        # включая образы без контейнеров
docker system prune --volumes # включая тома

docker container prune        # удалить остановленные контейнеры
docker image prune -a         # удалить неиспользуемые образы
docker volume prune           # удалить неиспользуемые тома
docker network prune          # удалить неиспользуемые сети

docker system df              # сколько места занимает Docker
```

---

##  Шпаргалка команд

### Контейнеры

| Команда | Описание |
|---|---|
| `docker run -d -p 8080:80 --name web nginx` | Запустить nginx в фоне |
| `docker ps` | Активные контейнеры |
| `docker ps -a` | Все контейнеры |
| `docker stop web` | Остановить |
| `docker rm web` | Удалить |
| `docker exec -it web bash` | Войти внутрь |
| `docker logs -f web` | Следить за логами |

### Образы

| Команда | Описание |
|---|---|
| `docker images` | Список образов |
| `docker pull postgres:16` | Скачать образ |
| `docker build -t myapp:1.0 .` | Собрать из Dockerfile |
| `docker rmi myapp:1.0` | Удалить образ |
| `docker tag myapp user/myapp:1.0` | Переименовать образ |

### Деплой

| Команда | Описание |
|---|---|
| `docker login` | Войти в Docker Hub |
| `docker push user/myapp:1.0` | Загрузить образ |
| `docker pull user/myapp:1.0` | Скачать образ |
