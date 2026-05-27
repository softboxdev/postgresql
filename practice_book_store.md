
---

# Полная подготовка ОС Ubuntu 24.04 для работы с PostgreSQL

## Для кого эта инструкция - в случае, если не установлено

- Вы никогда не работали с Linux/Ubuntu
- Вы не знаете, что такое терминал
- Вы хотите просто начать писать SQL-запросы


---

## Часть 0. Что вам понадобится

| Что | Где взять |
|-----|-----------|
| Ubuntu 24.04 | Установлена на компьютер или в виртуальной машине (VirtualBox, VMware) |
| Доступ в интернет | Для скачивания PostgreSQL |
| Права администратора | Пароль от учётной записи (sudo) |

---

## Часть 1. Открываем терминал

Терминал — это программа, где вы будете вводить команды.

### Способ 1 (самый простой)
Нажмите одновременно клавиши:
```
Ctrl + Alt + T
```

### Способ 2 (через меню)
1. Нажмите кнопку `Windows` (или `Super`) на клавиатуре
2. Напечатайте `terminal`
3. Нажмите `Enter`

### Как выглядит терминал
```
ваше_имя@ваш_компьютер:~$
```
Это **приглашение командной строки**. После него вы вводите команды.

✅ **Проверка:** Вы видите мигающий курсор. Всё работает.

---

## Часть 2. Обновляем систему (обязательно)

Введите команду **точно** как написано:

```bash
sudo apt update
```

**Что произойдёт:**
- Система спросит пароль — **введите свой пароль от Ubuntu** (символы не видны, это нормально)
- Нажмите `Enter`

**Ожидаемый результат:**
```
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
...
Reading package lists... Done
```

Теперь обновите пакеты:

```bash
sudo apt upgrade -y
```

**Что значит `-y`:** Автоматически соглашаться на все вопросы.

✅ **Проверка:** Команда завершилась без красных сообщений об ошибках.

---

## Часть 3. Устанавливаем PostgreSQL

Введите команду:

```bash
sudo apt install -y postgresql-16
```

**Что происходит:**
- Система скачивает PostgreSQL версии 16
- Устанавливает его как службу (автоматически запускается при старте)

**Ожидаемый результат:** Через 30–60 секунд появится приглашение командной строки `$`.

✅ **Проверка — убедимся, что установилось:**

```bash
psql --version
```

Должно вывести что-то вроде:
```
psql (PostgreSQL) 16.4 (Ubuntu 16.4-0ubuntu0.24.04.1)
```

---

## Часть 4. Проверяем, что сервер работает

PostgreSQL — это серверная программа, она должна работать в фоне.

```bash
sudo systemctl status postgresql
```

**Ожидаемый результат:** Вы видите зелёное слово `active (running)`

```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since ...
```

Если **не работает** (красное `inactive` или `failed`), запустите:

```bash
sudo systemctl start postgresql
```

✅ **Проверка:** Команда `status` показывает `active (running)`.

---

## Часть 5. Вход в PostgreSQL (первый раз)

PostgreSQL создаёт специального пользователя `postgres`. Переключитесь на него:

```bash
sudo -i -u postgres
```

**Что изменилось:**
- Приглашение командной строки стало:
```
postgres@ваш_компьютер:~$
```
- Вы теперь пользователь `postgres` (не ваш обычный пользователь)

Теперь запустите клиент PostgreSQL (`psql`):

```bash
psql
```

**Ожидаемый результат:** Вы видите:

```
psql (16.4)
Type "help" for help.

postgres=#
```

Поздравляю! Вы внутри PostgreSQL. 👏

---

## Часть 6. Создаём свою первую базу данных

Внутри `psql` (там, где `postgres=#`) введите:

```sql
CREATE DATABASE my_first_db;
```

**Важно:** Точка с запятой `;` в конце обязательна!

**Ожидаемый результат:**
```
CREATE DATABASE
```

Подключитесь к новой базе:

```sql
\c my_first_db;
```

**Ожидаемый результат:**
```
You are now connected to database "my_first_db" as user "postgres".
my_first_db=#
```

✅ **Проверка:** Приглашение изменилось на `my_first_db=#`.

---

## Часть 7. Создаём своего пользователя (чтобы не работать от postgres)

Работать от `postgres` небезопасно. Создадим обычного пользователя.

**Выйдите из `psql`** (только из psql, не из терминала):

```sql
\q
```

Вы вернулись к приглашению `postgres@...`

Теперь создайте пользователя (замените `myuser` и `mypassword` на свои):

