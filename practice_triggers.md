# Триггеры в PostgreSQL — полное объяснение для новичка

## Что такое триггер простыми словами

**Триггер** — это специальная хранимая функция, которая **автоматически запускается** (срабатывает) при выполнении определённых действий с таблицей: `INSERT`, `UPDATE` или `DELETE`.

### Аналогия из жизни

Представьте автоматическую дверь в супермаркете:

| Обычная дверь | Дверь с триггером |
|---------------|-------------------|
| Вы подходите, дверь не реагирует | Вы подходите → **триггер срабатывает** → дверь открывается |
| Нужно толкать руками | Всё происходит автоматически |
| Можно забыть закрыть | Закрывается сама |

**Триггер** — это как датчик движения: когда случается событие (кто-то подошёл), автоматически выполняется действие (открыть дверь).

---

## Зачем нужны триггеры

### Реальные задачи, которые решают триггеры

| Задача | Без триггера | С триггером |
|--------|--------------|-------------|
| **Автоматически ставить дату обновления** | Каждый раз писать вручную `UPDATE ... SET updated_at = NOW()` | Триггер сам добавляет дату при любом изменении |
| **Проверка остатков при продаже** | Приложение должно сначала проверить, потом продать (2 запроса) | Триггер проверяет и отклоняет продажу, если товара нет |
| **Логирование изменений** | Везде в коде добавлять `INSERT INTO log` | Триггер сам пишет в лог при каждом изменении |
| **Поддержание суммы заказа** | Пересчитывать вручную при каждой позиции | Триггер пересчитывает автоматически |

### Пример: без триггера (плохо)

Ваше приложение должно обновить `updated_at`:

```python
# Приложение на Python
cursor.execute("""
    UPDATE users 
    SET name = 'Новое имя', updated_at = NOW() 
    WHERE id = 1
""")
```

**Проблемы:**
- Забыли указать `updated_at` в одном месте — данные устарели
- 100 мест в коде — 100 раз нужно не забыть
- При смене логики нужно менять 100 мест

### Пример: с триггером (хорошо)

```sql
-- Создаётся один раз в БД
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

Теперь приложение пишет просто:
```python
cursor.execute("UPDATE users SET name = 'Новое имя' WHERE id = 1")
-- updated_at обновится АВТОМАТИЧЕСКИ!
```

---

## Основные понятия триггеров

### Время срабатывания

| Время | Когда срабатывает | Что можно делать |
|-------|-------------------|------------------|
| `BEFORE` | ДО выполнения операции | Изменить данные (`NEW`), проверить, отменить операцию |
| `AFTER` | ПОСЛЕ выполнения операции | Логировать, отправлять уведомления (данные уже сохранены) |
| `INSTEAD OF` | ВМЕСТО операции | Только для представлений (`VIEW`) — своя логика |

### События

| Событие | Когда срабатывает |
|---------|-------------------|
| `INSERT` | При добавлении строки |
| `UPDATE` | При изменении строки |
| `DELETE` | При удалении строки |
| `TRUNCATE` | При очистке таблицы |

### Для каждой строки или для всего запроса

| Тип | Срабатывает | Пример |
|-----|-------------|--------|
| `FOR EACH ROW` | Для КАЖДОЙ изменённой строки | UPDATE 100 строк → триггер вызовется 100 раз |
| `FOR EACH STATEMENT` | Один раз на запрос | UPDATE 100 строк → триггер вызовется 1 раз |

---

## Специальные переменные внутри триггера

Внутри триггерной функции доступны специальные переменные:

| Переменная | Что содержит | Доступна в |
|------------|--------------|------------|
| `NEW` | Новая версия строки (после изменений) | INSERT, UPDATE |
| `OLD` | Старая версия строки (до изменений) | UPDATE, DELETE |
| `TG_OP` | Какая операция: 'INSERT', 'UPDATE', 'DELETE' | Все |
| `TG_TABLE_NAME` | Имя таблицы | Все |

### Как получить данные из NEW и OLD

```sql
NEW.column_name  -- новая версия колонки
OLD.column_name  -- старая версия колонки
```

---

## Простейший пример: триггер на дату обновления

### Шаг 1. Создаём таблицу

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Шаг 2. Создаём триггерную функцию

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    -- NEW.updated_at — это новая версия колонки updated_at
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;  -- обязательно вернуть NEW!
END;
$$ LANGUAGE plpgsql;
```

