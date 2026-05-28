# Триггеры в PostgreSQL — полное объяснение

## Введение

**Триггер** (trigger) — это специальная функция, которая **автоматически запускается** при выполнении определённых операций с таблицей: `INSERT`, `UPDATE`, `DELETE` или `TRUNCATE`.

### Простая аналогия

| Аналогия | Описание |
|----------|----------|
| **Автоматическая дверь** | Вы подходите → дверь открывается (сработал триггер) |
| **Охранная сигнализация** | Кто-то открыл дверь → завыла сирена |
| **Автоответчик** | Звонок → автоматически включается запись |

**Триггер** — это правило: "когда происходит событие X, автоматически выполни действие Y".

---

## Часть 1. Основные понятия

### 1.1. Терминология

| Термин | Что означает | Пример |
|--------|--------------|--------|
| **Триггерное событие** | Когда срабатывает | `INSERT`, `UPDATE`, `DELETE` |
| **Триггерная функция** | Что выполняется | Функция, возвращающая `TRIGGER` |
| **Время срабатывания** | До или после события | `BEFORE` / `AFTER` / `INSTEAD OF` |
| **Частота срабатывания** | Для каждой строки или один раз | `FOR EACH ROW` / `FOR EACH STATEMENT` |

### 1.2. Жизненный цикл триггера

```
┌─────────────────────────────────────────────────────────────────┐
│                    ЖИЗНЕННЫЙ ЦИКЛ ТРИГГЕРА                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Пользователь выполняет INSERT/UPDATE/DELETE                │
│                         │                                        │
│                         ▼                                        │
│   2. PostgreSQL проверяет, есть ли триггер на эту операцию      │
│                         │                                        │
│                         ▼                                        │
│   3. Если есть BEFORE-триггер → выполнить его                   │
│                         │                                        │
│                         ▼                                        │
│   4. Выполняется сама операция (INSERT/UPDATE/DELETE)           │
│                         │                                        │
│                         ▼                                        │
│   5. Если есть AFTER-триггер → выполнить его                    │
│                         │                                        │
│                         ▼                                        │
│   6. Результат возвращается пользователю                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Часть 2. Синтаксис создания триггера

### 2.1. Два шага создания триггера

```sql
-- ШАГ 1: Создать триггерную функцию (возвращает TRIGGER)
CREATE OR REPLACE FUNCTION имя_функции()
RETURNS TRIGGER AS $$
BEGIN
    -- логика триггера
    RETURN NEW;  -- или OLD, или NULL
END;
$$ LANGUAGE plpgsql;

-- ШАГ 2: Привязать функцию к таблице и событию
CREATE TRIGGER имя_триггера
    {BEFORE | AFTER | INSTEAD OF}
    {INSERT | UPDATE | DELETE | TRUNCATE}
    ON имя_таблицы
    [FOR EACH ROW | FOR EACH STATEMENT]
    [WHEN (условие)]
    EXECUTE FUNCTION имя_функции();
```

### 2.2. Самый простой триггер

```sql
-- Таблица
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    name TEXT,
    created_at TIMESTAMP
);

-- ШАГ 1: Функция
CREATE OR REPLACE FUNCTION set_created_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.created_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ШАГ 2: Триггер
CREATE TRIGGER trg_set_created_at
    BEFORE INSERT ON test
    FOR EACH ROW
    EXECUTE FUNCTION set_created_at();

