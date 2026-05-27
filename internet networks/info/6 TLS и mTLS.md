# TLS / mTLS

> **Теги:** #networks #tls #mtls #https #security #конспект

> [!abstract] Связи
> [[main]] | [[main Internet Networks]]

---

## 🔹 TLS

TLS обеспечивает шифрование, целостность и аутентификацию сервера.

---

## 🔹 Handshake (упрощённо)

`ClientHello → ServerHello + cert → key exchange → encrypted channel`

---

## 🔹 mTLS

Обе стороны предъявляют сертификаты (client + server auth).

---

## 🔹 Итог

```
Шпаргалка TLS:
─────────────────────────────────────────
HTTPS = HTTP + TLS
TLS 1.2/1.3 only
X.509 certificates
mTLS for internal APIs
terminate TLS at Ingress/Nginx
```
