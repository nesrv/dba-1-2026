# Текст лектора — l14

## Резервное копирование: обзор

> По PDF `dba1_14_backup_overview`. Чуть больше исходника.
> Два мира: SQL-дампы vs файлы + WAL. Бэкап без плана restore — просто архив.

---

## Слайд 1 — Титул

Тема: **обзор резервного копирования** в PostgreSQL.

Сегодня — оба семейства: логика (`COPY` / `pg_dump` / `pg_dumpall`) и физика
(`pg_basebackup`, архив WAL, PITR). Дальше на курсе детали разворачиваются в DBA3.

---

## Слайд 2 — Темы

1. **Логическое** резервное копирование  
2. **Физическое** + непрерывная архивация WAL  

Практика: dump/restore, basebackup, потоковый архив + PITR.

---

## Слайд 3 — Логическое копирование

План блока:

- что такое логическая копия  
- копия **таблицы** (`COPY`)  
- копия **базы** (`pg_dump`)  
- копия **кластера** (`pg_dumpall`)

---

## Слайд 4 — Логическая копия

Суть: набор **SQL-команд**, который поднимает объект / БД / кластер с нуля.
По сути текстовый файл — можно вырезать таблицу, переименовать схему, поправить типы.

| Плюс | Минус |
|------|--------|
| отдельный объект / база / кластер | медленно на больших объёмах |
| другая **major**-версия PG | только момент дампа, не PITR |
| другая архитектура (x86 ↔ arm) | индексы пересоздаются заново |

Двоичная совместимость не нужна — нужна совместимость **команд**.

