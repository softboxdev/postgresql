# PL/pgSQL — Полный разбор языка процедурного программирования PostgreSQL

## Введение

| Конструкция | Синтаксис | Пример |
|-------------|-----------|--------|
| **Функция** | `CREATE FUNCTION ... RETURNS ... AS $$ ... $$ LANGUAGE plpgsql` | — |
| **Переменная** | `v_name TYPE := value;` | `v_id INT := 1;` |
| **IF** | `IF condition THEN ... END IF;` | `IF price > 100 THEN ...` |
| **CASE** | `CASE value WHEN 1 THEN ... END` | `CASE status WHEN 'new' THEN ...` |
| **FOR (число)** | `FOR i IN 1..10 LOOP ... END LOOP` | — |
| **FOR (запрос)** | `FOR rec IN SELECT ... LOOP ... END LOOP` | — |
| **WHILE** | `WHILE condition LOOP ... END LOOP` | — |
| **EXIT** | `EXIT WHEN condition;` | `EXIT WHEN i > 10;` |
| **SELECT INTO** | `SELECT col INTO var FROM ...` | — |
| **PERFORM** | `PERFORM query;` | `PERFORM 1 FROM books;` |
| **RETURN** | `RETURN value;` | `RETURN v_result;` |
| **RETURN QUERY** | `RETURN QUERY SELECT ...` | — |
| **EXCEPTION** | `BEGIN ... EXCEPTION WHEN ... THEN ... END;` | — |
| **RAISE** | `RAISE LEVEL 'msg', arg;` | `RAISE NOTICE 'ID: %', id;` |
| **ASSERT** | `ASSERT condition, 'message';` | `ASSERT stock > 0;` |
| **EXECUTE** | `EXECUTE sql_string USING ...;` | — |
| **Курсор** | `CURSOR FOR SELECT ...` | — |
| **Массив** | `ARRAY[1,2,3]` | — |

**PL/pgSQL** (Procedural Language/PostgreSQL) — это встроенный процедурный язык PostgreSQL, который позволяет писать сложную бизнес-логику, хранимые функции, триггеры и процедуры прямо внутри базы данных.

### Зачем нужен PL/pgSQL?

| Задача | Без PL/pgSQL (только SQL) | С PL/pgSQL |
|--------|--------------------------|------------|
| Условная логика | ❌ Невозможно | ✅ `IF/ELSE` |
| Циклы | ❌ Невозможно | ✅ `FOR/WHILE/LOOP` |
| Переменные | ❌ Только CTE | ✅ `DECLARE` |
| Обработка ошибок | ❌ Нет | ✅ `BEGIN...EXCEPTION` |
| Множество запросов | N запросов из приложения | 1 вызов функции |

---

## Часть 1. Обзор и конструкции языка

### 1.1. Базовая структура функции

```sql
CREATE OR REPLACE FUNCTION имя_функции(параметры)
RETURNS тип_возврата
LANGUAGE plpgsql
[IMMUTABLE | STABLE | VOLATILE]
AS $$
DECLARE
    -- Объявление переменных
    переменная1 тип;
    переменная2 тип := значение_по_умолчанию;
BEGIN
    -- Тело функции
    -- ... код ...
    RETURN значение;
END;
$$;
```

### 1.2. Переменные и типы данных

```sql
CREATE FUNCTION variable_example()
RETURNS TEXT AS $$
DECLARE
    -- Скалярные типы
    v_id INTEGER := 1;
    v_name VARCHAR(100) := 'Иван';
    v_price DECIMAL(10,2) := 99.99;
    v_created DATE := CURRENT_DATE;
    v_now TIMESTAMP := NOW();
    
    -- Составные типы (строка таблицы)
    v_book books%ROWTYPE;
    
    -- Поле из таблицы
    v_title books.title%TYPE;
    
    -- Массивы
    v_ids INTEGER[] := ARRAY[1, 2, 3];
    v_names TEXT[] := '{"Анна", "Иван", "Мария"}';
    
    -- RECORD (динамическая строка)
    v_record RECORD;
BEGIN
    RETURN 'Переменные объявлены';
END;
$$ LANGUAGE plpgsql;
```

### 1.3. Константы и алиасы

```sql
CREATE FUNCTION constants_example()
RETURNS TEXT AS $$
DECLARE
    VAT_RATE CONSTANT DECIMAL := 0.20;  -- Нельзя изменить
    user_id ALIAS FOR $1;               -- Алиас для параметра
BEGIN
    -- VAT_RATE := 0.25; -- ОШИБКА! Константу нельзя изменить
    RETURN format('VAT: %s%%, User ID: %s', VAT_RATE * 100, user_id);
END;
$$ LANGUAGE plpgsql;
```

### 1.4. Условные операторы (IF/THEN/ELSE)

```sql
CREATE FUNCTION discount_price(
    p_price DECIMAL,
    p_discount_level INTEGER
)
RETURNS DECIMAL AS $$
DECLARE
    v_discount DECIMAL;
BEGIN
    -- Простой IF
    IF p_discount_level <= 0 THEN
        RETURN p_price;
    END IF;
    
    -- IF/ELSE
    IF p_discount_level = 1 THEN
        v_discount := 0.05;  -- 5%
    ELSIF p_discount_level = 2 THEN
        v_discount := 0.10;  -- 10%
    ELSIF p_discount_level = 3 THEN
        v_discount := 0.15;  -- 15%
    ELSE
        v_discount := 0.20;  -- 20%
    END IF;
    
    RETURN p_price * (1 - v_discount);
END;
$$ LANGUAGE plpgsql;

-- Проверка
SELECT discount_price(1000, 2);  -- 900
```

### 1.5. Оператор CASE (два вида)

```sql
CREATE FUNCTION get_status_name(p_status_code INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_status TEXT;
BEGIN
    -- Простой CASE (по значению)
    v_status := CASE p_status_code
        WHEN 1 THEN 'Новый'
        WHEN 2 THEN 'В обработке'
        WHEN 3 THEN 'Отправлен'
        WHEN 4 THEN 'Доставлен'
        ELSE 'Неизвестно'
    END;
    
    -- Альтернатива с IF/ELSIF
    v_status := CASE
        WHEN p_status_code = 1 THEN 'Новый'
        WHEN p_status_code = 2 THEN 'В обработке'
        WHEN p_status_code > 4 THEN 'Ошибка'
        ELSE 'Другое'
    END;
    
    RETURN v_status;
END;
$$ LANGUAGE plpgsql;
```

### 1.6. Циклы (LOOP, WHILE, FOR)

#### LOOP с EXIT

```sql
CREATE FUNCTION sum_loop(p_max INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_counter INTEGER := 1;
    v_sum INTEGER := 0;
BEGIN
    LOOP
        v_sum := v_sum + v_counter;
        v_counter := v_counter + 1;
        EXIT WHEN v_counter > p_max;
    END LOOP;
    
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;

SELECT sum_loop(10);  -- 55
```

#### WHILE LOOP

```sql
CREATE FUNCTION sum_while(p_max INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_counter INTEGER := 1;
    v_sum INTEGER := 0;
BEGIN
    WHILE v_counter <= p_max LOOP
        v_sum := v_sum + v_counter;
        v_counter := v_counter + 1;
    END LOOP;
    
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;
```

#### FOR LOOP (числовой)