### Шаг 3. Создаём триггер

```sql
CREATE TRIGGER trigger_users_updated_at
    BEFORE UPDATE ON users        -- перед обновлением таблицы users
    FOR EACH ROW                  -- для каждой строки
    EXECUTE FUNCTION update_updated_at();
```

### Шаг 4. Проверяем

```sql
INSERT INTO users (name) VALUES ('Иван');
SELECT id, name, updated_at FROM users;
-- updated_at = 2024-01-15 10:00:00

UPDATE users SET name = 'Иван Петров' WHERE id = 1;
SELECT id, name, updated_at FROM users;
-- updated_at = 2024-01-15 10:00:05 (обновилось автоматически!)
```

---

## Практический пример 1: Проверка возраста

```sql
-- Таблица пользователей
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    birth_date DATE,
    age INTEGER
);

-- Триггерная функция: вычисляет возраст автоматически
CREATE OR REPLACE FUNCTION calculate_age()
RETURNS TRIGGER AS $$
BEGIN
    -- Вычисляем возраст на основе даты рождения
    NEW.age = EXTRACT(YEAR FROM age(CURRENT_DATE, NEW.birth_date));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Триггер: перед вставкой или обновлением
CREATE TRIGGER trigger_calculate_age
    BEFORE INSERT OR UPDATE OF birth_date ON users
    FOR EACH ROW
    EXECUTE FUNCTION calculate_age();

-- Проверка
INSERT INTO users (name, birth_date) VALUES ('Анна', '1990-05-15');
SELECT * FROM users;
-- age = 34 (вычислилось автоматически!)
```

---

## Практический пример 2: Проверка остатков товара

```sql
-- Таблица товаров
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    stock INTEGER CHECK (stock >= 0)
);

-- Таблица продаж
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER CHECK (quantity > 0),
    sale_date TIMESTAMP DEFAULT NOW()
);

-- Триггерная функция: проверяет остаток перед продажей
CREATE OR REPLACE FUNCTION check_stock()
RETURNS TRIGGER AS $$
DECLARE
    current_stock INTEGER;
BEGIN
    -- Получаем текущий остаток
    SELECT stock INTO current_stock 
    FROM products WHERE id = NEW.product_id;
    
    -- Если остатка недостаточно — отклоняем операцию
    IF current_stock < NEW.quantity THEN
        RAISE EXCEPTION 'Недостаточно товара! В наличии: %, заказано: %', 
            current_stock, NEW.quantity;
    END IF;
    
    -- Уменьшаем остаток
    UPDATE products 
    SET stock = stock - NEW.quantity 
    WHERE id = NEW.product_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Триггер: перед каждой вставкой в sales
CREATE TRIGGER trigger_check_stock
    BEFORE INSERT ON sales
    FOR EACH ROW
    EXECUTE FUNCTION check_stock();

-- Проверка
INSERT INTO products (name, stock) VALUES ('Ноутбук', 5);

-- Это сработает
INSERT INTO sales (product_id, quantity) VALUES (1, 3);

-- Это вызовет ошибку (заказано 10, в наличии 2)
INSERT INTO sales (product_id, quantity) VALUES (1, 10);
-- ERROR: Недостаточно товара! В наличии: 2, заказано: 10
```

---

## Практический пример 3: Логирование изменений (аудит)

