# 📦 Java — Библиотеки

> **Теги:** #java #library #hub

> [!abstract] Навигация
> [[main Java]]

---

## 🗺️ Файлы раздела

| Файл | Назначение | Ключевые концепции |
|------|-----------|---------------------|
| [[Jackson]] | JSON сериализация/десериализация | ObjectMapper, @JsonProperty, @JsonIgnore, полиморфизм |
| [[Lombok]] | Снижение бойлерплейта | @Data, @Builder, @Slf4j, @RequiredArgsConstructor |
| [[MapStruct]] | Маппинг между объектами | @Mapper, @Mapping, конвертация DTO ↔ Entity |

---

## 📚 Порядок изучения

```
[[Lombok]]       ← первым, т.к. используется везде
    ↓
[[Jackson]]      ← сериализация для REST
    ↓
[[MapStruct]]    ← маппинг DTO, требует понимания Jackson
```

---

## 🔗 Связанные темы

> [!tip] Смотри также
> - [[Spring_Web_REST]] — Jackson используется в Spring MVC автоматически
> - [[Spring_Data_JPA]] — MapStruct активно применяется с Entity ↔ DTO
> - [[02_OOP_Core]] — понимание ООП необходимо для MapStruct