```sql
CREATE FUNCTION sum_for(p_max INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_sum INTEGER := 0;
BEGIN
    FOR i IN 1..p_max LOOP
        v_sum := v_sum + i;
    END LOOP;
    
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;

-- Шаг 2
CREATE FUNCTION sum_even(p_max INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_sum INTEGER := 0;
BEGIN
    FOR i IN 2..p_max BY 2 LOOP
        v_sum := v_sum + i;
    END LOOP;
    
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;

-- Обратный порядок
CREATE FUNCTION sum_reverse(p_max INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_sum INTEGER := 0;
BEGIN
    FOR i IN REVERSE p_max..1 LOOP
        v_sum := v_sum + i;
    END LOOP;
    
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 2. Выполнение запросов

### 2.1. SELECT INTO (одиночная строка)

```sql
CREATE FUNCTION get_book_info(p_book_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_title books.title%TYPE;
    v_price books.price%TYPE;
    v_stock books.stock%TYPE;
    v_not_found BOOLEAN := FALSE;
BEGIN
    -- SELECT INTO с одной переменной
    SELECT title INTO v_title
    FROM books
    WHERE id = p_book_id;
    
    -- SELECT INTO с несколькими переменными
    SELECT title, price, stock 
    INTO v_title, v_price, v_stock
    FROM books
    WHERE id = p_book_id;
    
    -- Проверка, найдена ли строка
    IF NOT FOUND THEN
        RETURN 'Книга не найдена';
    END IF;
    
    RETURN format('Книга: %s, цена: %s, остаток: %s', 
                  v_title, v_price, v_stock);
END;
$$ LANGUAGE plpgsql;
```

### 2.2. SELECT INTO с RECORD

```sql
CREATE FUNCTION get_book_record(p_book_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_book RECORD;
BEGIN
    SELECT id, title, price, stock 
    INTO v_book
    FROM books
    WHERE id = p_book_id;
    
    IF NOT FOUND THEN
        RETURN 'Не найдено';
    END IF;
    
    RETURN format('ID: %s, Название: %s, Цена: %s, Остаток: %s',
                  v_book.id, v_book.title, v_book.price, v_book.stock);
END;
$$ LANGUAGE plpgsql;
```

### 2.3. SELECT INTO с %ROWTYPE

```sql
CREATE FUNCTION get_book_rowtype(p_book_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_book books%ROWTYPE;
BEGIN
    SELECT * INTO v_book
    FROM books
    WHERE id = p_book_id;
    
    IF NOT FOUND THEN
        RETURN 'Не найдено';
    END IF;
    
    RETURN format('Название: %s, Автор ID: %s', 
                  v_book.title, v_book.author_id);
END;
$$ LANGUAGE plpgsql;
```

### 2.4. PERFORM (для запросов без результата)

```sql
CREATE FUNCTION cleanup_temp_data()
RETURNS INTEGER AS $$
BEGIN
    -- PERFORM используется, когда результат не нужен
    PERFORM 1 FROM temp_logs WHERE created_at < NOW() - INTERVAL '7 days';
    
    IF FOUND THEN
        DELETE FROM temp_logs WHERE created_at < NOW() - INTERVAL '7 days';
        RETURN 1;  -- Удалено
    END IF;
    
    RETURN 0;  -- Ничего не удалено
END;
$$ LANGUAGE plpgsql;
```

### 2.5. RETURN QUERY (возврат набора строк)

```sql
CREATE FUNCTION get_expensive_books(p_min_price DECIMAL)
RETURNS TABLE(
    book_title VARCHAR,
    book_price DECIMAL,
    stock_quantity INTEGER
) AS $$
BEGIN
    RETURN QUERY
    SELECT title, price, stock
    FROM books
    WHERE price > p_min_price
    ORDER BY price DESC;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM get_expensive_books(300);
```

### 2.6. RETURN NEXT (построчное возвращение)

```sql
CREATE FUNCTION get_books_cursor()
RETURNS SETOF books AS $$
DECLARE
    v_book books%ROWTYPE;
BEGIN
    FOR v_book IN SELECT * FROM books ORDER BY price DESC LOOP
        -- Можно модифицировать данные перед возвратом
        IF v_book.price > 1000 THEN
            v_book.price := v_book.price * 0.9;  -- Скидка 10%
        END IF;
        RETURN NEXT v_book;
    END LOOP;
    
    RETURN;
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 3. Курсоры

### 3.1. Что такое курсор?

**Курсор** — это указатель на результирующий набор запроса, который позволяет обрабатывать строки **по одной**, а не все сразу.

### 3.2. Простой курсор (FOR LOOP) — автоматический

```sql
CREATE FUNCTION process_all_books()
RETURNS VOID AS $$
DECLARE
    v_book RECORD;
    v_counter INTEGER := 0;
BEGIN
    -- Автоматический курсор через FOR LOOP
    FOR v_book IN 
        SELECT id, title, price FROM books WHERE stock > 0
    LOOP
        v_counter := v_counter + 1;
        RAISE NOTICE '%: % (цена: %)', v_counter, v_book.title, v_book.price;
    END LOOP;
    
    RAISE NOTICE 'Всего обработано книг: %', v_counter;
END;
$$ LANGUAGE plpgsql;
```

### 3.3. Явный курсор (DECLARE CURSOR)

```sql
CREATE FUNCTION process_books_explicit()
RETURNS VOID AS $$
DECLARE
    -- Объявление курсора
    book_cursor CURSOR FOR
        SELECT id, title, price, stock FROM books WHERE stock > 0;
    
    v_book RECORD;
BEGIN
    -- Открытие курсора
    OPEN book_cursor;
    
    LOOP
        -- Получение следующей строки
        FETCH NEXT FROM book_cursor INTO v_book;
        EXIT WHEN NOT FOUND;
        
        RAISE NOTICE 'Книга: %, остаток: %', v_book.title, v_book.stock;
    END LOOP;
    
    -- Закрытие курсора
    CLOSE book_cursor;
END;
$$ LANGUAGE plpgsql;
```

### 3.4. Курсор с параметрами

```sql
CREATE FUNCTION process_books_by_category(p_min_price DECIMAL)
RETURNS VOID AS $$
DECLARE
    -- Курсор с параметрами
    book_cursor CURSOR (min_price DECIMAL) FOR
        SELECT id, title, price, stock 
        FROM books 
        WHERE price > min_price AND stock > 0
        ORDER BY price DESC;
    
    v_book RECORD;
BEGIN
    OPEN book_cursor(p_min_price);
    
    LOOP
        FETCH book_cursor INTO v_book;
        EXIT WHEN NOT FOUND;
        
        RAISE NOTICE '% - % руб.', v_book.title, v_book.price;
    END LOOP;
    
    CLOSE book_cursor;
END;
$$ LANGUAGE plpgsql;

-- Вызов
SELECT process_books_by_category(300);
```

### 3.5. Курсор с разными направлениями

```sql
CREATE FUNCTION cursor_directions()
RETURNS VOID AS $$
DECLARE
    book_cursor SCROLL CURSOR FOR
        SELECT title FROM books ORDER BY title;
    
    v_title VARCHAR;
BEGIN
    OPEN book_cursor;
    
    -- Первая строка
    FETCH FIRST FROM book_cursor INTO v_title;
    RAISE NOTICE 'FIRST: %', v_title;
    
    -- Последняя строка
    FETCH LAST FROM book_cursor INTO v_title;
    RAISE NOTICE 'LAST: %', v_title;
    
    -- 3 строки вперёд
    FETCH FORWARD 3 FROM book_cursor INTO v_title;
    RAISE NOTICE 'FORWARD 3: %', v_title;
    
    -- 2 строки назад
    FETCH BACKWARD 2 FROM book_cursor INTO v_title;
    RAISE NOTICE 'BACKWARD 2: %', v_title;
    
    CLOSE book_cursor;
END;
$$ LANGUAGE plpgsql;
```

### 3.6. Обновление данных через курсор

```sql
CREATE FUNCTION update_prices_with_cursor(p_discount DECIMAL)
RETURNS INTEGER AS $$
DECLARE
    -- Курсор с FOR UPDATE (блокировка строк)
    book_cursor CURSOR FOR
        SELECT id, price FROM books WHERE stock > 0
        FOR UPDATE;
    
    v_book RECORD;
    v_updated INTEGER := 0;
BEGIN
    OPEN book_cursor;
    
    LOOP
        FETCH book_cursor INTO v_book;
        EXIT WHEN NOT FOUND;
        
        UPDATE books 
        SET price = price * (1 - p_discount)
        WHERE CURRENT OF book_cursor;  -- Обновление текущей строки
        
        v_updated := v_updated + 1;
    END LOOP;
    
    CLOSE book_cursor;
    RETURN v_updated;
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 4. Динамические команды (EXECUTE)

### 4.1. Зачем нужна динамика?

Статический SQL не может:
- Использовать имя таблицы как параметр
- Строить WHERE-условия динамически
- Выполнять DDL команды

### 4.2. Простой EXECUTE

```sql
CREATE FUNCTION dynamic_table(p_table_name TEXT, p_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_result TEXT;
BEGIN
    -- Динамическое выполнение SQL
    EXECUTE format('SELECT title FROM %I WHERE id = %s', p_table_name, p_id)
    INTO v_result;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

-- Проверка
SELECT dynamic_table('books', 1);
```

### 4.3. EXECUTE с USING (безопасные параметры)

```sql
CREATE FUNCTION get_books_dynamic(
    p_table_name TEXT,
    p_min_price DECIMAL,
    p_max_price DECIMAL
)
RETURNS TABLE(title TEXT, price DECIMAL) AS $$
BEGIN
    -- USING предотвращает SQL-инъекции
    RETURN QUERY EXECUTE format(
        'SELECT title, price FROM %I WHERE price BETWEEN $1 AND $2',
        p_table_name
    ) USING p_min_price, p_max_price;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM get_books_dynamic('books', 300, 500);
```

### 4.4. Динамический INSERT

```sql
CREATE FUNCTION dynamic_insert(
    p_table_name TEXT,
    p_values TEXT[]
)
RETURNS VOID AS $$
DECLARE
    v_query TEXT;
    v_column_list TEXT;
    v_value_list TEXT;
BEGIN
    -- Формируем имена колонок
    SELECT string_agg(column_name, ',')
    INTO v_column_list
    FROM information_schema.columns
    WHERE table_name = p_table_name
    LIMIT array_length(p_values, 1);
    
    -- Формируем VALUES ($1, $2, ...)
    SELECT string_agg('$' || generate_series, ',')
    INTO v_value_list
    FROM generate_series(1, array_length(p_values, 1));
    
    v_query := format('INSERT INTO %I (%s) VALUES (%s)',
                      p_table_name, v_column_list, v_value_list);
    
    EXECUTE v_query USING p_values[1], p_values[2], p_values[3];
END;
$$ LANGUAGE plpgsql;
```

### 4.5. Динамический запрос с разным количеством параметров

```sql
CREATE FUNCTION flexible_filter(
    p_table_name TEXT,
    p_column_name TEXT,
    p_operator TEXT,
    p_value TEXT
)
RETURNS SETOF RECORD AS $$
DECLARE
    v_query TEXT;
    v_result RECORD;
BEGIN
    v_query := format(
        'SELECT * FROM %I WHERE %I %s $1',
        p_table_name, p_column_name, p_operator
    );
    
    FOR v_result IN EXECUTE v_query USING p_value
    LOOP
        RETURN NEXT v_result;
    END LOOP;
    
    RETURN;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM flexible_filter('books', 'price', '>', '300') 
AS (id INT, title TEXT, price DECIMAL, stock INT, author_id INT);
```

### 4.6. Динамическая DDL (CREATE/ALTER/DROP)

```sql
CREATE FUNCTION add_column_if_not_exists(
    p_table_name TEXT,
    p_column_name TEXT,
    p_column_type TEXT
)
RETURNS TEXT AS $$
DECLARE
    v_exists BOOLEAN;
BEGIN
    -- Проверяем, существует ли колонка
    SELECT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = p_table_name AND column_name = p_column_name
    ) INTO v_exists;
    
    IF NOT v_exists THEN
        EXECUTE format('ALTER TABLE %I ADD COLUMN %I %s',
                       p_table_name, p_column_name, p_column_type);
        RETURN format('Колонка %s добавлена', p_column_name);
    ELSE
        RETURN format('Колонка %s уже существует', p_column_name);
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 5. Массивы

### 5.1. Объявление и инициализация массивов

```sql
CREATE FUNCTION array_examples()
RETURNS TEXT AS $$
DECLARE
    -- Одномерные массивы
    v_numbers INTEGER[] := ARRAY[1, 2, 3, 4, 5];
    v_texts TEXT[] := '{"a", "b", "c"}';
    v_empty INTEGER[] := '{}';
    
    -- Многомерные массивы
    v_matrix INTEGER[3][3] := ARRAY[[1,2,3], [4,5,6], [7,8,9]];
    
    -- Массивы с типом таблицы
    v_book_ids books.id%TYPE[] := ARRAY[1, 2, 3];
BEGIN
    RETURN format('Числа: %s, Тексты: %s, Матрица[2][2]: %s',
                  v_numbers, v_texts, v_matrix[2][2]);
END;
$$ LANGUAGE plpgsql;
```

### 5.2. Работа с элементами массива

```sql
CREATE FUNCTION array_operations()
RETURNS TEXT AS $$
DECLARE
    v_arr INTEGER[] := ARRAY[10, 20, 30, 40, 50];
    v_value INTEGER;
    v_slice INTEGER[];
BEGIN
    -- Доступ по индексу (с 1!)
    v_value := v_arr[3];  -- 30
    
    -- Изменение элемента
    v_arr[2] := 25;
    
    -- Срез массива
    v_slice := v_arr[2:4];  -- {25,30,40}
    
    -- Добавление элемента в конец
    v_arr := v_arr || 60;  -- {10,25,30,40,50,60}
    
    -- Добавление в начало
    v_arr := ARRAY[5] || v_arr;  -- {5,10,25,30,40,50,60}
    
    -- Удаление последнего элемента
    v_arr := v_arr[1:array_length(v_arr, 1) - 1];
    
    RETURN format('Срез: %s, Финальный массив: %s', v_slice, v_arr);
END;
$$ LANGUAGE plpgsql;
```

### 5.3. Функции для работы с массивами

```sql
CREATE FUNCTION array_functions_demo()
RETURNS TEXT AS $$
DECLARE
    v_arr INTEGER[] := ARRAY[5, 2, 8, 1, 9, 3];
    v_result TEXT;
BEGIN
    -- array_length - длина массива
    RAISE NOTICE 'Длина: %', array_length(v_arr, 1);
    
    -- array_ndims - размерность
    RAISE NOTICE 'Размерность: %', array_ndims(v_arr);
    
    -- array_lower / array_upper - нижняя/верхняя граница
    RAISE NOTICE 'Нижняя граница: %', array_lower(v_arr, 1);
    RAISE NOTICE 'Верхняя граница: %', array_upper(v_arr, 1);
    
    -- array_to_string - преобразование в строку
    v_result := array_to_string(v_arr, ', ');
    
    -- string_to_array - из строки в массив
    v_arr := string_to_array('10,20,30,40', ',')::INTEGER[];
    
    -- array_cat - объединение массивов
    v_arr := array_cat(ARRAY[1,2], ARRAY[3,4]);
    
    -- array_prepend / array_append
    v_arr := array_prepend(0, v_arr);  -- в начало
    v_arr := array_append(v_arr, 5);    -- в конец
    
    -- array_remove - удаление всех вхождений
    v_arr := array_remove(ARRAY[1,2,3,2,1,2], 2);  -- {1,3,1}
    
    -- array_replace - замена
    v_arr := array_replace(ARRAY[1,2,3,2,1], 2, 99);  -- {1,99,3,99,1}
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;
```

### 5.4. Итерация по массиву

```sql
CREATE FUNCTION sum_array(p_arr INTEGER[])
RETURNS INTEGER AS $$
DECLARE
    v_sum INTEGER := 0;
    v_idx INTEGER;
BEGIN
    -- Через FOR с индексами
    FOR v_idx IN 1..array_length(p_arr, 1) LOOP
        v_sum := v_sum + p_arr[v_idx];
    END LOOP;
    
    -- Альтернатива: FOREACH
    -- FOREACH v_val IN ARRAY p_arr LOOP
    --     v_sum := v_sum + v_val;
    -- END LOOP;
    
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;

SELECT sum_array(ARRAY[1,2,3,4,5]);  -- 15
```

### 5.5. Массивы в запросах

```sql
-- Функция, принимающая массив ID
CREATE FUNCTION get_books_by_ids(p_book_ids INTEGER[])
RETURNS TABLE(title TEXT, price DECIMAL) AS $$
BEGIN
    RETURN QUERY
    SELECT b.title, b.price
    FROM books b
    WHERE b.id = ANY(p_book_ids)  -- ANY - проверка вхождения
    ORDER BY b.title;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM get_books_by_ids(ARRAY[1, 3, 5]);

-- Функция, возвращающая массив
CREATE FUNCTION get_book_titles(p_min_price DECIMAL)
RETURNS TEXT[] AS $$
DECLARE
    v_titles TEXT[];
BEGIN
    SELECT array_agg(title) INTO v_titles
    FROM books
    WHERE price > p_min_price;
    
    RETURN v_titles;
END;
$$ LANGUAGE plpgsql;

SELECT get_book_titles(300);  -- {'1984','Три товарища'}
```

---

## Часть 6. Обработка ошибок

### 6.1. Базовое исключение EXCEPTION

```sql
CREATE FUNCTION safe_division(p_numerator DECIMAL, p_denominator DECIMAL)
RETURNS DECIMAL AS $$
DECLARE
    v_result DECIMAL;
BEGIN
    v_result := p_numerator / p_denominator;
    RETURN v_result;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE WARNING 'Деление на ноль! Возвращаем NULL';
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;

SELECT safe_division(10, 0);  -- NULL с предупреждением
```

### 6.2. Обработка разных типов ошибок

```sql
CREATE FUNCTION safe_insert_book(
    p_title TEXT,
    p_price DECIMAL,
    p_stock INTEGER,
    p_author_id INTEGER
)
RETURNS TEXT AS $$
DECLARE
    v_book_id INTEGER;
BEGIN
    INSERT INTO books (title, price, stock, author_id)
    VALUES (p_title, p_price, p_stock, p_author_id)
    RETURNING id INTO v_book_id;
    
    RETURN format('Книга добавлена с ID: %s', v_book_id);
    
EXCEPTION
    -- Ошибка уникальности (если title UNIQUE)
    WHEN unique_violation THEN
        RETURN 'Ошибка: Книга с таким названием уже существует';
    
    -- Ошибка внешнего ключа (автор не существует)
    WHEN foreign_key_violation THEN
        RETURN 'Ошибка: Автор с указанным ID не найден';
    
    -- Ошибка CHECK-ограничения (price > 0, stock >= 0)
    WHEN check_violation THEN
        RETURN 'Ошибка: Цена должна быть > 0, остаток >= 0';
    
    -- Ошибка NOT NULL
    WHEN not_null_violation THEN
        RETURN 'Ошибка: Заполните все обязательные поля';
    
    -- Все остальные ошибки
    WHEN OTHERS THEN
        RETURN format('Неизвестная ошибка: %s', SQLERRM);
END;
$$ LANGUAGE plpgsql;
```

### 6.3. GET DIAGNOSTICS (получение информации об ошибке)

```sql
CREATE FUNCTION detailed_error_handler()
RETURNS VOID AS $$
DECLARE
    v_sqlstate TEXT;
    v_message TEXT;
    v_context TEXT;
BEGIN
    -- Генерируем ошибку
    PERFORM 1/0;
    
EXCEPTION
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_sqlstate = RETURNED_SQLSTATE,
            v_message = MESSAGE_TEXT,
            v_context = PG_EXCEPTION_CONTEXT;
        
        RAISE NOTICE 'SQLSTATE: %', v_sqlstate;
        RAISE NOTICE 'Сообщение: %', v_message;
        RAISE NOTICE 'Контекст: %', v_context;
        
        RAISE EXCEPTION 'Произошла ошибка: % (SQLSTATE: %)', v_message, v_sqlstate;
END;
$$ LANGUAGE plpgsql;
```

### 6.4. ASSERT (проверка условий)

```sql
CREATE FUNCTION transfer_stock(
    p_from_book_id INTEGER,
    p_to_book_id INTEGER,
    p_quantity INTEGER
)
RETURNS VOID AS $$
DECLARE
    v_current_stock INTEGER;
BEGIN
    -- Проверяем наличие товара
    SELECT stock INTO v_current_stock
    FROM books WHERE id = p_from_book_id;
    
    -- ASSERT - проверка (выкидывает ошибку при false)
    ASSERT v_current_stock >= p_quantity, 
           format('Недостаточно товара! В наличии: %s, запрошено: %s', 
                  v_current_stock, p_quantity);
    
    -- Сама операция
    UPDATE books SET stock = stock - p_quantity WHERE id = p_from_book_id;
    UPDATE books SET stock = stock + p_quantity WHERE id = p_to_book_id;
    
EXCEPTION
    WHEN assert_failure THEN
        RAISE WARNING 'Ошибка проверки: %', SQLERRM;
        ROLLBACK;
END;
$$ LANGUAGE plpgsql;
```

### 6.5. Транзакции внутри функции (SAVEPOINT)

```sql
CREATE FUNCTION batch_insert_books(p_books_data TEXT[][])
RETURNS JSONB AS $$
DECLARE
    v_idx INTEGER;
    v_inserted INTEGER := 0;
    v_failed INTEGER := 0;
    v_errors JSONB := '[]'::JSONB;
    v_book_title TEXT;
    v_book_price DECIMAL;
BEGIN
    FOR v_idx IN 1..array_length(p_books_data, 1) LOOP
        v_book_title := p_books_data[v_idx][1];
        v_book_price := p_books_data[v_idx][2]::DECIMAL;
        
        -- Устанавливаем точку сохранения для каждой итерации
        SAVEPOINT before_insert;
        
        BEGIN
            INSERT INTO books (title, price, stock) 
            VALUES (v_book_title, v_book_price, 0);
            
            v_inserted := v_inserted + 1;
            
        EXCEPTION
            WHEN OTHERS THEN
                -- Откатываем только к savepoint'у
                ROLLBACK TO SAVEPOINT before_insert;
                
                v_failed := v_failed + 1;
                v_errors := v_errors || 
                    jsonb_build_object(
                        'book', v_book_title,
                        'error', SQLERRM
                    );
        END;
    END LOOP;
    
    RETURN jsonb_build_object(
        'inserted', v_inserted,
        'failed', v_failed,
        'errors', v_errors
    );
END;
$$ LANGUAGE plpgsql;
```

### 6.6. RAISE (вывод сообщений)

```sql
CREATE FUNCTION logging_demo(p_level INTEGER)
RETURNS VOID AS $$
BEGIN
    -- DEBUG - только для разработки
    RAISE DEBUG 'DEBUG сообщение: %', p_level;
    
    -- LOG - общая информация
    RAISE LOG 'LOG сообщение: выполнен вызов с параметром %', p_level;
    
    -- INFO - информационные сообщения
    RAISE INFO 'INFO: Начало обработки (уровень %)', p_level;
    
    -- NOTICE - уведомления по умолчанию
    RAISE NOTICE 'NOTICE: Параметр = %', p_level;
    
    -- WARNING - предупреждения
    RAISE WARNING 'WARNING: Значение % может быть проблемным', p_level;
    
    -- EXCEPTION - ошибка с остановкой
    IF p_level < 0 THEN
        RAISE EXCEPTION 'Отрицательный уровень: %', p_level;
    END IF;
    
    -- EXCEPTION с определённым SQLSTATE
    IF p_level > 100 THEN
        RAISE EXCEPTION 'Слишком большой уровень: %', p_level
        USING ERRCODE = '22003';  -- numeric_value_out_of_range
    END IF;
    
    RAISE NOTICE 'Завершено успешно';
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 7. Триггеры

### 7.1. Базовая структура триггерной функции

```sql
-- Триггерная функция всегда возвращает TRIGGER
CREATE OR REPLACE FUNCTION my_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    -- Доступны специальные переменные:
    -- NEW - новая строка (для INSERT/UPDATE)
    -- OLD - старая строка (для UPDATE/DELETE)
    -- TG_OP - операция (INSERT/UPDATE/DELETE)
    -- TG_TABLE_NAME - имя таблицы
    
    -- Логика триггера
    
    RETURN NEW;  -- для BEFORE INSERT/UPDATE (обязательно)
    -- RETURN OLD;  -- для BEFORE DELETE
    -- RETURN NULL; -- для AFTER триггеров
END;
$$ LANGUAGE plpgsql;
```

### 7.2. BEFORE INSERT: автоматическое заполнение полей

```sql
-- Таблица с аудитом
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT DEFAULT current_user,
    changed_at TIMESTAMP DEFAULT NOW()
);

-- Триггерная функция
CREATE OR REPLACE FUNCTION trg_audit_log()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, 'INSERT', row_to_json(NEW));
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data)
        VALUES (TG_TABLE_NAME, 'UPDATE', row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;
        
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data)
        VALUES (TG_TABLE_NAME, 'DELETE', row_to_json(OLD));
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER books_audit
    AFTER INSERT OR UPDATE OR DELETE ON books
    FOR EACH ROW
    EXECUTE FUNCTION trg_audit_log();
