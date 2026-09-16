# Текст лектора — l6

## Архитектура: очистка (VACUUM)

> Меньше воды, больше autovacuum, pgstattuple и «почему файл 10 GB, а строк 100k».
> Если autovacuum выключили «для скорости» — поздравляю, вы купили bloat и wraparound.

---

## Слайд 1 — Титул

Тема: **очистка** — как Postgres живёт с последствиями MVCC.

MVCC дал изоляцию без блокировок на чтение. Цена — **мёртвые версии**, карты, статистика, freeze.
Сегодня — кто это убирает и что будет, если не убирать.

---

## Слайд 2 — Темы

Пять блоков:

1. Периодические задачи (не только «удалить dead tuple»)  
2. Автоочистка  
3. Ручной VACUUM / ANALYZE  
4. Разрастание (bloat)  
5. `VACUUM FULL`, `REINDEX`, `CONCURRENTLY`

Три демо — ручной vacuum, pgstattuple, перестроение. Практика в конце — с autovacuum on/off.

---

## Слайд 3 — Периодические задачи

**Главная задача:** вычистить исторические данные MVCC.

| где | что убираем |
|-----|-------------|
| heap (таблица) | **мёртвые** версии строк |
| index | записи, ссылающиеся на мёртвые версии |

**Dead tuple** — версия, которой уже **не нужен ни один snapshot**.

Если не чистить:

- файлы растут;
- seq scan и index scan тратят время на мёртвые ссылки;
- бэкапы и репликация страдают.

Доки: https://postgrespro.ru/docs/postgresql/16/routine-vacuuming

---

## Слайд 4 — Периодические задачи: карта видимости

**Visibility Map (VM)** — побитовая карта **страниц** таблицы.

Страница помечена, если **все** tuples на ней visible **во всех** snapshots (all-frozen / all-visible).

| применение | эффект |
|------------|--------|
| VACUUM | такие страницы можно **пропустить** — dead tuples там нет |
| Index-Only Scan | если все столбцы в индексе + VM all-visible → **heap fetch не нужен** |

**Только для таблиц.** У индексов VM нет — версионность живёт в heap.

Факт: без свежей VM index-only scans чаще лезут в таблицу → план «хуже, чем мог бы».

---

## Слайд 5 — Периодические задачи: карта свободного пространства

**Free Space Map (FSM)** — сколько пустого места **внутри** уже существующих страниц.

| факт | деталь |
|------|--------|
| после VACUUM | «дыры» в страницах → FSM растёт |
| при INSERT | ищем страницу с местом, не всегда в конец файла |
| структура | дерево (multi-level FSM) |
| индексы | FSM есть, но логика другая — в основном **полностью пустые** страницы после удаления всех items |

INSERT без FSM = больше page extend, файл растёт быстрее.

---

## Слайд 6 — Периодические задачи: обновление статистики

**ANALYZE** — случайная выборка, не full table scan.

| для кого | что даёт |
|----------|----------|
| planner | `reltuples`, histograms, ndistinct, correlation |
| вы | адекватный plan: seq vs index, join order |

Параметры:

```sql
SHOW default_statistics_target;   -- default 100
ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;
```

Устаревшая stats → plans «от балды» → запрос в 100× медленнее реальности — типичный прод-кейс.

VACUUM и ANALYZE часто идут вместе (`VACUUM ANALYZE`), но это **разные** задачи.

---

## Слайд 7 — Периодические задачи: заморозка

xid — **32 бита** → ~4 млрд tx → **wraparound**.

Модель времени **кольцевая**: для каждого xid половина номеров «в прошлом», половина «в будущем».

**Freeze** — пометить старые tuples как **frozen** → видны всем snapshots → xid можно переиспользовать.

| без freeze | с freeze |
|------------|----------|
| counter → 0 → chaos | старые строки «вне игры» xid |
| **shutdown** — txid exhaustion | autovacuum успевает |

VM-бит: «все tuples на странице frozen» — vacuum может страницу не сканировать.

Авария: `database is not accepting commands to avoid wraparound` — чинить **сейчас**, не в понедельник.

---

## Слайд 8 — Автоматическая очистка

Два процесса:

| процесс | роль |
|---------|------|
| **autovacuum launcher** | фоновый, постоянно жив; планирует workers |
| **autovacuum worker** | чистит таблицы **одной** БД, параллельно несколько workers |

Схема: postmaster → launcher → workers + backends → **shared buffers** → OS cache → disk.

Факты:

- постранично, **без** блокировки обычных SELECT/INSERT (в отличие от `VACUUM FULL`);
- частота ∝ активности таблицы (`autovacuum_vacuum_scale_factor`, thresholds);
- **не работает**, если `autovacuum=off` **или** `track_counts=off`.

