# Текст лектора — l7

## Архитектура: буферный кеш и журнал (WAL)

> Меньше воды, больше shared_buffers, LSN, checkpointer и «почему immediate shutdown — это симуляция краша».
> Без WAL Postgres был бы быстрым и мёртвым после power loss.

---

## Слайд 1 — Титул

Тема: **буферный кеш + WAL** — как Postgres одновременно **быстрый** и **живой после сбоя**.

Связка с l6: vacuum пишет на диск → это I/O через те же буферы и тот же журнал.

---

## Слайд 2 — Темы

Пять блоков:

1. Устройство буферного кеша  
2. Алгоритм вытеснения  
3. Журнал предзаписи (WAL)  
4. Контрольная точка  
5. Процессы (checkpointer, bgwriter, walwriter)

Две демо — кеш (`EXPLAIN BUFFERS`) и WAL (LSN, recovery). Практика — fast vs immediate shutdown.

---

## Слайд 3 — Буферный кеш

**shared_buffers** — массив слотов в shared memory.

| элемент | что внутри |
|---------|------------|
| страница | **8 KB** (меняется только при сборке) |
| метаданные | файл + номер страницы в файле |
| dirty | изменена в RAM, на диск ещё не сброшена |

Поток работы:

1. backend ищет страницу в кеше  
2. miss → read через ОС (может попасть в **OS page cache**)  
3. hit → работа без syscall  
4. modify → буфер **dirty**, запись на диск **отложена**

Блокировки в shared memory — не бесплатны. Чем меньше страниц трогает запрос, тем лучше.

Доки: https://postgrespro.ru/docs/postgresql/16/runtime-config-resource#GUC-SHARED-BUFFERS

---

## Слайд 4 — Вытеснение

Кеш конечен → **eviction**.

Алгоритм: **clock-sweep / LRU-подобный** — выгоняем «холодные» страницы.

| шаг | действие |
|-----|----------|
| выбрали victim | если dirty → **flush** на диск |
| слот свободен | read новой страницы |

«Горячий» working set обычно мал — при нормальном `shared_buffers` большинство запросов бьёт в hit.

Факт: backend сам может сбросить dirty page при eviction, если bgwriter/checkpointer не успели — отсюда latency spikes.

---

## Слайд 5 — Демонстрация

Демо: влияние буферного кеша на план и время.

```sql
CREATE DATABASE arch_wal_overview;
\c arch_wal_overview
CREATE TABLE t(n integer);
INSERT INTO t SELECT id FROM generate_series(1, 100_000);

SHOW shared_buffers;   -- дефолт 128MB — для прода мало
```

Перезапуск → холодный кеш:

```bash
sudo pg_ctlcluster 16 main restart
psql arch_wal_overview
```

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF, TIMING OFF)
SELECT * FROM t;
-- Buffers: shared read=443  — читали с диска

EXPLAIN (ANALYZE, BUFFERS, COSTS OFF, TIMING OFF)
SELECT * FROM t;
-- Buffers: shared hit=443   — всё из кеша
-- Execution Time меньше; Planning Time тоже (каталог в кеше)
```

| метрика | cold | warm |
|---------|------|------|
| `shared read` | много | ~0 |
| `shared hit` | мало | почти всё |
| время | выше | ниже |

---

## Слайд 6 — Журнал предзаписи (WAL)

**Проблема:** RAM умирает при сбое → dirty buffers пропадают.

**WAL** = поток записей «как повторить операцию». Запись WAL на диск **раньше**, чем изменённые data pages (**write-ahead**).

| защищает | не защищает |
|----------|-------------|
| heap/index pages | TEMP tables |
| clog (стatus tx) | UNLOGGED tables |

Каталог: `PGDATA/pg_wal/`. Файлы по **16 MB** (задаётся при initdb).

Доки: https://postgrespro.ru/docs/postgresql/16/wal-intro

---

## Слайд 7 — Демонстрация

Демо: устройство WAL, LSN, поток записей, роль при сбое.

```sql
SELECT pg_current_wal_lsn();          -- текущая позиция

UPDATE t SET n = 100_001 WHERE n = 1;

SELECT pg_current_wal_lsn();