```sql
-- Основная таблица
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    salary DECIMAL(10,2)
);

-- Таблица для логов
CREATE TABLE employees_log (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER,
    action VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by VARCHAR(100)
);

-- Триггерная функция для логирования
CREATE OR REPLACE FUNCTION log_employee_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'DELETE') THEN
        INSERT INTO employees_log (employee_id, action, old_data, changed_by)
        VALUES (OLD.id, 'DELETE', row_to_json(OLD), current_user);
        RETURN OLD;
        
    ELSIF (TG_OP = 'UPDATE') THEN
        INSERT INTO employees_log (employee_id, action, old_data, new_data, changed_by)
        VALUES (NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_user);
        RETURN NEW;
        
    ELSIF (TG_OP = 'INSERT') THEN
        INSERT INTO employees_log (employee_id, action, new_data, changed_by)
        VALUES (NEW.id, 'INSERT', row_to_json(NEW), current_user);
        RETURN NEW;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Триггер на все операции
CREATE TRIGGER trigger_employees_log
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION log_employee_changes();

-- Проверка
INSERT INTO employees (name, salary) VALUES ('Иван', 50000);
UPDATE employees SET salary = 55000 WHERE id = 1;
DELETE FROM employees WHERE id = 1;

-- Смотрим логи
SELECT * FROM employees_log;
```

---

## Практический пример 4: Автоматический расчёт суммы заказа

```sql
-- Таблица заказов
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    total DECIMAL(10,2) DEFAULT 0
);

-- Таблица позиций заказа
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    product_name VARCHAR(100),
    quantity INTEGER,
    price DECIMAL(10,2)
);

-- Триггерная функция: пересчитывает сумму заказа
CREATE OR REPLACE FUNCTION recalc_order_total()
RETURNS TRIGGER AS $$
DECLARE
    new_total DECIMAL;
BEGIN
    -- Вычисляем новую сумму
    SELECT COALESCE(SUM(quantity * price), 0) INTO new_total
    FROM order_items
    WHERE order_id = COALESCE(NEW.order_id, OLD.order_id);
    
    -- Обновляем заказ
    UPDATE orders 
    SET total = new_total 
    WHERE id = COALESCE(NEW.order_id, OLD.order_id);
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Триггер на все изменения в позициях
CREATE TRIGGER trigger_recalc_order
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION recalc_order_total();

-- Проверка
INSERT INTO orders DEFAULT VALUES;
INSERT INTO order_items (order_id, product_name, quantity, price) 
VALUES (1, 'Книга', 2, 500);
SELECT * FROM orders WHERE id = 1;  -- total = 1000

INSERT INTO order_items (order_id, product_name, quantity, price) 
VALUES (1, 'Ручка', 3, 50);
SELECT * FROM orders WHERE id = 1;  -- total = 1150
```

---

## Управление триггерами

### Посмотреть все триггеры

```sql
-- Все триггеры в текущей БД
SELECT tgname, relname, tgtype 
FROM pg_trigger 
JOIN pg_class ON tgrelid = pg_class.oid
WHERE tgname NOT LIKE 'pg%';

-- Триггеры конкретной таблицы
\d employees
```

### Включить/выключить триггер

```sql
-- Отключить триггер
ALTER TABLE employees DISABLE TRIGGER trigger_employees_log;

-- Включить снова
ALTER TABLE employees ENABLE TRIGGER trigger_employees_log;

-- Отключить все триггеры таблицы
ALTER TABLE employees DISABLE TRIGGER ALL;

-- Включить все
ALTER TABLE employees ENABLE TRIGGER ALL;
```

### Удалить триггер

```sql
DROP TRIGGER trigger_users_updated_at ON users;
```

---

## ВАЖНОЕ ограничение: триггеры и большие объёмы данных

Если вы делаете `UPDATE` 1 000 000 строк, и у вас триггер с `FOR EACH ROW`:

```sql
CREATE TRIGGER slow_trigger
    AFTER UPDATE ON big_table
    FOR EACH ROW  -- <- вызовется 1 000 000 раз!
    EXECUTE FUNCTION do_something();
```

**Что произойдёт:**
- Триггер выполнится 1 миллион раз
- Операция может длиться минуты вместо секунд
- БД может зависнуть

