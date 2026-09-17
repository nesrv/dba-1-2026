# Текст лектора — l15

## Репликация: обзор физической репликации

> По PDF `dba1_15_replica_overview_physical`. Чуть больше исходника.
> Клон файлов + бесконечный replay WAL. Split-brain — враг №1.

---

## Слайд 1 — Титул

Тема: **физическая (streaming) репликация** — обзор.

Связка с l14: `pg_basebackup` и replication protocol уже видели.
Сегодня — постоянный standby вместо разового restore. Детали — в DBA3.

---

## Слайд 2 — Темы

1. Задачи и виды репликации  
2. Физическая репликация  
3. Уровни журнала (`wal_level`)  
4. Варианты использования реплики  
5. Переключение на реплику (promote)

---

## Слайд 3 — Задачи и виды репликации

Один сервер = точка отказа и потолок масштаба. Несколько серверов с **одними данными**:

| задача | идея |
|--------|------|
| отказоустойчивость | сбой одного узла |
| высокая доступность | плановые работы без даунтайма |
| масштабирование | разнести нагрузку |

| вид | уровень синхронизации |
|-----|------------------------|
| **физическая** | страницы + статусы транзакций (WAL) |
| **логическая** | строки таблиц (l16) |

Репликация = процесс **синхронизации** этих данных.

---

## Слайд 4 — Физическая репликация

Механизм: мастер **шлёт WAL** → реплика **проигрывает** (как crash recovery, только бесконечно).

| особенность | |
|-------------|--|
| роли | master → replica, **односторонне** |
| совместимость | **бинарная**: та же major, та же платформа |
| granularity | **весь кластер**, не одна БД |

Применение «механическое» — реплика не «понимает» SQL, только байты WAL.

---

## Слайд 5 — Физическая репликация (схема)

Компоненты:

- **walsender** (master)  
- **walreceiver** + **startup** (replica)  
- опционально **архив WAL** как запасной источник  

Два канала доставки:

1. **streaming** — основной, минимальный lag (до нуля при sync);  
2. **file-based** — из archive, отстаёт до switch сегмента.

Практика: stream + archive fallback. Не получили запись по протоколу — пробуем файл из архива.