```sql
SELECT pid, datname, relid::regclass, phase
FROM pg_stat_progress_vacuum;

SELECT * FROM pg_stat_activity
WHERE backend_type LIKE 'autovacuum%';
```

Выключать autovacuum «для perf» = купить bloat + anti-wraparound panic. **Не выключать.**

---

## Слайд 9 — Запуск вручную

| задача | SQL | CLI |
|--------|-----|-----|
| vacuum таблиц | `VACUUM [VERBOSE] [ANALYZE] t;` | `vacuumdb -d db -t t` |
| vacuum всей БД | `VACUUM;` | `vacuumdb -d db` |
| только analyze | `ANALYZE t;` | `vacuumdb --analyze-only` |
| оба | `VACUUM ANALYZE;` | `vacuumdb --analyze` |

Когда руками:

- после массового `UPDATE`/`DELETE`;
- перед/после bulk load;
- one-shot таблица с `autovacuum_enabled = off` (как в демо).

Cron по расписанию ≠ autovacuum: не видит **активность**, легко промахнуться по частоте.

Доки: https://postgrespro.ru/docs/postgresql/16/sql-vacuum

---

## Слайд 10 — Демонстрация

### 1. Таблица без autovacuum

```sql
CREATE TABLE bloat (
  id integer GENERATED ALWAYS AS IDENTITY,
  d timestamptz
) WITH (autovacuum_enabled = off);

INSERT INTO bloat(d)
SELECT current_timestamp FROM generate_series(1, 100_000);

CREATE INDEX ON bloat(d);
```

### 2. UPDATE → dead tuples

```sql
UPDATE bloat SET d = d + interval '1 day' WHERE id <= 10000;
```

### 3. Ручной VACUUM

```sql
VACUUM (VERBOSE) bloat;
```

Смотреть в выводе:

- `tuples: 10000 removed, ... remain`
- `dead item identifiers removed` — index cleanup
- `removable cutoff` — horizon на момент vacuum

### 4. Наблюдение

```sql
SELECT * FROM pg_stat_all_tables WHERE relname = 'bloat';
-- n_dead_tup, last_vacuum, vacuum_count

SELECT * FROM pg_stat_progress_vacuum;  -- пока worker жив
```

---

## Слайд 11 — Проблема разрастания

**Обычный VACUUM не уменьшает файл на диске.** Освобождённое место → FSM → reuse при INSERT.

| причина bloat | механизм |
|---------------|----------|
| autovacuum отключён / lazy | dead tuples копятся |
| mass UPDATE одной tx | один большой «пул» мёртвых версий |
| long tx / idle in transaction | horizon стоит |
| index bloat | page split при INSERT в B-tree **не схлопывается** назад |

Негатив:

- disk / backup size ↑  
- seq scan читает пустые страницы  
- index scan — больше random I/O  

Доки: https://postgrespro.ru/docs/postgresql/16/pgstattuple

---

## Слайд 12 — Демонстрация

### pgstattuple — оценка bloat

```sql
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('bloat') \gx
-- tuple_percent, dead_tuple_percent, free_percent

SELECT * FROM pgstattuple_approx('bloat') \gx  -- быстрее на больших таблицах

SELECT * FROM pgstatindex('bloat_d_idx') \gx
-- leaf_pages, avg_leaf_density, deleted_pages
```

### Сценарий

```sql
UPDATE bloat SET d = d + interval '1 day' WHERE id % 2 = 0;  -- 50k rows
-- dead_tuple_percent ~30%, table_len вырос
```

**Когда радикальная очистка:**

- `dead_tuple_percent` / `free_percent` совсем плохие;
- index `leaf_pages` раздулись при нормальной fill factor;
- место на диске критично → `VACUUM FULL` / `pg_repack` / `REINDEX`.

Каталог bloat (грубо):

