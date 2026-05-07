# Управление сервисами и Bash-скрипты

##  Теория: systemd и сервисы

**systemd** — это система инициализации (PID 1) и менеджер сервисов в современных Linux. Он запускается первым при старте системы и управляет всеми остальными процессами.

```
Ядро Linux запускается
       │
       ▼
   systemd (PID 1)
       │
       ├── Сетевые сервисы (NetworkManager)
       ├── SSH сервер (sshd)
       ├── Веб-сервер (nginx)
       ├── База данных (postgresql)
       └── ... (все остальные сервисы)
```

### Дистрибутивы с systemd
Ubuntu, Debian, Fedora, CentOS, RHEL, Arch Linux — почти все современные дистрибутивы.

---

##  systemctl — управление сервисами

### Основные команды

```bash
sudo systemctl start nginx      # запустить сервис
sudo systemctl stop nginx       # остановить
sudo systemctl restart nginx    # перезапустить
sudo systemctl reload nginx     # перечитать конфиг (без рестарта)
sudo systemctl status nginx     # показать статус
```

### Автозапуск

```bash
sudo systemctl enable nginx      # включить автозапуск при старте
sudo systemctl disable nginx     # отключить автозапуск
systemctl is-enabled nginx       # проверить: включён ли автозапуск
systemctl is-active nginx        # проверить: работает ли сейчас
```

### Состояния сервиса

| Состояние | Описание |
|---|---|
| `active (running)` | ✅ Сервис работает |
| `active (exited)` | ✅ Выполнился и завершился (нормально) |
| `inactive (dead)` | ⏹ Остановлен |
| `failed` | ❌ Произошла ошибка |
| `activating` | 🔄 В процессе запуска |

```bash
systemctl --failed       # показать все сервисы в состоянии failed
systemctl list-units     # список всех загруженных юнитов
systemctl list-units --type=service  # только сервисы
```

### Пример вывода `systemctl status`

```
● nginx.service - A high performance web server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
   Active: active (running) since Mon 2024-01-01 10:00:00 UTC
 Main PID: 1234 (nginx)
    Tasks: 2 (limit: 4915)
   Memory: 3.5M
      CPU: 12ms
   CGroup: /system.slice/nginx.service
           ├─1234 nginx: master process
           └─1235 nginx: worker process
```

---

##  journalctl — просмотр логов

```bash
journalctl                          # все логи
journalctl -u nginx                 # логи конкретного сервиса
journalctl -f                       # следить в реальном времени
journalctl -n 50                    # последние 50 строк
journalctl --since "1 hour ago"     # за последний час
journalctl --since "2024-01-01"     # с конкретной даты
journalctl --since "2024-01-01 08:00:00" --until "2024-01-01 09:00:00"
journalctl -u nginx --since "1 hour ago"  # логи nginx за час
journalctl -p err                   # только ошибки
journalctl -p warning               # предупреждения и выше
journalctl --disk-usage             # сколько места занимают логи
```

---

##  Управление временем

```bash
timedatectl                        # показать текущие настройки
timedatectl list-timezones         # список временных зон
timedatectl set-timezone Europe/Moscow  # установить зону
timedatectl set-ntp true           # включить синхронизацию NTP
date                               # текущее время
date +"%Y-%m-%d %H:%M:%S"          # форматированный вывод
```

### Вывод `timedatectl`

| Поле | Описание |
|---|---|
| `Local time` | Локальное системное время |
| `Universal time` | Время UTC |
| `RTC time` | Аппаратное время (BIOS) |
| `Time zone` | Текущая временная зона |
| `System clock synchronized` | Синхронизация через NTP |
| `NTP service` | Статус NTP-сервиса |

---

## Bash-скрипты

### Структура скрипта

```bash
#!/bin/bash
# ↑ Shebang — указывает интерпретатор
# Это комментарий

echo "Привет, мир!"
```

```bash
nano hello.sh        # создать файл
chmod +x hello.sh    # сделать исполняемым
./hello.sh           # запустить
bash hello.sh        # запустить через bash явно
```

---

### Переменные

```bash
NAME="Linux"                      # объявить переменную
echo "Привет, $NAME!"             # использовать переменную
echo "Привет, ${NAME}!"           # явные границы переменной

# Системные переменные
echo $USER     # текущий пользователь
echo $HOME     # домашняя директория
echo $PWD      # текущая директория
echo $PATH     # пути поиска программ
echo $0        # имя скрипта
echo $1 $2     # аргументы командной строки
echo $#        # количество аргументов
echo $?        # код выхода последней команды (0 = успех)

# Подстановка команды
CURRENT_DATE=$(date)
echo "Сейчас: $CURRENT_DATE"
FILES=$(ls /home)
```