-- Проверка
INSERT INTO test (name) VALUES ('Иван');
SELECT * FROM test;
-- id | name | created_at
-- 1  | Иван | 2024-01-15 10:30:00  (заполнилось автоматически!)
```

---

## Часть 3. Специальные переменные в триггере

Внутри триггерной функции доступны специальные переменные:

| Переменная | Тип | Что содержит | Доступна в |
|------------|-----|--------------|------------|
| `NEW` | RECORD | Новая строка (после операции) | INSERT, UPDATE |
| `OLD` | RECORD | Старая строка (до операции) | UPDATE, DELETE |
| `TG_NAME` | TEXT | Имя триггера | Все |
| `TG_WHEN` | TEXT | 'BEFORE', 'AFTER', 'INSTEAD OF' | Все |
| `TG_OP` | TEXT | 'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE' | Все |
| `TG_TABLE_NAME` | TEXT | Имя таблицы | Все |
| `TG_TABLE_SCHEMA` | TEXT | Имя схемы | Все |
| `TG_LEVEL` | TEXT | 'ROW' или 'STATEMENT' | Все |
| `TG_NARGS` | INTEGER | Количество аргументов | Все |
| `TG_ARGV[]` | TEXT[] | Аргументы триггера | Все |

### 3.1. Пример использования специальных переменных

```sql
CREATE OR REPLACE FUNCTION log_trigger_info()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE '====== ИНФОРМАЦИЯ О ТРИГГЕРЕ ======';
    RAISE NOTICE 'Имя триггера: %', TG_NAME;
    RAISE NOTICE 'Время: %', TG_WHEN;
    RAISE NOTICE 'Операция: %', TG_OP;
    RAISE NOTICE 'Таблица: %.%', TG_TABLE_SCHEMA, TG_TABLE_NAME;
    RAISE NOTICE 'Уровень: %', TG_LEVEL;
    
    IF TG_OP = 'INSERT' THEN
        RAISE NOTICE 'NEW: %', NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        RAISE NOTICE 'OLD: %', OLD;
        RAISE NOTICE 'NEW: %', NEW;
    ELSIF TG_OP = 'DELETE' THEN
        RAISE NOTICE 'OLD: %', OLD;
    END IF;
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 4. Время срабатывания (BEFORE / AFTER / INSTEAD OF)

### 4.1. BEFORE (ДО операции)

**Срабатывает ДО того, как операция выполнена.**

```sql
-- BEFORE: можем изменить данные (NEW) или отменить операцию
CREATE OR REPLACE FUNCTION before_example()
RETURNS TRIGGER AS $$
BEGIN
    -- Можно изменить значение перед вставкой
    NEW.name := UPPER(TRIM(NEW.name));
    
    -- Можно отменить операцию
    IF NEW.name IS NULL OR NEW.name = '' THEN
        RAISE EXCEPTION 'Имя не может быть пустым';
        RETURN NULL;  -- отмена операции
    END IF;
    
    RETURN NEW;  -- обязательно вернуть NEW для INSERT/UPDATE
END;
$$ LANGUAGE plpgsql;
```

**Что можно делать в BEFORE:**
- ✅ Изменять `NEW` (новую строку)
- ✅ Отменить операцию, вернув `NULL`
- ✅ Выбросить исключение `RAISE EXCEPTION`
- ✅ Проверять данные

### 4.2. AFTER (ПОСЛЕ операции)

**Срабатывает ПОСЛЕ того, как операция выполнена.**

```sql
-- AFTER: данные уже сохранены, изменить нельзя
CREATE OR REPLACE FUNCTION after_example()
RETURNS TRIGGER AS $$
BEGIN
    -- Нельзя изменить NEW или OLD
    -- Можно выполнить дополнительные действия
    
    -- Логирование
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, action, new_data)
        VALUES (TG_TABLE_NAME, 'INSERT', row_to_json(NEW));
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, action, old_data)
        VALUES (TG_TABLE_NAME, 'DELETE', row_to_json(OLD));
    END IF;
    
    RETURN NULL;  -- AFTER-триггеры возвращают NULL
END;
$$ LANGUAGE plpgsql;
```

**Что можно делать в AFTER:**
- ✅ Логировать изменения
- ✅ Отправлять уведомления (NOTIFY)
- ✅ Обновлять агрегированные данные (но осторожно!)
- ❌ Нельзя изменить данные (операция уже выполнена)

### 4.3. INSTEAD OF (ВМЕСТО операции)

**Используется только для представлений (VIEW).**

```sql
-- Представление
CREATE VIEW active_users AS
SELECT * FROM users WHERE is_active = true;

-- Триггер для обновления через представление
CREATE OR REPLACE FUNCTION instead_of_example()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO users (id, name, is_active) 
        VALUES (NEW.id, NEW.name, true);
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE users SET name = NEW.name 
        WHERE id = OLD.id AND is_active = true;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM users WHERE id = OLD.id AND is_active = true;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_active_users
    INSTEAD OF INSERT OR UPDATE OR DELETE ON active_users
    FOR EACH ROW
    EXECUTE FUNCTION instead_of_example();
```

---

## Часть 5. Частота срабатывания (FOR EACH ROW / FOR EACH STATEMENT)

### 5.1. FOR EACH ROW (для каждой строки)