```bash
createuser --interactive
```

Ответьте на вопросы:
```
Enter name of role to add: myuser
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) y
Shall the new role be allowed to create more new roles? (y/n) n
```

Теперь задайте пароль:

```bash
psql -c "ALTER USER myuser WITH PASSWORD 'mypassword123';"
```

✅ **Проверка:** Команда вывела `ALTER ROLE`.

---

## Часть 8. Даём права пользователю на вашу БД

```bash
psql -c "GRANT ALL PRIVILEGES ON DATABASE my_first_db TO myuser;"
```

✅ **Проверка:** Вывело `GRANT`.

---

## Часть 9. Вход под своим пользователем

```bash
psql -d my_first_db -U myuser
```

**Ожидаемый результат:**
```
Password for user myuser: 
```

Введите пароль, который вы задали (`mypassword123`).

Вы должны увидеть:
```
my_first_db=>
```

Обратите внимание: `=>` вместо `=#`. Значит, вы обычный пользователь (не суперпользователь), это хорошо.

✅ **Проверка:** Вы внутри своей БД под своим пользователем.

---


## Часть 10. Как выйти и зайти обратно

### Выйти из `psql` (внутри базы)
```sql
\q
```

### Выйти из пользователя `postgres` (вернуться к себе)
```bash
exit
```

### Как зайти снова (короткий путь)
```bash
psql -d my_first_db -U myuser -h localhost
```

Пароль спросит.

---

## Часть 11. Полезные команды, которые нужно запомнить

| Команда | Что делает |
|---------|-------------|
| `psql -d имя_бд -U имя_пользователя` | Подключиться к БД |
| `\l` | Список всех баз данных |
| `\dt` | Список таблиц |
| `\d имя_таблицы` | Описание структуры таблицы |
| `\q` | Выйти из psql |
| `\c имя_бд` | Переключиться на другую БД |
| `\?` | Справка по командам psql |
| `Ctrl + L` | Очистить экран |

---

## Часть 12. Установка графического интерфейса (pgAdmin) — по желанию

Если не хотите работать в терминале, установите pgAdmin:

```bash
# Добавить репозиторий
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg

sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/noble pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'

# Установить
sudo apt update
sudo apt install -y pgadmin4-desktop
```

После установки найдите в меню приложений `pgAdmin 4`.

---

## Часть 13. Что делать, если что-то пошло не так

### Ошибка: `psql: error: could not connect to server`

**Причина:** PostgreSQL не запущен.

**Решение:**
```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql  # чтобы запускался при старте
```

### Ошибка: `FATAL: Peer authentication failed`

**Причина:** Проблема с файлом `pg_hba.conf`.

**Решение:**
```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```
Найдите строку:
```
local   all             all                                     peer
```
Замените `peer` на `md5`:
```
local   all             all                                     md5
```
Сохраните: `Ctrl+O`, `Enter`, `Ctrl+X`.  
Перезапустите:
```bash
sudo systemctl restart postgresql
```

### Ошибка: `permission denied` при командах

**Причина:** Забыли `sudo`.

**Решение:** Добавьте `sudo` в начало команды.

### Забыли пароль пользователя PostgreSQL

**Решение:**
```bash
sudo -u postgres psql
ALTER USER myuser WITH PASSWORD 'новый_пароль';
\q
```

---

## Часть 14. Чек-лист: что должно быть готово

Перед тем как писать SQL-запросы, проверьте:

- [ ] Ubuntu 24.04 установлена
- [ ] Терминал открывается (`Ctrl+Alt+T`)
- [ ] PostgreSQL установлен (`psql --version` работает)
- [ ] Сервер запущен (`sudo systemctl status postgresql` → active)
- [ ] Создана база данных (например, `my_first_db`)
- [ ] Создан свой пользователь (не `postgres`)
- [ ] Можете подключиться (`psql -d my_first_db -U myuser`)
- [ ] Можете выполнить `SELECT 1;` → выводит `1`

---

## Шпаргалка для быстрого старта (сохраните себе)

```bash
# === ОДИН РАЗ ПРИ ПЕРВОЙ НАСТРОЙКЕ ===
sudo apt update && sudo apt upgrade -y
sudo apt install -y postgresql-16

# === КАЖДЫЙ РАЗ, КОГДА ХОТИТЕ РАБОТАТЬ ===
psql -d имя_вашей_бд -U ваше_имя_пользователя -h localhost
# (введите пароль)

# === ПЕРВЫЕ SQL-ЗАПРОСЫ ===
CREATE TABLE users (id SERIAL, name TEXT);
INSERT INTO users (name) VALUES ('Анна');
SELECT * FROM users;
```

