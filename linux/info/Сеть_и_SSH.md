# Работа с сетью и SSH

##  Теория: Основы сети

### Ключевые понятия

| Понятие | Описание | Пример |
|---|---|---|
| **IP-адрес** | Уникальный адрес устройства | `192.168.1.100` |
| **Подсеть (subnet)** | Логическая группа устройств | `192.168.1.0/24` |
| **Маска подсети** | Определяет размер подсети | `255.255.255.0` |
| **Шлюз (gateway)** | Выход во внешний мир | `192.168.1.1` |
| **Порт** | Идентификатор сервиса | `80`, `443`, `22` |
| **DNS** | Преобразование имён в IP | `google.com → 142.250.x.x` |

### Модель OSI (упрощённо)

```
┌────────────────────────────────┐
│  7. Приложение  (HTTP, SSH)    │
│  4. Транспорт   (TCP, UDP)     │
│  3. Сеть        (IP, ICMP)     │
│  2. Канал       (Ethernet)     │
│  1. Физика      (кабель, Wi-Fi)│
└────────────────────────────────┘
```

### IPv4 vs IPv6

| | IPv4 | IPv6 |
|---|---|---|
| Формат | `192.168.1.1` | `2001:0db8::1` |
| Размер | 32 бита | 128 бит |
| Адресов | ~4 млрд | 340 ундециллионов |

### Важные порты

| Порт | Протокол |
|---|---|
| `22` | SSH |
| `80` | HTTP |
| `443` | HTTPS |
| `21` | FTP |
| `25` | SMTP (почта) |
| `53` | DNS |
| `3306` | MySQL |
| `5432` | PostgreSQL |

---

## 🔌 Сетевые интерфейсы

### `ip addr` — конфигурация интерфейсов

```bash
ip addr                                  # показать все интерфейсы
ip addr show eth0                        # конкретный интерфейс
sudo ip addr add 192.168.1.101/24 dev eth0   # добавить IP (до перезагрузки)
sudo ip addr del 192.168.1.101/24 dev eth0   # удалить IP
sudo ip link set eth0 up                 # включить интерфейс
sudo ip link set eth0 down               # отключить интерфейс
```

### Вывод `ip addr`

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    link/ether 08:00:27:xx:xx:xx
    inet 192.168.1.100/24 brd 192.168.1.255 scope global
    inet6 fe80::a00:27ff::1/64 scope link
```

| Поле | Описание |
|---|---|
| `lo` | Loopback интерфейс (127.0.0.1) |
| `eth0` / `enp0s3` | Ethernet-интерфейс |
| `wlan0` | Wi-Fi интерфейс |
| `inet` | IPv4-адрес |
| `inet6` | IPv6-адрес |

### `ifconfig` (устаревший, но встречается)

```bash
ifconfig              # показать интерфейсы
ifconfig eth0         # конкретный интерфейс
```

---

## 🗺️ Маршрутизация

```bash
ip route show                              # таблица маршрутов
ip route show match 192.168.1.0/24         # маршрут для подсети
sudo ip route add 10.0.0.0/24 via 192.168.1.1 dev eth0  # добавить маршрут
sudo ip route del 10.0.0.0/24              # удалить маршрут
```

### Вывод `ip route show`

```
default via 192.168.1.1 dev eth0 proto dhcp metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
```

| Запись | Смысл |
|---|---|
| `default via 192.168.1.1` | Пакеты "в интернет" → через шлюз 192.168.1.1 |
| `192.168.1.0/24 dev eth0` | Локальная сеть → напрямую через eth0 |

---

## 🔧 Диагностика сети

### `ping` — проверка связи

```bash
ping 8.8.8.8              # проверить связь с интернетом
ping google.com           # проверить DNS + связь
ping -c 4 192.168.1.1     # отправить 4 пакета
ping -i 0.5 host          # интервал 0.5 сек
```

Поля вывода:
- `icmp_seq` — номер запроса
- `ttl` — время жизни пакета (hop limit)
- `time` — задержка в миллисекундах

### `traceroute` — путь пакета

```bash
traceroute google.com     # маршрут до цели
tracepath google.com      # аналог, не требует root
```

### `netstat` — активные соединения (устарел → `ss`)

```bash
netstat -tun              # TCP/UDP соединения (числовые адреса)
netstat -tlnp             # слушающие порты с именами процессов
netstat -s                # статистика по протоколам
```

### `ss` — современная замена netstat

```bash
ss -tun                   # активные TCP/UDP
ss -tlnp                  # слушающие порты с процессами
ss -s                     # сводная статистика
ss -t state established   # только установленные соединения
```

| Флаг | Описание |
|---|---|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Только слушающие |
| `-n` | Числовые адреса |
| `-p` | Показать процессы |

---

## 🌍 DNS

### Типы DNS-записей

| Запись | Назначение | Пример |
|---|---|---|
| `A` | Домен → IPv4 | `google.com → 142.250.74.206` |
| `AAAA` | Домен → IPv6 | `google.com → 2a00:1450::200e` |
| `CNAME` | Псевдоним домена | `www → example.com` |
| `MX` | Почтовый сервер | `mail.example.com` |
| `TXT` | Текстовые данные (SPF и др.) | `v=spf1 include:...` |
| `NS` | Сервер имён | `ns1.example.com` |

### Публичные DNS-серверы

| Провайдер | Адрес |
|---|---|
| Google | `8.8.8.8`, `8.8.4.4` |
| Cloudflare | `1.1.1.1`, `1.0.0.1` |
| Yandex | `77.88.8.8`, `77.88.8.1` |

### `nslookup` — проверка DNS

```bash
nslookup google.com           # IP для домена
nslookup google.com 8.8.8.8   # через конкретный DNS-сервер
nslookup -type=MX google.com  # MX-записи
nslookup -type=TXT google.com # TXT-записи
```

### `dig` — расширенные DNS-запросы

```bash
dig google.com                # A-запись
dig google.com A              # явно A-запись
dig google.com AAAA           # IPv6
dig google.com MX             # почтовые серверы
dig google.com TXT            # TXT-записи
dig @8.8.8.8 google.com       # через конкретный DNS
dig +short google.com         # краткий вывод (только IP)
```

> [!TIP] dig vs nslookup
> `dig` предоставляет больше деталей: время запроса, DNS-сервер, флаги, TTL записей. Используйте `dig` для отладки DNS-проблем.

---

## 🔐 SSH — Secure Shell

### Теория

SSH — зашифрованный протокол для удалённого управления. Использует **асимметричное шифрование**:

```
Клиент                              Сервер
   │                                   │
   │──── публичный ключ сервера ───────│
   │    (рукопожатие, проверка сервера)│
   │                                   │
   │──── аутентификация (ключ/пароль)──│
   │                                   │
   │═══════ зашифрованный канал ═══════│
