# Текст лектора — l12

## Задачи администрирования: мониторинг

> Мониторинг = ОС снаружи + статистика и логи внутри Postgres.
> Без внешней системы вы видите «сейчас», но не «как было вчера в 03:14».

---

## Слайд 1 — Титул

Тема: **мониторинг** — от `ps` и `iostat` до `pg_stat_*` и pgBadger.

Postgres сам dashboard не рисует. Он даёт сырьё — выбираете Zabbix/PoWA/pg_profile или grep по логу.

---

## Слайд 2 — Темы

1. Средства **ОС**  
2. **Накопительная статистика**  
3. **Журнал сообщений**  
4. **Внешние** системы мониторинга  

Плюс практика: `pg_stat_*` после DELETE/VACUUM и deadlock в логе.

---

## Слайд 3 — Средства ОС

| Область | Инструменты |
|---------|-------------|
| Процессы | `ps`, `pgrep` |
| CPU/RAM/IO | `top`, `vmstat`, `iostat`, `sar` |
| Диск | `df`, `du`, квоты |

Postgres-параметры:

- `update_process_title=on` (default) — в `ps` видно `checkpointer`, `student dbname`, `idle in transaction`  
- `cluster_name` — отличить несколько кластеров на одном хосте  

Факты:

- Размер БД — и из SQL (`pg_database_size`), и `du $PGDATA/base/oid`  
- Квоты Unix — на пользователя ОС `postgres`; **ролевых квот** в Postgres нет  
- Один `postgres` на весь кластер — квота должна покрывать **все** БД  

Доки: monitoring-ps, diskusage.

---

## Слайд 4 — Накопительная статистика

Два главных источника «изнутри»:

1. **Stats** — счётчики в shared memory, views `pg_stat_*`  
2. **Server log** — события, ошибки, медленные запросы  

Stats — для трендов и «кто дёргает таблицу». Лог — для расследований и audit trail.

---

## Слайд 5 — Сбор статистики

| Параметр | Действие | Default |
|----------|----------|---------|
| `track_activities` | текущие команды | on |
| `track_counts` | таблицы/индексы | on |
| `track_functions` | вызовы функций | **off** |
| `track_io_timing` | время read/write блоков | **off** |
| `track_wal_io_timing` | время WAL IO | **off** |

Чем больше собираете — тем выше overhead. `track_io_timing` включайте осознанно (нужен для `blk_read_time`).

Сброс:

```sql
SELECT pg_stat_reset();              -- текущая БД
SELECT pg_stat_reset_shared('io');   -- shared counters
```

---

## Слайд 6 — Архитектура

Цепочка:

1. Backend копит stats **в транзакции**  
2. Пишет в shared memory — **не чаще 1 раза/сек** (compile-time)  
3. При **штатном** shutdown — flush в `PGDATA/pg_stat/`  
4. При crash — счётчики **обнуляются**  

`stats_fetch_consistency`:

| Значение | Поведение |
|----------|-----------|
| `none` | только shared mem |
| `cache` | кеш per-object (default) |
| `snapshot` | снимок всей БД в backend |

Кеш сбрасывается в конце транзакции или `pg_stat_clear_snapshot()`.

Факт: stats **слегка запаздывает** — для capacity planning норм, для «миллисекундной» отладки — нет.

---

## Слайд 7 — Демонстрация: нагрузка

```sql
ALTER SYSTEM SET track_io_timing = on;
SELECT pg_reload_conf();
```

```bash
pgbench -i admin_monitoring
pgbench -T 10 admin_monitoring   # ~325 tps
```

Перед тестом — сброс stats (см. слайд 5).

Строки vs страницы — два семейства views:

- `pg_stat_all_tables` — `n_tup_*`, `seq_scan`, `idx_scan`, vacuum/analyze counts  
- `pg_statio_all_tables` — `heap_blks_hit/read`, `idx_blks_*`  

Аналоги для инdexes: `pg_stat_all_indexes`, `pg_statio_all_indexes`.

**idx_scan = 0** долгое время → кандидат на удаление индекса (после проверки, что не nocturnal batch).

---

## Слайд 8 — Демонстрация: БД и IO

```sql
SELECT * FROM pg_stat_database
WHERE datname = 'admin_monitoring' \gx
-- xact_commit, deadlocks, temp_files, blk_*_time, sessions_*
```

