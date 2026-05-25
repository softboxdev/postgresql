
---

# MemoryContext в PostgreSQL: полный разбор

## 1. Проблема: почему недостаточно просто `malloc` и `free`

В обычном C-приложении вы пишете:
```c
void my_function() {
    char *buffer = malloc(1024);
    // ... работа
    free(buffer);
}
```

В серверном коде PostgreSQL (внутри процесса `postgres`) такой подход **катастрофически опасен** по трём причинам:

### 1.1. Исключения (ошибки) и longjmp

PostgreSQL использует `ereport()` / `elog()` для генерации ошибок. Когда происходит ошибка, управление передаётся через `sigsetjmp`/`longjmp` в точку начала обработки запроса.

**Что плохого в `malloc`:**
```c
char *buffer = malloc(1_000_000);
if (!buffer) ereport(ERROR, ...); // <- longjmp
// Если ошибка произошла ПОСЛЕ malloc, но ДО free, то free уже не вызовется
free(buffer); // <- никогда не выполнится при ошибке
```

`longjmp` просто выкидывает из функции, не вызывая деструкторы и не очищая память. Утечка гарантирована.

### 1.2. Производительность миллиона маленьких аллокаций

В цикле обработки 10 миллионов строк:
```c
for (int i = 0; i < 10_000_000; i++) {
    char *tmp = malloc(64);
    // ... работа
    free(tmp);
}
```

`malloc`/`free` 10 миллионов раз — это миллионы системных вызовов (`sbrk`/`mmap`), фрагментация кучи, и общая смерть производительности.

### 1.3. Отсутствие семантики транзакций

Память, выделенная в транзакции, должна автоматически освобождаться при ROLLBACK или COMMIT. Обычный `malloc` не знает про транзакции.

---

## 2. Решение: иерархические MemoryContext

`MemoryContext` — это абстракция, которая:
1. Группирует аллокации в логические области (контексты).
2. Позволяет удалить **весь контекст** целиком (и всю память в нём) одной операцией.
3. Освобождает память автоматически при ошибках (`longjmp`).
4. Привязан к транзакционной семантике.

### 2.1. Основные принципы

*   Каждый `MemoryContext` — это узел в дереве (может иметь родителя и детей).
*   У каждого контекста есть **тип** — конкретная стратегия выделения памяти (обычно `AllocSetContext`).
*   Выделение памяти через `palloc()` — выделяет из **текущего** контекста (`CurrentMemoryContext`).
*   Переключение контекста — `MemoryContextSwitchTo()`.

### 2.2. Структура контекста (упрощённо)

Из `src/include/utils/memutils.h`:

```c
typedef struct MemoryContextData
{
    NodeTag        type;           // идентификатор типа контекста (T_AllocSetContext и т.д.)
    MemoryContext  parent;         // указатель на родителя
    const char    *name;           // имя для отладки (например, "TransactionContext")
    void          *methods;        // виртуальные функции: alloc, free, delete, reset...
    struct MemoryContextData *firstchild; // первый дочерний контекст
    struct MemoryContextData *nextchild;  // следующий дочерний (список)
} MemoryContextData;
```

Реальная память хранится в специализированной структуре `AllocSetContext` (наследует `MemoryContextData`):

```c
typedef struct AllocSetContext
{
    MemoryContextData header;      // обязательная голова
    AllocBlock    blocks;          // список больших блоков памяти (через malloc)
    AllocChunk    freelist[ALLOCSET_NUM_FREELISTS]; // массивы мелких фрагментов
    Size          initBlockSize;   // размер первого блока
    Size          maxBlockSize;    // максимальный размер блока
    Size          nextBlockSize;   // следующий размер (удваивается)
    Size          allocChunkLimit; // порог для перехода в отдельный блок
} AllocSetContext;
```

---

## 3. Как работает аллокация: `palloc` внутри

Псевдокод (упрощённый из реального `src/backend/utils/mmgr/aset.c`):

```c
void * palloc(Size size)
{
    // 1. Проверка размера
    if (size == 0) return NULL;
    
    // 2. Определяем текущий контекст
    MemoryContext context = CurrentMemoryContext;
    
    // 3. Вызываем виртуальную функцию alloc этого контекста
    void *result = context->methods->alloc(context, size);
    
    // 4. В отладочной сборке заполняем 0x7F (для поиска ошибок)
    VALGRIND_MAKE_MEM_UNDEFINED(result, size);
    
    return result;
}
```

### 3.1. Аллокация в `AllocSetContext` (стратегия по умолчанию)

Вызов `AllocSetAlloc()` делает следующее:

