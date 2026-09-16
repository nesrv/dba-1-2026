# Текст лектора — l8

## Организация данных: базы данных и схемы

> Меньше воды, больше template0/template1, search_path и «почему SELECT * FROM t внезапно не находит таблицу».
> Кластер ≠ база ≠ схема — три разных namespace, не путать.

---

## Слайд 1 — Титул

Тема: **базы данных и схемы** — логическая иерархия Postgres.

```
кластер → базы → схемы → объекты (table, index, …)
```

Подключение всегда к **одной** БД; cross-database SELECT нет (кроме dblink/FDW).

---

## Слайд 2 — Темы

Четыре блока:

1. Базы данных и шаблоны  
2. Схемы и путь поиска  
3. Специальные схемы, временные объекты  
4. Управление базами, схемами и объектами

Демо — CREATE DATABASE, схемы, `search_path`, temp tables. Практика + `temp_buffers`.

---

## Слайд 3 — Кластер баз данных

При **initdb** создаются три БД:

| БД | роль |
|----|------|
| **template1** | шаблон по умолчанию; сюда кладут «наследуемое» (extensions, схемы) |
| **template0** | read-only шаблон; restore в чистую кодировку, «чистый» CREATE DATABASE |
| **postgres** | default DB для суперпользователя; утилиты ожидают её наличие |

Любая новая БД = **клон** существующей (`CREATE DATABASE … TEMPLATE …`).

Факт: положили `pgcrypto` в template1 → каждая новая БД его унаследует. Случайно засорили template1 — засорили все будущие БД.

Доки: https://postgrespro.ru/docs/postgresql/16/manage-ag-templatedbs

---

## Слайд 4 — Демонстрация

Демо: **Базы данных**.

```sql
\l
-- или
SELECT datname, datistemplate, datallowconn, datconnlimit
FROM pg_database;
```

| колонка | смысл |
|---------|-------|
| `datistemplate` | шаблон? |
| `datallowconn` | можно подключаться? (template0 = false) |
| `datconnlimit` | лимит сессий (-1 = без лимита) |

Создание из шаблона:

```sql
\c template1
CREATE EXTENSION pgcrypto;   -- попадёт во все новые БД

\c student
CREATE DATABASE db;
\c db
SELECT digest('Hello, world!', 'md5');   -- extension есть
```

CLI: `createdb mydb`

Управление:

```sql
ALTER DATABASE db RENAME TO appdb;
ALTER DATABASE appdb CONNECTION LIMIT 10;

SELECT pg_size_pretty(pg_database_size('appdb'));
-- «пустая» БД ~7–8 MB — это уже каталог + системные объекты
```

---

## Слайд 5 — Схемы

**Схема** = namespace объектов **внутри одной БД**.

| задача | как помогает |
|--------|--------------|
| логические группы | `app`, `billing`, `staging` |
| изоляция имён | два `users` в разных схемах — OK |

**Схема ≠ роль.** Совпадение имён — удобство (`"$user"` в search_path), не связь ownership.

Доки: https://postgrespro.ru/docs/postgresql/16/ddl-schemas

---

## Слайд 6 — Базы и схемы кластера

Диаграмма:

- клиент → **одна** БД  
- в БД: `pg_catalog`, `public`, пользовательские схемы  
- в схеме: таблицы, индексы, views…

`pg_catalog` — метаданные (l9). `public` — дефолт для объектов без квалификатора (если так настроен search_path).

---

## Слайд 7 — Демонстрация

Демо: **Схемы**.

```sql
\c appdb
\dn

CREATE SCHEMA app;
CREATE TABLE t(s text);
INSERT INTO t VALUES ('Я - таблица t');

\dt                    -- public.t

ALTER TABLE t SET SCHEMA app;

SELECT * FROM app.t;   -- OK
SELECT * FROM t;       -- ERROR: relation "t" does not exist
```

`ALTER TABLE … SET SCHEMA` — только каталог; **файлы на диске не переезжают** (`relfilenode` тот же).

---

## Слайд 8 — Путь поиска

**Квалифицированное имя:** `schema.object` — однозначно.

**Неквалифицированное:** ищем в **`search_path`**.

| механизм | деталь |
|----------|--------|
| `search_path` | список схем |
| `current_schemas(true)` | реальный путь + неявные (`pg_catalog`, `pg_temp`) |
| `current_schema()` | первая **не-системная** схема → куда попадёт CREATE |
| CREATE без схемы | в `current_schema()` |

Исключаются: несуществующие схемы, схемы без `USAGE`.

Аналогия: `search_path` ≈ `$PATH` в shell.

Доки: https://postgrespro.ru/docs/postgresql/16/runtime-config-client#GUC-SEARCH-PATH

---

## Слайд 9 — Специальные схемы

