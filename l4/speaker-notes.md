# Текст лектора — l4

## Архитектура: общее устройство PostgreSQL

> Меньше воды, больше схем и фактов.
> Обзор «как устроен зверь» — без залезания в vacuum/WAL/buffer pool (это дальше по курсу).

---

## Слайд 1 — Титул

Тема: **общее устройство PostgreSQL** — клиент-сервер, транзакции, пайплайн запроса, процессы, диск, расширяемость.

Факт: Postgres — не «один процесс с потоками», а **postmaster + N backend'ов + фоновые**. Это объясняет `max_connections`, память и пулы.

---

## Слайд 2 — Темы

Шесть блоков:

1. Клиент-серверный **протокол**  
2. **Транзакции** и ACID  
3. **Обработка запросов** — parse → plan → execute; PREPARE, курсоры  
4. **Процессы и память** — local vs shared  
5. **Хранение на диске** — страницы, buffer cache, WAL (обзор)  
6. **Расширяемость** — типы, индексы, FDW, background workers

Много демо в psql — SQL здесь учебный стенд, в проде то же через драйвер.

---

## Слайд 3 — Клиент и сервер

```
клиент (Python/psycopg2, Java/JDBC, psql/libpq)
        ↕ протокол
PostgreSQL (сервер)
```

| клиент | сервер |
|--------|--------|
| подключение | аутентификация (`pg_hba.conf`) |
| формирование SQL | выполнение запросов |
| управление транзакциями (BEGIN/COMMIT) | поддержка ACID на стороне данных |

Факты:

- Язык клиента не важен — возможности задаёт **протокол** + **драйвер**.
- Драйвер может сидеть на **libpq** или реализовывать протокол сам (JDBC часто свой стек).
- Подключение — к **одной базе** кластера, не «ко всем сразу».

Доки протокола: https://postgrespro.ru/docs/postgresql/16/protocol

---

## Слайд 4 — Транзакции

Последовательность операций с гарантиями **ACID**:

| свойство | смысл в Postgres |
|----------|------------------|
| **A**tomicity | всё или ничего — `COMMIT` / `ROLLBACK` |
| **C**onsistency | constraints + триггеры держат инварианты |
| **I**solation | параллельные tx не мешают (MVCC — отдельная тема) |
| **D**urability | после COMMIT переживает сбой (WAL) |

Поток:

```text
BEGIN → операции → COMMIT / ROLLBACK
```

Факт: **клиент** обычно решает границы транзакции. Сервер может — в хранимых процедурах (`COMMIT` в процедуре — отдельная история).

Доки: https://postgrespro.ru/docs/postgresql/16/transactions

---

## Слайд 5 — Демонстрация

### AUTOCOMMIT в psql

```text
\echo :AUTOCOMMIT    -- on по умолчанию
```

Каждая команда без `BEGIN` → **сразу COMMIT**. В JDBC аналог: `connection.setAutoCommit(true)`.

### Видимость изменений

```sql
CREATE TABLE t (id int, s text);
INSERT INTO t VALUES (1, 'foo');   -- autocommit → видно всем

BEGIN;
INSERT INTO t VALUES (2, 'bar'); -- другая сессия видит только (1,foo)
COMMIT;                            -- теперь (1,foo),(2,bar)
```

### AUTOCOMMIT off

```text
\set AUTOCOMMIT off
INSERT INTO t VALUES (3, 'baz');   -- tx началась неявно
COMMIT;
```

### SAVEPOINT

```sql
BEGIN;
SAVEPOINT sp;
INSERT INTO t VALUES (4, 'qux');
ROLLBACK TO sp;    -- не GOTO! только откат данных с sp
INSERT INTO t VALUES (4, 'xyz');
COMMIT;
```

Факт: **свои** uncommitted изменения tx **видит**; чужие — нет (уровень изоляции уточним позже).

---

## Слайд 6 — Выполнение запроса

Простой режим протокола — **весь результат сразу**:

| этап | откуда берёт инфу |
|------|-------------------|
| **разбор** (parse) | системный **каталог** (`pg_catalog`) |
| **переписывание** (rewrite) | **rules**, views |
| **планирование** (plan) | **статистика** (`pg_statistic`) |
| **выполнение** (execute) | **данные** на диске/в cache |