Доки: [backup-dump](https://postgrespro.ru/docs/postgresql/16/backup-dump).

---

## Слайд 5 — COPY: копия таблицы

**COPY** — быстрый обмен **строками** (не DDL). Быстрее пачки `INSERT`: меньше round-trip и разбора.

| | серверный `COPY` | клиентский `\copy` |
|--|------------------|---------------------|
| где | SQL на сервере | meta-команда psql |
| файл | на **сервере**, доступ у `postgres` | на **клиенте** |

Форматы: text, csv, binary. Параметры: разделитель, представление NULL и т. д.  
`COPY … FROM` **добавляет** строки — таблицу не очищает (нужен `TRUNCATE` отдельно).

Доки: [sql-copy](https://postgrespro.ru/docs/postgresql/16/sql-copy), [app-psql](https://postgrespro.ru/docs/postgresql/16/app-psql).

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
-- ввод, завершить \.
\pset null '<null>'
SELECT * FROM t;
```

Факт: пустая строка и NULL в обычном SELECT выглядят одинаково — в COPY NULL = `\N`.
Классическая ловушка на экзамене и в проде.

---

## Слайд 7 — pg_dump: копия базы

**pg_dump** — полноценная копия **одной БД**.

| формат | восстановление |
|--------|----------------|
| plain (SQL) | `psql -f` |
| custom / directory / tar | **pg_restore** (выбор объектов, `-j`) |

Фишки: `-t`, `-n`, `--data-only`, `--schema-only`, параллель при custom/directory.

**Восстановление:**

- новая БД из **`template0`** (не template1 — туда могли напихать объектов, они попадут в dump);  
- **роли и tablespace** создать заранее (они кластерные);  
- после restore — **`ANALYZE`**.

Доки: [pg_dump](https://postgrespro.ru/docs/postgresql/16/app-pgdump), [pg_restore](https://postgrespro.ru/docs/postgresql/16/app-pgrestore).

---

## Слайд 8 — pg_dumpall: копия кластера

**pg_dumpall** — весь кластер: все БД + **globals** (роли, tablespaces).

- только plain SQL → только `psql`;  
- **без параллелизма** (внутри — pg_dump по очереди);  
- запускать от **суперпользователя**.

На больших объёмах:

```bash
pg_dumpall --globals-only > globals.sql
pg_dump -j 4 -Fc -f db.dump mydb
```

Доки: [pg_dumpall](https://postgrespro.ru/docs/postgresql/16/app-pg-dumpall).

---

## Слайд 9 — Утилита pg_dump (демо)

```bash
pg_dump -d backup_overview --create
```

В дампе:

- `CREATE DATABASE … TEMPLATE = template0` (ключ `--create`);  
- DDL таблицы;  
- данные через **`COPY … FROM stdin`** (не INSERT — быстрее).

**`\restrict` / `\unrestrict`** (PG 16+): psql блокирует meta-команды `\…`, пока идёт restore —
чтобы `\` в данных не исполнил что-то «весёлое». Безопасный режим с одноразовым токеном.

---

## Слайд 10 — pg_dump: pipe в другую базу

```bash
pg_dump -d backup_overview --table=t | psql -d backup_overview2
```

Unix-pipe — нормальный паттерн «перекинуть объект без файла на диске».

Факт: plain + psql = прозрачно; custom + pg_restore = когда нужны `-j` и cherry-pick объектов.

---

## Слайд 11 — Физическое копирование

План блока:

- что такое физическая копия  
- холодная / горячая  
- протокол репликации  
- автономные копии  
- непрерывная архивация WAL

---

## Слайд 12 — Физическая копия

Механизм = **crash recovery**: файлы кластера + нужный WAL.

| Плюс | Минус |
|------|--------|
| быстрое восстановление | только **весь кластер** |
| PITR с архивом WAL | та же major + архитектура |
| горячая копия без stop | |

Холодный аккуратный shutdown → файлы согласованы, WAL часто не нужен.  
Горячий снимок → несогласованный; recovery доведёт. Архив WAL → состояние **на любой момент**.

Доки: [backup-file](https://postgrespro.ru/docs/postgresql/16/backup-file),
[continuous-archiving](https://postgrespro.ru/docs/postgresql/16/continuous-archiving).

---

## Слайд 13 — Горячо или холодно?

| | холодный (stop) | «грязный» stop / snapshot ОС | горячий |
|--|-----------------|------------------------------|---------|
| файлы | согласованы или + WAL | нужен WAL с checkpoint | несогласованы |
| WAL | часто не нужен | с последней CP | за время копирования |
| инструмент | `tar`, `cp` | snapshot + WAL | **pg_basebackup** |

«Просто скопировать `$PGDATA` на работающем сервере» — **нельзя**.
Нужен checkpoint + WAL или штатный `pg_basebackup`.

---

## Слайд 14 — Автономная копия

**pg_basebackup** = горячая **автономная** копия (base + WAL за время бэкапа).

Алгоритм:

1. подключение по протоколу репликации  
2. **checkpoint**  
3. копирование файлов кластера  
4. WAL с момента CP до конца копирования  

Restore: развернуть каталог → start → recovery → готово. Всё «в коробке» копии.

Доки: [pg_basebackup](https://postgrespro.ru/docs/postgresql/16/app-pgbasebackup).

---

## Слайд 15 — Протокол репликации

Не только для реплик — **бэкапы тоже через него**.

| компонент | роль |
|-----------|------|
| **wal_sender** | поток WAL / команды бэкапа |
| **replication slot** | «не удаляй WAL, пока клиент не прочитал» |
| `wal_level = replica` | достаточно для physical backup/stream |
| `max_wal_senders` | лимит одновременных wal_sender |

Доступ: роль с **REPLICATION** + строка `replication` в **pg_hba.conf**.  
Дефолты (PG 10+) уже позволяют локальный бэкап.

Доки: [protocol-replication](https://postgrespro.ru/docs/postgresql/16/protocol-replication).

---

## Слайд 16 — Автономная копия (схема)

Мастер пишет WAL, сегменты циклически удаляются.  
**pg_basebackup** забирает base + сегменты за время копирования — обычно на другой хост.

Слот не даёт мастеру выкинуть WAL, который ещё не доехал до клиента.

---

## Слайд 17 — Автономная резервная копия (демо)

```sql
SELECT name, setting FROM pg_settings
WHERE name IN ('wal_level','max_wal_senders');

SELECT type, database, user_name, address, auth_method
FROM pg_hba_file_rules()
WHERE 'replication' = ANY(database);
```

```bash
pg_lsclusters   # main :5432 online, replica :5433 down
rm -rf ~/tmp/basebackup
pg_basebackup --pgdata=~/tmp/basebackup --checkpoint=fast
```

**`--checkpoint=fast`**: dirty buffers пишутся без пауз (иначе spread до ~4.5 мин).
На демо — чтобы не ждать; в проде — осознанный пик IO.

---

## Слайд 18 — Восстановление (схема)

Base backup разворачивается на **другом** сервере → recovery → **независимый** инстанс
на момент конца бэкапа. Мастер ушёл вперёд — нормально. Это **fork**, не реплика.

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
Покажите обе стороны — иначе «реплика» в голове слушателей путается с restore.

---

## Слайд 20 — Архив журналов

Идея: base backup + **непрерывный архив WAL** → восстановление на **любой момент**.

| файловый архив | потоковый архив |
|----------------|-----------------|
| `archive_command` при **смене** сегмента | **pg_receivewal** по replication |
| задержка до fill сегмента | почти realtime |
| всё внутри PG | отдельный процесс ОС |

---

## Слайд 21 — Файловый архив журналов

```ini
archive_mode = on
archive_command = 'cp %p /archive/%f'   # пример; %p=path, %f=filename
```

Процесс **archiver**:

- сегмент заполнился → shell-команда;  
- exit 0 → сегмент можно удалить;  
- не 0 → retry, WAL копится на мастере.

Доки: [continuous-archiving](https://postgrespro.ru/docs/postgresql/16/continuous-archiving).

---

## Слайд 22 — Файловый архив (схема)

Мастер → archiver → **отдельное хранилище** (+ periodic base backups).

Архив **не на том же диске**, что PGDATA — иначе смерть диска убивает и данные, и бэкап.

---

## Слайд 23 — Потоковый архив журналов

**pg_receivewal**:

- replication + **слот** (в проде обязательно);  
- пишет сегменты как сервер; неполный — **`.partial`**;  
- старт: после последнего полного в каталоге, или текущий сегмент если пусто;  
- **не демонизируется** — systemd/supervisor ваш друг;  
- смена primary → перезапуск с новыми параметрами.

Доки: [pg_receivewal](https://postgrespro.ru/docs/postgresql/16/app-pgreceivewal).

---

## Слайд 24 — Потоковый архив (схема)

**wal_sender** на мастере ↔ **pg_receivewal** на архивном хосте.

Учитывайте слот в **`max_wal_senders`**: каждый receivewal + каждая реплика = sender.
Пишет **сразу**, не ждёт конца сегмента.

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
4. файл **`recovery.signal`** (содержимое игнорируется)  
5. start → managed recovery  

---

## Слайд 26 — Восстановление из архива (схема)

**restore_command** тянет `%f` из архива в `pg_wal/`.

Подводный камень файлового архива: **текущий незаполненный** сегмент на упавшем
мастере в архив **не попал**. Иногда его можно **руками** докинуть в `pg_wal` standby.
При сбое archive_command таких сегментов может быть несколько.

---

## Слайд 27 — Целевая точка восстановления

По умолчанию — **все доступные WAL**. Recovery target — остановиться на timestamp / LSN / name.

Максимальная потеря ≈ один неархивированный partial-сегмент (если не спасли руками).

---

## Слайд 28 — После восстановления

Recovery закончился → сервер **обычный primary**: пишет WAL, снова архивирует.

Failover-канон: новый primary должен быть **не слабее** старого по железу —
иначе «восстановились и умерли от нагрузки».

---

## Слайд 29 — Итоги

| логика | физика |
|--------|--------|
| COPY, pg_dump, pg_dumpall | pg_basebackup |
| SQL, гибкость, кросс-версия | файлы + WAL, та же major |
| момент дампа | PITR с архивом |

Три утилиты: **pg_dump**, **pg_basebackup**, **pg_receivewal** (или `archive_command`).

---

## Слайд 30 — Практика / Практика+

**Практика:**

1. БД + таблица  
2. `pg_dump -f … --create` → DROP → `psql -f`  
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

1. логика vs физика — когда что;  
2. `template0`, globals, `ANALYZE` после restore;  
3. автономный base vs base + архив;  
4. `\restrict`, `.partial`, `recovery.signal`.

Бэкап, который ни разу не восстанавливали, — вера, не инженерия.
