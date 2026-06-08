# Load Balancing

> **Теги:** #networks #load-balancing #nginx #proxy #high-availability #конспект

> [!abstract] Связи
> [[main]] | [[main Internet Networks]]

---

## 🔹 Зачем балансировка

Распределение трафика между несколькими инстансами для HA и scale.

---

## 🔹 L4 vs L7

- L4: TCP, быстрее
- L7: HTTP-aware routing (path/header/cookie)

---

## 🔹 Алгоритмы

- round robin
- least connections
- ip hash
- weighted

---

## 🔹 Итог

```
Шпаргалка Load Balancing:
─────────────────────────────────────────
L4 = transport
L7 = application routing
health checks required
sticky vs stateless
Nginx/Ingress/Cloud LB
```