---


**«Книжный магазин»**:

- Связи «многие ко многим» (книги ↔ авторы через связующую таблицу).
- Агрегацию (`COUNT`, `SUM`).
- Группировку (`GROUP BY`).

Задание разбито на **схему данных** + **интерфейс запросов** (без UI, только SQL как интерфейс к данным).

---

# Упражнение: Приложение «Книжный магазин»

## Контекст (очень простой)

Вы делаете бэкенд для небольшого книжного магазина.  
Нужно хранить:

- Книги (название, цена, количество на складе)
- Авторов (имя, фамилия)
- Связь «книга — автор» (у книги может быть несколько авторов)
- Покупателей (имя, email)
- Продажи (какой покупатель, какую книгу, когда купил)

---

## Часть 1. Схема данных (SQL DDL)

Выполните в `psql` после подключения к вашей БД (например, `bookstore`).

```sql
-- 1. Авторы
CREATE TABLE authors (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL
);

-- 2. Книги
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0)
);

-- 3. Связь "многие ко многим": книга ←→ автор
CREATE TABLE book_authors (
    book_id INTEGER REFERENCES books(id) ON DELETE CASCADE,
    author_id INTEGER REFERENCES authors(id) ON DELETE CASCADE,
    PRIMARY KEY (book_id, author_id)
);

-- 4. Покупатели
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL
);

-- 5. Продажи (чеки)
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    book_id INTEGER REFERENCES books(id),
    customer_id INTEGER REFERENCES customers(id),
    sale_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_sale DECIMAL(10,2) NOT NULL
);
```

> ✅ **Проверка:**  
> ```sql
> \dt
> ```
> Должно быть 5 таблиц.

---

## Часть 2. Заполнение тестовыми данными (минимально, но осмысленно)

```sql
-- Авторы
INSERT INTO authors (first_name, last_name) VALUES
('Джордж', 'Оруэлл'),
('Лев', 'Толстой'),
('Эрих Мария', 'Ремарк');

-- Книги
INSERT INTO books (title, price, stock) VALUES
('1984', 350.00, 10),
('Анна Каренина', 450.00, 5),
('Три товарища', 400.00, 7);

-- Связи книг с авторами
INSERT INTO book_authors (book_id, author_id) VALUES
(1, 1), -- 1984 → Оруэлл
(2, 2), -- Анна Каренина → Толстой
(3, 3); -- Три товарища → Ремарк

-- Покупатели
INSERT INTO customers (name, email) VALUES
('Алексей Иванов', 'alex@example.com'),
('Елена Смирнова', 'elena@example.com');

-- Продажи (кто, что, когда, сколько, по какой цене)
INSERT INTO sales (book_id, customer_id, quantity, price_at_sale) VALUES
(1, 1, 1, 350.00),
(2, 2, 1, 450.00),
(3, 1, 2, 400.00);
```

✅ **После этого у вас есть работающие данные.**

---

## Часть 3. Интерфейс — это SQL-запросы (ваши задания)

Вы — «бэкенд». Клиент (условный веб-интерфейс) задаёт вопросы, вы отвечаете SQL-запросами.

Напишите запросы для каждого пункта.

---

### 3.1. Показать все книги с их авторами

**Результат:**  
`title | first_name | last_name`

<details>
<summary>Подсказка</summary>

```sql
SELECT b.title, a.first_name, a.last_name
FROM books b
JOIN book_authors ba ON b.id = ba.book_id
JOIN authors a ON a.id = ba.author_id;
```
</details>

---

### 3.2. Какая книга продалась лучше всего (по количеству экземпляров)?

<details>
<summary>Подсказка</summary>

```sql
SELECT b.title, SUM(s.quantity) AS total_sold
FROM sales s
JOIN books b ON b.id = s.book_id
GROUP BY b.id
ORDER BY total_sold DESC
LIMIT 1;
```
</details>

---

### 3.3. Выручка по каждой книге (price_at_sale * quantity)

<details>
<summary>Подсказка</summary>

```sql
SELECT b.title, SUM(s.price_at_sale * s.quantity) AS revenue
FROM sales s
JOIN books b ON b.id = s.book_id
GROUP BY b.id
ORDER BY revenue DESC;
```
</details>

---

### 3.4. Какой покупатель потратил больше всего денег?

<details>
<summary>Подсказка</summary>