Доки: [high-availability](https://postgrespro.ru/docs/postgresql/16/high-availability).

---

## Слайд 6 — Уровни журнала

**`wal_level`:**

| значение | crash recovery | hot backup / physical replica |
|----------|----------------|------------------------------|
| `minimal` | да | **нет** (часть изменений сразу на диск, мимо WAL) |
| `replica` | да | **да** (default с PG 10) |

До PG 10 default был `minimal` — репликация «из коробки» не заводилась.
Сменили на `replica`, потому что backup/replication — повседневность.

Логическая репликация — **`logical`**, это l16.

---

## Слайд 7 — Настройка физической репликации

Минимальный чеклист (PG 10+ дефолты уже ок):

```sql
SELECT name, setting FROM pg_settings
WHERE name IN ('wal_level','max_wal_senders');
-- replica, 10

SELECT type, user_name, address, auth_method
FROM pg_hba_file_rules()
WHERE 'replication' = ANY(database);
```

Развёртывание реплики:

```bash
rm -rf ~/tmp/backup
pg_basebackup --pgdata=~/tmp/backup -R --checkpoint=fast
```

**`-R`**: пишет `primary_conninfo` в `postgresql.auto.conf` + **`standby.signal`**
→ режим постоянного recovery (не обычный restore).

Дальше: stop replica → `mv` в `$PGDATA` → `chown postgres` → start.

---

## Слайд 8 — Процессы и pg_stat_replication

На реплике:

```text
startup recovering... waiting for ...
walreceiver
```

На мастере:

```text
walsender ... START_REPLICATION
```

Мониторинг:

```sql
SELECT * FROM pg_stat_replication \gx
-- sent_lsn, write_lsn, flush_lsn, replay_lsn
-- sync_state: async | sync | ...
```

`*_lsn` — где WAL на каждом этапе pipeline.  
`application_name` часто = `cluster_name` реплики (например `16/replica`).

---

## Слайд 9 — Использование реплики

**Hot standby** (default `hot_standby = on`):

| можно | нельзя |
|-------|--------|
| SELECT, COPY TO, cursors | INSERT/UPDATE/DELETE/TRUNCATE |
| SET, BEGIN/COMMIT | DDL, temp tables |
| pg_basebackup с реплики | VACUUM, ANALYZE, REINDEX |
| | GRANT/REVOKE, nextval, FOR UPDATE |

Триггеры и advisory locks на реплике **не срабатывают**.

`hot_standby = off` → **warm standby**: подключений нет вообще.

Доки: [hot-standby](https://postgrespro.ru/docs/postgresql/16/hot-standby).

---

## Слайд 10 — Использование реплики (демо)

```sql
-- master
CREATE DATABASE replica_overview_physical;
CREATE TABLE test(id int PRIMARY KEY, descr text);
INSERT INTO test VALUES (1, 'Раз');

-- replica :5433
SELECT * FROM test;   -- пусто → после INSERT на master → 'Раз'

INSERT INTO test VALUES (2, 'Два');
-- ERROR: cannot execute INSERT in a read-only transaction
```

Read-only — фича, не баг. Бэкап с реплики — ок, помните про lag.

---

## Слайд 11 — Надёжность: синхронная репликация

Аналог `synchronous_commit`, но для **реплики**:

- **async** — master не ждёт replica;  
- **sync** — commit ждёт, пока WAL **принят** синхронной standby.

Надёжнее (данные на втором узле даже при смерти primary), но **медленнее**.
Replica упала — commits **висят**, пока не вернётся (или не уберёте из `synchronous_standby_names`).

Есть промежуточные уровни sync — детали в DBA3.

---

## Слайд 12 — Долгие аналитические запросы

Проблема на master: long SELECT держит **vacuum horizon** → bloat.

Решение: **отчёты на реплику**.

Конфликт recovery vs query:

1. vacuum на master удалил версии строк, нужные SELECT на replica;  
2. exclusive lock на master vs query на replica.

«Отчётную» реплику настраивают так, чтобы конфликтующие WAL **откладывались** —
реплика может **отставать**; для аналитики обычно ок.

Параметры: **`max_standby_streaming_delay`**, **`hot_standby_feedback`**.

---

## Слайд 13 — Несколько реплик

Несколько standby → **read scaling** для коротких OLTP-read.

Обратная связь (feedback) + короткие запросы → master не удалит строки, нужные replica
(как будто запросы крутились на master).

Нюанс: **нет глобальной read-consistency** между репликами.
Прочитали с A и B — можете увидеть разные эпохи (даже при sync).
Балансировку PG **не делает** — Patroni, HAProxy, pgpool, app-side routing.

---

## Слайд 14 — Каскадная репликация

Replica A → Replica B → …: меньше **walsender**-нагрузки на master, меньше дублирования трафика.

Минусы:

- больше **lag** дальше по цепочке;  
- **синхронная** каскадная репликация **не поддерживается** (sync только с прямой standby);  
- feedback от **всех** узлов всё равно идёт на master.

---

## Слайд 15 — Отложенная репликация

**Delayed standby** — применяет WAL с **задержкой** (например 1 час).

«Машина времени» без полноценного PITR из архива:

- откатить `DROP TABLE` / кривой UPDATE;  
- быстрее, чем restore base + WAL.

Снимков «как pg_dump на прошлое» в PG нет. Настройки — DBA3.

---

## Слайд 16 — Переключение на реплику

| тип | когда |
|-----|-------|
| **плановый** | maintenance master, switchover |
| **аварийный** | master мёртв, failover |

По умолчанию — **ручной** promote. Автомат — Patroni, repmgr, Pacemaker…

Главное: приложение ходит **только на одного** primary. Иначе **split-brain** —
два независимых мира данных, склеить почти невозможно.

---

## Слайд 17 — Переключение на реплику (демо)

```sql
SELECT pg_is_in_recovery();  -- t на replica
```

```bash
sudo pg_ctlcluster 16 replica promote
# или SELECT pg_promote();  -- с PG 12
```

```sql
SELECT pg_is_in_recovery();  -- f
INSERT INTO test VALUES (2, 'Два');  -- теперь можно писать
```

Два **независимых** кластера. Обратно «склеить» без боли нельзя — подчеркните это вслух.

---

## Слайд 19 — Итоги / Практика

*(в презентации `slide19.png`, без slide18)*

**Итоги:**

- физическая реплика = WAL stream (или файлы) + replay;  
- весь кластер, одна major, master→replica;  
- hot standby, sync/async, cascade, delay — сценарии.

**Практика 1 — Sync:**

```sql
-- на мастере (имя = cluster_name реплики)
ALTER SYSTEM SET synchronous_standby_names = '"16/replica"';
SELECT pg_reload_conf();
SELECT sync_state FROM pg_stat_replication;  -- sync

-- stop replica → CREATE TABLE на master зависает
-- start replica → CREATE TABLE завершается
```

`synchronous_commit` по умолчанию `on`, но без `synchronous_standby_names`
синхронизация только с локальным диском.

**Практика 2 — Конфликты:**

```sql
-- на реплике
ALTER SYSTEM SET max_standby_streaming_delay = 0;
SELECT pg_reload_conf();
SELECT pg_sleep(5), count(*) FROM test;  -- долгий запрос

-- на мастере параллельно: DELETE + VACUUM
-- → canceling statement due to conflict with recovery

-- потом на реплике:
ALTER SYSTEM SET hot_standby_feedback = on;
-- VACUUM на master: "dead but not yet removable" — запрос не убивается
```

Мораль: `max_standby_streaming_delay` откладывает **replay на реплике**;  
`hot_standby_feedback` откладывает **vacuum на мастере**.

Не забудьте `RESET synchronous_standby_names` после лабы.

---

## Финал

1. `-R` + `standby.signal` vs разовый restore;  
2. hot standby — только read;  
3. `pg_stat_replication`, promote / split-brain;  
4. sync ждёт replica; feedback vs `max_standby_streaming_delay`.

Физическая реплика — фундамент HA. Логическая — в l16, другие компромиссы.