```text
клиент --запрос--> сервер --результат--> клиент (все строки)
```

SQL **декларативный** — «что хотим», не «как читать». Планировщик решает seq scan vs index.

Доки: https://postgrespro.ru/docs/postgresql/16/query-path

---

## Слайд 7 — Подготовка операторов

**Extended query protocol** — разбить пайплайн:

```text
PREPARE:  разбор + переписывание → дерево запроса (кеш)
EXECUTE:  привязка параметров → планирование → выполнение
```

| плюс | деталь |
|------|--------|
| не парсить каждый раз | разбор один раз |
| защита от SQL-injection | параметры ≠ конкатенация строк |
| generic vs custom plan | с параметрами план может пересчитываться |

Факт: без параметров при PREPARE может запомниться **и план**. С `$1` — планировщик смотрит на значения; иногда перестаёт планировать заново (см. `pg_prepared_statements`: `generic_plans` / `custom_plans`).

Доки: https://postgrespro.ru/docs/postgresql/16/sql-prepare

---

## Слайд 8 — Демонстрация

```sql
PREPARE q(integer) AS
  SELECT * FROM t WHERE id = $1;

EXECUTE q(1);

SELECT * FROM pg_prepared_statements \gx
-- name, statement, parameter_types, prepare_time, generic_plans, custom_plans
```

Показать **дерево** — на уровне курса достаточно `EXPLAIN (VERBOSE)` или упомянуть, что parse tree внутренний.

Вопрос аудитории: «как это в Python/Java?» → `cursor.execute("... WHERE id=%s", (1,))` — драйвер шлёт bind/execute по протоколу.

Факт: **Simple query** (`psql -c "SELECT..."`) — parse+plan+execute за один заход. **Extended** — то, что делают ORM и prepared statements.

---

## Слайд 9 — Курсоры

Не всегда нужны **все строки сразу** — extended protocol + **курсоры** (на сервере ≈ **portal**).

```text
подготовка → привязка → выполнение → получение результата (порциями)
                                              ↑___________|
```

«Окно» над результатом — сдвигается при FETCH.

- Запрос в курсоре **неявно подготавливается**
- Удобно для больших выборок **при правильном fetch size** (не по 1 строке на миллион — убьёте производительность)

Доки: https://postgrespro.ru/docs/postgresql/16/sql-declare

---

## Слайд 10 — Демонстрация

### Все строки сразу

```sql
SELECT * FROM t ORDER BY id;   -- 4 rows одним пакетом
```

### Server-side cursor

```sql
BEGIN;
DECLARE c CURSOR FOR SELECT * FROM t ORDER BY id;

FETCH c;       -- 1 row
FETCH 2 c;     -- 2 rows
FETCH 2 c;     -- остаток
FETCH 2 c;     -- (0 rows) — не ошибка, просто конец

CLOSE c;       -- можно не CLOSE — закроется на COMMIT
COMMIT;
```

Исключение: `DECLARE ... WITH HOLD` — живёт после COMMIT.

Факт: в приложениях курсоры часто **скрыты** в API (`fetchmany(n)`), но механизм тот же.

---

## Слайд 11 — Процессы и память

```
postmaster
  ├── backend (на клиента) + локальная память
  └── фоновые процессы (writer, checkpointer, autovacuum, …)
         ↕
    общая память (shared buffers, locks, …)
         ↕
       диск
```

**Локальная память backend:**

- разобранные запросы и планы  
- состояние курсоров / portals  
- cache системного каталога  
- work_mem для sort/hash join  

**Postmaster:** слушает порт, **fork** backend на новое соединение, следит за детьми — упал backend → перезапуск; если общие данные могли пострадать → **restart всего кластера**.

Факт: между запросами состояние (prepared statements, temp tables, SET) живёт в **backend-процессе** — отсюда боль пулов.

---

## Слайд 12 — Много клиентов

**1 клиент = 1 backend-процесс** (пока без пула).

| проблема | решение Postgres |
|----------|------------------|
| конкуренция за объекты в RAM | короткие **блокировки** в shared memory |
| конкуренция за строки таблиц | **MVCC** + snapshot isolation |
| writers vs readers | версии строк; блокируем только конфликтующие UPDATE |

