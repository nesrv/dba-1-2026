# Текст лектора — l16

## Репликация: обзор логической репликации

> По PDF `dba1_16_replica_overview_logical`. Чуть больше исходника.
> Строки, не байты. Pub/sub. Major можно менять — DDL и конфликты ваши.

---

## Слайд 1 — Титул

Тема: **логическая репликация** — публикация/подписка на уровне строк таблиц.

Контраст с l15: не весь `$PGDATA`, а **выбранные таблицы**; подписчик **может писать** локально.

---

## Слайд 2 — Темы

1. Логическая репликация — механизм  
2. Уровни журнала (`logical`)  
3. Публикации и подписки  
4. Конфликты и replica identity  
5. Сценарии: консолидация, upgrade, master-master  

---

## Слайд 3 — Логическая репликация

| физическая (l15) | логическая |
|------------------|------------|
| один пишет, остальные replay WAL | **любой** может писать |
| байты WAL | **декодированные изменения строк** |
| весь кластер | **отдельные таблицы** |
| бинарная совместимость | совместимость **протокола** |

Publisher: читает свой WAL → **logical decoding** → шлёт подписчикам.  
Subscriber: **logical replication worker** применяет изменения.

Один сервер может и публиковать, и подписываться — произвольные потоки данных.

Доки: [logical-replication](https://postgrespro.ru/docs/postgresql/16/logical-replication).

---

## Слайд 4 — Логическая репликация (схема)

**walsender** на publisher ↔ **logical replication worker** на subscriber.

Фишка: subscriber может тянуть изменения с **физической реплики** publisher'а —
разгрузить primary (logical decoding на standby).

Доки: [publication](https://postgrespro.ru/docs/postgresql/16/logical-replication-publication),
[subscription](https://postgrespro.ru/docs/postgresql/16/logical-replication-subscription).

---

## Слайд 5 — Публикации и подписки

**Publication:**

- одна или несколько таблиц **одной БД**;  
- с PG 15: **column list** + **row filter**;  
- INSERT/UPDATE/DELETE/TRUNCATE (по настройке);  
- после **COMMIT** — построчно;  
- слот **logical replication**.

**Subscription:**

- initial sync (`copy_data`);  
- apply **без SQL-планирования** — прямое изменение строк;  
- **конфликты** с локальными данными возможны.

DDL **не едет** — таблицы на subscriber создаёте сами.
Одна SQL-команда на publisher → много однострочных сообщений на subscriber.

---

## Слайд 6 — Уровни журнала

**`wal_level`:** `minimal` < `replica` < **`logical`**

Для logical в WAL нужны: **идентификаторы строк**, факты изменения схем/типов
(чтобы подписчик всегда знал структуру объектов).

Default = `replica` → на publisher:

```sql
ALTER SYSTEM SET wal_level = logical;
-- restart кластера, не reload
```

Доки: [protocol-logical-replication](https://postgrespro.ru/docs/postgresql/16/protocol-logical-replication).

---

## Слайд 7 — Логическая репликация (демо, часть 1)

Два **независимых** сервера (base backup **без `-R`**):

```bash
pg_basebackup --pgdata=~/tmp/backup --checkpoint=fast
# → /var/lib/postgresql/16/replica, start
```

```sql
-- publisher :5432
CREATE DATABASE replica_overview_logical_dba;
CREATE TABLE test(id int PRIMARY KEY, descr text);
INSERT INTO test VALUES (1,'Раз'),(2,'Два');

ALTER SYSTEM SET wal_level = logical;  -- restart main
CREATE PUBLICATION test_pub FOR TABLE test;
\dRp+
```

Без `-R` — не standby, а fork. Иначе logical поверх physical recovery не то, что нужно для демо.

---

## Слайд 8 — Логическая репликация (демо, часть 2)

```sql
-- subscriber :5433
CREATE SUBSCRIPTION test_sub
  CONNECTION 'port=5432 user=student dbname=replica_overview_logical_dba'
  PUBLICATION test_pub;
-- NOTICE: created replication slot "test_sub" on publisher
\dRs
```

```sql
-- publisher
INSERT INTO test VALUES (3,'Три');

SELECT * FROM pg_stat_subscription \gx
-- received_lsn, latest_end_time, pid → apply worker
```

Процесс: `logical replication apply worker for subscription …`.
`pid` пустой — worker умер / конфликт / подписка остановлена.

---

## Слайд 9 — Конфликты

**Replica identity** — как найти строку для UPDATE/DELETE:

| режим | |
|-------|--|
| **DEFAULT** | PK |
| USING INDEX | unique NOT NULL |
| FULL | все столбцы |
| NOTHING | только INSERT (system catalog default) |

Конфликт = нарушение **UNIQUE/FK** на subscriber → replication **останавливается**
до ручного fix. Авто-разрешения **нет**.

Доки: [conflicts](https://postgrespro.ru/docs/postgresql/16/logical-replication-conflicts),
[ALTER TABLE](https://postgrespro.ru/docs/postgresql/16/sql-altertable).

---

## Слайд 10 — Конфликты (демо)

```sql
-- subscriber
INSERT INTO test VALUES (4, 'Четыре (локально)');

-- publisher
INSERT INTO test VALUES (4, 'Четыре'), (5, 'Пять');
-- subscriber: pk=4 конфликт, pg_stat_subscription.pid пустой

DELETE FROM test WHERE id = 4;  -- на subscriber
-- репликация ожила, строки 4 и 5 доехали
```

Удаление подписки:

```sql
DROP SUBSCRIPTION test_sub;
-- NOTICE: dropped replication slot on publisher
```

Иначе **слот** на publisher → WAL не чистится → диск кончится.
Это частый прод-инцидент после «временно потестили logical».

---

## Слайд 11 — Ограничения

**Не реплицируется:**

- DDL;  
- **sequences** (`nextval`);  
- large objects;  
- views, matviews, foreign tables.

**Не умеет:** auto conflict resolution.

Sequences: разнести **диапазоны** или UUID. Иначе PK-коллизии при локальной insert на subscriber.

Реплицируются обычные и секционированные таблицы.

Доки: [restrictions](https://postgrespro.ru/docs/postgresql/16/logical-replication-restrictions).

---

## Слайд 12 — Особенности

- **TRUNCATE** + FK: в publication должны быть **все** связанные таблицы;  
- **неактивный слот** → WAL копится, vacuum horizon страдает;  
- массовый UPDATE/DELETE на publisher → **много** row-сообщений на subscriber;  
- долгие транзакции на publisher → изменения видны только после commit;
  параметр подписки **`streaming`** помогает (накапливать / применять раньше);  
- соединение publisher↔subscriber должно быть **стабильным**.

---

## Слайд 13 — Консолидация данных

Сценарий: региональные филиалы → **центральный** хаб.

- на филиалах **publication**;  
- на центре **subscription** на каждый филиал (по worker на подписку);  
- триггеры на центре для нормализации.

Обратно: справочники с центра на регионы.

Иногда проще **batch ETL**, чем logical 24/7 — бизнес решает.

---

## Слайд 14–15 — Обновление серверов

Задача: **major upgrade** без даунтайма. Физическая реплика **не поможет** (разная major).

Схема:

1. новый сервер (напр. 16.2);  
2. logical sync ← старый (13.6), initial sync;  
3. cutover приложения (внешние средства);  
4. старый off.

Реально процесс **сложный** (sequences, extensions, types) — DBA2 «Обновление сервера».
Logical — инструмент, не кнопка «upgrade без боли».

---

## Слайд 16 — Master-master

**Bidirectional** (PG 16+): publish + subscribe на **одних таблицах** на обоих узлах.

Требования приложения:

- разные **key ranges** или **UUID**;  
- нет **global distributed transactions** в PG;  
- конфликты — **вручную**.

Нет встроенного auto-failover / add-node для multi-master — только внешние оркестраторы.

---

## Слайд 17 — Итоги

- логическая реплика = **строки таблиц**, pub/sub;  
- **разнонаправленность** возможна, бинарная совместимость не обязательна;  
- **`wal_level=logical`**, слоты, replica identity;  
- DDL, sequences, auto-conflict — **слабые места** — учитывайте при дизайне.

---

## Слайд 18 — Практика

**1. Репликация на одном сервере** (две БД):

`CREATE SUBSCRIPTION` **зависает** — ждёт конца своей транзакции при auto-create slot
(слот создаётся только после завершения всех активных транзакций, включая текущую).

Fix:

```sql
SELECT pg_create_logical_replication_slot('test_slot', 'pgoutput');
CREATE SUBSCRIPTION test
  CONNECTION 'user=student dbname=replica_overview_logical_dba'
  PUBLICATION test
  WITH (slot_name = test_slot, create_slot = false);
```

Доки: [CREATE SUBSCRIPTION](https://postgrespro.ru/docs/postgresql/16/sql-createsubscription).

**2. Двунаправленная** (два сервера):

```sql
CREATE SUBSCRIPTION test
  CONNECTION 'port=5433 ...' PUBLICATION test
  WITH (copy_data = false, origin = none);
-- и зеркально на втором
```

- `copy_data = false` — данные уже есть (клон);  
- `origin = none` — не реплицировать изменения, пришедшие по другой подписке (**anti-loop**).

UPDATE без PK:

```text
ERROR: cannot update table "test" because it does not have a replica identity
```

```sql
ALTER TABLE test ADD PRIMARY KEY (id);  -- на обоих
```

Потом `UPDATE … SET id = id + 100` на обоих — диапазоны уезжают согласованно.

---

## Финал

1. physical vs logical — файл vs строка;  
2. `wal_level=logical` + **restart**;  
3. слоты и `DROP SUBSCRIPTION` (не оставляйте сирот);  
4. конфликты, sequences, `origin=none` для bi-di.

Logical — гибкий трубопровод данных. Физический — фундамент HA. В проде часто **оба**.