```sql
SELECT c.name, SUM(s.price_at_sale * s.quantity) AS total_spent
FROM sales s
JOIN customers c ON c.id = s.customer_id
GROUP BY c.id
ORDER BY total_spent DESC
LIMIT 1;
```
</details>

---

### 3.5. Вывести список покупателей, которые купили книгу «1984»

<details>
<summary>Подсказка</summary>

```sql
SELECT DISTINCT c.name
FROM sales s
JOIN books b ON b.id = s.book_id
JOIN customers c ON c.id = s.customer_id
WHERE b.title = '1984';
```
</details>

---

### 3.6. Показать остатки на складе (только книги с остатком > 0)

<details>
<summary>Подсказка</summary>

```sql
SELECT title, stock FROM books WHERE stock > 0;
```
</details>

---

### 3.7. Снизить цену всех книг на 10% (UPDATE)

> Это **действие**, а не запрос на вывод. Выполните его в БД.

<details>
<summary>Подсказка</summary>

```sql
UPDATE books SET price = price * 0.9;
```
</details>

---

### 3.8. Добавить второго автора для книги «1984» (например, «Имя Фамилия»)

> Вставьте в `book_authors`

<details>
<summary>Подсказка</summary>

```sql
INSERT INTO authors (first_name, last_name) VALUES ('Имя', 'Фамилия');
INSERT INTO book_authors (book_id, author_id)
VALUES (
    (SELECT id FROM books WHERE title = '1984'),
    (SELECT id FROM authors WHERE first_name = 'Имя' AND last_name = 'Фамилия')
);
```
</details>

---

### 3.9. Показать книги, у которых нет ни одной продажи

<details>
<summary>Подсказка</summary>

```sql
SELECT b.title
FROM books b
LEFT JOIN sales s ON b.id = s.book_id
WHERE s.id IS NULL;
```
</details>

---

### 3.10. Самый популярный автор по количеству проданных книг

<details>
<summary>Подсказка</summary>

```sql
SELECT a.first_name, a.last_name, SUM(s.quantity) AS sold
FROM authors a
JOIN book_authors ba ON a.id = ba.author_id
JOIN books b ON b.id = ba.book_id
JOIN sales s ON s.book_id = b.id
GROUP BY a.id
ORDER BY sold DESC
LIMIT 1;
```
</details>

---

## Часть 4. Контрольные вопросы (чтобы убедиться, что вы поняли)

| № | Вопрос | Ваш ответ (устно или письменно) |
|---|--------|--------------------------------|
| 1 | Зачем нужна таблица `book_authors`? | |
| 2 | Что произойдёт с `sales`, если удалить книгу? | |
| 3 | В чём разница между `stock` и `quantity` в продажах? | |
| 4 | Почему в `sales` хранится `price_at_sale`, а не берётся из `books.price`? | |

<details>
<summary>Ответы</summary>

1. Чтобы реализовать связь «многие ко многим» (у книги несколько авторов, у автора — несколько книг).  
2. Ничего, если `ON DELETE RESTRICT` — ошибка. В нашей схеме `REFERENCES books(id)` без `ON DELETE CASCADE` — удаление книги будет запрещено, если есть продажи.  
3. `stock` — сколько сейчас есть на складе, `quantity` — сколько купили.  
4. Цена книги может меняться, но цена продажи должна оставаться исторической.
</details>

---

## Ожидаемый результат выполнения

После выполнения упражнения вы:

- ✅ Спроектируете схему с отношениями 1:N и M:N
- ✅ Напишете `SELECT` с `JOIN`, `GROUP BY`, `SUM`, `COUNT`
- ✅ Выполните `UPDATE` и `INSERT`
- ✅ Используете подзапросы и `LEFT JOIN` для поиска отсутствующих данных

---

## Если вы хотите проверить себя полностью

Выполните этот скрипт как «экзамен» (без подсказок):

```sql
-- 1. Создать таблицы (уже сделано)
-- 2. Вставить минимум 3 книги, 3 авторов, 2 покупателей, 5 продаж

-- 3. Написать запрос:
--    "Название книги, автор, суммарная выручка по этой книге"

-- 4. Написать запрос:
--    "Покупатели, которые купили больше 2 книг (по сумме quantity)"
```

---

## Как сдать упражнение (если нужно отчитаться)

```bash
pg_dump bookstore --inserts > bookstore_solution.sql
```

Или просто сохраните все ваши `SELECT`-запросы в текстовый файл.

Готово. Вы только что сделали полноценное приложение «Книжный магазин» на уровне схемы и SQL-интерфейса.