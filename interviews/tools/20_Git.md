# Git

> **Теги:** #interviews #tools #git #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Зачем VCS (Version Control System)

**VCS** — система контроля версий.

### Зачем нужен Git

- **История изменений** — можно откатиться к любой версии
- **Коллаборация** — несколько разработчиков работают над одним проектом
- **Ветвление** — параллельная разработка фич
- **Резервное копирование** — код хранится на сервере
- **Code review** — просмотр изменений перед слиянием

### Централизованные vs Распределённые

| Характеристика | Централизованные (SVN) | Распределённые (Git) |
|----------------|------------------------|---------------------|
| Сервер | Обязателен | Не обязателен |
| Офлайн работа | Нет | Да |
| Скорость | Медленнее | Быстрее |
| Ветвление | Сложное | Простое |

---

## 🔹 .gitignore

**.gitignore** — файл с правилами исключения файлов из Git.

### Пример

```gitignore
# .gitignore
target/
*.log
*.class
.DS_Store
node_modules/
.env
```

### Синтаксис

| Паттерн | Описание |
|---------|----------|
| `*.log` | Все файлы с расширением .log |
| `target/` | Директория target |
| `*.class` | Все .class файлы |
| `!important.log` | Исключение (включить important.log) |
| `**/node_modules/**` | Любая node_modules директория |

### Примеры

```gitignore
# Игнорировать все .log файлы
*.log

# Но не important.log
!important.log

# Игнорировать директорию
target/

# Игнорировать во всех поддиректориях
**/node_modules/**

# Игнорировать конкретный файл
.env
```

---

## 🔹 Merge vs Cherry-pick

### Merge

**Merge** — объединение веток.

```bash
# Слияние feature в main
git checkout main
git merge feature
```

**Характеристики:**
- Сохраняет историю обеих веток
- Создаёт merge commit
- Используется для объединения больших изменений

### Cherry-pick

**Cherry-pick** — применение конкретного commit из другой ветки.

```bash
# Применить commit abc123 из feature в main
git checkout main
git cherry-pick abc123
```

**Характеристики:**
- Применяет только один commit
- Не сохраняет историю ветки
- Используется для backporting (переноса исправления)

### Сравнение

| Характеристика | Merge | Cherry-pick |
|----------------|-------|-------------|
| Объединение | Веток | Конкретного commit |
| История | Сохраняется | Не сохраняется |
| Использование | Обычное слияние | Backporting |

---

## 🔹 Tag

**Tag** — метка для конкретного commit (обычно для релизов).

### Типы тегов

#### Lightweight tag

```bash
git tag v1.0.0
```

- Простой указатель на commit
- Без дополнительной информации

#### Annotated tag

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

- Полноценный объект
- Содержит сообщение, автора, дату
- Рекомендуется для релизов

### Команды

```bash
# Создать annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Посмотреть все теги
git tag

# Посмотреть детали тега
git show v1.0.0

# Удалить тег локально
git tag -d v1.0.0

# Удалить тег удалённо
git push origin --delete v1.0.0

# Отправить теги на remote
git push origin --tags
```

---

## 🔹 Diff

**Diff** — просмотр различий между версиями.

### Команды

```bash
# Различия между рабочей директорией и staging
git diff

# Различия между staging и последним commit
git diff --staged
# или
git diff --cached

# Различия между двумя commit
git diff abc123 def456

# Различия между двумя ветками
git diff main feature

# Различия конкретного файла
git diff file.txt

# Различия в кратком формате
git diff --stat
```

### Пример вывода

```diff
diff --git a/src/main.java b/src/main.java
index 1234567..abcdefg 100644
--- a/src/main.java
+++ b/src/main.java
@@ -1,5 +1,5 @@
 public class Main {
-    public static void main(String[] args) {
+    public static void main(String[] args) throws Exception {
         System.out.println("Hello");
     }
 }
```

---

## 🔹 Rebase vs Merge

### Rebase

**Rebase** — перебазирование изменений.

```bash
# Rebase feature на main
git checkout feature
git rebase main
```

**Характеристики:**
- Линейная история
- Переписывает историю
- Конфликты решаются по одному

### Merge

**Merge** — слияние изменений.

```bash
# Merge feature в main
git checkout main
git merge feature
```

**Характеристики:**
- Сохраняет историю веток
- Создаёт merge commit
- История не линейная

### Сравнение

| Характеристика | Rebase | Merge |
|----------------|--------|-------|
| История | Линейная | Не линейная |
| Переписывание истории | Да | Нет |
| Конфликты | По одному | Все сразу |
| Рекомендация | Локальные ветки | Public ветки |

> [!warning] Не используйте rebase на public ветках (переписывает историю)

---

## 🔹 Reset vs Revert

### Reset

**Reset** — перемещение HEAD на другой commit (изменяет историю).

```bash
# Мягкий reset (сохраняет изменения в staging)
git reset --soft HEAD~1

# Жёсткий reset (удаляет изменения)
git reset --hard HEAD~1
```

**Характеристики:**
- Изменяет историю
- Опасно для public веток
- Используется локально

### Revert

**Revert** — создание нового commit, который отменяет изменения (сохраняет историю).

```bash
# Отменить последний commit
git revert HEAD

# Отменить конкретный commit
git revert abc123
```

**Характеристики:**
- Сохраняет историю
- Безопасно для public веток
- Создаёт новый commit

### Сравнение

| Характеристика | Reset | Revert |
|----------------|-------|--------|
| История | Изменяет | Сохраняет |
| Безопасность | Опасно | Безопасно |
| Public ветки | Нет | Да |
| Использование | Локально | Public |

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **VCS:** история изменений, коллаборация, ветвление
> - **.gitignore:** исключение файлов (*.log, target/, node_modules/)
> - **Merge** — объединение веток, **Cherry-pick** — конкретный commit
> - **Tag:** Lightweight (простой), Annotated (с информацией, для релизов)
> - **Diff:** git diff (рабочая), git diff --staged (staging), git diff abc123 def456
> - **Rebase** — линейная история (локально), **Merge** — сохраняет историю (public)
> - **Reset** — изменяет историю (локально), **Revert** — сохраняет историю (public)

```
VCS:
История изменений, коллаборация, ветвление
Git → распределённый, быстрый

.gitignore:
*.log → все .log файлы
target/ → директория
!important.log → исключение

Merge vs Cherry-pick:
Merge → объединение веток
Cherry-pick → конкретный commit (backporting)

Tag:
Lightweight → простой указатель
Annotated → с информацией (релизы)

Diff:
git diff → рабочая директория vs staging
git diff --staged → staging vs last commit
git diff abc123 def456 → два commit

Rebase vs Merge:
Rebase → линейная история (локально)
Merge → сохраняет историю (public)

Reset vs Revert:
Reset → изменяет историю (локально)
Revert → сохраняет историю (public)
```
