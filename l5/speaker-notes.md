# Текст лектора — l5

## Архитектура: изоляция и многоверсионность (MVCC)

> Меньше воды, больше xmin/xmax и фактов.
> MVCC — не «магия Postgres», а способ не убить прод блокировками на каждом SELECT.

---

## Слайд 1 — Титул

Тема: **изоляция и многоверсионность**.

С этого места Postgres перестаёт быть «чёрным ящиком, который ест SQL».
Дальше всё про bloat, vacuum и «почему мой UPDATE висит» опирается на сегодняшний материал.

---

## Слайд 2 — Темы

Шесть блоков:

1. Многоверсионность  
2. Снимок данных  
3. Уровни изоляции  
4. Очистка и горизонт (задел на l6)  
5. Блокировки  
6. Статус транзакций (clog)

Две демо-сессии — там xmin/xmax и блокировки. Потом практика на изоляции и DDL.

---

## Слайд 3 — Многоверсионность

**Идея:** одна логическая строка → несколько **версий** в heap-файле.

| метка | смысл |
|-------|--------|
| `xmin` | xid, **создавший** версию |
| `xmax` | xid, **удаливший** версию (`0` = версия жива) |

«Время» версии = номер транзакции (xid), растёт монотонно.

**UPDATE** в Postgres = **DELETE старой версии + INSERT новой**. Физически строка не перезаписывается на месте.

Три сценария параллелизма:

| кто с кем | что происходит |
|-----------|----------------|
| read + read | без конфликта |
| write + write | очередь — второй ждёт row lock |
| read + write | **MVCC**: читатель видит свою версию, писатель — свою |

Альтернативы, которые Postgres **не** выбрал:

- блокировать всё подряд → производительность в минус;
- dirty read → видеть незакоммиченное → откат ломает картину мира.

Доки: https://postgrespro.ru/docs/postgresql/16/mvcc-intro

---

## Слайд 4 — Снимок данных

**Снимок (snapshot)** = согласованный срез БД на момент времени.

Не копия таблиц — **два числа**:

| компонент | зачем |
|-----------|--------|
| xid последней **зафиксированной** tx на момент снимка | «до какого xid мир считается committed» |
| список **активных** tx на этот момент | не показывать их незакоммиченные изменения |

Правило видимости версии (упрощённо):

- `xmin` committed **до** снимка и tx не в списке активных;
- `xmax` пуст или tx удаления ещё не видна снимку.

На диаграмме: строка 2 удалена, но tx со снимком «до удаления» ещё её видит — **это фича**, не баг.

---

## Слайд 5 — Уровни изоляции

Стандарт SQL — 4 уровня. Postgres реализует 3 «по-настоящему».

| уровень | в Postgres | когда строится снимок |
|---------|------------|------------------------|
| Read Uncommitted | **нет** (= Read Committed) | — |
| **Read Committed** | **default** | на **каждый** SQL-оператор |
| Repeatable Read | да | на **первый** оператор tx |
| Serializable | да | SSI, полная изоляция |

Факты:

- RC: два одинаковых `SELECT` подряд могут вернуть **разное** (non-repeatable read).
- RR: картина стабильна внутри tx, но возможна **ошибка сериализации** (`40001`).
- Serializable: пишете как будто одни; приложение **обязано** уметь retry.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;
SHOW transaction_isolation;
```

Доки: https://postgrespro.ru/docs/postgresql/16/transaction-iso

---

## Слайд 6 — Демонстрация

Четыре пункта — показать в двух psql-сеансах.

### 1. Несколько версий одной строки

```sql
CREATE TABLE t (s text);
INSERT INTO t VALUES ('Первая версия');  -- autocommit, xmin = prev xid

