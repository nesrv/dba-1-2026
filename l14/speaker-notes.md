# Текст лектора — l14

## Резервное копирование: обзор

> Два мира: SQL-дампы vs копия файлов + WAL.
> Логика — гибко и кросс-версионно. Физика — быстро и до точки во времени.

---

## Слайд 1 — Титул

Тема: **обзор резервного копирования** в PostgreSQL.

Факт: «бэкап» без плана восстановления — это просто архив на диске. Сегодня — оба семейства инструментов и как они стыкуются с WAL.

---

## Слайд 2 — Темы

Два больших блока:

1. **Логическое** резервное копирование  
2. **Физическое** + архив WAL

Потом практика и практика+ — там pg_dump, pg_basebackup и PITR «на коленке».

---

## Слайд 3 — Логическое копирование

План секции:

- что такое логическая копия  
- копия **таблицы** (`COPY`)  
- копия **базы** (`pg_dump`)  
- копия **кластера** (`pg_dumpall`)

---

## Слайд 4 — Логическая копия

Суть: текстовый набор **SQL-комmand**, который поднимает объект с нуля.

| плюс | минус |
|------|-------|
| отдельный объект / база / кластер | медленно на больших объёмах |
| другая **major**-версия PG | только момент дампа, не PITR |
| другая архитектура (x86 ↔ arm) | индексы пересоздаются заново |
| можно править дамп руками | |

Файл — обычный текст: вырезать таблицу, переименовать схему, мигрировать выборочно.

Доки: https://postgrespro.ru/docs/postgresql/16/backup-dump

---

## Слайд 5 — COPY: копия таблицы

**COPY** — быстрый обмен **строками** (не DDL).

| | серверный `COPY` | клиентский `\copy` |
|--|------------------|---------------------|
| где | SQL на сервере | meta-команда psql |
| файл | на **сервере**, доступ у `postgres` | на **клиенте** |
| скорость | >> пачки `INSERT` | то же по сути |

Форматы: text, csv, binary. NULL в text — `\N`, пустая строка — отдельное значение.

Доки: https://postgrespro.ru/docs/postgresql/16/sql-copy

---

## Слайд 6 — COPY (демо)

```sql
CREATE DATABASE backup_overview;
\c backup_overview
CREATE TABLE t(id numeric, s text);
INSERT INTO t VALUES (1, 'Привет!'), (2, ''), (3, NULL);

COPY t TO STDOUT;
-- 1	Привет!
-- 2
-- 3	\N

TRUNCATE t;
COPY t FROM STDIN;
-- ввод строк, завершить \.
```

Лайфхак: `\pset null '<null>'` — и NULL в SELECT видно явно.

Пустая строка и NULL в psql без `\pset` выглядят одинаково — на экзамене и в проде это классическая ловушка.

---

## Слайд 7 — pg_dump: копия базы

**pg_dump** — полноценная копия **одной БД**.

| формат | восстановление |
|--------|----------------|
| plain (SQL) | `psql -f dump.sql` |
| custom/directory/tar | **pg_restore** (выбор объектов, `-j` параллель) |

Фишки: `-t`, `-n`, `--data-only`, `--schema-only`, `-j` при custom/directory.

**Восстановление:**

- новая БД из **`template0`** (не template1 — туда могли напихать объектов)  
- **роли и tablespace** создать заранее (они кластерные)  
- после restore — **`ANALYZE`**

Доки: https://postgrespro.ru/docs/postgresql/16/app-pgdump  
https://postgrespro.ru/docs/postgresql/16/app-pgrestore

---

## Слайд 8 — pg_dumpall: копия кластера

**pg_dumpall** — весь кластер: все БД + **globals** (роли, tablespaces).

- только plain SQL → только `psql`  
- **без параллелизма** (внутри — pg_dump по очереди)  
- запускать от **суперпользователя**

На больших объёмах:

```bash
pg_dumpall --globals-only > globals.sql
pg_dump -j 4 -Fc -f db.dump mydb   # по базам отдельно
```

Доки: https://postgrespro.ru/docs/postgresql/16/app-pg-dumpall

---

## Слайд 9 — Утилита pg_dump (демо)

```bash
pg_dump -d backup_overview --create
```

В дампе:

- `CREATE DATABASE ... TEMPLATE = template0`  
- DDL таблицы  
- данные через **`COPY ... FROM stdin`** (не INSERT — быстрее)

