# Текст лектора — l9

## Организация данных: системный каталог

> Меньше воды, больше pg_class, `\d+` и «psql meta-команды — это просто SELECT с sugar».
> Каталог — это Postgres про Postgres. GUI рисует дерево отсюда.

---

## Слайд 1 — Титул

Тема: **системный каталог** — метаданные кластера, хранящиеся **внутри** кластера.

Всё, что показывает pgAdmin/DBeaver в navigator — ultimately `pg_catalog` + views.

---

## Слайд 2 — Темы

Четыре блока:

1. Что такое каталог и как обращаться  
2. Объекты каталога и расположение  
3. Правила именования  
4. Специальные типы (`oid`, `reg*`)

Демо — `\dt`, `\d+`, `ECHO_HIDDEN`. Практика — `pg_class`, `pg_views`, `\dnS`.

---

## Слайд 3 — Системный каталог

**Каталог** = таблицы + views с описанием **всех** объектов СУБД.

| доступ | как |
|--------|-----|
| SQL | `SELECT` / DDL как обычно |
| psql | `\d*`, `\l`, `\dn` … (describe) |

Схемы:

| схема | роль |
|-------|------|
| **pg_catalog** | native Postgres metadata |
| **information_schema** | SQL-standard; переносимее, но беднее |

С PG14+ — PK и UNIQUE на большинстве catalog tables (раньше было «на честном слове»).

Доки: https://postgrespro.ru/docs/postgresql/16/catalogs  
Шпаргалка курса: `catalogs.pdf`

---

## Слайд 4 — Общие объекты кластера

Два уровня:

| где | примеры |
|-----|---------|
| **per-database** каталог | `pg_class`, `pg_namespace` — свой набор в каждой БД |
| **cluster-wide** | `pg_database`, roles, tablespaces — видны из любой БД |

Cluster-wide таблицы физически **вне** конкретной БД, но читаются через `pg_catalog` любой сессии.

Диаграмма: `pg_database` — общий список БД; внутри `appdb`/`postgres` — свои объекты в схемах.

---

## Слайд 5 — Правила именования

| правило | пример |
|---------|--------|
| префикс **`pg_`** | `pg_database`, `pg_class` |
| префикс столбца ≈ имя таблицы | `pg_database.datname`, `pg_class.relname` |
| регистр | **всегда lower** в каталоге |

**Не создавать** свои объекты с `pg_` — словите коллизии и странные баги.

Исключения: `oid`, общие имена вроде `relname`.

---

## Слайд 6 — Демонстрация

Демо: **Некоторые объекты системного каталога**.

```sql
CREATE DATABASE data_catalog;
\c data_catalog

CREATE TABLE employees(
  id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text, manager integer
);
CREATE VIEW top_managers AS
  SELECT * FROM employees WHERE manager IS NULL;
CREATE TEMP TABLE emp_salaries(employee integer, salary numeric);
```

Ключевые таблицы:

```sql
SELECT * FROM pg_database WHERE datname = 'data_catalog' \gx
SELECT * FROM pg_namespace WHERE nspname = 'public' \gx

SELECT relname, relkind, relnamespace, relfilenode, relowner, relpersistence
FROM pg_class WHERE relname ~ '^(emp|top)';
```

| relkind | объект |
|---------|--------|
| `r` | table |
| `v` | view |
| `i` | index |
| `S` | sequence |
| `rel persistence t` | temporary |

Удобные views:

```sql
SELECT schemaname, tablename FROM pg_tables WHERE schemaname ~ '(public|pg_temp.+)';
SELECT * FROM pg_views WHERE schemaname = 'public';
```

psql shortcuts:

```text
\dt          \dt+         \dtvis
\dv public.* \d top_managers \d+ top_managers
\dfS pg*size \sf pg_catalog.pg_database_size(oid)
```

**`ECHO_HIDDEN`** — показать SQL под капотом:

```sql
\set ECHO_HIDDEN on
\dt employees
\unset ECHO_HIDDEN
```

Catalog integrity: `pg_get_catalog_foreign_keys()` — pseudo-FK между catalog tables.

Факт: много temp objects → bloat в catalog tables → autovacuum на `pg_class`/`pg_attribute` важен.

---

## Слайд 7 — Специальные типы данных

**`oid`** — 32-bit object id, autoincrement, PK большинства catalog tables (~4 млрд, не бесконечность).

**`reg*`** — typed oid aliases:

| тип | для |
|-----|-----|
| `regclass` | relation (pg_class) |
| `regtype` | type |
| `regnamespace` | schema |
| `regproc` / `regprocedure` | function |

Cast: `'employees'::regclass` → oid без subquery в `pg_class`.

Доки: https://postgrespro.ru/docs/postgresql/16/datatype-oid

---

## Слайд 8 — Демонстрация

Демо: **Тип oid и reg-типы**.

```sql
-- без regclass
SELECT a.attname, a.atttypid
FROM pg_attribute a
WHERE a.attrelid = (SELECT oid FROM pg_class WHERE relname = 'employees')
  AND a.attnum > 0;

-- с regclass
SELECT a.attname, a.atttypid
FROM pg_attribute a
WHERE a.attrelid = 'employees'::regclass
  AND a.attnum > 0;

SELECT a.attname, a.atttypid::regtype
FROM pg_attribute a
WHERE a.attrelid = 'employees'::regclass
  AND a.attnum > 0;
-- integer, text, integer

\dT reg*
```

`reg*` ломается, если имя ambiguous (две `employees` в разных схемах) — тогда только qualified `'app.t'::regclass`.

---

## Слайд 9 — Итоги

1. Каталог = meta-in-DB.  
2. SQL + psql `\d*`.  
3. Часть tables per-DB, часть cluster-wide.  
4. `oid` / `reg*` — язык каталога.

---

## Слайд 10 — Практика

Задания:

1. `\d pg_class`  
2. `\d+ pg_tables`  
3. DB + temp table → `\dnS` (все схемы)  
4. Views в `information_schema`  
5. Какие запросы делает `\d+ pg_views`?

```sql
\d pg_class
\d+ pg_tables

CREATE DATABASE data_catalog;
\c data_catalog
CREATE TEMP TABLE t(n integer);

\dnS
SELECT pg_my_temp_schema()::regnamespace;

\dv information_schema.*

\set ECHO_HIDDEN on
\d+ pg_views
\set ECHO_HIDDEN off
-- ~5 queries: find relation, attrs, viewdef, rules…
```

| задача | команда |
|--------|---------|
| все схемы incl. system | `\dnS` |
| temp schema name | `pg_my_temp_schema()` |
| psql SQL peek | `\set ECHO_HIDDEN on` |

`\d+ pg_views` — не один SELECT; цепочка к `pg_class`, `pg_attribute`, `pg_get_viewdef`.

---

## Финал

На выход:

1. `pg_catalog` vs `information_schema` — когда что.  
2. `pg_database` (cluster) vs `pg_class`/`pg_namespace` (per-DB).  
3. Naming: `pg_`, column prefixes, lowercase.  
4. `\dt`/`\d+` = thin wrapper над catalog SQL.  
5. `'table'::regclass`, `::regtype` — must-have для ad-hoc queries.  
6. Temp sessions → catalog bloat; `\dnS` для полной картины схем.

Дальше по курсу — углубление в объекты, права, storage; каталог останется фоном для любой диагностики.