**Срабатывает для КАЖДОЙ изменённой строки.**

```sql
CREATE OR REPLACE FUNCTION row_level_trigger()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Обработана строка: %', NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_row_level
    BEFORE UPDATE ON users
    FOR EACH ROW  -- ← для каждой строки
    EXECUTE FUNCTION row_level_trigger();

-- UPDATE 100 строк → триггер вызовется 100 раз!
UPDATE users SET status = 'active' WHERE age > 18;
```

### 5.2. FOR EACH STATEMENT (для всего оператора)

**Срабатывает ОДИН раз на весь запрос.**

```sql
CREATE OR REPLACE FUNCTION statement_level_trigger()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Оператор выполнен, затронуто % строк', TG_NARGS;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_statement_level
    AFTER UPDATE ON users
    FOR EACH STATEMENT  -- ← один раз на запрос
    EXECUTE FUNCTION statement_level_trigger();

-- UPDATE 100 строк → триггер вызовется 1 раз!
UPDATE users SET status = 'active' WHERE age > 18;
```

### 5.3. Сравнение (важно для производительности!)

| Характеристика | FOR EACH ROW | FOR EACH STATEMENT |
|----------------|--------------|-------------------|
| **Срабатываний** | N раз (на каждую строку) | 1 раз |
| **Производительность** | Медленнее при больших UPDATE | Быстро |
| **Доступ к данным** | Есть NEW/OLD для каждой строки | Нет прямого доступа к строкам |
| **Когда использовать** | Логирование, валидация, модификация | Подсчёт изменений, аудит |

```sql
-- ПРИМЕР: ПЛОХО (очень медленно для 1 млн строк)
CREATE TRIGGER bad_trigger
    AFTER UPDATE ON large_table
    FOR EACH ROW  -- 1 000 000 вызовов!
    EXECUTE FUNCTION log_row();

-- ПРИМЕР: ХОРОШО (быстро даже для 1 млн строк)
CREATE TRIGGER good_trigger
    AFTER UPDATE ON large_table
    FOR EACH STATEMENT  -- 1 вызов!
    EXECUTE FUNCTION log_statement();
```

---

## Часть 6. Условные триггеры (WHEN)

### 6.1. Синтаксис

```sql
CREATE TRIGGER имя_триггера
    BEFORE|AFTER операция ON таблица
    FOR EACH ROW
    WHEN (условие)  -- ← только если условие истинно
    EXECUTE FUNCTION функция();
```

### 6.2. Примеры

```sql
-- Триггер срабатывает только при изменении цены
CREATE TRIGGER trg_price_change
    BEFORE UPDATE OF price ON products  -- только при обновлении колонки price
    FOR EACH ROW
    WHEN (OLD.price IS DISTINCT FROM NEW.price)  -- цена реально изменилась
    EXECUTE FUNCTION log_price_change();

-- Триггер для дорогих товаров
CREATE TRIGGER trg_expensive_products
    AFTER INSERT ON products
    FOR EACH ROW
    WHEN (NEW.price > 10000)
    EXECUTE FUNCTION notify_admin();

-- Триггер для новых пользователей из Москвы
CREATE TRIGGER trg_moscow_users
    AFTER INSERT ON users
    FOR EACH ROW
    WHEN (NEW.city = 'Москва')
    EXECUTE FUNCTION send_welcome_sms();
```

### 6.3. WHEN vs IF внутри триггера

```sql
-- Вариант 1: WHEN (выполняется только при условии)
CREATE TRIGGER trg_with_when
    BEFORE UPDATE ON users
    FOR EACH ROW
    WHEN (OLD.email IS DISTINCT FROM NEW.email)  -- PostgreSQL проверяет условие
    EXECUTE FUNCTION log_email_change();

-- Вариант 2: IF (триггер вызывается всегда, но может ничего не делать)
CREATE FUNCTION log_email_change_if()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.email IS DISTINCT FROM NEW.email THEN
        INSERT INTO email_log...;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Разница: WHEN эффективнее (триггерная функция не вызывается, если условие ложно)
```

---

## Часть 7. Полный практический пример

### 7.1. Задача: автоматический аудит изменений