**Случай 1: Маленький размер (≤ 8 КБ по умолчанию)**
1. Округляет размер до степени двойки (8, 16, 32, 64, 128, ... 8192).
2. Идёт в соответствующий `freelist[index]` (массив свободных чанков).
3. Если есть свободный чанк — возвращает его (очень быстро, просто перемещая указатель).
4. Если нет — запрашивает новый блок у `malloc` (обычно 8 КБ) и нарезает его на чанки нужного размера.

**Случай 2: Большой размер (> 8 КБ)**
1. Выделяет отдельный блок через `malloc` (ровно нужного размера + служебная информация).
2. Этот блок не попадает в freelist, будет освобождён только при `pfree()` или удалении контекста.

**Визуализация:**
```
AllocSetContext
├── blocks (список выделенных блоков от malloc)
│   ├── Block 1 (8 КБ) -> чанки 64 байта, 128 байт, 256...
│   ├── Block 2 (8 КБ) -> чанки...
│   └── Block 3 (16 КБ) -> чанки...
└── freelist[0] (8 байт) -> chunk -> chunk -> NULL
    freelist[1] (16 байт) -> chunk -> NULL
    freelist[2] (32 байта) -> NULL
    ...
```

### 3.2. Почему это быстро

*   **Нет системных вызовов** в `palloc` для 99% случаев — просто перемещение указателя в уже выделенном блоке.
*   **Размеры фиксированы** (степени двойки) — нет фрагментации и быстрый поиск в freelist.
*   **Освобождение контекста целиком** — просто `free()` каждого блока через `malloc`, без обхода каждого чанка.

---

## 4. Иерархия MemoryContext в процессе PostgreSQL

При старте процесса `postgres` создаётся предопределённое дерево контекстов:

```
TopMemoryContext (живёт весь процесс)
├── CacheMemoryContext (кэш системных каталогов, никогда не сбрасывается)
├── MessageContext (сетевые буферы сообщений)
├── TopTransactionContext (данные текущей транзакции, живут до COMMIT/ROLLBACK)
│   └── CurTransactionContext (временные данные в рамках транзакции)
├── ErrorContext (для обработки ошибок, всегда доступен)
└── PortalMemory (для порталов - подготовленных запросов)
    └── PortalHeapMemory (данные конкретного портала)
```

### 4.1. Жизненный цикл транзакции и памяти

1. **Клиент отправляет `BEGIN`:**
   - `TopTransactionContext` создаётся (или очищается от предыдущей транзакции).
   - `CurrentMemoryContext` устанавливается на `CurTransactionContext`.

2. **Выполняется запрос:**
   - Все `palloc` внутри вашей функции попадают в `CurTransactionContext`.
   - Если вы создаёте временные структуры для одного запроса — создайте дочерний контекст от `CurTransactionContext`.

3. **Ошибка в запросе (`ereport(ERROR)`):**
   - Выполняется `longjmp` к началу обработки.
   - `CurTransactionContext` полностью удаляется (все дочерние рекурсивно).
   - `CurrentMemoryContext` переключается на `TopTransactionContext` (чистый лист).

4. **`COMMIT` или `ROLLBACK`:**
   - Удаляется `CurTransactionContext` (и всё, что в нём выделяли ваши функции).
   - `TopTransactionContext` очищается, но остаётся жив для следующей транзакции.

---

## 5. Практические примеры использования в серверном коде

### 5.1. Безопасная временная память внутри функции

```c
Datum my_function(PG_FUNCTION_ARGS)
{
    // Вход: CurrentMemoryContext указывает на CurTransactionContext
    // Если мы выделим память напрямую, она проживёт до конца транзакции
    
    // Правильный подход: создаём временный контекст
    MemoryContext old_ctx = CurrentMemoryContext;
    MemoryContext tmp_ctx = AllocSetContextCreate(
        old_ctx,                     // родитель
        "my_function_tmp",
        ALLOCSET_DEFAULT_SIZES
    );
    
    MemoryContextSwitchTo(tmp_ctx);
    
    // Теперь вся память внутри tmp_ctx
    char *buffer = palloc(1024);
    HeapTuple *rows = palloc(sizeof(HeapTuple) * 1000);
    
    // ... интенсивные вычисления ...
    
    // Освобождаем всю временную память одной командой
    MemoryContextSwitchTo(old_ctx);
    MemoryContextDelete(tmp_ctx);  // <- все palloc внутри буфера и rows освобождены
    
    PG_RETURN_INT32(result);
}
```

**Что будет, если не удалить tmp_ctx:**
Память останется висеть до конца транзакции. При миллионе вызовов функции вы получите утечку в миллион контекстов — процесс умрёт от `out of memory`.