```

### Подключение

```bash
ssh username@hostname          # подключиться
ssh username@192.168.1.100     # по IP
ssh -p 2222 user@host          # нестандартный порт
ssh -i ~/.ssh/mykey user@host  # с конкретным ключом
ssh -L 8080:localhost:80 user@host  # туннель (проброс порта)
```

### Генерация SSH-ключей

```bash
ssh-keygen                             # интерактивно
ssh-keygen -t ed25519 -C "email@mail.com"  # ed25519 (рекомендуется)
ssh-keygen -t rsa -b 4096              # RSA 4096 бит (классика)
```

Создаются два файла:

| Файл | Описание |
|---|---|
| `~/.ssh/id_ed25519` | **Приватный ключ** — никому не давать! |
| `~/.ssh/id_ed25519.pub` | **Публичный ключ** — размещается на серверах |

### Копирование ключа на сервер

```bash
ssh-copy-id username@hostname                    # стандартный способ
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host   # конкретный ключ

# Вручную:
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### Конфигурация SSH (`~/.ssh/config`)

```
Host myserver
    HostName 192.168.1.100
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
```

После этого достаточно: `ssh myserver`

### Конфигурация SSH-сервера (`/etc/ssh/sshd_config`)

```bash
sudo nano /etc/ssh/sshd_config
```

| Параметр | Значение | Описание |
|---|---|---|
| `Port` | `22` | Порт SSH |
| `PermitRootLogin` | `no` | Запретить вход под root |
| `PasswordAuthentication` | `no` | Только ключи (рекомендуется) |
| `PubkeyAuthentication` | `yes` | Разрешить вход по ключу |
| `AllowUsers` | `user1 user2` | Белый список пользователей |

```bash
sudo systemctl restart sshd   # применить изменения
```

---

## 🔨 Netcat — "швейцарский нож" сети

```bash
nc -zv 192.168.1.1 22         # проверить открыт ли порт
nc -zv host 1-1000            # сканировать диапазон портов
nc -l 1234                    # слушать на порту 1234 (сервер)
nc host 1234                  # подключиться к порту 1234 (клиент)
nc -u host 53                 # UDP соединение
```

---

## 🔗 Передача файлов

```bash
# scp — безопасное копирование через SSH
scp file.txt user@host:/tmp/           # локально → удалённо
scp user@host:/tmp/file.txt ./         # удалённо → локально
scp -r folder/ user@host:/tmp/         # директория
scp -P 2222 file.txt user@host:/tmp/   # нестандартный порт

# rsync — синхронизация (умнее scp)
rsync -av folder/ user@host:/backup/   # синхронизировать папку
rsync -av --delete folder/ user@host:/ # удалять лишние файлы
rsync -avz folder/ user@host:/         # с сжатием
```

---

## 📊 Схема диагностики сетевых проблем

```
Не работает интернет?
         │
    ping 127.0.0.1 ──── нет → сеть сломана на уровне ОС
         │ да
    ping шлюза ──────── нет → проблема в локальной сети
         │ да
    ping 8.8.8.8 ─────  нет → интернет не работает (ISP)
         │ да
    ping google.com ─── нет → проблема DNS
         │ да
    Всё ОК!
```