---

### Условия `if`

```bash
if [ условие ]; then
    echo "Истина"
elif [ другое_условие ]; then
    echo "Другое"
else
    echo "Ложь"
fi
```

#### Операторы сравнения

| Числа | Строки | Файлы |
|---|---|---|
| `-eq` равно | `=` равно | `-f` файл существует |
| `-ne` не равно | `!=` не равно | `-d` директория существует |
| `-lt` меньше | `-z` пустая строка | `-e` путь существует |
| `-gt` больше | `-n` непустая строка | `-r` можно читать |
| `-le` меньше/равно | | `-x` можно выполнять |
| `-ge` больше/равно | | `-s` файл непустой |

```bash
#!/bin/bash
HOUR=$(date +%H)

if [ $HOUR -lt 12 ]; then
    echo "Доброе утро!"
elif [ $HOUR -lt 18 ]; then
    echo "Добрый день!"
else
    echo "Добрый вечер!"
fi
```

---

### Циклы

#### `for` — перебор списка

```bash
for i in {1..5}; do
    echo "Итерация $i"
done

# Перебор файлов
for file in /etc/*.conf; do
    echo "Файл: $file"
done

# Перебор аргументов
for arg in "$@"; do
    echo "Аргумент: $arg"
done
```

#### `while` — пока условие верно

```bash
COUNTER=0
while [ $COUNTER -lt 5 ]; do
    echo "Счётчик: $COUNTER"
    COUNTER=$((COUNTER + 1))
done

# Чтение файла построчно
while read line; do
    echo "Строка: $line"
done < file.txt
```

#### Управление циклами

```bash
break     # прервать цикл
continue  # перейти к следующей итерации
```

---

### Функции

```bash
greet() {
    local name=$1          # local — локальная переменная
    echo "Привет, $name!"
}

greet "Мир"
greet "Linux"
```

---

### Пример скрипта: Проверка доступности сайтов

```bash
#!/bin/bash

SITES=("example.com" "google.com" "nonexistent.website")

for SITE in "${SITES[@]}"; do
    if ping -c 1 $SITE &> /dev/null; then
        echo "✅ $SITE — доступен"
    else
        echo "❌ $SITE — недоступен"
    fi
done
```

---

##  Планирование задач

### cron — периодические задачи

```bash
crontab -e    # редактировать расписание текущего пользователя
crontab -l    # показать текущее расписание
crontab -r    # удалить всё расписание
sudo crontab -e -u username   # расписание другого пользователя
```

#### Формат crontab

```
# ┌──── минута (0-59)
# │ ┌─── час (0-23)
# │ │ ┌── день месяца (1-31)
# │ │ │ ┌─ месяц (1-12)
# │ │ │ │ ┌ день недели (0-7, 0=7=воскресенье)
# │ │ │ │ │
  * * * * * команда
```

#### Примеры расписаний

| Расписание | Описание |
|---|---|
| `* * * * *` | Каждую минуту |
| `0 * * * *` | Каждый час |
| `0 0 * * *` | Каждый день в полночь |
| `0 9 * * 1` | Каждый понедельник в 9:00 |
| `*/5 * * * *` | Каждые 5 минут |
| `0 0 1 * *` | Первое число каждого месяца |
| `0 0 * * 0` | Каждое воскресенье в полночь |

```bash
# Пример записи в crontab:
0 2 * * * /home/user/backup.sh >> /var/log/backup.log 2>&1
```

### at — одноразовые задачи

```bash
at 15:30                   # выполнить в 15:30
at now + 1 hour            # через час
at midnight                # в полночь

# В интерактивном режиме вводим команды, Ctrl+D — конец
echo "backup.sh" | at 02:00

atq                        # список запланированных задач
atrm 5                     # удалить задачу №5
```

---

##  Шаблон хорошего bash-скрипта

```bash
#!/bin/bash
set -e          # остановить при ошибке
set -u          # ошибка на неопределённые переменные
set -o pipefail # ошибка пайпа — это тоже ошибка

# === Константы ===
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="/var/log/myscript.log"

# === Функции ===
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# === Основная логика ===
log "Скрипт запущен"
# ... ваш код
log "Скрипт завершён"
```