```

### 7.3. BEFORE UPDATE: автоматическое обновление даты

```sql
CREATE OR REPLACE FUNCTION trg_update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER books_update_timestamp
    BEFORE UPDATE ON books    FOR EACH ROW
    EXECUTE FUNCTION trg_update_timestamp();
```

### 7.4. BEFORE INSERT/UPDATE: валидация данных

```sql
CREATE OR REPLACE FUNCTION trg_validate_book()
RETURNS TRIGGER AS $$
BEGIN
    -- Проверка цены
    IF NEW.price <= 0 THEN
        RAISE EXCEPTION 'Цена книги должна быть больше 0. Текущее значение: %', NEW.price;
    END IF;
    
    -- Проверка остатка
    IF NEW.stock < 0 THEN
        RAISE EXCEPTION 'Остаток не может быть отрицательным. Значение: %', NEW.stock;
    END IF;
    
    -- Проверка названия
    IF NEW.title IS NULL OR length(NEW.title) < 2 THEN
        RAISE EXCEPTION 'Название книги слишком короткое или отсутствует';
    END IF;
    
    -- Автоматическая нормализация
    NEW.title := initcap(trim(NEW.title));
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER books_validate
    BEFORE INSERT OR UPDATE ON books
    FOR EACH ROW
    EXECUTE FUNCTION trg_validate_book();
