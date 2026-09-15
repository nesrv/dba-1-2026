# Текст лектора — l10

## Организация данных: табличные пространства

> Логика — схемы и базы. Физика — табличные пространства и каталоги на диске.
> Разводить их умеют не все, а зря — это про I/O и аварийное «куда положить тяжёлое».

---

## Слайд 1 — Титул

Тема: **табличные пространства (ТП)** — как Postgres раскладывает файлы по диску.

Факт: одна БД может жить в нескольких ТП, одно ТП — обслуживать несколько БД. Это не «один диск = одна база».

---

## Слайд 2 — Темы

Три блока + практика:

1. ТП и каталоги на диске  
2. Создание / изменение / удаление  
3. Перемещение данных между ТП  

Демо плотное — там весь `CREATE TABLESPACE`, `ALTER … SET TABLESPACE` и подводные камни удаления.

---

## Слайд 3 — Табличные пространства

ТП = **где на файловой системе** лежат данные.

| ТП | Назначение |
|----|------------|
| `pg_default` | по умолчанию для объектов БД |
| `pg_global` | общие для **кластера** объекты каталога |

Типичный кейс: архив на медленных дисках, OLTP — на быстрых SSD.

У каждой БД есть **ТП по умолчанию** — туда падают новые объекты и системный каталог этой БД, если не указано иное. Можно сменить: `ALTER DATABASE … SET TABLESPACE`.

`pg_global` — особый: shared-каталог, не «ещё одна база».

Доки: https://postgrespro.ru/docs/postgresql/16/manage-ag-tablespaces

---

## Слайд 4 — Каталоги

ТП на диске — это **путь к каталогу**:

| ТП | Путь относительно PGDATA |
|----|--------------------------|
| `pg_global` | `global/` |
| `pg_default` | `base/<dboid>/` |
| пользовательское | `pg_tblspc/<tsoid>` → symlink на ваш каталог |

Внутри пользовательского ТП ещё уровень **версии сервера** (`PG_16_…/dboid/`) — чтобы major-upgrade не смешивал файлы разных версий.

Каждый объект — **отдельные файлы** (плюс слои `_fsm`, `_vm` — на l11).

Факт: `pg_tblspc/` — только symlinks. Сам каталог ТП вы создаёте сами, Postgres туда не «заходит» без `CREATE TABLESPACE`.

---

## Слайд 5 — Демонстрация

### Системные и пользовательские ТП

```sql
SELECT * FROM pg_tablespace;
-- pg_default (1663), pg_global (1664)
```

Создание — каталог **пустой**, владелец ОС = `postgres`:

```bash
sudo -u postgres mkdir /var/lib/postgresql/ts_dir
```

```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';
\db
\db+
```

БД с ТП по умолчанию:

```sql
CREATE DATABASE appdb TABLESPACE ts;
\c appdb
CREATE TABLE t1(id int GENERATED ALWAYS AS IDENTITY, name text);
CREATE TABLE t2(n numeric) TABLESPACE pg_default;
```

Проверка:

```sql
SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';
-- t1: пустое поле = ТП по умолчанию (ts)
-- t2: pg_default явно
```

Индекс в другом ТП:

```sql
CREATE INDEX ON t1(id) TABLESPACE pg_default;
```

Сессионно без `TABLESPACE` в DDL:

```sql
SET default_tablespace = 'ts';
SET temp_tablespaces = 'ts';   -- несколько значений → random pick
CREATE TEMP TABLE temp(s text);
```

Одно ТП — объекты **разных** БД:

```sql
CREATE DATABASE configdb;   -- default pg_default
\c configdb
CREATE TABLE t(n int) TABLESPACE ts;
```

---

## Слайд 6 — Демонстрация (продолжение)

### Перемещение — это **физическое копирование**

```sql
ALTER TABLE t1 SET TABLESPACE pg_default;
REINDEX (TABLESPACE ts) TABLE t1;
ALTER TABLE ALL IN TABLESPACE pg_default SET TABLESPACE ts;
```

На время операции объект **заблокирован полностью**. Не путать с `SET SCHEMA`.

Размер:

```sql
SELECT pg_size_pretty(pg_tablespace_size('ts'));
\db+
```

Почему «пустое» ТП уже мегабайты? Если `ts` — default для `appdb`, там **системный каталог** БД.

### Удаление — только пустое, без CASCADE

```sql
DROP TABLESPACE ts;  -- ERROR: not empty
```

Разведка по кластеру:

```sql
SELECT oid FROM pg_tablespace WHERE spcname = 'ts';
SELECT datname FROM pg_database
WHERE oid IN (SELECT pg_tablespace_databases(16386));
```

В каждой БД — объекты с `reltablespace = oid` или `0` (= default ТП БД).

Смена default БД **переносит всё физически**:

```sql
\c postgres
ALTER DATABASE appdb SET TABLESPACE pg_default;
DROP TABLESPACE ts;
```

```bash
sudo -u postgres rm -rf /var/lib/postgresql/ts_dir
```

Факт: `DROP TABLESPACE` не знает про объекты в других БД — отсюда запрет CASCADE.

---

## Слайд 7 — Итоги

Коротко:

- ТП — инструмент **физической** организации хранения  
- Логика (БД, схемы) и физика (ТП) **независимы**  
- Перенос между ТП = копирование файлов + блокировка  

На память: `\db+`, `pg_tablespace_size`, `pg_tablespace_databases`.

---

## Слайд 8 — Практика / Практика+

### Практика — ответ на вопрос со слайда

Почему без `TABLESPACE` default = `pg_default`?  
→ Новые БД клонируются из `template1`, у которого default = `pg_default`.

Шаги:

```bash
sudo -u postgres mkdir /var/lib/postgresql/ts_dir
```

```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';
ALTER DATABASE template1 SET TABLESPACE ts;
CREATE DATABASE db;
SELECT spcname FROM pg_tablespace
WHERE oid = (SELECT dattablespace FROM pg_database WHERE datname = 'db');
-- ts
```

Symlink:

```bash
sudo -u postgres ls -l $PGDATA/pg_tblspc/<tsoid>
```

Уборка:

```sql
ALTER DATABASE template1 SET TABLESPACE pg_default;
DROP DATABASE db;
DROP TABLESPACE ts;
```

### Практика+ — `random_page_cost` на SSD

Дефолты под HDD:

```sql
\dconfig *page_cost
-- random_page_cost = 4, seq_page_cost = 1
```

Для SSD в конкретном ТП:

```sql
ALTER TABLESPACE pg_default SET (random_page_cost = 1.1);
\db+   -- Options: {random_page_cost=1.1}
```

Планировщик чаще выберет index scan. Глобально — в `postgresql.conf`. Подробнее — курс QPT.

---

## Финал

1. `pg_default` / `pg_global` / свой каталог + `pg_tblspc`  
2. Пустое `tablespace` в `pg_tables` = default БД  
3. `ALTER … SET TABLESPACE` — тяжёлая физическая операция  
4. `template1` задаёт default для новых БД  
5. На ТП можно вешать `random_page_cost` под тип диска