`numbackends` — сколько backend'ов **сейчас** на этой БД.

```sql
CHECKPOINT;
SELECT backend_type, sum(hits), sum(reads), sum(writes)
FROM pg_stat_io GROUP BY 1;
```

Появилось в PG16 — IO **по типу процесса** (client, checkpointer, …).

Есть варианты views: `_all_`, `_user_`, `_sys_`, плюс `pg_stat_xact_*` для **текущей** транзакции.

---

## Слайд 9 — Текущие активности

View **`pg_stat_activity`** — «кто что делает прямо сейчас».

Зависит от `track_activities` (on по умолчанию).

Поля, на которые смотрят первыми:

| Поле | Смысл |
|------|-------|
| `state` | active / idle / idle in transaction / … |
| `wait_event_type` + `wait_event` | чего ждём (Lock, IO, Client…) |
| `query` | текст (может обрезаться) |
| `pg_blocking_pids(pid)` | кто держит lock |

Демо — на следующих слайдах.

---

## Слайд 10 — Демонстрация: блокировка

Сценарий: два сеанса, одна строка.

```sql
-- сеанс 1
BEGIN;
UPDATE t SET n = n + 1;

-- сеанс 2
UPDATE t SET n = n + 2;   -- висит
```

```sql
SELECT pid, query, state, wait_event, wait_event_type,
       pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE backend_type = 'client backend' \gx
```

Сеанс 1: `idle in transaction` — **опасное** состояние: держит snapshot, мешает vacuum.

Таймауты:

- `idle_in_transaction_session_timeout`  
- `idle_session_timeout`  

---

## Слайд 11 — Демонстрация: снять блокировку

```sql
SELECT pid FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

SELECT locktype, transactionid, pid, mode, granted
FROM pg_locks WHERE …;
```

| Действие | Функция |
|----------|---------|
| Прервать **запрос** | `pg_cancel_backend(pid)` |
| Убить **сеанс** | `pg_terminate_backend(pid)` |

```sql
SELECT pg_terminate_backend(b.pid)
FROM unnest(pg_blocking_pids(77683)) AS b(pid);
```

`pg_blocking_pids` возвращает **массив** — `unnest` на случай нескольких блокировщиков.

Подробнее про lock modes — DBA2.

---

## Слайд 12 — Демонстрация: все процессы

```sql
SELECT pid, backend_type, backend_start, state
FROM pg_stat_activity;
```

Сравнение с ОС:

```bash
head -1 $PGDATA/postmaster.pid   # PID postmaster
ps -o pid,command --ppid <postmaster_pid>
```

В `ps` — те же checkpointer, walwriter, autovacuum launcher + клиенты с `cluster_name` и `application_name`.

---

## Слайд 13 — Выполнение команд

Progress views — для **долгих** операций:

| Команда | View |
|---------|------|
| VACUUM | `pg_stat_progress_vacuum` |
| CREATE INDEX / REINDEX | `pg_stat_progress_create_index` |
| ANALYZE | `pg_stat_progress_analyze` |
| CLUSTER / VACUUM FULL | `pg_stat_progress_cluster` |
| COPY | `pg_stat_progress_copy` |
| base backup | `pg_stat_progress_basebackup` |

Опрос в цикле из второго сеанса — «сколько % уже прошло».

Доки: progress-reporting.

---

## Слайд 14 — Дополнительная статистика

Расширения (часть — в shared_preload_libraries):

| Extension | Зачем |
|-----------|-------|
| `pg_stat_statements` | топ запросов, время, calls |
| `pgstattuple` | bloat, dead/live tuples |
| `pg_buffercache` | что в shared_buffers |
| `pg_wait_sampling` | профиль ожиданий |
| `pg_stat_kcache` | CPU/IO per query |

`pg_profile` / **pgpro_pwr** — снимки и diff между периодами.

DBA2/DEV2 — углубление. Wiki: https://wiki.postgresql.org/wiki/Monitoring

---

## Слайд 15 — Журнал сообщений

Второй столп мониторинга — **server log**.

Не путать с **WAL** — это про recovery, не про «кто и когда SELECT *».

Три решения админа:

1. **Куда** писать (`log_destination`, collector)  
2. **Что** писать (`log_min_messages`, `log_statement`, …)  
3. **Как ротировать** и **как читать** (grep, pgBadger)

---

## Слайд 16 — Настройка журнала

**Приёмники** (`log_destination`):

| Значение | Куда |
|----------|------|
| `stderr` | stderr (default) |
| `csvlog` / `jsonlog` | через collector |
| `syslog` / `eventlog` | OS log |

**Collector** (`logging_collector=on`):

- собирает со **всех** процессов  
- не теряет сообщения (но может стать bottleneck)  
- пишет в `log_directory` / `log_filename`  

Без collector csvlog/jsonlog недоступны.

---

## Слайд 17 — Информация в журнале

Выборочно включают:

| Что | Параметр |
|-----|----------|
| Уровень сообщений | `log_min_messages` |
| Медленные запросы | `log_min_duration_statement` |
| Любая duration | `log_duration` |
| Checkpoints | `log_checkpoints` |
| Connect/disconnect | `log_connections` / `log_disconnections` |
| Lock waits | `log_lock_waits` |
| SQL текст | `log_statement` |
| Temp files | `log_temp_files` |

Default — **почти всё off**, иначе IO убьёт диск. Включайте точечно под задачу расследования.

`application_name` в connection string — потом фильтруете grep'ом.

---

## Слайд 18 — Ротация файлов

Встроенная (с collector):

| Параметр | Назначение |
|----------|------------|
| `log_filename` | маска (`postgresql-%H.log`) |
| `log_rotation_age` | по времени |
| `log_rotation_size` | по размеру, KB |
| `log_truncate_on_rotation` | перезаписывать старые |

Примеры:

- `'postgresql-%H.log'` + `1h` → 24 файла/сутки  
- `'postgresql-%a.log'` + `1d` → 7 файлов/неделю  

Альтернатива — **logrotate** (`/etc/logrotate.d/postgresql-common` в Ubuntu).

---

## Слайд 19 — Анализ журнала

Простое:

```bash
sudo grep FATAL /var/log/postgresql/postgresql-16-main.log | tail
```

Для slow queries:

```sql
ALTER SYSTEM SET log_min_duration_statement = 0;
SELECT pg_reload_conf();
```

```bash
sudo tail -1 .../postgresql-16-main.log
-- LOG: duration: 348 ms  statement: SELECT ...
```

**pgBadger** — de-facto standard, но хочет **английские** сообщения и определённый формат лога.

Книга: https://edu.postgrespro.ru/monitoring.pdf

---

## Слайд 20 — Внешний мониторинг

Универсальные: **Zabbix**, Munin, Cacti, Datadog, New Relic…

PostgreSQL-ориентированные: **PoWA**, PGObserver, OPM, pg_profile.

Postgres даёт метрики — **хранение, графики, алерты** делает внешняя система.

Минимальный набор алертов: disk %, connections, replication lag, autovacuum stuck, deadlocks ↑.

---

## Слайд 21 — Итоги / Практика / Практика+

### Итоги

- Мониторинг = **ОС + pg_stat_* + log + (желательно) внешняя система**  
- Stats запаздывает и сбрасывается при crash  
- `pg_stat_activity` — первый инструмент при «всё висит»  

### Практика — stats и deadlock

```sql
CREATE TABLE t(n numeric);
INSERT INTO t SELECT 1 FROM generate_series(1,1000);
DELETE FROM t;
-- n_tup_ins=1000, n_tup_del=1000, n_dead_tup=1000, n_live_tup=0

VACUUM;
-- n_dead_tup=0, vacuum_count=1
```

Deadlock на двух строках — в логе `ERROR: deadlock detected` + DETAIL с pid и statement.

### Практика+ — pg_stat_statements

```sql
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
-- restart
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time
FROM pg_stat_statements ORDER BY calls DESC LIMIT 5;
```

Не забудьте `RESET shared_preload_libraries` после лабы, если не нужен постоянно.

---

## Финал

1. `pg_stat_activity` + `pg_blocking_pids` — при блокировках  
2. `pg_stat_all_tables` / `pg_statio_*` — кто и как читает таблицы  
3. Лог — включать **точечно**, ротировать всегда  
4. Stats ≠ real-time; после crash счётчики с нуля  
5. Dashboard — **вне** Postgres, данные — из него