```sql
-- ============================================
-- ШАГ 1: Таблицы
-- ============================================

-- Основная таблица
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    position TEXT,
    salary DECIMAL(10,2),
    department TEXT,
    updated_at TIMESTAMP
);

-- Таблица аудита
CREATE TABLE employees_audit (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER,
    action TEXT,
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT NOW()
);

-- ============================================
-- ШАГ 2: Триггерная функция
-- ============================================

CREATE OR REPLACE FUNCTION audit_employees()
RETURNS TRIGGER AS $$
DECLARE
    v_user TEXT := current_user;
BEGIN
    -- Для INSERT
    IF TG_OP = 'INSERT' THEN
        INSERT INTO employees_audit (employee_id, action, new_data, changed_by)
        VALUES (NEW.id, 'INSERT', row_to_json(NEW), v_user);
        RETURN NEW;
    
    -- Для UPDATE
    ELSIF TG_OP = 'UPDATE' THEN
        -- Логируем только если данные реально изменились
        IF NEW IS DISTINCT FROM OLD THEN
            INSERT INTO employees_audit (employee_id, action, old_data, new_data, changed_by)
            VALUES (NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), v_user);
        END IF;
        
        -- Обновляем updated_at
        NEW.updated_at := NOW();
        RETURN NEW;
    
    -- Для DELETE
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO employees_audit (employee_id, action, old_data, changed_by)
        VALUES (OLD.id, 'DELETE', row_to_json(OLD), v_user);
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- ============================================
-- ШАГ 3: Триггеры
-- ============================================

-- Триггер на INSERT
CREATE TRIGGER employees_insert_audit
    AFTER INSERT ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_employees();

-- Триггер на UPDATE
CREATE TRIGGER employees_update_audit
    BEFORE UPDATE ON employees  -- BEFORE, чтобы обновить updated_at
    FOR EACH ROW
    EXECUTE FUNCTION audit_employees();

-- Триггер на DELETE
CREATE TRIGGER employees_delete_audit
    AFTER DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_employees();

-- ============================================
-- ШАГ 4: Проверка
-- ============================================

INSERT INTO employees (name, position, salary, department)
VALUES ('Иван Петров', 'Разработчик', 100000, 'IT');

UPDATE employees SET salary = 120000 WHERE name = 'Иван Петров';

DELETE FROM employees WHERE name = 'Иван Петров';

SELECT * FROM employees_audit;
```

---

## Часть 8. Управление триггерами

### 8.1. Просмотр триггеров

```sql
-- Все триггеры в базе
SELECT 
    tgname AS trigger_name,
    relname AS table_name,
    tgtype,
    tgisconstraint,
    tgenabled
FROM pg_trigger
JOIN pg_class ON tgrelid = pg_class.oid
WHERE tgname NOT LIKE 'pg%';

-- Триггеры конкретной таблицы
\d employees

-- Детальная информация
SELECT 
    tgname,
    proname AS function_name,
    tgtype::bit(16) AS trigger_type_bits
FROM pg_trigger t
JOIN pg_proc p ON t.tgfoid = p.oid
WHERE tgname = 'employees_update_audit';
```

### 8.2. Включение/выключение триггеров

```sql
-- Отключить конкретный триггер
ALTER TABLE employees DISABLE TRIGGER employees_update_audit;

-- Включить обратно
ALTER TABLE employees ENABLE TRIGGER employees_update_audit;

-- Отключить ВСЕ триггеры на таблице
ALTER TABLE employees DISABLE TRIGGER ALL;

-- Включить все триггеры
ALTER TABLE employees ENABLE TRIGGER ALL;

-- Временно отключить для массовой операции (быстро!)
ALTER TABLE employees DISABLE TRIGGER ALL;
-- ... массовая вставка 1 млн строк ...
ALTER TABLE employees ENABLE TRIGGER ALL;
```

### 8.3. Удаление триггера

```sql
DROP TRIGGER employees_insert_audit ON employees;
DROP TRIGGER IF EXISTS employees_update_audit ON employees;
```

---

## Часть 9. Порядок выполнения триггеров

```sql
-- Создаём несколько триггеров на одно событие
CREATE TRIGGER trigger_1 BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION func1();
CREATE TRIGGER trigger_2 BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION func2();
CREATE TRIGGER trigger_3 BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION func3();

-- Порядок выполнения: в алфавитном порядке имён триггеров
-- trigger_1, trigger_2, trigger_3

-- Можно управлять порядком с помощью имён:
CREATE TRIGGER z_final BEFORE UPDATE ON users ...;  -- выполнится последним
CREATE TRIGGER a_first BEFORE UPDATE ON users ...;   -- выполнится первым
```

