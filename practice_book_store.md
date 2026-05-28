
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



## Часть 06. Как выйти и зайти обратно

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

## Часть 07. Полезные команды, которые нужно запомнить

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

## Часть 08. Установка графического интерфейса (pgAdmin) — по желанию

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

## Что делать, если что-то пошло не так

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

## Чек-лист: что должно быть готово

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
# Практическое задание: PL/pgSQL в книжном магазине

## Цель работы

Отработать все основные конструкции PL/pgSQL на реальных данных книжного магазина: переменные, условия, циклы, курсоры, динамические команды, массивы, обработку ошибок, триггеры и отладку.

---

## Подготовка (схема и данные)

Перед выполнением заданий выполните скрипт из предыдущего упражнения (схема + тестовые данные).

```bash
# Подключитесь к PostgreSQL
sudo -u postgres psql

# Создайте базу (если ещё нет)
CREATE DATABASE bookstore;
\c bookstore

# Выполните скрипт создания таблиц и заполнения данными
-- (скопируйте SQL из Части 1 и Части 2 выше)
```

**Проверьте, что всё готово:**
```sql
\dt
SELECT * FROM books;
SELECT * FROM authors;
SELECT * FROM customers;
SELECT * FROM sales;
```

---

## Часть 1. Обзор и конструкции языка (10 баллов)

### Задание 1.1. Функция со скидкой

Создайте функцию `apply_discount(price DECIMAL, discount_percent DECIMAL)`, которая возвращает цену со скидкой.

**Требования:**
- Использовать `DECLARE` для объявления переменной `discount_amount`
- Использовать `IF/ELSE`: если скидка больше 50%, выдать предупреждение через `RAISE NOTICE`, но скидку всё равно применить
- Вернуть новую цену (не может быть меньше 0)

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION apply_discount(price DECIMAL, discount_percent DECIMAL)
RETURNS DECIMAL AS $$
DECLARE
    discount_amount DECIMAL;
    new_price DECIMAL;
BEGIN
    IF discount_percent > 50 THEN
        RAISE NOTICE 'Внимание: скидка %%% превышает 50%%!', discount_percent;
    END IF;
    
    discount_amount := price * discount_percent / 100;
    new_price := price - discount_amount;
    
    IF new_price < 0 THEN
        new_price := 0;
    END IF;
    
    RETURN new_price;
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
SELECT apply_discount(1000, 20);  -- 800
SELECT apply_discount(100, 60);   -- 40 с предупреждением
```

---

### Задание 1.2. CASE для категории цены

Создайте функцию `price_category(price DECIMAL)`, которая возвращает:
- `'Бюджетный'` — если цена < 300
- `'Средний'` — если цена от 300 до 800
- `'Премиум'` — если цена > 800

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION price_category(price DECIMAL)
RETURNS TEXT AS $$
BEGIN
    RETURN CASE
        WHEN price < 300 THEN 'Бюджетный'
        WHEN price <= 800 THEN 'Средний'
        ELSE 'Премиум'
    END;
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Часть 2. Выполнение запросов (15 баллов)

### Задание 2.1. SELECT INTO с проверкой FOUND

Создайте функцию `get_book_by_id(p_id INTEGER)`, которая:
- Возвращает `title` и `price` книги
- Если книга не найдена, возвращает сообщение `'Книга не найдена'`
- Используйте `SELECT INTO` и проверку `FOUND`

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION get_book_by_id(p_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_title books.title%TYPE;
    v_price books.price%TYPE;
BEGIN
    SELECT title, price INTO v_title, v_price
    FROM books
    WHERE id = p_id;
    
    IF NOT FOUND THEN
        RETURN 'Книга не найдена';
    END IF;
    
    RETURN format('Книга: %s, цена: %s руб.', v_title, v_price);
END;
$$ LANGUAGE plpgsql;
```
</details>

---

### Задание 2.2. RETURN QUERY — список дорогих книг

Создайте функцию `get_expensive_books(p_min_price DECIMAL)`, которая возвращает таблицу с колонками: `title`, `price`, `category`.