**Как решить:**
1. Используйте `FOR EACH STATEMENT`, если возможно
2. Временно отключайте триггер для массовых операций
3. Используйте `DISABLE TRIGGER` → сделать UPDATE → `ENABLE TRIGGER`

---

## Типичные ошибки новичков

| Ошибка | Почему | Решение |
|--------|--------|---------|
| Забыл `RETURN NEW` в `BEFORE` триггере | Строка не сохраняется | Всегда возвращайте `NEW` |
| Забыл `RETURN OLD` в `DELETE` триггере | Ошибка выполнения | Возвращайте `OLD` |
| Бесконечный цикл триггеров | Триггер вызывает сам себя | Добавьте проверку `IF TG_OP = 'UPDATE'` |
| Слишком много логики в триггере | Медленная работа | Оставляйте только необходимое |
| Не учёл NULL в `NEW` | Неправильные данные | Используйте `IF NEW.column IS NOT NULL` |

### Пример бесконечного цикла (НЕ ДЕЛАЙТЕ ТАК)

```sql
-- Этот триггер вызовет сам себя бесконечно!
CREATE FUNCTION bad_trigger() RETURNS TRIGGER AS $$
BEGIN
    UPDATE users SET name = NEW.name WHERE id = NEW.id;  -- снова вызовет триггер
    RETURN NEW;
END; $$ LANGUAGE plpgsql;
```

### Правильное решение

```sql
CREATE FUNCTION good_trigger() RETURNS TRIGGER AS $$
BEGIN
    -- Делаем что-то, что не вызывает триггер
    INSERT INTO log (user_id, action) VALUES (NEW.id, 'update');
    RETURN NEW;
END; $$ LANGUAGE plpgsql;
```

---

## Полная шпаргалка по созданию триггера

```sql
-- Шаг 1: Создать функцию (обязательно RETURNS TRIGGER)
CREATE OR REPLACE FUNCTION my_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    -- Ваша логика здесь
    -- Доступно: NEW, OLD, TG_OP, TG_TABLE_NAME
    
    RETURN NEW;  -- для BEFORE INSERT/UPDATE
    -- RETURN OLD;  -- для BEFORE DELETE
    -- RETURN NULL; -- для AFTER-триггеров
END;
$$ LANGUAGE plpgsql;

-- Шаг 2: Привязать функцию к таблице
CREATE TRIGGER my_trigger_name
    {BEFORE | AFTER | INSTEAD OF}
    {INSERT | UPDATE | DELETE | TRUNCATE}
    ON table_name
    [FOR EACH ROW | FOR EACH STATEMENT]
    EXECUTE FUNCTION my_trigger_function();
```

---

## Когда использовать триггеры (и когда НЕ использовать)

### ✅ Хорошо использовать:
- Автоматическое обновление `updated_at`
- Поддержание согласованности данных (каскадные обновления)
- Аудит и логирование изменений
- Автоматические вычисления (total, age, full_name)
- Валидация сложных бизнес-правил
- Ограничения, которые нельзя выразить через `CHECK`

### ❌ Плохо использовать:
- Вместо нормализации данных (переделайте схему)
- Для массовых операций с миллионами строк
- Для вызова внешних API (HTTP, очереди) — медленно и ненадёжно
- Для логики, которая часто меняется (проще в приложении)
- Когда можно обойтись `GENERATED ALWAYS` (вычисляемые колонки)

---

## Итог: что нужно запомнить

1. **Триггер** — это функция, которая запускается автоматически при `INSERT/UPDATE/DELETE`
2. **BEFORE** — можно изменить данные (`NEW`), **AFTER** — нельзя, но запись уже сохранена
3. **FOR EACH ROW** — для каждой строки, **FOR EACH STATEMENT** — один раз на запрос
4. Всегда проверяйте, не создаёте ли **бесконечный цикл**
5. **Триггеры невидимы** в коде приложения — об этом легко забыть

---
