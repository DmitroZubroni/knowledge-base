# 17 — Сериализация (JSON vs Java Serialization)

> **Теги:** #java #programming #serialization #json #jackson #конспект

> [!abstract] Связи
> [[main]] | [[main Java]]

---

## 🔹 Java Serialization vs JSON

| | Java Serializable | JSON (Jackson) |
|---|------------------|----------------|
| Формат | binary | text |
| Interop | Java-only | cross-language |
| API use | legacy only | modern REST |

---

## 🔹 Pitfalls

- не сериализовать JPA entity наружу
- не использовать Java serialization для HTTP API
- versioning: `serialVersionUID` и backward compatibility

---

## 🔹 Итог

```
Шпаргалка Serialization:
─────────────────────────────────────────
REST -> JSON/Jackson
Serializable -> legacy
DTO instead of Entity
watch cycles/lazy fields
```