**`\restrict` / `\unrestrict`** (PG 16+): psql блокирует meta-команды `\...` пока идёт restore — чтобы `\` в данных не устроил сюрприз.

---

## Слайд 10 — pg_dump: pipe в другую базу

Копия одной таблицы между базами:

```bash
pg_dump -d backup_overview --table=t | psql -d backup_overview2
```

Unix-pipe — нормальный паттерн для «перекинуть объект без файла».

Факт: plain-дамп + psql = самый прозрачный путь; custom + pg_restore = когда нужен `-j` и cherry-pick объектов.

---

## Слайд 11 — Физическое копирование

План секции:

- физическая копия  
- холодная / горячая  
- протокол репликации  
- автономные копии  
- непрерывная архивация WAL

---

## Слайд 12 — Физическая копия

Механизм = **crash recovery**: файлы кластера + нужный WAL.

| плюс | минус |
|------|-------|
| быстрое восстановление | только **весь кластер** |
| PITR с архивом WAL | та же major + архитектура |
| горячая копия без stop | |

Согласованный снимок при аккуратном shutdown — WAL не нужен. Горячий снимок — **несогласованный**, recovery доведёт.

Доки: https://postgrespro.ru/docs/postgresql/16/backup-file  
https://postgrespro.ru/docs/postgresql/16/continuous-archiving

---

## Слайд 13 — Горячо или холодно?

| | холодный (stop) | «грязный» stop / snapshot ОС | горячий (работает) |
|--|-----------------|------------------------------|---------------------|
| файлы | согласованы или + WAL | нужен WAL с checkpoint | несогласованы |
| WAL | часто не нужен | с последней CP | за время копирования |
| инструмент | `tar`, `cp` | snapshot + WAL | **pg_basebackup** |

«Просто скопировать `$PGDATA` на работающем сервере» — **нельзя**. Нужен checkpoint + WAL или pg_basebackup.

---

## Слайд 14 — Автономная копия

**pg_basebackup** = горячая **автономная** копия (base + WAL за время бэкапа).

Алгоритм:

1. подключение по протоколу репликации  
2. **checkpoint**  
3. копирование файлов кластера  
4. WAL с момента CP до конца копирования  

Restore: развернуть каталог → `pg_ctl start` → recovery → готово.

Доки: https://postgrespro.ru/docs/postgresql/16/app-pgbasebackup

---

## Слайд 15 — Протокол репликации

Не только для реплик — **бэкапы тоже через него**.

| компонент | роль |
|-----------|------|
| **wal_sender** | отдаёт поток WAL / команды бэкапа |
| **replication slot** | «не удаляй WAL, пока клиент не прочитал» |
| `wal_level = replica` | достаточно для physical backup/stream |
| `max_wal_senders` | лимит одновременных wal_sender |

Доступ: роль с **REPLICATION**, строка `replication` в **pg_hba.conf**.

Доки: https://postgrespro.ru/docs/postgresql/16/protocol-replication

---

## Слайд 16 — Автономная копия (схема)

Картинка: мастер пишет WAL, **pg_basebackup** забирает base + сегменты.

WAL на мастере **переиспользуется** (старые сегменты удаляются) — слот не даёт удалить то, что ещё не доехало до бэкапа/реплики.

---

## Слайд 17 — Автономная резервная копия (демо)

Проверки:

```sql
SELECT name, setting FROM pg_settings
WHERE name IN ('wal_level','max_wal_senders');