### 5.2. Кэширование между вызовами функций

```c
static MemoryContext cache_ctx = NULL;
static HTAB *my_cache = NULL;

Datum cached_lookup(PG_FUNCTION_ARGS)
{
    if (cache_ctx == NULL) {
        // Первый вызов в этом процессе
        cache_ctx = AllocSetContextCreate(
            TopMemoryContext,  // живёт вечно
            "my_permanent_cache",
            ALLOCSET_DEFAULT_SIZES
        );
        
        MemoryContext old = MemoryContextSwitchTo(cache_ctx);
        
        // Создаём хеш-таблицу в этом контексте
        HASHCTL ctl = { ... };
        my_cache = hash_create("my_cache", 100, &ctl, HASH_ELEM | HASH_BLOBS);
        
        MemoryContextSwitchTo(old);
    }
    
    // Используем кэш
    void *result = hash_search(my_cache, ...);
    PG_RETURN_POINTER(result);
}
```

**Важно:** Кэш живёт в `TopMemoryContext`, поэтому не очищается между транзакциями. Нужно продумать механизм инвалидации (обычно через `CacheRegisterRelcacheCallback`).

### 5.3. Работа с большими объёмами данных в цикле

```c
Datum process_million_rows(PG_FUNCTION_ARGS)
{
    MemoryContext old_ctx = CurrentMemoryContext;
    MemoryContext per_row_ctx = AllocSetContextCreate(
        old_ctx,
        "per_row",
        ALLOCSET_SMALL_SIZES  // 1 КБ начальный блок
    );
    
    for (int i = 0; i < 1000000; i++) {
        MemoryContextSwitchTo(per_row_ctx);
        
        // Выделяем память для одной строки
        char *row_data = palloc(512);
        // ... обработка ...
        
        // Очищаем всё, что выделили для строки
        MemoryContextReset(per_row_ctx);  // <- быстрее, чем удалять и создавать заново
        
        // Переключаемся назад
        MemoryContextSwitchTo(old_ctx);
    }
    
    MemoryContextDelete(per_row_ctx);
    PG_RETURN_VOID();
}
```

**Почему это важно:** Без `MemoryContextReset` на каждой итерации память бы росла до конца транзакции, и на 1 миллионной строке процесс занял бы гигабайты.

---

## 6. Внутреннее устройство: как работает удаление контекста

Вызов `MemoryContextDelete(context)`:

```c
void MemoryContextDelete(MemoryContext context)
{
    // 1. Рекурсивно удалить всех детей
    MemoryContext child = context->firstchild;
    while (child) {
        MemoryContext next = child->nextchild;
        MemoryContextDelete(child);
        child = next;
    }
    
    // 2. Вызвать метод удаления конкретного контекста
    context->methods->delete(context);
    
    // 3. Отсоединить от родителя
    if (context->parent) {
        // убрать из списка children у родителя
    }
    
    // 4. Освободить память под сам MemoryContextData
    pfree(context);
}
```

Метод `delete` для `AllocSetContext`:
```c
static void AllocSetDelete(MemoryContext context)
{
    AllocSet set = (AllocSet) context;
    
    // Пройти по всем блокам и вызвать free() для каждого
    AllocBlock block = set->blocks;
    while (block) {
        AllocBlock next = block->next;
        free(block);  // <- системный вызов
        block = next;
    }
    
    // Не нужно чистить freelist — блоки уже удалены
}
```

**Ключевое:** Удаление контекста с 1 миллионом маленьких объектов требует всего `O(число блоков)` операций (обычно десятки), а не миллион.

---

## 7. Типичные ошибки и их последствия

### 7.1. Утечка памяти через `TopMemoryContext`

```c
void bad_function() {
    MemoryContext old = MemoryContextSwitchTo(TopMemoryContext);
    char *leak = palloc(1024);  // <- навсегда!
    MemoryContextSwitchTo(old);
}
```

**Результат:** Каждый вызов функции добавляет 1 КБ в `TopMemoryContext`. Процесс умрёт через N вызовов.

### 7.2. Использование `malloc` вместо `palloc`

```c
void dangerous() {
    char *buf = malloc(1024);
    if (some_error_condition)
        ereport(ERROR, ...);  // <- утечка buf, free не вызовется
    free(buf);  // <- не выполнится при ошибке
}
```

**Результат:** Утечка при любом `ERROR` (а это может быть `ERROR: out of memory`, `ERROR: division by zero` и т.д.).

### 7.3. Забыл переключить контекст обратно