```

### 7.5. INSTEAD OF (для представлений)

```sql
-- Создаём представление
CREATE VIEW books_with_authors AS
SELECT b.id, b.title, b.price, 
       a.first_name, a.last_name
FROM books b
LEFT JOIN book_authors ba ON b.id = ba.book_id
LEFT JOIN authors a ON a.id = ba.author_id;

-- Триггер для обновления через представление
CREATE OR REPLACE FUNCTION trg_update_books_view()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        UPDATE books SET title = NEW.title, price = NEW.price
        WHERE id = OLD.id;
        RETURN NEW;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_books_view
    INSTEAD OF UPDATE ON books_with_authors
    FOR EACH ROW
    EXECUTE FUNCTION trg_update_books_view();
```

### 7.6. Триггер на уровне STATEMENT

```sql
-- Счётчик изменений
CREATE TABLE change_counter (
    table_name TEXT PRIMARY KEY,
    insert_count INTEGER DEFAULT 0,
    update_count INTEGER DEFAULT 0,
    delete_count INTEGER DEFAULT 0,
    last_change TIMESTAMP
);

CREATE OR REPLACE FUNCTION trg_count_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO change_counter (table_name, insert_count, update_count, delete_count, last_change)
    VALUES (TG_TABLE_NAME, 0, 0, 0, NOW())
    ON CONFLICT (table_name) DO UPDATE
    SET 
        insert_count = change_counter.insert_count + CASE WHEN TG_OP = 'INSERT' THEN 1 ELSE 0 END,
        update_count = change_counter.update_count + CASE WHEN TG_OP = 'UPDATE' THEN 1 ELSE 0 END,
        delete_count = change_counter.delete_count + CASE WHEN TG_OP = 'DELETE' THEN 1 ELSE 0 END,
        last_change = NOW();
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- FOR EACH STATEMENT (выполняется один раз на запрос)
CREATE TRIGGER books_count_changes
    AFTER INSERT OR UPDATE OR DELETE ON books
    FOR EACH STATEMENT
    EXECUTE FUNCTION trg_count_changes();
```

### 7.7. Условные триггеры (WHEN)

```sql
-- Триггер срабатывает только при изменении цены
CREATE TRIGGER books_price_change_log
    AFTER UPDATE OF price ON books
    FOR EACH ROW
    WHEN (OLD.price IS DISTINCT FROM NEW.price)  -- Только если цена изменилась
    EXECUTE FUNCTION trg_audit_log();

-- Триггер для дорогих книг
CREATE TRIGGER expensive_books_alert
    AFTER INSERT ON books
    FOR EACH ROW
    WHEN (NEW.price > 10000)
    EXECUTE FUNCTION trg_notify_admin();