**Полный порядок для UPDATE:**

```
1. BEFORE STATEMENT триггеры
2. BEFORE EACH ROW триггеры (по одному на строку)
3. Сама операция UPDATE
4. AFTER EACH ROW триггеры
5. AFTER STATEMENT триггеры
```

---

## Часть 10. Типичные ошибки и как их избежать

### 10.1. Забыл RETURN в BEFORE-триггере

```sql
-- ❌ ОШИБКА!
CREATE FUNCTION bad_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := NOW();
    -- забыли RETURN NEW!
END;
$$ LANGUAGE plpgsql;

-- ✅ ПРАВИЛЬНО
CREATE FUNCTION good_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;  -- обязательно!
END;
$$ LANGUAGE plpgsql;
```

### 10.2. Бесконечный цикл (триггер вызывает сам себя)

```sql
-- ❌ ОПАСНО! Бесконечный цикл
CREATE FUNCTION dangerous() RETURNS TRIGGER AS $$
BEGIN
    -- Обновление той же таблицы вызовет тот же триггер снова!
    UPDATE same_table SET count = count + 1 WHERE id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ✅ ПРАВИЛЬНО: избегаем рекурсии
CREATE FUNCTION safe() RETURNS TRIGGER AS $$
BEGIN
    -- Проверяем, не вызван ли триггер рекурсивно
    IF NOT pg_trigger_depth() = 1 THEN
        RETURN NEW;
    END IF;
    
    UPDATE same_table SET count = count + 1 WHERE id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 10.3. Медленный триггер на больших объёмах

```sql
-- ❌ ПЛОХО: FOR EACH ROW на миллионах строк
CREATE TRIGGER slow_trigger
    AFTER UPDATE ON large_table
    FOR EACH ROW  -- 1 млн вызовов!
    EXECUTE FUNCTION log_row();

-- ✅ ХОРОШО: FOR EACH STATEMENT
CREATE TRIGGER fast_trigger
    AFTER UPDATE ON large_table
    FOR EACH STATEMENT  -- 1 вызов!
    EXECUTE FUNCTION log_statement();
```

---

## Шпаргалка

```sql
-- ============================================
-- БАЗОВЫЙ ТРИГГЕР (BEFORE INSERT)
-- ============================================
CREATE FUNCTION set_created_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.created_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_created_at
    BEFORE INSERT ON users
    FOR EACH ROW
    EXECUTE FUNCTION set_created_at();

-- ============================================
-- ТРИГГЕР-ЛОГ (AFTER)
-- ============================================
CREATE FUNCTION log_deletion()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO logs (table_name, row_id, deleted_at)
    VALUES (TG_TABLE_NAME, OLD.id, NOW());
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_deletion
    AFTER DELETE ON users
    FOR EACH ROW
    EXECUTE FUNCTION log_deletion();

-- ============================================
-- УСЛОВНЫЙ ТРИГГЕР (WHEN)
-- ============================================
CREATE TRIGGER trg_price_change
    BEFORE UPDATE OF price ON products
    FOR EACH ROW
    WHEN (OLD.price != NEW.price)
    EXECUTE FUNCTION log_price_change();

-- ============================================
-- УПРАВЛЕНИЕ ТРИГГЕРАМИ
-- ============================================
ALTER TABLE users DISABLE TRIGGER trg_name;
ALTER TABLE users ENABLE TRIGGER trg_name;
ALTER TABLE users DISABLE TRIGGER ALL;
DROP TRIGGER trg_name ON users;
```

---

## Итог: когда использовать триггеры

| Задача | Использовать триггер? | Почему |
|--------|----------------------|--------|
| Автоматическое обновление `updated_at` | ✅ Да | Идеальное применение |
| Аудит изменений | ✅ Да | Лучшее решение |
| Каскадное обновление | ⚠️ Осторожно | Может вызвать циклы |
| Сложная валидация | ✅ Да | Лучше чем CHECK для сложной логики |
| Вызов внешнего API | ❌ Нет | Триггер должен быть быстрым |
| Массовая обработка данных | ❌ Нет | Лучше явный SQL |
| Бизнес-логика (часто меняется) | ❌ Нет | Тяжело отлаживать |