```c
void bug() {
    MemoryContext tmp = AllocSetContextCreate(...);
    MemoryContextSwitchTo(tmp);
    char *data = palloc(100);
    // Забыли: MemoryContextSwitchTo(old);
    return;  // CurrentMemoryContext остался tmp!
}
```

**Результат:** Следующая функция, которая вызовет `palloc`, выделит память в `tmp`, что приведёт к странным ошибкам или падению при попытке удалить `tmp` раньше времени.

### 7.4. Сохранение указателя после удаления контекста

```c
char *global_ptr;

void store() {
    MemoryContext tmp = AllocSetContextCreate(...);
    MemoryContextSwitchTo(tmp);
    global_ptr = palloc(100);
    MemoryContextSwitchTo(old);
    MemoryContextDelete(tmp);  // <- global_ptr теперь висячий указатель
}

void use() {
    strcpy(global_ptr, "hello");  // <- SEGFAULT или порча памяти
}
```

**Результат:** Неопределённое поведение, падение процесса PostgreSQL.

---

## 8. Отладка проблем с MemoryContext

### 8.1. Включение проверок в сборке

```bash
./configure --enable-cassert --enable-debug
make
```

С этими флагами:
- `palloc` забивает память 0x7F.
- `pfree` забивает 0xFE.
- Проверяется, что `pfree` вызывается на корректном указателе.

### 8.2. Дамп всех контекстов

```c
MemoryContextStats(TopMemoryContext);
```

Вывод в лог:
```
TopMemoryContext: 1048576 total, 123456 in 42 blocks
  CacheMemoryContext: 524288 total, 98765 in 12 blocks
  my_function_tmp: 8192 total, 4096 in 1 block
```

### 8.3. Valgrind с PostgreSQL

```bash
valgrind --leak-check=full --track-origins=yes postgres -D /path/to/data
```

Valgrind поймает:
- Утечки памяти (даже в контекстах).
- Использование после `pfree`.
- Выход за границы аллоцированного блока.

---

## 9. Сравнение с другими подходами

| Механизм | Скорость | Автоочистка при ошибке | Транзакционная семантика | Контроль над временем жизни |
|----------|----------|------------------------|--------------------------|----------------------------|
| `malloc`/`free` | Средняя | Нет | Нет | Ручной |
| `alloca` (стек) | Очень высокая | Да (при выходе из функции) | Нет | Пока функция не завершится |
| `palloc` без контекста | Высокая | Да (при ошибке) | Да | До конца транзакции |
| `palloc` + временный контекст | Высокая | Да | Да | Контролируемый |

---

## 10. Итог: ментальная модель

Запомните три правила:

1. **В серверном коде PostgreSQL всегда используйте `palloc`/`pfree`, никогда `malloc`/`free`.**
2. **Создавайте временные контексты для циклов и больших объёмов данных.**
3. **Всегда переключайте `CurrentMemoryContext` обратно через `MemoryContextSwitchTo(old)`.**

`MemoryContext` — это не просто менеджер памяти, это фундамент надёжности PostgreSQL. Он превращает ручное управление памятью в декларативное: «вся память, выделенная в этом логическом блоке, должна быть освобождена, когда блок завершится».

Понимание этой системы позволяет писать расширения на C, которые живут годы без утечек и падений — именно так работает ядро PostgreSQL самого по себе.

Три интерфейса — `spi.h`, `tuptable.h` и `indexam.h` — представляют три разных уровня погружения в ядро PostgreSQL:

1.  **`spi.h`** — высокий уровень: выполнить SQL из C-функции (как «встроенный клиент» внутри базы).
2.  **`tuptable.h`** — представление данных: как вернуть из C-функции целую таблицу (SRF).
3.  **`indexam.h`** — глубокий уровень: создать свой тип индекса (свой access method).

Разберём каждый максимально подробно, с примерами, внутренними структурами и типичными ошибками.

---

# 1. SPI (Server Programming Interface) — `spi.h`

## 1.1. Зачем нужен SPI

SPI — это мост между вашим C-кодом (расширением) и ядром PostgreSQL. Он позволяет:

- Выполнять любые SQL-запросы (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`…).
- Получать результаты в виде строк.
- Использовать параметризованные запросы (безопасно, без инъекций).
- Работать в контексте текущей транзакции (ваш код и SPI-запросы — в одной транзакции).

**Ключевое отличие от клиентского драйвера (libpq):** SPI не использует сеть, не парсит протокол FE/BE, не создаёт отдельный процесс. Он напрямую вызывает парсер, планировщик и исполнитель PostgreSQL, работая с памятью в текущем процессе.

## 1.2. Подключение и инициализация

```c
#include "executor/spi.h"

Datum my_spi_function(PG_FUNCTION_ARGS)
{
    // 1. Подключиться к SPI (получить доступ к executor'у)
    int ret = SPI_connect();
    if (ret != SPI_OK_CONNECT)
        ereport(ERROR, (errmsg("SPI_connect failed: %d", ret)));
    
    // 2. Здесь будет выполнение запросов
    
    // 3. Отключиться
    SPI_finish();
    
    PG_RETURN_VOID();
}
```

**Важно:** `SPI_connect()` переключает `CurrentMemoryContext` на специальный контекст `SPIProcContext`. Не пытайтесь `palloc` что-то до `SPI_finish()` и сохранить указатель — память будет освобождена.

## 1.3. Выполнение простого SELECT

```c
int ret = SPI_execute("SELECT * FROM users WHERE id = 1", false, 0);
if (ret != SPI_OK_SELECT)
    ereport(ERROR, (errmsg("SPI_execute failed: %d", ret)));

// Получить количество строк
int nrows = SPI_processed;
// Получить указатель на результаты (массив строк)
SPITupleTable *tuptable = SPI_tuptable;
HeapTuple tuple = tuptable->vals[0];  // первая строка
TupleDesc tupdesc = tuptable->tupdesc; // описание колонок
```

**Структура `SPITupleTable`:**
```c
typedef struct SPITupleTable
{
    MemoryContext  tuptabcxt;    // контекст, где живут строки
    HeapTuple     *vals;         // массив указателей на строки
    uint64         numvals;      // количество строк
    TupleDesc      tupdesc;      // описание колонок
    // ... служебные поля
} SPITupleTable;
```

## 1.4. Безопасные параметризованные запросы (без инъекций)

Никогда не делайте так:
```c
char query[256];
sprintf(query, "SELECT * FROM users WHERE name = '%s'", user_input); // УЯЗВИМО!
SPI_execute(query, false, 0);
```

Правильный способ — `SPI_prepare` + `SPI_execute_plan`:

```c
// 1. Подготовить план (один раз, можно кэшировать)
const char *query = "SELECT * FROM users WHERE name = $1 AND age > $2";
SPIPlanPtr plan = SPI_prepare(query, 2, NULL);
if (plan == NULL)
    ereport(ERROR, (errmsg("SPI_prepare failed")));

// 2. Заполнить параметры
Datum values[2];
bool nulls[2];
values[0] = CStringGetTextDatum(user_input); // параметр $1
nulls[0] = false;
values[1] = Int32GetDatum(18); // параметр $2
nulls[1] = false;

// 3. Выполнить
int ret = SPI_execute_plan(plan, values, nulls, false, 0);
```

## 1.5. Полный пример: функция, которая логирует действие в другую таблицу

```c
#include "postgres.h"
#include "fmgr.h"
#include "executor/spi.h"
#include "utils/builtins.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(log_user_action);

Datum log_user_action(PG_FUNCTION_ARGS)
{
    int32 user_id = PG_GETARG_INT32(0);
    text *action = PG_GETARG_TEXT_PP(1);
    
    // Подключиться к SPI
    if (SPI_connect() != SPI_OK_CONNECT)
        ereport(ERROR, (errmsg("SPI_connect failed")));
    
    // Подготовить INSERT (с параметрами)
    SPIPlanPtr plan = SPI_prepare(
        "INSERT INTO audit_log (user_id, action, created_at) VALUES ($1, $2, now())",
        2,  // два параметра
        NULL
    );
    
    if (plan == NULL) {
        SPI_finish();
        ereport(ERROR, (errmsg("Failed to prepare INSERT")));
    }
    
    Datum values[2];
    bool nulls[2];
    values[0] = Int32GetDatum(user_id);
    nulls[0] = false;
    values[1] = PointerGetDatum(action);
    nulls[1] = false;
    
    int ret = SPI_execute_plan(plan, values, nulls, false, 0);
    
    SPI_finish();
    
    if (ret != SPI_OK_INSERT)
        ereport(ERROR, (errmsg("INSERT failed: %d", ret)));
    
    PG_RETURN_VOID();
}
```

## 1.6. Получение скалярного значения (одна строка, одна колонка)

```c
int ret = SPI_execute("SELECT COUNT(*) FROM users", false, 0);
if (ret == SPI_OK_SELECT && SPI_processed > 0) {
    HeapTuple tuple = SPI_tuptable->vals[0];
    Datum count_datum = SPI_getbinval(tuple, SPI_tuptable->tupdesc, 1, &isnull);
    int64 count = DatumGetInt64(count_datum);
}
```

## 1.7. Типичные ошибки с SPI

| Ошибка | Проявление | Решение |
|--------|-----------|---------|
| Забыли `SPI_connect()` | Паника с `cannot execute SPI without connection` | Всегда вызывайте перед `SPI_execute` |
| Не вызвали `SPI_finish()` | Утечка памяти (`SPIProcContext` растёт) | В конце обязательно `SPI_finish()` |
| Сохранили указатель из `SPI_tuptable` после `SPI_finish` | Указатель на освобождённую память | Скопируйте строки через `heap_copytuple()` |
| Используете `SPI_execute` внутри цикла без контекста | Память растёт | Создайте временный контекст и `MemoryContextReset` |

---

# 2. Tuptable — возврат таблицы из C-функции (`tuptable.h`)

## 2.1. Что такое SRF (Set Returning Function)

Функция, которая возвращает не скаляр, а **таблицу** (множество строк). В SQL это выглядит так:

```sql
SELECT * FROM my_table_function('param');
```

## 2.2. Три способа реализации SRF

| Способ | Сложность | Производительность | Когда использовать |
|--------|-----------|-------------------|--------------------|
| `RETURNS TABLE` через SQL | Низкая | Высокая | Простые запросы |
| `RETURNS SETOF record` + `RETURN NEXT` в PL/pgSQL | Средняя | Средняя | Маленькие наборы |
| **C-функция с tuptable (SRF)** | Высокая | Очень высокая | Миллионы строк, сложная логика |

## 2.3. Базовая структура SRF на C

```c
#include "funcapi.h"
#include "tuptable.h"

PG_FUNCTION_INFO_V1(my_srf);

Datum my_srf(PG_FUNCTION_ARGS)
{
    FuncCallContext *funcctx;
    
    // Фаза 1: инициализация (первый вызов)
    if (SRF_IS_FIRSTCALL()) {
        MemoryContext old_ctx;
        funcctx = SRF_FIRSTCALL_INIT();
        
        // Создаём контекст для хранения состояния между вызовами
        old_ctx = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);
        
        // Инициализируем наше состояние
        MyState *state = palloc(sizeof(MyState));
        state->counter = 0;
        state->max_rows = PG_GETARG_INT32(0);
        
        funcctx->user_fctx = state;
        
        // Описание возвращаемой таблицы (TupleDesc)
        TupleDesc tupdesc;
        if (get_call_result_type(fcinfo, NULL, &tupdesc) != TYPEFUNC_COMPOSITE)
            ereport(ERROR, (errmsg("function returning record called in wrong context")));
        
        funcctx->tuple_desc = tupdesc;
        
        MemoryContextSwitchTo(old_ctx);
    }
    
    // Фаза 2: генерация строк (каждый вызов возвращает одну строку)
    funcctx = SRF_PERCALL_SETUP();
    MyState *state = (MyState *) funcctx->user_fctx;
    
    if (state->counter < state->max_rows) {
        Datum values[2];
        bool nulls[2];
        
        // Формируем строку
        values[0] = Int32GetDatum(state->counter);
        nulls[0] = false;
        values[1] = CStringGetTextDatum("hello");
        nulls[1] = false;
        
        HeapTuple tuple = heap_form_tuple(funcctx->tuple_desc, values, nulls);
        HeapTupleHeader header = DatumGetHeapTupleHeader(HeapTupleGetDatum(tuple));
        
        state->counter++;
        SRF_RETURN_NEXT(funcctx, header);
    } else {
        SRF_RETURN_DONE(funcctx);
    }
}
```

## 2.4. Детальный разбор `FuncCallContext`

```c
typedef struct FuncCallContext
{
    // Обязательные поля
    TupleDesc       tuple_desc;        // Описание колонок возвращаемой таблицы
    MemoryContext   multi_call_memory_ctx; // Контекст для состояния (живёт между вызовами)
    
    // Опциональные для производительности
    AttInMetadata  *attinmeta;         // Для конвертации строк из text
    
    // Пользовательские данные (вы заполняете)
    void           *user_fctx;         // Указатель на ваше состояние
} FuncCallContext;
```

**Важно:** SRF-функция вызывается многократно для одного SQL-запроса:
1.  Первый вызов: инициализация, создание контекста, возврат первой строки (через `SRF_RETURN_NEXT`).
2.  Второй вызов: восстановление состояния из `user_fctx`, возврат второй строки.
3.  N-й вызов: когда строки закончились — `SRF_RETURN_DONE`.

## 2.5. Пример: генератор чисел Фибоначчи (с сохранением состояния)

```c
typedef struct
{
    int64 prev;
    int64 curr;
    int64 count;
    int64 max;
} FibState;