BEGIN;
SELECT pg_current_xact_id();   -- PG14+: xid; в старых — txid_current()
SELECT *, xmin, xmax FROM t;
-- xmin = insert-xid, xmax = 0
```

### 2. Системные столбцы xmin / xmax

| поле | значение |
|------|----------|
| `xmin` | кто создал версию |
| `xmax = 0` | версия актуальна |
| `xmax > 0` | кто-то пометил на удаление (может быть незакоммиченный UPDATE) |

**Не тащите xmin/xmax в прод-код.** Это учебный рентген, не API.

### 3. Видимость в снимке

Сессия B:

```sql
BEGIN;
UPDATE t SET s = 'Вторая версия';
-- B видит xmin=свой; A всё ещё «Первая версия», xmax=xid(B)
COMMIT;
-- A после COMMIT B видит «Вторая версия»; первая версия — мёртвая
```

### 4. Уровни изоляции на практике

Повторить delete-сценарий из практики (слайд 12) — RC vs RR.

---

## Слайд 7 — Очистка и ее горизонт

MVCC копит **мёртвые версии**. Их нельзя удалить, пока жив хоть один снимок их «помнит».

**Горизонт очистки (vacuum horizon)** — минимальный xid, для которого все версии с меньшим `xmax` уже никому не нужны.

| факт | следствие |
|------|-----------|
| горизонт **один на БД** | idle in transaction в одной схеме бьёт по всей базе |
| долгий SELECT / idle tx | horizon не двигается → bloat |
| за horizon | VACUUM может вычистить dead tuples |

Спойлер l6: чистит autovacuum / `VACUUM`. Сейчас — **почему** это вообще нужно.

Доки: https://postgrespro.ru/docs/postgresql/16/routine-vacuuming

---

## Слайд 8 — Блокировки

MVCC минимизирует блокировки, но не отменяет.

### Строки

| операция | блокирует |
|----------|-----------|
| SELECT | **никого** |
| UPDATE/DELETE | другие **write** на ту же строку; **read** — нет |

### Таблицы

- DDL / `VACUUM FULL` / некоторые `ALTER` → **Access Exclusive** → таблица недоступна.
- `DROP TABLE` при открытой tx с SELECT → второй сеанс **ждёт**.

### Время жизни

- lock живёт до **COMMIT/ROLLBACK**;
- после `SAVEPOINT` + `ROLLBACK TO` — locks после savepoint снимаются.

Доки: https://postgrespro.ru/docs/postgresql/16/explicit-locking

---

## Слайд 9 — Демонстрация

### 1–2. Конфликт двух UPDATE

Сессия A:

```sql
BEGIN;
UPDATE t SET s = 'Третья версия' RETURNING *;
-- не COMMIT
```

Сессия B:

```sql
BEGIN;
UPDATE t SET s = 'Четвертая версия';  -- висит, ждёт A
```

A `COMMIT` → B отрабатывает. Классический **row-level exclusive lock**.

### 3. Чтение не блокирует

Пока B ждёт — третий сеанс:

```sql
SELECT * FROM t;  -- ответ сразу, MVCC
```

### 4. Ожидание и разблокировка

Смотреть блокировки:

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity WHERE wait_event IS NOT NULL;

SELECT l.locktype, l.mode, l.granted, a.query
FROM pg_locks l JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted;
```

`lock_timeout` / `statement_timeout` — ваши друзья в демо, чтобы не зависнуть навечно.

---

## Слайд 10 — Статус транзакций

**clog** (Commit Log, `pg_xact/`) — 2 бита на tx:

| статус | биты |
|--------|------|
| in progress | — |
| committed | «зафиксирована» |
| aborted | «прервана» |

- буферы в **shared memory** — не ходим на диск на каждый commit;
- **ROLLBACK так же быстр, как COMMIT** — физически данные не откатываются, меняется только статус в clog;
- aborted-версии строк **остаются на диске** до VACUUM, но **невидимы** для новых снимков.

Факт для собеседований: «Postgres откатывает транзакцию мгновенно» — правда, но bloat от aborted tx всё равно копится.

---

## Слайд 11 — Итоги

Коротко:

- в heap — **несколько версий** строки;
- tx работает со **снимком**, не с «текущей таблицей»;
- уровень изоляции = **когда** строится снимок;
- dead tuples за horizon → **VACUUM**;
- **readers don't block writers, writers don't block readers** (на уровне строк).

---

## Слайд 12 — Практика

### Задание 1 — Read Committed

```sql
-- сессия 1
CREATE TABLE t (n integer);
INSERT INTO t VALUES (42);
BEGIN;                    -- default RC
SELECT * FROM t;          -- 1 row

-- сессия 2
DELETE FROM t; COMMIT;

-- сессия 1, тот же SELECT
SELECT * FROM t;          -- 0 rows — видит committed delete
COMMIT;
```

**Ответ:** 0 строк. Новый снимок на каждый оператор.

### Задание 2 — Repeatable Read

```sql
INSERT INTO t VALUES (42);

-- сессия 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM t;          -- 1 row

-- сессия 2
DELETE FROM t; COMMIT;

-- сессия 1
SELECT * FROM t;          -- 1 row — снимок tx не меняется
COMMIT;
```

**Отличие:** RR держит snapshot tx; RC — per-statement.

---

## Слайд 13 — Практика+

### 1. DDL + COMMIT

```sql
-- сессия 1
BEGIN;
CREATE TABLE t1 (n integer);
INSERT INTO t1 VALUES (42);
-- не COMMIT

-- сессия 2
SELECT * FROM t1;
-- ERROR: relation "t1" does not exist
```

После `COMMIT` в сессии 1 — таблица видна.

**Факт:** DDL transactional в Postgres — `CREATE` откатывается с `ROLLBACK`.

### 2. DDL + ROLLBACK

```sql
BEGIN;
CREATE TABLE t2 (n integer);
INSERT INTO t2 VALUES (42);
ROLLBACK;
-- t2 не существует ни для кого
```

### 3. DROP vs открытая tx

```sql
-- сессия 1
BEGIN;
SELECT * FROM t1;

-- сессия 2
DROP TABLE t1;   -- блокируется, пока tx1 жива
```

После `COMMIT`/`ROLLBACK` сессии 1 — `DROP` проходит.

---

## Финал

На выход:

1. UPDATE = delete+insert версии; `xmin`/`xmax`.  
2. Snapshot = committed xid + active list.  
3. RC (default) vs RR vs Serializable — **когда** snapshot.  
4. Vacuum horizon — долгие tx = bloat.  
5. Чтение не блокирует; два UPDATE — да.  
6. ROLLBACK = flip bit в clog, не «откат байтов».

Дальше l6 — кто физически выносит мусор и почему `VACUUM FULL` — это уже хирургия.