| схема | поведение |
|-------|-----------|
| **public** | в search_path по умолчанию; с PG15 права на CREATE ужесточены |
| **"$user"** | схема с именем роли; если есть — объекты туда |
| **pg_catalog** | системный каталог; если не в path — **всё равно первая** при поиске |
| **information_schema** | SQL-standard view на каталог |

Создали `CREATE SCHEMA student` → `current_schema()` = `student`, не `public`.

---

## Слайд 10 — Демонстрация

Демо: **Путь поиска**, `current_schema`.

```sql
SHOW search_path;              -- "$user", public

SELECT current_schemas(true);  -- {pg_catalog, public}

SET search_path = public, app;
SELECT * FROM t;               -- нашла app.t

ALTER DATABASE appdb SET search_path = public, app;
\c appdb
SHOW search_path;

SELECT current_schema();       -- public (первая non-system)
```

Уровни настройки: session → `ALTER DATABASE` → `ALTER ROLE` → cluster (`postgresql.conf` / `ALTER SYSTEM`).

---

## Слайд 11 — Специальные схемы

**Временные таблицы:**

| свойство | значение |
|----------|----------|
| scope | session или transaction (`ON COMMIT …`) |
| WAL | **нет** (после crash — пусто) |
| shared_buffers | **нет** — local buffers backend'а |
| schema | `pg_temp_N`, alias **`pg_temp`** |

Если `pg_temp` не в path — подставляется **первой** (перекрывает одноимённые объекты!).

После disconnect — объекты умирают, схема `pg_temp_N` остаётся для reuse.

---

## Слайд 12 — Демонстрация

Демо: **Временные таблицы и pg_temp**.

```sql
CREATE TEMP TABLE t(s text);
\dt   -- pg_temp_3.t — номер свой у каждого backend

SELECT current_schemas(true);
-- {pg_temp_3, pg_catalog, public, app}

INSERT INTO t VALUES ('временная');
SELECT * FROM app.t;      -- обычная
SELECT * FROM pg_temp.t;  -- temp

CREATE VIEW v AS SELECT * FROM pg_temp.t;
-- NOTICE: temporary view

\c appdb   -- reconnect
SELECT * FROM pg_temp.t;
-- ERROR: relation "pg_temp.t" does not exist
```

Удаление:

```sql
DROP SCHEMA app CASCADE;   -- схема + объекты
DROP DATABASE appdb;       -- только без активных коннектов
```

---

## Слайд 13 — Итоги

1. Кластер → БД → схема → объект.  
2. Новая БД = clone template (обычно template1).  
3. Имя объекта: явно или через `search_path`.  
4. `public`, `pg_catalog`, `pg_temp` — особые правила.

---

## Слайд 14 — Практика

Задания:

1. CREATE DATABASE + connect  
2. `pg_database_size`  
3. Схемы `app` + `<username>`, таблицы в обеих  
4. Рост размера БД  
5. `search_path` — обе схемы видны, приоритет у user-схемы

```sql
CREATE DATABASE data_databases;
\c data_databases
SELECT pg_size_pretty(pg_database_size('data_databases')) \gset
SELECT pg_database_size('data_databases') AS oldsize \gset

CREATE SCHEMA app;
CREATE SCHEMA student;   -- = имя пользователя курса

CREATE TABLE a(s text); INSERT INTO a VALUES ('student');
CREATE TABLE app.a(s text); INSERT INTO app.a VALUES ('app');

SELECT pg_size_pretty(pg_database_size('data_databases') - :oldsize);

ALTER DATABASE data_databases SET search_path = "$user", app, public;
\c data_databases
SELECT * FROM a;   -- student (приоритет)
SELECT * FROM c;   -- app.c
```

---

## Слайд 15 — Практика+

Задание: `temp_buffers` = **4× default** для каждой новой сессии БД.

```sql
SELECT name, setting, unit FROM pg_settings WHERE name = 'temp_buffers';
-- default 1024 × 8kB = 8MB

ALTER DATABASE data_databases SET temp_buffers = '32MB';
\c data_databases
SHOW temp_buffers;

\drds   -- настройки уровня DB/role → pg_db_role_setting
```

`temp_buffers` — **local** cache temp tables; overflow → temp files на диск. Тюнить, если temp tables в hot path.

Доки: https://postgrespro.ru/docs/postgresql/16/sql-alterdatabase

---

## Финал

На выход:

1. template0 / template1 / postgres — знать назначение.  
2. `CREATE DATABASE` = clone; размер «пустой» БД ≠ 0.  
3. Схема ≠ user; `ALTER … SET SCHEMA` — логический move.  
4. `search_path`, `current_schemas()`, `current_schema()`.  
5. Temp = no WAL, no shared_buffers, `pg_temp` shadowing.  
6. `ALTER DATABASE … SET` — per-DB defaults (`temp_buffers`, `search_path`).

Следующая тема (l9) — системный каталог: где живут метаданные и как их читать.