Datum fibonacci_srf(PG_FUNCTION_ARGS)
{
    FuncCallContext *funcctx;
    FibState *state;
    
    if (SRF_IS_FIRSTCALL()) {
        funcctx = SRF_FIRSTCALL_INIT();
        MemoryContext old_ctx = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);
        
        state = palloc(sizeof(FibState));
        state->prev = 0;
        state->curr = 1;
        state->count = 0;
        state->max = PG_GETARG_INT64(0);
        
        funcctx->user_fctx = state;
        
        // Описание: одно поле BIGINT
        TupleDesc tupdesc = CreateTemplateTupleDesc(1);
        TupleDescInitEntry(tupdesc, 1, "fibonacci", INT8OID, -1, 0);
        funcctx->tuple_desc = BlessTupleDesc(tupdesc);
        
        MemoryContextSwitchTo(old_ctx);
    }
    
    funcctx = SRF_PERCALL_SETUP();
    state = (FibState *) funcctx->user_fctx;
    
    if (state->count < state->max) {
        Datum values[1];
        bool nulls[1] = {false};
        
        // Для первого вызова возвращаем 0, для второго — 1, далее сумму
        int64 result;
        if (state->count == 0)
            result = 0;
        else if (state->count == 1)
            result = 1;
        else {
            int64 next = state->prev + state->curr;
            state->prev = state->curr;
            state->curr = next;
            result = next;
        }
        
        values[0] = Int64GetDatum(result);
        HeapTuple tuple = heap_form_tuple(funcctx->tuple_desc, values, nulls);
        state->count++;
        
        SRF_RETURN_NEXT(funcctx, HeapTupleGetDatum(tuple));
    } else {
        SRF_RETURN_DONE(funcctx);
    }
}
```

## 2.6. Производительность: возврат SETOF vs TABLE

```sql
-- Плохо: одна строка за раз (много вызовов)
SELECT * FROM slow_c_srf();