```sql
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup,0), 3) AS dead_ratio
FROM pg_stat_all_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

---

## Слайд 13 — Перестроение объектов

Когда обычный VACUUM уже не вернёт место **ОС**:

| команда | что делает | блокировка |
|---------|------------|------------|
| `VACUUM FULL t` | переписывает таблицу + indexes compact | **AccessExclusive** — таблица мёртва для всех |
| `vacuumdb --full` | то же из shell | то же |
| `REINDEX INDEX CONCURRENTLY idx` | новый index, swap | write на table, read index ok |
| `REINDEX t` | все indexes таблицы | блок index + **write** table |

`VACUUM FULL` нужен **2× disk** на время работы (новый файл + старый).

Альтернатива без long lock: **pg_repack** (стороннее расширение).

---

## Слайд 14 — Перестроение объектов: REINDEX CONCURRENTLY

```sql
REINDEX INDEX CONCURRENTLY bloat_d_idx;
REINDEX TABLE CONCURRENTLY bloat;
```

| плюс | минус |
|------|-------|
| не блокирует DML на таблице | дольше, больше I/O |
| можно в prod днём | **не транзакционно** — упал посередине → дочистить руками |
| | **нет** для system catalogs |
| | **нет** для EXCLUDE constraints |

При fail — смотреть invalid index:

```sql
SELECT indexrelid::regclass, indisvalid
FROM pg_index WHERE NOT indisvalid;
```

---

## Слайд 15 — Демонстрация

### REINDEX CONCURRENTLY

```sql
-- после mass UPDATE, index раздулся
REINDEX TABLE CONCURRENTLY bloat;
SELECT * FROM pgstatindex('bloat_d_idx') \gx
-- leaf_pages ↓, deleted_pages → 0
```

### VACUUM FULL

```sql
VACUUM FULL bloat;
SELECT * FROM pgstattuple('bloat') \gx
-- tuple_percent ↑, table_len ↓ — место вернулось ОС
```

**Выбор стратегии:**

| ситуация | инструмент |
|----------|------------|
| routine dead tuples | autovacuum / `VACUUM` |
| stats устарела | `ANALYZE` |
| index bloat, prod online | `REINDEX CONCURRENTLY` |
| table bloat, окно обслуживания | `VACUUM FULL` или pg_repack |
| wraparound warning | `VACUUM FREEZE` / смотреть `age(relfrozenxid)` |

```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(oid)),
       age(relfrozenxid) AS xid_age
FROM pg_class JOIN pg_namespace n ON n.oid = relnamespace
WHERE relkind = 'r' AND n.nspname = 'public'
ORDER BY age(relfrozenxid) DESC;
```

---

## Слайд 16 — Итоги

- MVCC → dead tuples **неизбежны** → нужен VACUUM.  
- VACUUM = vacuum + VM + FSM + (опционально) ANALYZE + freeze.  
- **Autovacuum must run**, tuning — DBA2.  
- Обычный VACUUM **не shrink** файл; bloat лечится `FULL` / repack / reindex.  
- Freeze — не «теория», а **анти-остановка** кластера.

---

## Слайд 17 — Практика

### 1. Autovacuum off (учебная песочница!)

```sql
ALTER SYSTEM SET autovacuum = off;
SELECT pg_reload_conf();

SELECT pid, backend_type FROM pg_stat_activity
WHERE backend_type = 'autovacuum launcher';
-- 0 rows после reload (launcher уйдёт)
```

### 2. Таблица + 100k строк

```sql
CREATE DATABASE arch_vacuum_overview;
\c arch_vacuum_overview

CREATE TABLE t (n numeric);
CREATE INDEX t_n ON t(n);
INSERT INTO t SELECT random() FROM generate_series(1, 100_000);
```

### 3. Bloat без vacuum

```sql
\set SIZE 'SELECT pg_size_pretty(pg_table_size(''t'')) AS table_size,
                 pg_size_pretty(pg_indexes_size(''t'')) AS index_size \g (footer=off)'

:SIZE
UPDATE t SET n = n WHERE n < 0.5;   -- ~50% rows, 3 раза подряд
:SIZE
-- table ~4.3 MB → ~11 MB, index тоже растёт
```

### 4. VACUUM FULL

```sql
VACUUM FULL t;
:SIZE
-- ~4.3 MB / ~3.1 MB — файл shrink, index компактнее
```

### 5. С обычным VACUUM после каждого UPDATE

```sql
UPDATE t SET n = n WHERE n < 0.5;
VACUUM t;
:SIZE
-- рост один раз → plateau ~6.5 MB / ~4.6 MB
```

**Вывод:** массовые изменения — **батчами + vacuum между**, не одной гигантской tx.

### 6. Вернуть autovacuum

```sql
ALTER SYSTEM RESET autovacuum;
SELECT pg_reload_conf();
```

---

## Финал

На выход:

1. Dead tuple → VACUUM; index entries тоже.  
2. VM / FSM / ANALYZE / freeze — **часть** vacuum-процесса.  
3. `autovacuum=off` — табу (кроме осознанных lab).  
4. `VACUUM` reuse space; `VACUUM FULL` / `REINDEX` — shrink.  
5. `REINDEX CONCURRENTLY` — для prod; `FULL` — только в окно.  
6. Long tx = horizon = bloat — связка с l5.

Следующая тема (l7) — WAL и буферы: где vacuum пишет и почему это I/O.
