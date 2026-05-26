# Nginx

> **Теги:** #tools #nginx #конспект  

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Что такое Nginx

> [!note] Определение
> **Nginx** — высокопроизводительный **web server** и **reverse proxy**. Event-driven архитектура — тысячи concurrent connections на мало памяти.

| Роль | Описание |
|------|----------|
| **Web server** | Статика (HTML, CSS, JS, images) |
| **Reverse proxy** | Принимает запросы, проксирует на backend (Spring Boot) |
| **Load balancer** | Распределение между несколькими upstream |
| **SSL termination** | HTTPS на Nginx, HTTP к backend |
| **API Gateway lite** | Rate limit, gzip, routing |

```
Client ──HTTPS──► Nginx (:443)
                    │
                    ├── /api/*  ──► Spring Boot :8080
                    ├── /static ──► files on disk
                    └── /admin  ──► another upstream
```

---

## 🔹 Базовая структура nginx.conf

```nginx
# /etc/nginx/nginx.conf
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;
    keepalive_timeout 65;
    gzip on;

    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# /etc/nginx/conf.d/app.conf
upstream spring_backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    keepalive 32;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/ssl/certs/api.crt;
    ssl_certificate_key /etc/ssl/private/api.key;

    location /api/ {
        proxy_pass http://spring_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/static/;
        expires 30d;
    }
}
```

> [!warning] proxy_pass trailing slash
> `location /api/` + `proxy_pass http://backend/;` — `/api/users` → `/users` на backend.
> Без slash — `/api/users` передаётся как есть.

---

## 🔹 Reverse proxy к Spring Boot

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 5s;
    proxy_read_timeout 60s;
    proxy_send_timeout 60s;
}
```

**Spring Boot — доверять forwarded headers:**

```yaml
server:
  forward-headers-strategy: framework
  # или native при proxy_set_header X-Forwarded-*
```

> [!tip] Зачем proxy
> Spring Boot на 8080 внутри сети; Nginx снаружи на 80/443 — SSL, static, rate limit.

---

## 🔹 Load balancing

```nginx
upstream backend {
    least_conn;                    # алгоритм
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080 backup;   # только если primary down
}
```

| Алгоритм | Когда |
|----------|-------|
| `round-robin` (default) | Равномерная нагрузка |
| `least_conn` | Долгие соединения |
| `ip_hash` | Sticky по IP клиента |
| `hash $request_uri` | Sticky по URI |

---

## 🔹 Gzip

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
gzip_comp_level 5;
```

> [!note] Spring Boot
> Можно gzip и в embedded Tomcat, но на Nginx — меньше нагрузка на JVM.

---

## 🔹 Rate limiting

```nginx
# http block
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://spring_backend;
    }
}
```

| Параметр | Смысл |
|----------|-------|
| `rate=10r/s` | 10 запросов в секунду |
| `burst=20` | Кратковременный всплеск |
| `nodelay` | Не задерживать burst-запросы в очереди |

**Ответ при превышении:** HTTP 503.

---

## 🔹 SSL termination

```nginx
listen 443 ssl http2;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers HIGH:!aNULL:!MD5;
ssl_prefer_server_ciphers on;

# Let's Encrypt (certbot)
# ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
```

Backend получает HTTP — Nginx добавляет `X-Forwarded-Proto: https`.

---

## 🔹 Health check (passive)

```nginx
upstream backend {
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
}
```

> [!tip] Active health checks
> Nginx Plus или внешний orchestrator (k8s readiness). Spring Actuator: `/actuator/health`.

---

## 🔹 Типичные ошибки

### ❌ Забыть X-Forwarded-* headers
Spring видит `http` вместо `https` → redirect loops, wrong links.

### ❌ Маленький `client_max_body_size`
Upload файлов — 413 Request Entity Too Large.

```nginx
client_max_body_size 20M;
```

### ❌ proxy_read_timeout слишком мал
Long-running запросы обрываются — увеличить для reports/export.

---

## 🔹 Итог

1. Nginx — reverse proxy + SSL + static + load balance.
2. `upstream` + `proxy_pass` — маршрутизация на Spring Boot.
3. `X-Forwarded-*` + `forward-headers-strategy` в Boot.
4. `limit_req_zone` — rate limiting.
5. `gzip`, `client_max_body_size` — production tuning.

```
Шпаргалка Nginx:
─────────────────────────────────────────
upstream backend { server host:port; }
proxy_pass http://backend;
X-Forwarded-For / X-Forwarded-Proto
ssl_certificate + listen 443 ssl
limit_req_zone + limit_req burst=
gzip on; gzip_types application/json;
client_max_body_size 20M;
forward-headers-strategy: framework  → Spring Boot
```
