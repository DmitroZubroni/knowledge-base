# MongoDB

> **Теги:** #nosql #mongodb #document #spring-data #конспект

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[NoSQL_DB]]

---

## 🔹 Что такое MongoDB

MongoDB — document-oriented NoSQL БД (BSON документы в коллекциях).

---

## 🔹 CRUD

```javascript
db.products.insertOne({ name: "Widget", price: 19.9 })
db.products.find({ price: { $gte: 10 } })
db.products.updateOne({ name: "Widget" }, { $set: { price: 17.9 } })
db.products.deleteMany({ price: { $lt: 5 } })
```

---

## 🔹 Aggregation

`$match → $group → $sort → $lookup`

---

## 🔹 Spring Data MongoDB

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
```

---

## 🔹 Итог

```
Шпаргалка Mongo:
─────────────────────────────────────────
document model (BSON)
CRUD + indexes
aggregation pipeline
MongoRepository
use when schema flexible
```