SELECT type, database, user_name, address, auth_method
FROM pg_hba_file_rules()
WHERE 'replication' = ANY(database);
```

```bash
pg_lsclusters   # main :5432, replica :5433 down
rm -rf ~/tmp/basebackup
pg_basebackup --pgdata=~/tmp/basebackup --checkpoint=fast
```

**`--checkpoint=fast`** — сброс dirty buffers без пауз (до ~4.5 мин spread по умолчанию). На демо — чтобы не ждать.

---

## Слайд 18 — Восстановление (схема)

Base backup разворачивается на **другом** сервере → recovery → **независимый** инстанс на момент конца бэкапа.

Мастер ушёл вперёд — это нормально. Это не реплика, а **fork** состояния.

---

## Слайд 19 — Восстановление (демо)

```bash
sudo pg_ctlcluster 16 replica status   # down
sudo rm -rf /var/lib/postgresql/16/replica
sudo mv ~/tmp/basebackup /var/lib/postgresql/16/replica
sudo chown -R postgres:postgres /var/lib/postgresql/16/replica
sudo pg_ctlcluster 16 replica start
```

В каталоге: `backup_label`, `backup_manifest`, `pg_wal/` с WAL за бэкап.

Два сервера **независимы**: INSERT на main не виден на replica и наоборот.

---

## Слайд 20 — Архив журналов

Идея: base backup + **непрерывный архив WAL** → восстановление на **любой момент**.

| файловый архив | потоковый архив |
|----------------|-----------------|
| `archive_command` при **смене** сегмента | **pg_receivewal** по replication |
| задержка до fill сегмента | почти realtime |
| всё внутри PG | отдельный процесс/сервис ОС |

---

## Слайд 21 — Файловый архив журналов

```ini
archive_mode = on
archive_command = 'cp %p /archive/%f'   # пример; %p=path, %f=filename
```

Процесс **archiver**:

- сегмент заполнился → shell-команда  
- exit 0 → сегмент можно удалить с мастера  
- не 0 → retry, WAL копится  

Доки: https://postgrespro.ru/docs/postgresql/16/continuous-archiving

---

## Слайд 22 — Файловый архив (схема)

Мастер → archiver → **отдельное хранилище**. Там же лежат periodic base backups.

Архив обычно **не на том же диске**, что PGDATA — иначе смысл теряется при смерти сервера.

---

## Слайд 23 — Потоковый архив журналов

**pg_receivewal**:

- replication + **слот** (обязательно в проде)  
- пишет сегменты как на мастере; неполный — **`.partial`**  
- старт: после последнего полного в каталоге, или текущий сегмент если пусто  
- **не демонизируется** — systemd/supervisor ваш друг

Доки: https://postgrespro.ru/docs/postgresql/16/app-pgreceivewal

---

## Слайд 24 — Потоковый архив (схема)

**wal_sender** на мастере ↔ **pg_receivewal** на архивном хосте.

Учитывайте слот в **`max_wal_senders`** — каждый receivewal + каждая реплика = sender.

---

## Слайд 25 — Базовая копия + архив

При работающем архиве base backup **без WAL внутри**:

```bash
pg_basebackup --wal-method=none ...
```

Restore:

1. развернуть base  
2. **`restore_command`** (обратная archive_command)  
3. целевая точка (**recovery target**)  
4. файл **`recovery.signal`**  
5. start → managed recovery → promote или stop on target  

---

## Слайд 26 — Восстановление из архива (схема)

**restore_command** тянет `%f` из архива в `pg_wal/`.

Подводный камень: **текущий незаполненный** сегмент на упавшем мастере в файловый архив **не попал**. Иногда его можно **руками** докинуть.

---

## Слайд 27 — Целевая точка восстановления

По умолчанию — **все доступные WAL**. Recovery target — остановиться в нужный **timestamp/LSN/name**.

Максимальная потеря ≈ один неархивированный partial-сегмент (если не спасли руками).

---

## Слайд 28 — После восстановления

Recovery закончился → сервер **обычный primary**: пишет WAL, снова архивирует.

Failover-кандон: новый primary должен быть **не слабее** старого, иначе «мы восстановились и умерли от нагрузки».

---

## Слайд 29 — Итоги

| логика | физика |
|--------|--------|
| COPY, pg_dump, pg_dumpall | pg_basebackup |
| SQL, гибкость | файлы + WAL |
| момент дампа | PITR с архивом |

Три утилиты на память: **pg_dump**, **pg_basebackup**, **pg_receivewal** (или archive_command).

---

## Слайд 30 — Практика / Практика+

**Практика:**

1. БД + таблица  
2. `pg_dump -f ... --create` → DROP → `psql -f`  
3. `pg_basebackup` → изменить main → restore на **replica :5433** → старые данные  

**Практика+** (потоковый архив + PITR):

```bash
mkdir /var/lib/postgresql/archive
pg_receivewal --create-slot --slot=archive
pg_receivewal -D /var/lib/postgresql/archive --slot=archive &

pg_basebackup --wal-method=none --pgdata=~/tmp/backup --checkpoint=fast
# ... данные на main ...

# restore на replica:
echo "restore_command = 'cp /var/lib/postgresql/archive/%f %p || cp /var/lib/postgresql/archive/%f.partial %p'" \
  | sudo tee .../postgresql.auto.conf
touch .../recovery.signal
```

`.partial` в restore_command — must have для streaming archive.

Уборка: `pkill pg_receivewal`, drop slot, stop replica.

---

## Финал

На выход:

1. логика vs физика — когда что;  
2. template0, globals, ANALYZE после pg_restore;  
3. автономный base backup vs base + архив;  
4. `\restrict`, `.partial`, recovery.signal.

Бэкап, который вы ни разу не восстанавливали, — вера, не инженерия.