**Требования:**
- Использовать `RETURNS TABLE(...)`
- Использовать `RETURN QUERY`
- Колонка `category` вычисляется через функцию `price_category()` из задания 1.2

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION get_expensive_books(p_min_price DECIMAL)
RETURNS TABLE(
    title TEXT,
    price DECIMAL,
    category TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT b.title::TEXT, b.price, price_category(b.price)
    FROM books b
    WHERE b.price > p_min_price
    ORDER BY b.price DESC;
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
SELECT * FROM get_expensive_books(300);
```

---

### Задание 2.3. PERFORM для запросов без результата

Создайте функцию `cleanup_zero_stock()`, которая:
- Удаляет книги с нулевым остатком (stock = 0)
- Использует `PERFORM` для проверки, есть ли такие книги
- Выводит `RAISE NOTICE` с количеством удалённых книг

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION cleanup_zero_stock()
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    -- Проверяем, есть ли книги с нулевым остатком
    PERFORM 1 FROM books WHERE stock = 0 LIMIT 1;
    
    IF NOT FOUND THEN
        RAISE NOTICE 'Нет книг с нулевым остатком';
        RETURN 0;
    END IF;
    
    WITH deleted AS (
        DELETE FROM books WHERE stock = 0 RETURNING *
    )
    SELECT COUNT(*) INTO deleted_count FROM deleted;
    
    RAISE NOTICE 'Удалено книг: %', deleted_count;
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Часть 3. Курсоры (15 баллов)

### Задание 3.1. Простой курсор с LOOP

Создайте функцию `increase_price_for_author(p_author_id INTEGER, p_percent DECIMAL)`, которая:
- Использует курсор для перебора книг указанного автора
- Увеличивает цену каждой книги на `p_percent` процентов
- Выводит `RAISE NOTICE` для каждой обновлённой книги

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION increase_price_for_author(
    p_author_id INTEGER,
    p_percent DECIMAL
)
RETURNS INTEGER AS $$
DECLARE
    book_cursor CURSOR FOR
        SELECT b.id, b.title, b.price
        FROM books b
        JOIN book_authors ba ON b.id = ba.book_id
        WHERE ba.author_id = p_author_id
        FOR UPDATE;
    
    v_book RECORD;
    v_new_price DECIMAL;
    v_counter INTEGER := 0;
BEGIN
    OPEN book_cursor;
    
    LOOP
        FETCH book_cursor INTO v_book;
        EXIT WHEN NOT FOUND;
        
        v_new_price := v_book.price * (1 + p_percent / 100);
        
        UPDATE books SET price = v_new_price
        WHERE CURRENT OF book_cursor;
        
        v_counter := v_counter + 1;
        RAISE NOTICE 'Книга "%s": цена была %.2f, стала %.2f',
            v_book.title, v_book.price, v_new_price;
    END LOOP;
    
    CLOSE book_cursor;
    RETURN v_counter;
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
SELECT * FROM books WHERE id IN (SELECT book_id FROM book_authors WHERE author_id = 1);
SELECT increase_price_for_author(1, 10);  -- увеличить книги Оруэлла на 10%
SELECT * FROM books WHERE id IN (SELECT book_id FROM book_authors WHERE author_id = 1);
```

---

### Задание 3.2. Курсор с параметрами

Создайте функцию `process_sales_batch(p_batch_size INTEGER)`, которая:
- Использует курсор с параметром для перебора продаж
- Обрабатывает продажи порциями (batch)
- Для каждой продажи выводит информацию

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION process_sales_batch(p_batch_size INTEGER)
RETURNS INTEGER AS $$
DECLARE
    sales_cursor CURSOR (batch INTEGER) FOR
        SELECT id, book_id, customer_id, quantity
        FROM sales
        LIMIT batch;
    
    v_sale RECORD;
    v_total_processed INTEGER := 0;
BEGIN
    OPEN sales_cursor(p_batch_size);
    
    LOOP
        FETCH sales_cursor INTO v_sale;
        EXIT WHEN NOT FOUND;
        
        RAISE NOTICE 'Продажа ID: %, книга: %, покупатель: %, кол-во: %',
            v_sale.id, v_sale.book_id, v_sale.customer_id, v_sale.quantity;
        
        v_total_processed := v_total_processed + 1;
    END LOOP;
    
    CLOSE sales_cursor;
    RETURN v_total_processed;
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Часть 4. Динамические команды (EXECUTE) (10 баллов)

### Задание 4.1. Динамический SELECT

Создайте функцию `dynamic_select(p_table_name TEXT, p_column_name TEXT, p_id INTEGER)`, которая:
- Динамически выполняет запрос `SELECT column_name FROM table_name WHERE id = p_id`
- Использует `FORMAT` с `%I` для безопасной подстановки имён
- Использует `%L` для безопасной подстановки значения
- Возвращает значение в виде TEXT

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION dynamic_select(
    p_table_name TEXT,
    p_column_name TEXT,
    p_id INTEGER
)
RETURNS TEXT AS $$
DECLARE
    v_result TEXT;
    v_query TEXT;
BEGIN
    v_query := FORMAT('SELECT %I FROM %I WHERE id = %L', 
                      p_column_name, p_table_name, p_id);
    
    RAISE NOTICE 'Выполняется запрос: %', v_query;
    
    EXECUTE v_query INTO v_result;
    
    RETURN COALESCE(v_result, 'Не найдено');
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
SELECT dynamic_select('books', 'title', 1);   -- '1984'
SELECT dynamic_select('authors', 'first_name', 2);  -- 'Лев'
```

---

### Задание 4.2. Динамическое обновление

Создайте функцию `dynamic_update(p_table_name TEXT, p_column_name TEXT, p_value TEXT, p_id INTEGER)`, которая:
- Динамически обновляет указанную колонку
- Возвращает количество обновлённых строк
- Использует `EXECUTE` с `USING` для безопасности

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION dynamic_update(
    p_table_name TEXT,
    p_column_name TEXT,
    p_value TEXT,
    p_id INTEGER
)
RETURNS INTEGER AS $$
DECLARE
    v_query TEXT;
    v_result INTEGER;
BEGIN
    v_query := FORMAT('UPDATE %I SET %I = $1 WHERE id = $2', 
                      p_table_name, p_column_name);
    
    RAISE NOTICE 'Выполняется запрос: %', v_query;
    
    EXECUTE v_query USING p_value, p_id;
    GET DIAGNOSTICS v_result = ROW_COUNT;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Часть 5. Массивы (10 баллов)

### Задание 5.1. Функция с массивом ID

Создайте функцию `get_books_by_ids(p_book_ids INTEGER[])`, которая:
- Принимает массив ID книг
- Возвращает таблицу с названиями и ценами
- Использует `= ANY()` для поиска

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION get_books_by_ids(p_book_ids INTEGER[])
RETURNS TABLE(title TEXT, price DECIMAL) AS $$
BEGIN
    RETURN QUERY
    SELECT b.title, b.price
    FROM books b
    WHERE b.id = ANY(p_book_ids)
    ORDER BY b.id;
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
SELECT * FROM get_books_by_ids(ARRAY[1, 3]);
```

---

### Задание 5.2. Работа с массивом в цикле

Создайте функцию `sum_prices_by_ids(p_book_ids INTEGER[])`, которая:
- Принимает массив ID книг
- В цикле проходит по массиву, находит цену каждой книги и суммирует
- Возвращает общую сумму

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION sum_prices_by_ids(p_book_ids INTEGER[])
RETURNS DECIMAL AS $$
DECLARE
    v_total DECIMAL := 0;
    v_price DECIMAL;
    v_idx INTEGER;
BEGIN
    FOR v_idx IN 1..array_length(p_book_ids, 1) LOOP
        SELECT price INTO v_price
        FROM books
        WHERE id = p_book_ids[v_idx];
        
        IF FOUND THEN
            v_total := v_total + v_price;
            RAISE NOTICE 'Книга ID %: цена % (сумма: %)', 
                p_book_ids[v_idx], v_price, v_total;
        ELSE
            RAISE NOTICE 'Книга ID % не найдена', p_book_ids[v_idx];
        END IF;
    END LOOP;
    
    RETURN v_total;
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Часть 6. Обработка ошибок (10 баллов)

### Задание 6.1. Безопасное удаление с обработкой ошибок

Создайте функцию `safe_delete_book(p_book_id INTEGER)`, которая:
- Удаляет книгу по ID
- Если книга не найдена, возвращает сообщение
- Если возникает ошибка внешнего ключа (книга есть в продажах), перехватывает её и возвращает понятное сообщение

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION safe_delete_book(p_book_id INTEGER)
RETURNS TEXT AS $$
BEGIN
    DELETE FROM books WHERE id = p_book_id;
    
    IF NOT FOUND THEN
        RETURN format('Книга с ID % не найдена', p_book_id);
    END IF;
    
    RETURN format('Книга с ID % успешно удалена', p_book_id);
    
EXCEPTION
    WHEN foreign_key_violation THEN
        RETURN format('Нельзя удалить книгу с ID %: есть связанные продажи', p_book_id);
    WHEN OTHERS THEN
        RETURN format('Ошибка при удалении: %', SQLERRM);
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
SELECT safe_delete_book(1);  -- ошибка (есть продажи)
SELECT safe_delete_book(99); -- не найдена
```

---

### Задание 6.2. ASSERT для проверки

Создайте функцию `create_order(p_customer_id INTEGER, p_book_id INTEGER, p_quantity INTEGER)`, которая:
- Проверяет, что количество > 0 (через `ASSERT`)
- Проверяет, что товар есть в наличии
- Если товара недостаточно, выбрасывает исключение

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION create_order(
    p_customer_id INTEGER,
    p_book_id INTEGER,
    p_quantity INTEGER
)
RETURNS INTEGER AS $$
DECLARE
    v_price DECIMAL;
    v_stock INTEGER;
BEGIN
    ASSERT p_quantity > 0, 'Количество должно быть положительным';
    
    SELECT price, stock INTO v_price, v_stock
    FROM books WHERE id = p_book_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Книга с ID % не найдена', p_book_id;
    END IF;
    
    IF v_stock < p_quantity THEN
        RAISE EXCEPTION 'Недостаточно товара. В наличии: %, заказано: %',
            v_stock, p_quantity;
    END IF;
    
    UPDATE books SET stock = stock - p_quantity WHERE id = p_book_id;
    
    INSERT INTO sales (book_id, customer_id, quantity, price_at_sale)
    VALUES (p_book_id, p_customer_id, p_quantity, v_price)
    RETURNING id INTO p_book_id;
    
    RETURN p_book_id;
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Часть 7. Триггеры (15 баллов)

### Задание 7.1. BEFORE INSERT — автоматическое обновление даты

Добавьте в таблицу `books` колонку `updated_at`:
```sql
ALTER TABLE books ADD COLUMN updated_at TIMESTAMP DEFAULT NOW();
```

Создайте триггер, который автоматически обновляет `updated_at` при каждом изменении книги.

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
-- Функция триггера
CREATE OR REPLACE FUNCTION update_books_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Триггер
CREATE TRIGGER books_update_timestamp
    BEFORE UPDATE ON books
    FOR EACH ROW
    EXECUTE FUNCTION update_books_timestamp();
```
</details>

**Проверка:**
```sql
UPDATE books SET price = 400 WHERE id = 1;
SELECT id, title, price, updated_at FROM books;
```

---

### Задание 7.2. AFTER INSERT — аудит продаж

Создайте таблицу `sales_audit`:
```sql
CREATE TABLE sales_audit (
    id SERIAL PRIMARY KEY,
    sale_id INTEGER,
    book_title TEXT,
    customer_name TEXT,
    quantity INTEGER,
    total DECIMAL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

Создайте триггер, который при каждой новой продаже записывает информацию в `sales_audit`.

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION audit_new_sale()
RETURNS TRIGGER AS $$
DECLARE
    v_book_title TEXT;
    v_customer_name TEXT;
BEGIN
    SELECT title INTO v_book_title FROM books WHERE id = NEW.book_id;
    SELECT name INTO v_customer_name FROM customers WHERE id = NEW.customer_id;
    
    INSERT INTO sales_audit (sale_id, book_title, customer_name, quantity, total)
    VALUES (NEW.id, v_book_title, v_customer_name, NEW.quantity, 
            NEW.quantity * NEW.price_at_sale);
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sales_audit_trigger
    AFTER INSERT ON sales
    FOR EACH ROW
    EXECUTE FUNCTION audit_new_sale();
```
</details>

**Проверка:**
```sql
INSERT INTO sales (book_id, customer_id, quantity, price_at_sale) 
VALUES (1, 1, 2, 350.00);
SELECT * FROM sales_audit;
```

---

### Задание 7.3. BEFORE UPDATE — валидация цены

Создайте триггер, который:
- Не позволяет установить цену ниже 100 рублей
- Не позволяет повысить цену более чем на 50% за одно обновление

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION validate_book_price()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.price < 100 THEN
        RAISE EXCEPTION 'Цена книги не может быть ниже 100 руб. (попытка: %)', NEW.price;
    END IF;
    
    IF NEW.price > OLD.price * 1.5 THEN
        RAISE EXCEPTION 'Цену нельзя повышать более чем на 50%% за раз (было: %, стало: %)',
            OLD.price, NEW.price;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER books_validate_price
    BEFORE UPDATE ON books
    FOR EACH ROW
    EXECUTE FUNCTION validate_book_price();
```
</details>

**Проверка:**
```sql
UPDATE books SET price = 50 WHERE id = 1;   -- ошибка!
UPDATE books SET price = 1000 WHERE id = 1; -- ошибка! (более 50% от 350)
```

---

## Часть 8. Отладка (15 баллов)

### Задание 8.1. Функция с отладочным выводом

Создайте функцию `debug_sales_summary()`, которая:
- Выводит `RAISE NOTICE` с информацией о:
  - Общем количестве продаж
  - Общей выручке
  - Самой популярной книге
- Использует разные уровни (`DEBUG`, `LOG`, `INFO`, `NOTICE`)

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION debug_sales_summary()
RETURNS VOID AS $$
DECLARE
    v_total_sales INTEGER;
    v_total_revenue DECIMAL;
    v_best_book TEXT;
    v_best_count INTEGER;
BEGIN
    RAISE DEBUG 'Начало выполнения debug_sales_summary()';
    
    SELECT COUNT(*), COALESCE(SUM(quantity * price_at_sale), 0)
    INTO v_total_sales, v_total_revenue
    FROM sales;
    
    RAISE INFO 'Всего продаж: %, общая выручка: % руб.', 
        v_total_sales, v_total_revenue;
    
    SELECT b.title, SUM(s.quantity)
    INTO v_best_book, v_best_count
    FROM sales s
    JOIN books b ON b.id = s.book_id
    GROUP BY b.id
    ORDER BY SUM(s.quantity) DESC
    LIMIT 1;
    
    RAISE NOTICE 'Самая популярная книга: "%" (продано % экз.)', 
        v_best_book, v_best_count;
    
    RAISE LOG 'Завершение выполнения debug_sales_summary()';
END;
$$ LANGUAGE plpgsql;
```
</details>

**Проверка:**
```sql
-- Включить вывод DEBUG
SET client_min_messages = 'DEBUG';
SELECT debug_sales_summary();
```

---

### Задание 8.2. Функция с ASSERT для отладки

Создайте функцию `debug_check_integrity()`, которая:
- Проверяет, что у каждой книги есть хотя бы один автор
- Проверяет, что все продажи ссылаются на существующие книги
- Использует `ASSERT` для проверок
- При ошибке выводит детальную информацию

```sql
-- Ваш код здесь
```

<details>
<summary>Решение</summary>

```sql
CREATE OR REPLACE FUNCTION debug_check_integrity()
RETURNS TEXT AS $$
DECLARE
    v_books_without_authors INTEGER;
    v_orphan_sales INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO v_books_without_authors
    FROM books b
    LEFT JOIN book_authors ba ON b.id = ba.book_id
    WHERE ba.book_id IS NULL;
    
    ASSERT v_books_without_authors = 0, 
        format('Найдено книг без авторов: %s', v_books_without_authors);
    
    SELECT COUNT(*)
    INTO v_orphan_sales
    FROM sales s
    LEFT JOIN books b ON b.id = s.book_id
    WHERE b.id IS NULL;
    
    ASSERT v_orphan_sales = 0,
        format('Найдено продаж с несуществующими книгами: %s', v_orphan_sales);
    
    RETURN 'Целостность данных в порядке';
EXCEPTION
    WHEN assert_failure THEN
        RETURN format('Ошибка целостности: %', SQLERRM);
END;
$$ LANGUAGE plpgsql;
```
</details>

---

## Проверка выполнения

### Создайте отчёт о выполненной работе

```sql
-- Список всех созданных функций
\df

-- Список триггеров
SELECT tgname, relname FROM pg_trigger WHERE tgname NOT LIKE 'pg%';

-- Проверка всех функций (должны выполняться без ошибок)
SELECT apply_discount(1000, 20);
SELECT price_category(500);
SELECT get_book_by_id(1);
SELECT * FROM get_expensive_books(300);
SELECT cleanup_zero_stock();
SELECT increase_price_for_author(1, 5);
SELECT process_sales_batch(10);
SELECT dynamic_select('books', 'title', 1);
SELECT dynamic_update('books', 'price', '999', 1);
SELECT * FROM get_books_by_ids(ARRAY[1, 3]);
SELECT sum_prices_by_ids(ARRAY[1, 2, 3]);
SELECT safe_delete_book(99);
SELECT create_order(1, 3, 1);
SELECT debug_sales_summary();
SELECT debug_check_integrity();
```



## Сдача задания

1. Сохраните все ваши решения в файл `plpgsql_bookstore.sql`
2. Выполните все проверочные запросы
3. Убедитесь, что нет ошибок
4. Отправьте файл на проверку

```bash
# Дамп всех функций
pg_dump bookstore --schema-only --file=plpgsql_bookstore.sql
```