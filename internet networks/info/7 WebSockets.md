# WebSockets

> **Теги:** #networks #websockets #realtime #http #конспект

> [!abstract] Связи
> [[main]] | [[main Internet Networks]]

---

## 🔹 HTTP vs WebSocket

WebSocket = persistent full-duplex channel после HTTP Upgrade.

---

## 🔹 Handshake

`Upgrade: websocket` + `101 Switching Protocols`.

---

## 🔹 Когда использовать

- chat
- live notifications
- realtime dashboards

---

## 🔹 Итог

```
Шпаргалка WebSocket:
─────────────────────────────────────────
Upgrade header
101 Switching Protocols
full-duplex realtime
STOMP in Spring (optional)
sticky sessions / broker for scale
```