-- Хорошо: материализовать результат в CTE
WITH data AS (SELECT * FROM slow_c_srf())
SELECT * FROM data JOIN other ON ...;
```

---

# 3. Index Access Method (`indexam.h`) — свой тип индекса

## 3.1. Что такое Index Access Method

Это **самый низкий уровень** расширения PostgreSQL. Вы создаёте новый тип индекса, который может быть использован планировщиком так же, как стандартные B-Tree, Hash, GiST, GIN, BRIN.

Примеры кастомных индексов в реальном мире:
- **PostGIS** — R-Tree для геоданных через GiST.
- **pg_trgm** — триграммный индекс для `LIKE '%text%'`.
- **RUM** — индекс для полнотекстового поиска с позициями.

## 3.2. Что нужно реализовать

PostgreSQL ожидает от вас структуру `IndexAmRoutine` (в современном API):

```c
typedef struct IndexAmRoutine
{
    NodeTag     type;
    
    // Параметры индекса
    bool        amcanorder;      // может ли выдавать сортировку?
    bool        amcanunique;     // поддерживает ли UNIQUE?
    bool        amcanmulticol;   // многоколоночный?
    bool        amcanreturn;     // может ли возвращать значения (covering index)?
    
    // Функции, которые вы обязаны реализовать
    aminsert_function aminsert;          // вставка
    ambulkdelete_function ambulkdelete;  // удаление
    amvacuumcleanup_function amvacuumcleanup;
    amcostestimate_function amcostestimate; // оценка стоимости сканирования
    amoptions_function amoptions;
    
    amscanbegin_function amscanbegin;    // начало сканирования
    amscanend_function amscanend;        // конец сканирования
    amrescan_function amrescan;          // сброс сканирования
    
    amgettuple_function amgettuple;      // получить следующую строку (одна)
    amgetbitmap_function amgetbitmap;    // получить все строки в битмап
    
    ambuild_function ambuild;            // построить индекс с нуля
    ambuildempty_function ambuildempty;  // пустой индекс
} IndexAmRoutine;
```

## 3.3. Упрощённый пример: индекс с фиксированным хешем (только для демонстрации)

Этот индекс хранит ключ в хеш-таблице в памяти (не для продакшена!).

```c
#include "postgres.h"
#include "access/genam.h"
#include "access/htup_details.h"
#include "catalog/index.h"
#include "utils/hsearch.h"

// Структура нашего "индекса" (на самом деле хеш-таблица в памяти)
typedef struct
{
    HTAB *hash;
    MemoryContext ctx;
} MyIndexData;

