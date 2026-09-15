# Текст лектора — l15

## Репликация: обзор физической репликации

> Клон файлов + бесконечный replay WAL.
> Реплика читает, мастер пишет. Split-brain — ваш враг №1.

---

## Слайд 1 — Титул

Тема: **физическая (streaming) репликация** — обзор.

Связка с l14: pg_basebackup и replication protocol мы уже видели. Сегодня — постоянный standby вместо разового restore.

---

## Слайд 2 — Темы

1. Задачи и виды репликации  
2. Физическая репликация  
3. Уровни журнала (`wal_level`)  
4. Сценарии использования реплики  
5. Переключение на реплику (promote)

Детали — в DBA3; здесь карта местности.

---

## Слайд 3 — Задачи и виды репликации

Зачем несколько серверов с **одними данными**:

| задача | идея |
|--------|------|
| отказоустойчивость | сбой одного узла |
| высокая доступность | плановые работы без даунтайма |
| масштабирование | разнести read/write |

| вид | уровень синхронизации |
|-----|------------------------|
| **физическая** | страницы + статусы транзакций (WAL) |
| **логическая** | строки таблиц (следующая лекция l16) |

---

## Слайд 4 — Физическая репликация

Механизм: мастер **шлёт WAL** → реплика **проигрывает** (как crash recovery, только бесконечно).

| особенность | |
|-------------|--|
| роли | master → replica, **односторонне** |
| совместимость | **бинарная**: та же major, та же платформа |
| гранularity | **весь кластер**, не одна БД |

«Механическое» применение — реплика не «понимает» SQL, только байты WAL.

---

## Слайд 5 — Физическая репликация (схема)

Компоненты:

- **walsender** (master)  
- **walreceiver** + **startup** (replica)  
- опционально **архив WAL** как запасной источник  

Два канала доставки:

1. **streaming** — основной, минимальный lag  
2. **file-based** — из archive, отстаёт до switch сегмента  

На практике: stream + archive fallback.

Доки: https://postgrespro.ru/docs/postgresql/16/high-availability

---

## Слайд 6 — Уровни журнала

**`wal_level`:**

| значение | crash recovery | hot backup / physical replica |
|----------|----------------|------------------------------|
| `minimal` | да | **нет** (часть изменений в data, мимо WAL) |
| `replica` | да | **да** (default с PG 10) |

До PG 10 default был `minimal` — репликация «не заводилась» из коробки.

Логическая репликация — **`logical`**, это уже l16.

---

## Слайд 7 — Настройка физической репликации

Минимальный чеклист:

```sql
SELECT name, setting FROM pg_settings
WHERE name IN ('wal_level','max_wal_senders');
-- replica, 10 — ок

SELECT type, user_name, address, auth_method
FROM pg_hba_file_rules()
WHERE 'replication' = ANY(database);
```

Развёртывание реплики:

```bash
rm -rf ~/tmp/backup
pg_basebackup --pgdata=~/tmp/backup -R --checkpoint=fast
```

**`-R`**: пишет `primary_conninfo` + **`standby.signal`** → режим постоянного recovery.

Дальше: stop replica cluster → mv в `$PGDATA` → chown postgres → start.

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

`*_lsn` — где WAL на каждом этапе pipeline. `sync_state` — sync/async/quorum (детали в DBA3).

---

## Слайд 9 — Использование реплики

**Hot standby** (default `hot_standby = on`):

| можно | нельзя |
|-------|--------|
| SELECT, COPY TO, cursors | INSERT/UPDATE/DELETE/TRUNCATE |
| SET, BEGIN/COMMIT | DDL, temp tables |
| pg_basebackup с реплики | VACUUM, ANALYZE, REINDEX |
| | GRANT/REVOKE, nextval, FOR UPDATE |

`hot_standby = off` → **warm standby**: подключений нет вообще.

Доки: https://postgrespro.ru/docs/postgresql/16/hot-standby

---

## Слайд 10 — Использование реплики (демо)