Callout на слайде: **MVCC** — основа A+C+I; **блокировки** — для shared structures.

Факт: MVCC ≠ «блокировок нет» — они есть, но другого класса и короче по времени, чем «lock whole table on SELECT».

Подробности — тема «Изоляция и многоверсионность».

---

## Слайд 13 — Пул соединений

Когда **много клиентов** или **частые connect/disconnect**:

```
N клиентов → [ pgBouncer / Odyssey / app server pool ] → M backend'ов (M << N)
```

| плюс | минус |
|------|-------|
| стабильное число backend'ов | **один backend — много клиентов по очереди** |
| быстрый «коннект» к пулу | local state (prepared stmts, SET, temp) **протекает** между клиентами |
| pause/resume для maintenance | transaction pooling vs session pooling — разные режимы |

Факт: pgBouncer умеет **pause** клиентов без disconnect — удобно для rolling restart Postgres.

Курс DEV2 — детали режимов pool'а.

---

## Слайд 14 — Хранение данных

**Единица на диске — страница (block), обычно 8 KB.**

```
┌─────────────┐
│  заголовок  │
│  данные     │  ← может быть free space
└─────────────┘
     ↕
 буферный кеш (shared_buffers)  ↔  файл на диске (страницы)
```

Факты:

- Размер страницы (8/16/32 KB) — **на этапе сборки**, не runtime knob.
- Изменения сначала в **buffer cache**, на диск — позже (background writer).
- HDD/SSD медленные → кеш окупается при повторном чтении.

Детали layout файлов по таблицам — отдельная тема «Организация данных».

---

## Слайд 15 — Хранение данных

Полная картина с **ОС** и **WAL**:

```
backend → WAL (durability, sync)     ← долговечность (D в ACID)
backend ↔ buffer cache (shared) ↔ OS page cache ↔ файлы данных
```

| слой | роль |
|------|------|
| buffer cache (Postgres) | все backend'ы видят одни и те же страницы |
| OS cache | второй шанс не ходить на диск |
| WAL | после COMMIT пережить crash; replay при recovery |

Факт: данные и WAL — **обычные файлы**, доступ через **syscall'ы ОС**, не raw device (в типичной установке).

Темы «Buffer pool» и «WAL» — отдельные лекции с цифрами и tuning.

---

## Слайд 16 — Расширяемость

Postgres — **platform**, не монолит:

| что расширяем | примеры |
|---------------|---------|
| типы данных | `jsonb`, PostGIS geometry |
| операторы, функции, триггеры | свой домен + индекс |
| access methods (индексы) | GiST, GIN, BRIN, bloom |
| языки | PL/pgSQL, PL/Python, … |
| background workers | pg_cron, logical replication workers |
| FDW | postgres_fdw, file_fdw, сторонние |
| расширения (`CREATE EXTENSION`) | часто **без restart** |

Большинство extension'ов — shared library + SQL glue.

Курсы DBA2/DEV2 — писать свои и ставить чужие без страха.

---

## Слайд 17 — Итоги

Пять тезисов со слайда — развернуть одной фразой каждый:

1. **Сервер** управляет **кластером** (много БД, один PGDATA).  
2. **Протокол** — connect, query, transactions (simple + extended).  
3. **1 клиент ≈ 1 backend-процесс** — следствие для connections и RAM.  
4. **Данные** — файлы ОС, страницы 8K, через buffer cache.  
5. **Кеш** — local (parse, catalog) + shared (buffers) + OS page cache.

Дальше по курсу: углубляем MVCC, vacuum, WAL, buffer pool — сегодня был **каркас**.

---

## Финал

На выход:

1. **Клиент** держит транзакции; **сервер** держит ACID и данные  
2. **Запрос:** parse → rewrite → plan → execute; **PREPARE** кеширует ранние шаги  
3. **Курсоры** — результат порциями, не всё в RAM клиента  
4. **postmaster + backends + bgworkers + shared memory**  
5. **MVCC + WAL + buffer cache** — три кита, которые всплывут снова

Если запомнили одну фразу: Postgres — **мультипроцессный**, **MVCC-ный**, **расширяемый** — и всё остальное из курса вешается на эти крючки.