// Функция вставки
bool myinsert(Relation rel, Datum *values, bool *isnull,
              ItemPointer heap_tid, Relation heapRel,
              IndexUniqueCheck checkUnique, bool indexUnchanged,
              IndexInfo *indexInfo)
{
    MyIndexData *data = (MyIndexData *) rel->rd_amcache;
    
    if (!data) {
        // Первая вставка — инициализируем хеш-таблицу
        data = MemoryContextAllocZero(rel->rd_indexcxt, sizeof(MyIndexData));
        HASHCTL ctl = { .keysize = sizeof(Datum), .entrysize = sizeof(ItemPointerData) };
        data->hash = hash_create("my_hash", 1024, &ctl, HASH_ELEM | HASH_BLOBS);
        rel->rd_amcache = data;
    }
    
    Datum key = values[0];  // предполагаем одноколоночный индекс
    
    // Вставляем в хеш (ключ → TID)
    ItemPointerData *existing = hash_search(data->hash, &key, HASH_ENTER, NULL);
    *existing = *heap_tid;
    
    return true;
}

// Функция сканирования: поиск по ключу
bool mygettuple(IndexScanDesc scan, ScanDirection dir)
{
    MyIndexData *data = (MyIndexData *) scan->indexRelation->rd_amcache;
    
    // Получить ключ сканирования (что ищем)
    ScanKey key = scan->keyData;
    Datum search_value = key->sk_argument;
    
    ItemPointerData *found = hash_search(data->hash, &search_value, HASH_FIND, NULL);
    if (!found)
        return false;  // не найдено
    
    // Установить TID результата
    scan->xs_heaptid = *found;
    scan->xs_recheck = false;  // точное совпадение, перепроверка не нужна
    
    return true;  // найдена одна строка
}

// Регистрация метода доступа (в _PG_init)
void _PG_init(void)
{
    IndexAmRoutine *amroutine = makeNode(IndexAmRoutine);
    
    amroutine->amcanorder = false;
    amroutine->amcanunique = false;
    amroutine->amcanmulticol = false;
    amroutine->amcanreturn = false;
    
    amroutine->aminsert = myinsert;
    amroutine->amgettuple = mygettuple;
    amroutine->amgetbitmap = NULL;  // не поддерживаем битмап-скан
    // ... заполнить остальные заглушки
    
    // Зарегистрировать новый метод доступа
    RegisterIndexAccessMethod(amroutine);
}
```

## 3.4. Основные ограничения и сложности

| Аспект | Сложность | Что нужно учесть |
|--------|-----------|------------------|
| **Долговечность** | Высокая | Индекс должен жить на диске, восстанавливаться после падения, участвовать в WAL |
| **Конкурентность** | Очень высокая | Ваш код обязан брать буферные блокировки, работать с MVCC |
| **Восстановление** | Высокая | Нужно уметь строить индекс из сырых данных (`ambuild`) |
| **Оценка стоимости** | Средняя | `amcostestimate` — без правильной оценки планировщик не будет использовать ваш индекс |
| **Бинарная совместимость** | Кошмар | API `indexam.h` меняется между мажорными версиями PostgreSQL |

## 3.5. Когда реально писать свой индекс (а не использовать GiST/GIN)

**Да, стоит писать свой индекс:**
- Вы создаёте абсолютно новый тип данных (например, `mydate` с календарём древних майя).
- Вам нужен индекс с принципиально другой сложностью (например, O(1) для всех операций).
- Вы портируете алгоритм из академической статьи.

**Нет, используйте GiST или GIN:**
- Ваш индекс — это дерево или хеш (уже есть B-Tree и Hash).
- Вы индексируете геоданные, массивы, JSON, текст (есть GiST/GIN).
- Вы хотите поддержку `LIKE`, полнотекстовый поиск (есть pg_trgm, gin_tsvector).

---

## Итоговая таблица сравнения трёх интерфейсов

| Характеристика | `spi.h` | `tuptable.h` (SRF) | `indexam.h` |
|----------------|---------|--------------------|-------------|
| **Что делает** | Выполняет SQL из C | Возвращает таблицу из C | Создаёт новый тип индекса |
| **Уровень абстракции** | Высокий (как клиент) | Средний | Очень низкий (ядро) |
| **Сложность реализации** | Низкая | Средняя | Экстремально высокая |
| **Работа с памятью** | Автоматическая (SPI контексты) | Ручное управление | Полный контроль + буферы |
| **Требует C** | Нет (можно PL/pgSQL обойтись) | Да для сложных случаев | Да, только C |
| **Переносимость версий** | Высокая (API стабилен) | Высокая | Очень низкая (ломается в каждой мажорной версии) |
| **Типичные применения** | Логирование, аудит, сложные валидации | Генераторы данных, ETL, экспорт | PostGIS, поисковые движки, уникальные типы данных |

---