```sql
-- master
CREATE DATABASE replica_overview_physical;
CREATE TABLE test(id int PRIMARY KEY, descr text);
INSERT INTO test VALUES (1, 'Раз');

-- replica :5433
SELECT * FROM test;   -- пусто → INSERT на master → 'Раз'

INSERT INTO test VALUES (2, 'Два');
-- ERROR: cannot execute INSERT in a read-only transaction
```

Реплика **read-only** — это фича, не баг.

---

## Слайд 11 — Надёжность: синхронная репликация

Аналог `synchronous_commit` для **реплики**:

- async — master не ждёт replica  
- sync — commit ждёт, пока WAL **принят** синхронной standby  

Надёжнее (данные на втором узле), но **медленнее**. Replica упала — commits **висят**, пока не вернётся.

Схема на слайде: sync replication между master и одной standby.

---

## Слайд 12 — Долгие аналитические запросы

Проблема на master: long SELECT держит **vacuum horizon** → bloat.

Решение: **отчёты на реплику**.

Конфликт recovery vs query:

1. vacuum на master удалил версии строк, нужные SELECT на replica  
2. exclusive lock на master vs query на replica  

Параметры: **`max_standby_streaming_delay`**, **`hot_standby_feedback`**. Реплика может **отставать** — для отчётов обычно ок.

---

## Слайд 13 — Несколько реплик

Несколько standby → **read scaling** для OLTP-read.

Нюанс: **нет глобальной read-consistency** между репликами. Прочитали с A и B — можете увидеть разные эпохи.

Обратная связь (feedback) + короткие read-запросы → master не удалит строки, нужные replica.

Балансировку read-трафика PG **не делает** — Patroni, HAProxy, pgpool, app-side routing.

---

## Слайд 14 — Каскадная репликация

Replica A → Replica B → …: меньше **walsender**-нагрузки на master, меньше дублирования трафика.

Минусы:

- больше **lag** дальше по цепочке  
- **синхронная** каскадная репликация **не поддерживается** (sync только с прямой standby)  
- feedback от **всех** узлов всё равно идёт на master  

---

## Слайд 15 — Отложенная репликация

**Delayed standby** — применяет WAL с **задержкой** (например 1 час).

«Машина времени» без полноценного PITR из архива:

- откатить `DROP TABLE` idiot-user  
- быстрее, чем restore base + WAL  

Настройки — DBA3. Снимков «как pg_dump на прошлое» в PG нет.

---

## Слайд 16 — Переключение на реплику

| тип | когда |
|-----|-------|
| **плановый** | maintenance master, switchover |
| **аварийный** | master мёртв, failover |

По умолчанию — **ручной** promote. Автомат — Patroni, repmgr, Pacemaker и т.д.

Главное: приложение должно ходить **только на одного** primary. Иначе **split-brain** — два независимых мира данных.

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

Два **независимых** кластера. Обратно «склеить» без боли нельзя.

---

## Слайд 19 — Итоги / Практика
*(в презентации файл `slide19.png`, без slide18)*

**Итоги:**

- физическая реплика = WAL stream + replay  
- весь кластер, одна major, master→replica  
- hot standby, sync/async, cascade, delay — сценарии  

**Практика:**

1. **Sync:**  
   `ALTER SYSTEM SET synchronous_standby_names = '"16/replica"';`  
   `pg_reload_conf();` — `sync_state = sync`  
   stop replica → транзакция на master **блокируется** до start replica  

2. **Конфликты:**  
   `max_standby_streaming_delay = 0` → long query на replica + VACUUM на master →  
   `canceling statement due to conflict with recovery`  
   `hot_standby_feedback = on` → vacuum на master **не удаляет** пока нужно replica  

Подсказка: `pg_sleep(5)` в SELECT для искусственно долгого запроса.

---

## Финал

На память:

1. `-R` + `standby.signal` vs разовый restore;  
2. hot standby — только read;  
3. `pg_stat_replication`, promote / split-brain;  
4. sync ждёт replica, feedback vs max_standby_streaming_delay.

Физическая реплика — фундамент HA. Логическая — в l16, там другие компромиссы.