SELECT '0/237BD40'::pg_lsn - '0/2378D28'::pg_lsn AS bytes;
-- разница LSN = объём WAL в байтах
```

LSN — **64-bit offset** в потоке WAL, пишется как `high/low`.

```sql
SELECT * FROM pg_ls_waldir() ORDER BY name LIMIT 10;
```

Имена файлов: `TTTTTTTTTTTTTTTTLLLLLLLLLLLLLLLL` (timeline + log segment).

---

## Слайд 8 — Контрольная точка

**Checkpoint (КТ):** периодический сброс **всех** dirty buffers (+ clog).

| эффект | зачем |
|--------|-------|
| data до КТ на диске | recovery короче |
| старые WAL можно удалить | диск не раздувается |

**Recovery после сбоя:**

1. старт с **последней завершённой** КТ  
2. redo WAL-записей, если page на диске старее  
3. tx без commit record в WAL → **abort**

КТ на больших `shared_buffers` — тяжёлая; Postgres **размазывает** flush по времени.

На слайде: между двумя КТ — «необходимые файлы журнала»; после сбоя — redo с последней КТ.

---

## Слайд 9 — Демонстрация

Демо: recovery, dirty buffers, checkpoint, redo после рестарта.

Симуляция **краша** (не делать на проде):

```bash
sudo kill -QUIT $(sudo head -n 1 /var/lib/postgresql/16/main/postmaster.pid)
sudo pg_ctlcluster 16 main start
```

```sql
SELECT min(n), max(n) FROM t;
-- UPDATE до kill восстановился
```

Обычный stop делает КТ → recovery не нужен. `kill -QUIT` / `immediate` — как обрыв питания.

После КТ PostgreSQL удаляет WAL, не нужный для recovery.

---

## Слайд 10 — Производительность

WAL быстрее прямой записи data pages:

- записи **меньше** страницы (8 KB)  
- **последовательная** запись — HDD не страдает

| режим | кто пишет | надёжность |
|-------|-----------|------------|
| **sync commit** | backend ждёт WAL flush + **fsync** | commit = на диске |
| **async** | **walwriter** с задержкой | окно потери последних commits |

Оба режима **сосуществуют**: длинная tx → WAL async; при flush data page без WAL на диске → **принудительный sync WAL**.

Параметры: `synchronous_commit`, `wal_writer_delay`, `commit_delay` (редко трогают).

---

## Слайд 11 — Основные процессы

| процесс | роль |
|---------|------|
| **walwriter** | async flush WAL buffers → disk |
| **checkpointer** | periodic full dirty flush (КТ) |
| **bgwriter** | proactive partial dirty flush |
| **backend** | sync WAL at commit; flush evicted dirty page |

Схема: shared memory (`wal`, `clog`, buffer cache) → OS cache → files.

Если bgwriter ленится — backends сами пишут dirty pages → **pg_stat_bgwriter** и latency.

Поиск процессов:

```bash
sudo cat /var/lib/postgresql/16/main/postmaster.pid   # PID postmaster
sudo ps -o pid,command --ppid <pid>
# checkpointer, background writer, walwriter
```

---

## Слайд 12 — Уровни журнала

Параметр **`wal_level`** (нужен restart):

| уровень | что в WAL | для чего |
|---------|-----------|----------|
| **minimal** | только crash recovery | почти не используют |
| **replica** (default) | + info для backup/replication | physical replica, pg_basebackup |
| **logical** | + декодируемые row changes | logical replication, CDC |

Повышение уровня → **больше WAL** → следить за диском и replication slots.

---

## Слайд 13 — Итоги

Коротко:

1. Buffer cache режет random I/O — смотреть `BUFFERS` в EXPLAIN.  
2. Надёжность = WAL (write-ahead).  
3. Checkpoint ограничивает объём redo.  
4. WAL переиспользуется: recovery, backup, replication.

---

## Слайд 14 — Практика

Задания:

1. Найти процессы buffer cache / WAL через ОС.  
2. Stop **fast** → start → читать server log.  
3. Stop **immediate** → start → сравнить log.

```bash
# 1
sudo head -n 1 /var/lib/postgresql/16/main/postmaster.pid
sudo ps -o pid,command --ppid <pid>

# 2 fast — делает checkpoint
sudo pg_ctlcluster 16 main stop
sudo pg_ctlcluster 16 main start
# log: checkpoint starting: shutdown immediate ... database system was shut down

# 3 immediate — как сбой
sudo pg_ctlcluster 16 main stop -m immediate --skip-systemctl-redirect
sudo pg_ctlcluster 16 main start
# log: database system was interrupted; automatic recovery in progress
```

| режим | checkpoint | старт |
|-------|------------|-------|
| **fast** | да | сразу ready |
| **immediate** | нет | recovery + redo |

Лог: `/var/log/postgresql/postgresql-16-main.log`

---

## Финал

На выход:

1. `shared_buffers` + OS cache = два уровня; `hit/read` в EXPLAIN.  
2. WAL first, pages later — основа durability.  
3. LSN, `pg_wal/`, `pg_ls_waldir()`.  
4. Checkpoint = граница redo; `checkpointer` / `bgwriter` / `walwriter`.  
5. `wal_level`: replica для физики, logical для logical rep.  
6. `stop fast` ≠ `stop immediate` — разница видна в log за 30 секунд.

Следующая тема (l8) — базы, схемы, `search_path`: логическая иерархия кластера.
