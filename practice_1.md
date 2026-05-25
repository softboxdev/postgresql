
---

# БД «Список контактов»

## Цель работы

Научиться устанавливать PostgreSQL, создавать БД, таблицы и выполнять базовые SQL-запросы (`INSERT`, `SELECT`, `UPDATE`, `DELETE`).

---

## Часть 1. Установка PostgreSQL (2 минуты)

Откройте терминал (Ctrl+Alt+T) и выполните команды:

```bash
# Установить PostgreSQL
sudo apt update
sudo apt install -y postgresql-16

# Проверить, что сервер запущен
sudo systemctl status postgresql
```

**Ожидаемый результат:** В выводе команды `systemctl status` должно быть написано `active (running)`.

---

## Часть 2. Создание базы данных (3 минуты)

Переключитесь на пользователя `postgres` и создайте БД:

```bash
# Переключиться на пользователя postgres
sudo -u postgres psql
```

Внутри `psql` выполните:

```sql
-- Создать базу данных
CREATE DATABASE phonebook;

-- Подключиться к ней
\c phonebook;

-- Создать пользователя (опционально)
CREATE USER myuser WITH PASSWORD 'mypass123';

-- Дать права на БД
GRANT ALL PRIVILEGES ON DATABASE phonebook TO myuser;
```

**Ожидаемый результат:** Вы видите сообщение `CREATE DATABASE` и приглашение `phonebook=#`.

---

## Часть 3. Создание таблицы (5 минут)

Всё ещё внутри `psql` выполните:

```sql
-- Создать таблицу контактов
CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(100),
    age INTEGER
);
```

**Проверка:** Посмотрите список таблиц:

```sql
\dt
```

**Ожидаемый результат:** Вы видите одну таблицу `contacts`.

---

## Часть 4. Добавление данных (INSERT) (5 минут)

Добавьте несколько контактов:

```sql
-- Добавить одного человека
INSERT INTO contacts (name, phone, email, age) 
VALUES ('Иван Петров', '+7-999-123-45-67', 'ivan@example.com', 25);

-- Добавить второго
INSERT INTO contacts (name, phone, email, age) 
VALUES ('Мария Сидорова', '+7-888-555-12-34', 'maria@example.com', 30);

-- Добавить третьего (без email)
INSERT INTO contacts (name, phone, age) 
VALUES ('Алексей Смирнов', '+7-777-333-22-11', 28);
```

**Проверка:** Посчитайте, сколько записей добавилось:

```sql
SELECT COUNT(*) FROM contacts;
```

**Ожидаемый результат:** `3`.

---

## Часть 5. Просмотр данных (SELECT) (5 минут)

### 5.1. Посмотреть все данные

```sql
SELECT * FROM contacts;
```

### 5.2. Посмотреть только имена и телефоны

```sql
SELECT name, phone FROM contacts;
```

### 5.3. Найти контакты старше 25 лет

```sql
SELECT name, age FROM contacts WHERE age > 25;
```

### 5.4. Отсортировать по имени

```sql
SELECT name, phone FROM contacts ORDER BY name;
```

**Ожидаемый результат:** Вы видите таблицу с тремя строками (в разных форматах).

---

## Часть 6. Изменение данных (UPDATE) (3 минуты)

### 6.1. Изменить телефон Ивана

```sql
UPDATE contacts 
SET phone = '+7-999-000-00-00' 
WHERE name = 'Иван Петров';
```

### 6.2. Добавить email Алексею

```sql
UPDATE contacts 
SET email = 'alexey@example.com' 
WHERE name = 'Алексей Смирнов';
```

**Проверка:** Посмотрите на изменённые данные:

```sql
SELECT * FROM contacts;
```

---

## Часть 7. Удаление данных (DELETE) (2 минуты)

### 7.1. Удалить Марию

```sql
DELETE FROM contacts WHERE name = 'Мария Сидорова';
```

### 7.2. Проверить, что осталось

```sql
SELECT * FROM contacts;
```

**Ожидаемый результат:** Осталось 2 записи: Иван и Алексей.

---

## Часть 8. Выход и бэкап (3 минуты)

### 8.1. Выйти из psql

```sql
\q
```

### 8.2. Сделать резервную копию БД

```bash
# Выйдите в обычную командную строку (не psql)
pg_dump phonebook > phonebook_backup.sql
```

### 8.3. Проверить, что файл создался

```bash
ls -la phonebook_backup.sql
```

**Ожидаемый результат:** Вы видите файл размером примерно 2-4 КБ.

---

## Часть 9. Восстановление из бэкапа (бонус, 5 минут)

### 9.1. Удалить текущую БД

```bash
sudo -u postgres psql
```

```sql
DROP DATABASE phonebook;
\q
```

### 9.2. Восстановить из бэкапа

```bash
sudo -u postgres psql < phonebook_backup.sql
```

### 9.3. Проверить, что данные восстановились

```bash
sudo -u postgres psql -d phonebook -c "SELECT * FROM contacts;"
```

**Ожидаемый результат:** Вы снова видите Ивана и Алексея.

---

## Задание для самостоятельного выполнения (5 задач)

Напишите SQL-запросы для следующих задач (прямо в `psql`):

| № | Задание | Ваш запрос |
|---|---------|------------|
| 1 | Добавить новый контакт: «Ольга Кузнецова», телефон `+7-666-444-33-22`, email `olga@example.com`, возраст 35 | `INSERT INTO ...` |
| 2 | Найти всех, кто младше 30 лет | `SELECT ...` |
| 3 | Изменить возраст Алексея на 29 | `UPDATE ...` |
| 4 | Удалить контакт с телефоном `+7-999-123-45-67` | `DELETE ...` |
| 5 | Вывести все имена и email, отсортированные по возрасту (от молодых к старым) | `SELECT ... ORDER BY ...` |

**Решение (не подглядывайте сразу):**

<details>
<summary>Ответы</summary>

```sql
-- 1
INSERT INTO contacts (name, phone, email, age) 
VALUES ('Ольга Кузнецова', '+7-666-444-33-22', 'olga@example.com', 35);

-- 2
SELECT * FROM contacts WHERE age < 30;

-- 3
UPDATE contacts SET age = 29 WHERE name = 'Алексей Смирнов';

-- 4
DELETE FROM contacts WHERE phone = '+7-999-123-45-67';

-- 5
SELECT name, email FROM contacts ORDER BY age ASC;
```
</details>

---



## Часто встречающиеся ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `psql: command not found` | PostgreSQL не установлен | Выполните `sudo apt install postgresql-16` |
| `FATAL: database "phonebook" does not exist` | Не создали БД | Выполните `CREATE DATABASE phonebook;` |
| `ERROR: relation "contacts" does not exist` | Не создали таблицу | Выполните `CREATE TABLE contacts ...` |
| `ERROR: column "Name" does not exist` | Имена колонок чувствительны к регистру | Используйте строчные буквы: `name`, а не `Name` |
| `ERROR: syntax error at or near "..."` | Опечатка в SQL | Проверьте кавычки, запятые и точки с запятой `;` |

---

## Сдача задания

Сохраните все выполненные команды в файл `my_work.sql`:

```bash
# Выйдите из psql, затем выполните:
sudo -u postgres pg_dump phonebook --inserts > my_work.sql
```

Или просто скопируйте все ваши SQL-команды в текстовый файл.