```

---

## Часть 8. Отладка

### 8.1. RAISE NOTICE (основной способ)

```sql
CREATE FUNCTION debug_example(p_id INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_result INTEGER;
    v_counter INTEGER := 0;
BEGIN
    RAISE NOTICE 'Начало функции: p_id = %', p_id;
    
    FOR i IN 1..p_id LOOP
        v_counter := v_counter + i;
        RAISE DEBUG 'Итерация %: v_counter = %', i, v_counter;
    END LOOP;
    
    SELECT COUNT(*) INTO v_result FROM books;
    RAISE NOTICE 'Количество книг в БД: %', v_result;
    
    RAISE NOTICE 'Конец функции, результат = %', v_counter;
    RETURN v_counter;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Ошибка: %', SQLERRM;
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

### 8.2. Настройка уровней логирования

```sql
-- Включение DEBUG вывода
SET client_min_messages = 'DEBUG';

-- Только NOTICE и выше
SET client_min_messages = 'NOTICE';

-- Только WARNING и выше
SET client_min_messages = 'WARNING';

-- Только ERROR
SET client_min_messages = 'ERROR';
```

### 8.3. ASSERT для проверок

```sql
CREATE FUNCTION debug_assert(p_value INTEGER)
RETURNS INTEGER AS $$
BEGIN
    -- Проверка с отладочным сообщением
    ASSERT p_value > 0, 'Значение должно быть положительным';
    ASSERT p_value < 1000, 'Значение слишком большое';
    
    RETURN p_value * 2;
EXCEPTION
    WHEN assert_failure THEN
        RAISE NOTICE 'ASSERT failed: %', SQLERRM;
        RETURN -1;
END;
$$ LANGUAGE plpgsql;
```

### 8.4. PERFORM для отладки без результатов

```sql
CREATE FUNCTION debug_perform()
RETURNS VOID AS $$
DECLARE
    v_count INTEGER;
BEGIN
    -- PERFORM выполняет запрос, но не возвращает результат
    PERFORM 1 FROM books WHERE price > 1000;
    
    IF FOUND THEN
        RAISE NOTICE 'Есть книги дороже 1000 рублей';
    ELSE
        RAISE NOTICE 'Нет книг дороже 1000 рублей';
    END IF;
    
    -- Для получения количества используем SELECT INTO
    SELECT COUNT(*) INTO v_count FROM books WHERE price < 500;
    RAISE NOTICE 'Книг дешевле 500 рублей: %', v_count;
END;
$$ LANGUAGE plpgsql;
```

### 8.5. Условная компиляция (условная отладка)

```sql
CREATE FUNCTION conditional_debug(p_value INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_debug CONSTANT BOOLEAN := current_setting('myapp.debug', true)::BOOLEAN;
BEGIN
    IF v_debug THEN
        RAISE NOTICE 'DEBUG: начало функции, p_value = %', p_value;
    END IF;
    
    -- Основная логика
    -- ...
    
    IF v_debug THEN
        RAISE NOTICE 'DEBUG: конец функции';
    END IF;
    
    RETURN p_value * 2;
END;
$$ LANGUAGE plpgsql;

-- Включение отладки
SET myapp.debug = 'on';
SELECT conditional_debug(10);

-- Выключение отладки
SET myapp.debug = 'off';
```

### 8.6. Логирование в таблицу

```sql
-- Таблица для логов
CREATE TABLE plpgsql_log (
    id SERIAL PRIMARY KEY,
    function_name TEXT,
    message TEXT,
    debug_data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Функция логирования
CREATE OR REPLACE FUNCTION log_message(
    p_function TEXT,
    p_message TEXT,
    p_data JSONB DEFAULT NULL
)
RETURNS VOID AS $$
BEGIN
    INSERT INTO plpgsql_log (function_name, message, debug_data)
    VALUES (p_function, p_message, p_data);
    
    -- Также выводим в консоль
    RAISE NOTICE '[%] %', p_function, p_message;
END;
$$ LANGUAGE plpgsql;

-- Использование
CREATE FUNCTION process_books()
RETURNS VOID AS $$
DECLARE
    v_book RECORD;
    v_start_time TIMESTAMP;
BEGIN
    PERFORM log_message('process_books', 'Начало обработки');
    v_start_time := clock_timestamp();
    
    FOR v_book IN SELECT * FROM books LOOP
        PERFORM log_message('process_books', 
                           format('Обработана книга: %s', v_book.title),
                           jsonb_build_object('id', v_book.id, 'price', v_book.price));
    END LOOP;
    
    PERFORM log_message('process_books', 
                       format('Завершено. Время: %s ms', 
                              extract(epoch from (clock_timestamp() - v_start_time)) * 1000));
END;
$$ LANGUAGE plpgsql;
```

---

## Часть 9. Практические примеры (всё вместе)

### Пример 1: Сложная бизнес-логика

```sql
CREATE OR REPLACE FUNCTION process_order(
    p_customer_id INTEGER,
    p_items JSONB  -- [{book_id: 1, quantity: 2}, ...]
)
RETURNS JSONB AS $$
DECLARE
    v_order_id INTEGER;
    v_total DECIMAL := 0;
    v_item RECORD;
    v_book_price DECIMAL;
    v_book_stock INTEGER;
    v_result JSONB;
BEGIN
    -- Создаём заказ
    INSERT INTO orders (customer_id, status, order_date)
    VALUES (p_customer_id, 'new', NOW())
    RETURNING id INTO v_order_id;
    
    -- Обрабатываем каждую позицию
    FOR v_item IN SELECT * FROM jsonb_to_recordset(p_items) AS x(book_id INT, quantity INT)
    LOOP
        -- Получаем информацию о книге
        SELECT price, stock INTO v_book_price, v_book_stock
        FROM books WHERE id = v_item.book_id;
        
        -- Проверяем остаток
        IF v_book_stock < v_item.quantity THEN
            RAISE EXCEPTION 'Недостаточно товара. Книга ID: %, остаток: %, заказано: %',
                v_item.book_id, v_book_stock, v_item.quantity;
        END IF;
        
        -- Добавляем позицию
        INSERT INTO order_items (order_id, book_id, quantity, price_at_moment)
        VALUES (v_order_id, v_item.book_id, v_item.quantity, v_book_price);
        
        -- Уменьшаем остаток
        UPDATE books SET stock = stock - v_item.quantity
        WHERE id = v_item.book_id;
        
        -- Считаем сумму
        v_total := v_total + (v_book_price * v_item.quantity);
    END LOOP;
    
    -- Обновляем сумму заказа
    UPDATE orders SET total_amount = v_total WHERE id = v_order_id;
    
    -- Формируем результат
    v_result := jsonb_build_object(
        'order_id', v_order_id,
        'total', v_total,
        'status', 'success'
    );
    
    RETURN v_result;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Логируем ошибку
        INSERT INTO error_log (function_name, error_message, error_data, error_time)
        VALUES ('process_order', SQLERRM, 
                jsonb_build_object('customer_id', p_customer_id, 'items', p_items),
                NOW());
        
        RETURN jsonb_build_object(
            'status', 'error',
            'message', SQLERRM
        );
END;
$$ LANGUAGE plpgsql;
```

### Пример 2: Пакетная обработка с курсором

```sql
CREATE OR REPLACE FUNCTION apply_discount_to_old_books(
    p_days INTEGER,
    p_discount DECIMAL
)
RETURNS INTEGER AS $$
DECLARE
    book_cursor CURSOR FOR
        SELECT id, title, price, created_at
        FROM books
        WHERE created_at < NOW() - (p_days || ' days')::INTERVAL
        FOR UPDATE;
    
    v_book RECORD;
    v_updated INTEGER := 0;
    v_new_price DECIMAL;
BEGIN
    OPEN book_cursor;
    
    LOOP
        FETCH book_cursor INTO v_book;
        EXIT WHEN NOT FOUND;
        
        v_new_price := v_book.price * (1 - p_discount);
        
        UPDATE books 
        SET price = v_new_price
        WHERE CURRENT OF book_cursor;
        
        v_updated := v_updated + 1;
        
        RAISE NOTICE 'Книга "%s": старая цена %s, новая цена %s',
            v_book.title, v_book.price, v_new_price;
    END LOOP;
    
    CLOSE book_cursor;
    
    RETURN v_updated;
END;
$$ LANGUAGE plpgsql;
```

---



| Конструкция | Синтаксис | Пример |
|-------------|-----------|--------|
| **Функция** | `CREATE FUNCTION ... RETURNS ... AS $$ ... $$ LANGUAGE plpgsql` | — |
| **Переменная** | `v_name TYPE := value;` | `v_id INT := 1;` |
| **IF** | `IF condition THEN ... END IF;` | `IF price > 100 THEN ...` |
| **CASE** | `CASE value WHEN 1 THEN ... END` | `CASE status WHEN 'new' THEN ...` |
| **FOR (число)** | `FOR i IN 1..10 LOOP ... END LOOP` | — |
| **FOR (запрос)** | `FOR rec IN SELECT ... LOOP ... END LOOP` | — |
| **WHILE** | `WHILE condition LOOP ... END LOOP` | — |
| **EXIT** | `EXIT WHEN condition;` | `EXIT WHEN i > 10;` |
| **SELECT INTO** | `SELECT col INTO var FROM ...` | — |
| **PERFORM** | `PERFORM query;` | `PERFORM 1 FROM books;` |
| **RETURN** | `RETURN value;` | `RETURN v_result;` |
| **RETURN QUERY** | `RETURN QUERY SELECT ...` | — |
| **EXCEPTION** | `BEGIN ... EXCEPTION WHEN ... THEN ... END;` | — |
| **RAISE** | `RAISE LEVEL 'msg', arg;` | `RAISE NOTICE 'ID: %', id;` |
| **ASSERT** | `ASSERT condition, 'message';` | `ASSERT stock > 0;` |
| **EXECUTE** | `EXECUTE sql_string USING ...;` | — |
| **Курсор** | `CURSOR FOR SELECT ...` | — |
| **Массив** | `ARRAY[1,2,3]` | — |

---

# PL/pgSQL — Полный разбор синтаксиса для новичка (с комментарием каждого слова)

## Введение для абсолютного новичка

**PL/pgSQL** — это язык, на котором вы пишете программы, которые работают **внутри** базы данных PostgreSQL.

Представьте, что вы пишете рецепт:
- **SQL** — это "возьми муку", "возьми яйца" (простые команды)
- **PL/pgSQL** — это "смешай муку с яйцами, если тесто жидкое, добавь муки, повтори 3 раза" (алгоритм)

---


'192.168.1.1' = '192.168.1.1' -> true

'192.168.1.1' <> '10.2.10.1' -> true

'192.168.1.1' < '192.168.1.2' -> true

'192.168.1.1' > '192.168.1.1' -> false

'192.168.1.1' << '192.168.1.0/24' -> true

'192.168.1.0/24' <<= '192.168.1.0/24' -> true

'192.168.1.0/24' >> '192.168.1.10' -> true

'192.168.1.0/24' >>= '192.168.1.0/24' -> true

```
CREATE DATABASE mydb
    ENCODING 'UTF8'
    LC_COLLATE 'ru_RU.UTF-8'
    LC_CTYPE 'ru_RU.UTF-8'



```


```
CREATE TYPE person AS (
    first_name TEXT,
    last_name TEXT,
    age INTEGER

    );




```
```
CREATE FUNCTION get_address(customer_id INTEGER)
RETURNS person
LANGUAGE plpgsql
AS $$
DECLARE 
    result person;
BEGIN 
    SELECT addr INO result
    FROM customers
    WHERE id = customer_id
    RETURN result;
END;
$$;







```

```
BEGIN;

DECLARE cursor_name CURSOR FOR
    SELECT * FROM users WHERE age > 18;

OPEN cursor_name;

LOOP
   FETCH 100 FROM cursor_name;
   EXIT WHEN NOT FOUND;
END LOOP;
CLOSE cursor_name;

COMMIT;
```

## Часть 1. Самая простая функция (разбор каждого символа)

```sql
CREATE OR REPLACE FUNCTION hello_world()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN 'Привет, мир!';
END;
$$;
```

### Разберём каждое слово:

| Слово | Что означает | Комментарий |
|-------|--------------|-------------|
| `CREATE` | создать | Команда на создание чего-либо |
| `OR` | или | Ключевое слово |
| `REPLACE` | заменить | Вместе `CREATE OR REPLACE` = "создать или заменить, если уже есть" |
| `FUNCTION` | функция | То, что мы создаём (программа, которая возвращает значение) |
| `hello_world` | привет_мир | Имя нашей функции (вы придумываете сами) |
| `()` | круглые скобки | Здесь будут параметры (сейчас пусто) |
| `RETURNS` | возвращает | Какое значение вернёт функция |
| `TEXT` | текст | Тип возвращаемого значения (строка) |
| `LANGUAGE` | язык | На каком языке написана функция |
| `plpgsql` | пэл-пи-эс-кю-эль | Название языка (PostgreSQL Procedural Language) |
| `AS` | как | Начинается тело функции |
| `$$` | два доллара | Символы-ограничители (внутри них код) |
| `BEGIN` | начало | Начало блока кода |
| `RETURN` | вернуть | Команда вернуть значение |
| `'Привет, мир!'` | строка | Значение, которое возвращаем |
| `;` | точка с запятой | Конец команды |
| `END` | конец | Конец блока кода |
| `$$` | два доллара | Закрытие ограничителя |
| `;` | точка с запятой | Конец всей команды CREATE |

### Как вызвать функцию:

```sql
SELECT hello_world();
```

**Разбор:**
- `SELECT` — выбрать
- `hello_world()` — вызвать нашу функцию
- `;` — конец команды

**Результат:** `'Привет, мир!'`

---

## Часть 2. Функция с параметрами

```sql
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN a + b;
END;
$$;
```

### Разбор параметров:

| Слово | Что означает |
|-------|--------------|
| `add_numbers` | имя функции (сложить_числа) |
| `(` | начало списка параметров |
| `a` | имя первого параметра |
| `INTEGER` | тип первого параметра (целое число) |
| `,` | разделитель параметров |
| `b` | имя второго параметра |
| `INTEGER` | тип второго параметра |
| `)` | конец списка параметров |

### Разбор тела функции:

| Код | Что происходит |
|-----|----------------|
| `RETURN` | вернуть результат |
| `a + b` | вычислить сумму a и b |
| `;` | конец команды |

### Как вызвать:

```sql
SELECT add_numbers(5, 3);  -- Результат: 8
```

---

## Часть 3. Переменные (DECLARE)

```sql
CREATE OR REPLACE FUNCTION calculate_discount(price DECIMAL, discount_percent DECIMAL)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
DECLARE
    discount_amount DECIMAL;
    final_price DECIMAL;
BEGIN
    discount_amount := price * (discount_percent / 100);
    final_price := price - discount_amount;
    
    RETURN final_price;
END;
$$;
```

### Разбор DECLARE (объявление переменных):

| Слово | Что означает |
|-------|--------------|
| `DECLARE` | объявить (начало секции переменных) |
| `discount_amount` | имя переменной (сумма_скидки) |
| `DECIMAL` | тип переменной (дробное число) |
| `;` | конец объявления |
| `final_price` | имя переменной (итоговая_цена) |
| `DECIMAL` | тип переменной |
| `;` | конец объявления |

### Разбор присваивания:

| Код | Что означает |
|-----|--------------|
| `discount_amount` | переменная, которой присваиваем |
| `:=` | оператор присваивания (положи значение) |
| `price` | параметр (цена) |
| `*` | умножить |
| `(` | открыть скобку |
| `discount_percent` | параметр (процент скидки) |
| `/` | разделить |
| `100` | число 100 |
| `)` | закрыть скобку |
| `;` | конец команды |

**Как это читать:** 
"Переменной discount_amount присвоить значение price умножить на (discount_percent разделить на 100)"

### Как вызвать:

```sql
SELECT calculate_discount(1000, 20);  -- Результат: 800
```

---

## Часть 4. Условия (IF/THEN/ELSE)

```sql
CREATE OR REPLACE FUNCTION check_age(age INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    result TEXT;
BEGIN
    IF age < 18 THEN
        result := 'Ребёнок';
    ELSIF age < 65 THEN
        result := 'Взрослый';
    ELSE
        result := 'Пенсионер';
    END IF;
    
    RETURN result;
END;
$$;
```

### Разбор условия (построчно):

**Строка 1:** `IF age < 18 THEN`
| Слово | Что означает |
|-------|--------------|
| `IF` | если |
| `age` | переменная (возраст) |
| `<` | меньше |
| `18` | число 18 |
| `THEN` | тогда (начало блока, который выполнится, если условие верно) |

**Строка 2:** `result := 'Ребёнок';`
- Присвоить переменной result строку 'Ребёнок'

**Строка 3:** `ELSIF age < 65 THEN`
| Слово | Что означает |
|-------|--------------|
| `ELSIF` | иначе если (сокращение от ELSE IF) |
| `age` | возраст |
| `<` | меньше |
| `65` | число 65 |
| `THEN` | тогда |

**Строка 4:** `result := 'Взрослый';`
- Присвоить result строку 'Взрослый'

**Строка 5:** `ELSE`
| Слово | Что означает |
|-------|--------------|
| `ELSE` | иначе (во всех остальных случаях) |

**Строка 6:** `result := 'Пенсионер';`
- Присвоить result строку 'Пенсионер'

**Строка 7:** `END IF;`
| Слово | Что означает |
|-------|--------------|
| `END` | конец |
| `IF` | блока IF |
| `;` | конец команды |

### Как вызвать:

```sql
SELECT check_age(15);  -- 'Ребёнок'
SELECT check_age(30);  -- 'Взрослый'
SELECT check_age(70);  -- 'Пенсионер'
```

---

## Часть 5. SELECT INTO (получение данных из таблицы)

Предположим, у нас есть таблица:

```sql
CREATE TABLE products (
    id INTEGER,
    name TEXT,
    price DECIMAL
);

INSERT INTO products VALUES (1, 'Ноутбук', 50000);
```

**Функция, которая читает из таблицы:**

```sql
CREATE OR REPLACE FUNCTION get_product_name(product_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    product_name TEXT;
BEGIN
    SELECT name INTO product_name
    FROM products
    WHERE id = product_id;
    
    RETURN product_name;
END;
$$;
```

### Разбор SELECT INTO:

| Слово | Что означает |
|-------|--------------|
| `SELECT` | выбрать |
| `name` | колонка name (название товара) |
| `INTO` | в переменную |
| `product_name` | имя переменной (куда сохранить) |
| `FROM` | из |
| `products` | таблица products |
| `WHERE` | где |
| `id` | колонка id |
| `=` | равно |
| `product_id` | параметр функции |

**Как это читать:**
"Выбрать колонку name из таблицы products, где колонка id равна параметру product_id, и сохранить результат в переменную product_name"

### FOUND (проверка, найдена ли строка):

```sql
CREATE OR REPLACE FUNCTION get_product_name_safe(product_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    product_name TEXT;
BEGIN
    SELECT name INTO product_name
    FROM products
    WHERE id = product_id;
    
    IF NOT FOUND THEN
        RETURN 'Товар не найден';
    END IF;
    
    RETURN product_name;
END;
$$;
```

**Что такое `FOUND`:** 
- Специальная переменная
- `TRUE` (истина), если последний SELECT нашёл строку
- `FALSE` (ложь), если не нашёл

**Что такое `NOT FOUND`:**
- Проверка "НЕ найдено" (если строка НЕ существует)

---

## Часть 6. Циклы (LOOP)

### 6.1. Простой LOOP (бесконечный с выходом)

```sql
CREATE OR REPLACE FUNCTION count_to_ten()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    counter INTEGER := 1;
    result TEXT := '';
BEGIN
    LOOP
        result := result || counter || ' ';
        counter := counter + 1;
        
        EXIT WHEN counter > 10;
    END LOOP;
    
    RETURN result;
END;
$$;
```

**Разбор цикла:**

| Строка | Код | Что означает |
|--------|-----|--------------|
| 1 | `LOOP` | начало цикла |
| 2 | `result := result \|\| counter \|\| ' ';` | добавить счётчик к результату |
| 3 | `counter := counter + 1;` | увеличить счётчик на 1 |
| 4 | `EXIT WHEN counter > 10;` | выход из цикла, если счётчик больше 10 |
| 5 | `END LOOP;` | конец цикла |

**Что значит `||`:** оператор склеивания строк (конкатенации)

### 6.2. FOR LOOP (счётчик от и до)

```sql
CREATE OR REPLACE FUNCTION sum_numbers(n INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    total INTEGER := 0;
BEGIN
    FOR i IN 1..n LOOP
        total := total + i;
    END LOOP;
    
    RETURN total;
END;
$$;
```

**Разбор FOR LOOP:**

| Слово | Что означает |
|-------|--------------|
| `FOR` | для |
| `i` | переменная-счётчик |
| `IN` | в диапазоне |
| `1` | от 1 |
| `..` | до (две точки) |
| `n` | до n |
| `LOOP` | выполнять цикл |
| `total := total + i;` | тело цикла |
| `END LOOP;` | конец цикла |

**Как это читать:**
"Для i от 1 до n выполнять: total = total + i"

**Результат:** `sum_numbers(10)` = 55 (сумма чисел от 1 до 10)

### 6.3. FOR LOOP (обратный порядок)

```sql
FOR i IN REVERSE 10..1 LOOP
    -- i будет: 10, 9, 8, ..., 1
END LOOP;
```

### 6.4. WHILE LOOP

```sql
CREATE OR REPLACE FUNCTION power(base INTEGER, exponent INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    result INTEGER := 1;
    counter INTEGER := 0;
BEGIN
    WHILE counter < exponent LOOP
        result := result * base;
        counter := counter + 1;
    END LOOP;
    
    RETURN result;
END;
$$;
```

**Разбор WHILE:**

| Слово | Что означает |
|-------|--------------|
| `WHILE` | пока |
| `counter < exponent` | условие (пока счётчик меньше степени) |
| `LOOP` | выполнять цикл |
| (тело цикла) | result = result * base; counter = counter + 1 |
| `END LOOP;` | конец цикла |

**Как это читать:**
"Пока counter меньше exponent, выполнять: умножить result на base, увеличить counter на 1"

**Результат:** `power(2, 3)` = 8 (2³)

---

## Часть 7. CASE (множественный выбор)

```sql
CREATE OR REPLACE FUNCTION get_day_name(day_number INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    day_name TEXT;
BEGIN
    CASE day_number
        WHEN 1 THEN day_name := 'Понедельник';
        WHEN 2 THEN day_name := 'Вторник';
        WHEN 3 THEN day_name := 'Среда';
        WHEN 4 THEN day_name := 'Четверг';
        WHEN 5 THEN day_name := 'Пятница';
        WHEN 6 THEN day_name := 'Суббота';
        WHEN 7 THEN day_name := 'Воскресенье';
        ELSE day_name := 'Неверный номер дня';
    END CASE;
    
    RETURN day_name;
END;
$$;
```

**Разбор CASE:**

| Слово | Что означает |
|-------|--------------|
| `CASE` | выбор (начало) |
| `day_number` | переменная, которую проверяем |
| `WHEN 1 THEN` | если значение равно 1, то |
| `day_name := 'Понедельник';` | присвоить Понедельник |
| `WHEN 2 THEN` | если значение равно 2, то |
| ... | ... |
| `ELSE` | иначе (если ни одно условие не подошло) |
| `END CASE;` | конец выбора |

**Как это читать:**
"Проверить значение day_number: если 1 → Понедельник, если 2 → Вторник, ..., иначе → Неверный номер"

---

## Часть 8. RETURN NEXT и RETURN QUERY (возврат нескольких строк)

```sql
CREATE OR REPLACE FUNCTION get_numbers(limit_count INTEGER)
RETURNS SETOF INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1..limit_count LOOP
        RETURN NEXT i;
    END LOOP;
    
    RETURN;
END;
$$;
```

**Разбор:**

| Слово | Что означает |
|-------|--------------|
| `RETURNS SETOF INTEGER` | возвращает МНОЖЕСТВО целых чисел (не одно число) |
| `RETURN NEXT i` | вернуть ОДНО значение и продолжить (не выходить из функции) |
| `RETURN` | завершить функцию (без значения) |

**Как это работает:** Функция возвращает строки по одной, как таблицу.

**Вызов:**
```sql
SELECT * FROM get_numbers(5);
-- Результат:
-- 1
-- 2
-- 3
-- 4
-- 5
```

### RETURN QUERY (возврат результатов запроса)

```sql
CREATE OR REPLACE FUNCTION get_expensive_books(min_price DECIMAL)
RETURNS SETOF products
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT * FROM products WHERE price > min_price;
    
    RETURN;
END;
$$;
```

**Разбор:**
| Слово | Что означает |
|-------|--------------|
| `RETURN QUERY` | вернуть результат запроса |
| `SELECT * FROM products` | запрос |
| `WHERE price > min_price` | условие |

---

## Часть 9. Обработка ошибок (EXCEPTION)

```sql
CREATE OR REPLACE FUNCTION divide_numbers(a DECIMAL, b DECIMAL)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN a / b;
EXCEPTION
    WHEN division_by_zero THEN
        RETURN NULL;
END;
$$;
```

**Разбор:**

| Слово | Что означает |
|-------|--------------|
| `BEGIN` | начало блока (здесь выполняется нормальный код) |
| `RETURN a / b;` | попытаться разделить |
| `EXCEPTION` | исключение (блок для обработки ошибок) |
| `WHEN division_by_zero THEN` | если произошла ошибка "деление на ноль" |
| `RETURN NULL;` | вернуть NULL вместо ошибки |
| `END;` | конец блока |

**Что такое `division_by_zero`:** это название ошибки (условие)

**Другие условия ошибок:**
- `unique_violation` — нарушение уникальности
- `foreign_key_violation` — нарушение внешнего ключа
- `check_violation` — нарушение CHECK
- `OTHERS` — все остальные ошибки

---

## Часть 10. RAISE (вывод сообщений)

```sql
CREATE OR REPLACE FUNCTION debug_demo(value INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Начало функции. Значение = %', value;
    
    IF value < 0 THEN
        RAISE EXCEPTION 'Ошибка: значение не может быть отрицательным (%)', value;
    END IF;
    
    RAISE NOTICE 'Всё хорошо, значение = %', value;
    
    RETURN value * 2;
END;
$$;
```

**Разбор RAISE:**

| Слово | Что означает |
|-------|--------------|
| `RAISE` | поднять (вывести сообщение) |
| `NOTICE` | уровень сообщения (уведомление) |
| `'Начало функции. Значение = %'` | текст сообщения, % — место для подстановки |
| `, value` | значение, которое подставляется вместо % |

**Уровни сообщений (от самых важных до самых неважных):**
- `EXCEPTION` — ошибка (останавливает функцию)
- `WARNING` — предупреждение
- `NOTICE` — уведомление (видно по умолчанию)
- `INFO` — информация
- `DEBUG` — отладка (не видно без настройки)

---

## Часть 11. Массивы

```sql
CREATE OR REPLACE FUNCTION sum_array(arr INTEGER[])
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    total INTEGER := 0;
    i INTEGER;
BEGIN
    FOR i IN 1..array_length(arr, 1) LOOP
        total := total + arr[i];
    END LOOP;
    
    RETURN total;
END;
$$;
```

**Разбор синтаксиса массива:**

| Код | Что означает |
|-----|--------------|
| `arr INTEGER[]` | параметр arr — массив целых чисел (квадратные скобки означают массив) |
| `array_length(arr, 1)` | длина массива arr по первому измерению |
| `arr[i]` | элемент массива с индексом i (индексы начинаются с 1) |

**Вызов:**
```sql
SELECT sum_array(ARRAY[1, 2, 3, 4, 5]);  -- Результат: 15
SELECT sum_array('{10,20,30}'::INTEGER[]);  -- Результат: 60
```

### Конструктор массива ARRAY:

| Синтаксис | Что означает |
|-----------|--------------|
| `ARRAY[1,2,3]` | создать массив из чисел 1,2,3 |
| `'{1,2,3}'::INTEGER[]` | привести строку к типу массив (другой способ) |

---

## Часть 12. Динамические команды (EXECUTE)

```sql
CREATE OR REPLACE FUNCTION execute_dynamic(table_name TEXT, column_name TEXT, value TEXT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    result TEXT;
BEGIN
    EXECUTE format('SELECT %I FROM %I WHERE id = %L', column_name, table_name, value)
    INTO result;
    
    RETURN result;
END;
$$;
```

**Разбор EXECUTE:**

| Слово | Что означает |
|-------|--------------|
| `EXECUTE` | выполнить строку как SQL-команду |
| `format(...)` | функция форматирования строки |
| `%I` | вставить имя (таблицы/колонки) с защитой от SQL-инъекций |
| `%L` | вставить строковое значение с кавычками |
| `INTO result` | сохранить результат в переменную |

**Почему нельзя просто написать `SELECT * FROM table_name`:** 
Потому что PostgreSQL не подставит значение переменной как имя таблицы. Нужен EXECUTE.

---

## Часть 13. Триггеры (специальная функция)

```sql
CREATE OR REPLACE FUNCTION trg_update_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$;
```

**Специальные слова для триггеров:**

| Слово | Что означает |
|-------|--------------|
| `RETURNS TRIGGER` | функция возвращает TRIGGER (специальный тип) |
| `NEW` | новая строка (для INSERT/UPDATE) |
| `OLD` | старая строка (для UPDATE/DELETE) |
| `TG_OP` | операция ('INSERT', 'UPDATE', 'DELETE') |
| `TG_TABLE_NAME` | имя таблицы |

**Создание триггера:**

```sql
CREATE TRIGGER trigger_name
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION trg_update_timestamp();
```

**Разбор CREATE TRIGGER:**

| Слово | Что означает |
|-------|--------------|
| `CREATE TRIGGER` | создать триггер |
| `trigger_name` | имя триггера |
| `BEFORE` | перед выполнением операции |
| `UPDATE` | при обновлении |
| `ON products` | на таблице products |
| `FOR EACH ROW` | для каждой строки |
| `EXECUTE FUNCTION` | выполнить функцию |
| `trg_update_timestamp()` | имя функции-триггера |

---

## Часть 14. Полный пример с комментариями

```sql
-- =============================================================================
-- Функция: create_order
-- Описание: Создаёт новый заказ для покупателя
-- Параметры:
--   p_customer_id - ID покупателя
--   p_items - массив товаров в формате JSON: [{"id":1,"qty":2}, ...]
-- Возвращает: ID созданного заказа или NULL при ошибке
-- =============================================================================

CREATE OR REPLACE FUNCTION create_order(
    p_customer_id INTEGER,          -- ID покупателя (целое число)
    p_items JSONB                   -- Список товаров (JSON-объект)
)
RETURNS INTEGER                     -- Вернём ID заказа (целое число)
LANGUAGE plpgsql                    -- Язык: PL/pgSQL
AS $$                               -- Начало тела функции
DECLARE
    -- Объявляем переменные
    v_order_id INTEGER;             -- ID нового заказа
    v_total DECIMAL := 0;           -- Общая сумма заказа (начинается с 0)
    v_item RECORD;                  -- Для перебора товаров (запись)
    v_book_price DECIMAL;           -- Цена книги
    v_book_stock INTEGER;           -- Остаток книги
BEGIN
    -- ========================================================================
    -- Шаг 1: Создаём запись о заказе
    -- ========================================================================
    INSERT INTO orders (customer_id, status, order_date)
    VALUES (p_customer_id, 'new', NOW())
    RETURNING id INTO v_order_id;
    -- ↑ RETURNING id - вернуть автоматически сгенерированный ID
    -- ↑ INTO v_order_id - сохранить в переменную
    
    -- ========================================================================
    -- Шаг 2: Обрабатываем каждый товар из JSON-массива
    -- ========================================================================
    FOR v_item IN 
        SELECT * FROM jsonb_to_recordset(p_items) AS x(book_id INT, quantity INT)
        -- ↑ jsonb_to_recordset - превращает JSON-массив в строки таблицы
        -- ↑ book_id INT - каждая строка имеет поле book_id (целое)
        -- ↑ quantity INT - и поле quantity (целое)
    LOOP
        -- 2.1: Получаем цену и остаток книги
        SELECT price, stock 
        INTO v_book_price, v_book_stock
        FROM books 
        WHERE id = v_item.book_id;
        
        -- 2.2: Проверяем, достаточно ли товара на складе
        IF v_book_stock < v_item.quantity THEN
            RAISE EXCEPTION 'Недостаточно товара! Книга ID: %, остаток: %, заказано: %',
                v_item.book_id, v_book_stock, v_item.quantity;
            -- ↑ RAISE EXCEPTION - выбрасываем ошибку (заказ не создастся)
        END IF;
        
        -- 2.3: Добавляем позицию в таблицу order_items
        INSERT INTO order_items (order_id, book_id, quantity, price_at_moment)
        VALUES (v_order_id, v_item.book_id, v_item.quantity, v_book_price);
        
        -- 2.4: Уменьшаем остаток на складе
        UPDATE books 
        SET stock = stock - v_item.quantity
        WHERE id = v_item.book_id;
        
        -- 2.5: Добавляем к общей сумме
        v_total := v_total + (v_book_price * v_item.quantity);
        
    END LOOP;  -- Конец цикла по товарам
    
    -- ========================================================================
    -- Шаг 3: Обновляем сумму заказа
    -- ========================================================================
    UPDATE orders 
    SET total_amount = v_total 
    WHERE id = v_order_id;
    
    -- ========================================================================
    -- Шаг 4: Возвращаем ID созданного заказа
    -- ========================================================================
    RETURN v_order_id;
    
-- ============================================================================
-- Блок обработки ошибок (если что-то пошло не так)
-- ============================================================================
EXCEPTION
    WHEN OTHERS THEN
        -- Логируем ошибку в таблицу error_log
        INSERT INTO error_log (function_name, error_message, error_data, error_time)
        VALUES (
            'create_order',                 -- имя функции
            SQLERRM,                        -- текст ошибки
            jsonb_build_object(             -- данные, которые привели к ошибке
                'customer_id', p_customer_id,
                'items', p_items
            ),
            NOW()                           -- время ошибки
        );
        
        -- Выводим уведомление
        RAISE NOTICE 'Ошибка при создании заказа: %', SQLERRM;
        
        -- Возвращаем NULL (сигнал об ошибке)
        RETURN NULL;
END;                                    -- Конец функции
$$;                                     -- Конец тела функции
```

**Как вызвать эту функцию:**

```sql
SELECT create_order(
    1,  -- покупатель с ID=1
    '[{"book_id": 1, "quantity": 2}, {"book_id": 3, "quantity": 1}]'::JSONB
);
```

---

## Шпаргалка: таблица всех изученных слов

| Слово/символ | Русский смысл | Где используется |
|--------------|---------------|------------------|
| `CREATE` | создать | создание функции |
| `OR` | или | CREATE OR REPLACE |
| `REPLACE` | заменить | CREATE OR REPLACE |
| `FUNCTION` | функция | CREATE FUNCTION |
| `RETURNS` | возвращает | тип возврата |
| `LANGUAGE` | язык | указание PL/pgSQL |
| `AS` | как | начало тела |
| `$$` | (ограничитель) | вместо кавычек |
| `DECLARE` | объявить | секция переменных |
| `BEGIN` | начало | начало блока кода |
| `END` | конец | конец блока |
| `:=` | присвоить | оператор присваивания |
| `=` | равно | сравнение |
| `<>` или `!=` | не равно | сравнение |
| `<` | меньше | сравнение |
| `>` | больше | сравнение |
| `IF` | если | условие |
| `THEN` | тогда | условие |
| `ELSE` | иначе | условие |
| `ELSIF` | иначе если | условие |
| `END IF` | конец условия | условие |
| `CASE` | выбор | множественный выбор |
| `WHEN` | когда | CASE |
| `END CASE` | конец выбора | CASE |
| `LOOP` | цикл | начало цикла |
| `END LOOP` | конец цикла | цикл |
| `FOR` | для | цикл FOR |
| `WHILE` | пока | цикл WHILE |
| `EXIT` | выйти | выход из цикла |
| `CONTINUE` | продолжить | переход к следующей итерации |
| `RETURN` | вернуть | выход из функции |
| `RETURN NEXT` | вернуть строку | SRF функция |
| `RETURN QUERY` | вернуть запрос | SRF функция |
| `SELECT INTO` | выбрать в переменную | чтение данных |
| `PERFORM` | выполнить | выполнить без результата |
| `FOUND` | найдено | проверка результата SELECT |
| `NOT FOUND` | не найдено | проверка |
| `EXCEPTION` | исключение | обработка ошибок |
| `WHEN OTHERS THEN` | при любой ошибке | обработка ошибок |
| `RAISE` | поднять | вывод сообщения |
| `NOTICE` | уведомление | уровень сообщения |
| `EXCEPTION` | исключение | уровень сообщения (ошибка) |
| `WARNING` | предупреждение | уровень сообщения |
| `DEBUG` | отладка | уровень сообщения |
| `ASSERT` | проверить | проверка условия |
| `EXECUTE` | выполнить | динамический SQL |
| `format()` | форматировать | форматирование строк |
| `%I` | идентификатор | placeholder для format() |
| `%L` | литерал | placeholder для format() |
| `%s` | строка | placeholder для format() |
| `NEW` | новый | в триггере (новая строка) |
| `OLD` | старый | в триггере (старая строка) |
| `TG_OP` | операция | в триггере (INSERT/UPDATE/DELETE) |
| `RETURNS TRIGGER` | возвращает триггер | тип триггерной функции |
| `ARRAY[]` | массив | создание массива |
| `array_length()` | длина массива | функция |
| `JSONB` | JSON-объект | тип данных |
| `->` | извлечь | оператор JSON |
| `->>` | извлечь как текст | оператор JSON |

---

