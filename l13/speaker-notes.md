# Текст лектора — l13

## Управление доступом: обзор

> По PDF `dba1_13_access_overview`. Чуть больше исходника: конкретика и факты.
> Голос — старший разработчик: роли ≠ юзеры ОС, `PUBLIC` ≠ схема `public`.

---

## Слайд 1 — Титул

Тема: **управление доступом**. Обзор ролей, подключения и привилегий.

До сих пор на курсе жили суперпользователями — разграничение «не мешало».
Сейчас включаем три слоя: **кто** (роль) → **пускают ли** (`pg_hba`) → **что можно** (GRANT).

---

## Слайд 2 — Темы

1. Роли и атрибуты  
2. Подключение к серверу  
3. Парольная аутентификация  
4. Привилегии и управление ими  
5. Категории ролей  
6. Групповые и предопределённые роли  
7. Привилегии по умолчанию  
8. Привилегии и подпрограммы  

Практика в конце — writer/reader и peer + `pg_ident`.

---

## Слайд 3 — Роли и атрибуты

Роль в Postgres — две роли сразу (каламбур уместный):

1. **пользователь СУБД** — подключается клиентом;  
2. **группа** — в неё включают другие роли (удобно для ACL).

Формально роль **не связана** с пользователем ОС. Но `psql` от `student` по умолчанию
лезет в роль `student` — отсюда путаница «я же student в Linux…».

При `initdb` создаётся одна начальная роль-суперпользователь (обычно `postgres`).
Дальше — сами.

Атрибуты задают общие свойства (не привязанные к конкретному объекту). Часто пара:
`CREATEDB` / `NOCREATEDB`, `LOGIN` / `NOLOGIN` и т. д.

| Атрибут | Смысл |
|---------|--------|
| `LOGIN` | можно подключаться (= «пользователь») |
| `SUPERUSER` | проверки ACL не выполняются |
| `CREATEDB` | создавать базы |
| `CREATEROLE` | создавать роли |
| … | и другие (`REPLICATION`, `BYPASSRLS`, `INHERIT`…) |

`NOLOGIN` — типичная «групповая» роль: сама не коннектится, в неё включают логины.

Доки: [database-roles](https://postgrespro.ru/docs/postgresql/16/database-roles),
[role-attributes](https://postgrespro.ru/docs/postgresql/16/role-attributes).

---

## Слайд 4 — Роли и атрибуты (демо)

Имя роли вынесено в prompt — важно видеть, *от кого* идут команды.

```sql
CREATE ROLE alice LOGIN PASSWORD 'alice';
\du
CREATE DATABASE access_overview;
\c access_overview
```

В `\du` у `student` и `postgres` — Superuser, Create role, Create DB, Replication, Bypass RLS.
Поэтому раньше о GRANT не думали: суперпользователь **не проверяет** привилегии.

Факт: `CREATE USER` = сахар над `CREATE ROLE … LOGIN`. Отдельной сущности «user» нет.

---

## Слайд 5 — Подключение к серверу

Файл **`pg_hba.conf`** (host-based authentication). Изменения — после reload
(`pg_reload_conf()` / reload утилиты управления), как у `postgresql.conf`.

Алгоритм:

1. строки смотрятся **сверху вниз**;  
2. берётся **первая** подходящая (тип, БД, пользователь, адрес).

```text
# TYPE  DATABASE  USER  ADDRESS         METHOD
local   all       postgres               peer
local   all       all                    peer
host    all       all   127.0.0.1/32     scram-sha-256
host    all       all   ::1/128          scram-sha-256
```

| Поле | Варианты |
|------|----------|
| TYPE | `local` = Unix-сокет; `host` = TCP/IP |
| DATABASE | `all` или имя БД |
| USER | `all` или имя роли |
| ADDRESS | для host: IP/маска, домен, `all`; для local — нет |

`listen_addresses` — *где слушаем*; `pg_hba` — *кого пускаем*.
Часто ставят `listen_addresses='*'` и дальше режут доступ в pg_hba.

Доки: [client-authentication](https://postgrespro.ru/docs/postgresql/16/client-authentication).

---

## Слайд 6 — Подключение: аутентификация

3. По найденной строке — аутентификация указанным **METHOD** + проверка `LOGIN` и `CONNECT`.  
4. Успех → пускаем; иначе → **запрет** (остальные строки уже не смотрят).  
   Нет ни одной строки → тоже запрет.

Поэтому записи идут **от частных к общим**. Общий `trust` сверху съест всё ниже.

| METHOD | Смысл |
|--------|--------|
| `trust` | пустить без вопросов |
| `reject` | всегда отказать |
| `scram-sha-256` | пароль (актуальный) |
| `md5` | пароль, устаревший |
| `peer` | имя ОС == имя роли (можно map) |

Факт: `-h localhost` (TCP) и без `-h` (socket) — **разные** строки pg_hba.
Классика: «через socket пускает, через TCP — нет».

Доки: [auth-methods](https://postgrespro.ru/docs/postgresql/16/auth-methods).

---

## Слайд 7 — Парольная аутентификация

На сервере:

- пароль при `CREATE ROLE` / `ALTER ROLE … PASSWORD`;  
- без пароля при парольном методе — отказ;  
- хеш в **`pg_authid`**.

На клиенте:

| Способ | Заметка |
|--------|---------|
| вручную | интерактивный prompt |
| `PGPASSWORD` | удобно, но небезопасно (видно в env/`/proc`) |
| `~/.pgpass` | `хост:порт:база:роль:пароль`; права **600**, иначе libpq **игнорирует** файл |

Несколько баз/хостов — `.pgpass` обычно удобнее, чем `PGPASSWORD`.

---

## Слайд 8 — Подключение (демо)

Нужны и `LOGIN`, и подходящая строка в pg_hba.

```sql
SHOW hba_file;
-- часто /etc/postgresql/16/main/pg_hba.conf (Ubuntu), не обязательно внутри PGDATA

SELECT type, database, user_name, address, auth_method
FROM pg_hba_file_rules();
```

Подключение alice по TCP (localhost → обычно scram):

```sql
ALTER ROLE alice PASSWORD 'alicepass';
```

```bash
psql 'host=localhost user=alice dbname=access_overview password=alicepass'
\conninfo
```

Смотрите в `\conninfo`: host vs socket, SSL — сразу видно, какой путь сработал.

---

## Слайд 9 — Привилегии (таблицы)

Привилегии связывают **роли** и **объекты**: что можно делать.

Для таблиц / представлений:

| Priv | Действие |
|------|----------|
| `SELECT` | чтение |
| `INSERT` | вставка |
| `UPDATE` | изменение |
| `DELETE` | удаление строк |
| `TRUNCATE` | опустошение (отдельная priv!) |
| `REFERENCES` | быть целью FK |
| `TRIGGER` | создавать триггеры |

Часть — **на уровне столбцов** (`GRANT SELECT (n) ON t TO bob`).

Доки: [ddl-priv](https://postgrespro.ru/docs/postgresql/16/ddl-priv), [GRANT](https://postgrespro.ru/docs/postgresql/16/sql-grant).

---

## Слайд 10 — Привилегии (другие объекты)

| Объект | Привилегии | Смысл |
|--------|------------|--------|
| **DATABASE** | `CONNECT`, `CREATE`, `TEMPORARY` | подключение; создание схем; временные таблицы |
| **SCHEMA** | `USAGE`, `CREATE` | ходить к объектам; создавать объекты в схеме |
| **TABLESPACE** | `CREATE` | создавать объекты в ТП |
| **SEQUENCE** | `SELECT`, `UPDATE`, `USAGE` | `currval` / `nextval` / `setval` (разные сочетания) |

Имя схемы для temp заранее неизвестно (`pg_temp_NNN`) — поэтому `TEMPORARY` на уровне **БД**.

Мантра: **USAGE схемы**, потом priv на таблицу. Без USAGE часто «permission denied for schema».

С PG15 у `PUBLIC` на схему `public` по умолчанию нет `CREATE` — только `USAGE`
(владелец БД — через `pg_database_owner`). Раньше любой мог создать таблицу в public.

---

## Слайд 11 — Категории ролей

1. **SUPERUSER** — полный доступ, проверки ACL не выполняются.  
2. **Владелец объекта** — изначально создатель (можно сменить).  
   Члены роли-владельца тоже считаются владельцами.  
   Полный набор priv можно отозвать, но остаются «нерегламентируемые» права:
   DROP, GRANT/REVOKE, ALTER…  
3. **Остальные** — только выданные привилегии.

Проверка: функции `has_*_privilege`  
([functions-info](https://postgrespro.ru/docs/postgresql/16/functions-info)).

```sql
SELECT has_table_privilege('bob', 'alice.t1', 'SELECT');
```

Факт: «отозвать у себя SELECT как у owner» бессмысленно для защиты —
owner всё равно может вернуть priv себе. Нужна отдельная роль без ownership.

---

## Слайд 12 — Управление привилегиями

Выдаёт / отзывает **владелец** объекта (и суперпользователь):

```sql
GRANT SELECT ON TABLE t TO bob;
REVOKE DELETE ON t FROM bob;
```

Синтаксис гибкий: отдельные priv / `ALL`, один объект / `ALL TABLES IN SCHEMA` и т. д.

В psql:

| Команда | Что |
|---------|-----|
| `\dp` / `\z` | ACL таблиц |
| `\dn+` | схемы |
| `\ddp` | default privileges |
| `\drg` | membership |

Доки: [GRANT](https://postgrespro.ru/docs/postgresql/16/sql-grant), [REVOKE](https://postgrespro.ru/docs/postgresql/16/sql-revoke).

---

## Слайд 13 — Привилегии (демо, часть 1)

Alice без `CREATE` на БД:

```sql
CREATE SCHEMA alice;
-- ERROR: permission denied for database access_overview

GRANT CREATE ON DATABASE access_overview TO alice;  -- от student
-- alice снова: CREATE SCHEMA alice; → ok
```

Владелец схемы — полный доступ, `search_path` подхватывает `alice`.

```sql
CREATE TABLE t1(n numeric);
CREATE TABLE t2(n numeric, who text DEFAULT current_user);
```

Bob:

```sql
CREATE ROLE bob LOGIN PASSWORD 'bobpass';
SELECT * FROM alice.t1;
-- ERROR: permission denied for schema alice
```

`\dn+`: пустое Access privileges у `alice` = только owner; у `public` видно `=U/…` (псевдороль public).

Формат ACL: `роль=привилегии/кем_выдано`. Без имени роли = `public`.  
Для схем: `U`=USAGE, `C`=CREATE.

```sql
GRANT CREATE, USAGE ON SCHEMA alice TO bob;
SELECT * FROM alice.t1;
-- ERROR: permission denied for table t1   -- схема есть, таблицы нет
```

---

## Слайд 14 — Привилегии (демо, часть 2)

```sql
GRANT SELECT, UPDATE ON alice.t1 TO bob;
GRANT SELECT (n), INSERT ON alice.t2 TO bob;
\dp alice.*
```

Буквы ACL (не все очевидны):  
`a`=INSERT, `r`=SELECT, `w`=UPDATE, `d`=DELETE, `D`=TRUNCATE,  
`x`=REFERENCES, `t`=TRIGGER. Столбцы — в Column privileges.

```sql
ALTER ROLE bob SET search_path = public, alice;
-- \c  (переподключение)
UPDATE t1 SET n = n + 1;   -- ok
DELETE FROM t1;            -- denied
INSERT INTO t2(n) VALUES (100);  -- ok
SELECT n FROM t2;          -- ok
SELECT * FROM t2;          -- denied: нет SELECT на who
```

Факт: `SELECT *` падает, если в `*` попал запрещённый столбец — так и задумано.
ORM с `SELECT *` + column grants = боль.

---

## Слайд 15 — Включение роли в роль

```sql
GRANT dba TO bob;      -- включить
REVOKE dba FROM bob;   -- исключить
```

Отдельного типа «GROUP» нет — группа = роль (часто `NOLOGIN`).
Роль может быть в нескольких ролях; вложенность ок, **циклы запрещены**.

По умолчанию привилегии группы **наследуются**. С `NOINHERIT` нужен `SET ROLE`.
Атрибуты ролей не наследуются — но можно переключиться и воспользоваться ими.

**`PUBLIC`** — псевдороль, неявно включает **всех**.  
`GRANT … TO PUBLIC` = выдали вообще всем (включая будущие роли).  
Не путать со схемой `public`.

Доки: [role-membership](https://postgrespro.ru/docs/postgresql/16/role-membership).

---

## Слайд 16 — Предопределённые роли

Готовые «батарейки» для админских задач без SUPERUSER (`\duS` — полный список):

| Роль | Зачем |
|------|--------|
| `pg_read_all_settings` | все параметры |
| `pg_read_all_stats` | статистика |
| `pg_stat_scan_tables` | мониторинг / блокировки таблиц |
| `pg_read_all_data` | SELECT/USAGE почти везде (с PG14) |
| `pg_write_all_data` | запись везде |
| `pg_read_server_files` / `pg_write_server_files` | файлы на сервере |
| `pg_execute_server_program` | запуск программ на сервере |
| `pg_monitor` | агрегат для мониторинга |
| … | список растёт с версиями |

Свои админские роли тоже ок (бэкап и т. п.).

Демо из курса: `GRANT pg_read_all_data TO bob` → Bob читает `t2` целиком;
`\drg` показывает `INHERIT, SET`. Потом `REVOKE`.

Доки: [predefined-roles](https://postgrespro.ru/docs/postgresql/16/predefined-roles).

---

## Слайд 17 — Подпрограммы

Единственная object-priv: **`EXECUTE`**.

| Режим | Права тела |
|-------|------------|
| `SECURITY INVOKER` (default) | вызывающего |
| `SECURITY DEFINER` | владельца функции |

DEFINER = способ дать *действие*, не открывая таблицу напрямую.
Аналог setuid. Опасно, если `EXECUTE` у PUBLIC и владелец с широкими правами.

Доки: [CREATE FUNCTION](https://postgrespro.ru/docs/postgresql/16/sql-createfunction).

---

## Слайд 18 — Привилегии по умолчанию

У **`PUBLIC`** из коробки широко:

- **CONNECT** к любой новой БД (поэтому alice подключилась без явного GRANT);  
- доступ к системному каталогу;  
- **EXECUTE** на любые (новые) подпрограммы.

Privileges на новые объекты у PUBLIC появляются **автоматически**.
Просто `REVOKE EXECUTE ON ALL … FROM PUBLIC` мало: следующая функция снова даст EXECUTE public.

Механизм **`ALTER DEFAULT PRIVILEGES`** — правила на *будущие* объекты
(в т. ч. чтобы не раздавать EXECUTE public).

Доки: [ALTER DEFAULT PRIVILEGES](https://postgrespro.ru/docs/postgresql/16/sql-alterdefaultprivileges).

Важно: default privileges привязаны к **роли-создателю** (`FOR ROLE`).
Миграции под другой ролью — правила могут не сработать.

---

## Слайд 19 — Подпрограммы и default priv (демо)

```sql
-- alice
CREATE FUNCTION foo() RETURNS SETOF t2 AS $$
  SELECT * FROM t2;
$$ LANGUAGE sql STABLE;

-- bob: EXECUTE у public есть, но INVOKER → нет доступа к t2
SELECT foo();  -- permission denied for table t2
```

Нюанс курса: если bob создаст свою `public.t2` (нужен `GRANT CREATE ON SCHEMA public`,
с PG15 явно), INVOKER-функция у разных ролей пойдёт в **разные** таблицы
из-за разного `search_path`. Поучительно и опасно.

```sql
ALTER FUNCTION foo() SECURITY DEFINER;
-- bob после DROP своей t2 читает таблицу alice через foo()
```

Гигиена:

```sql
REVOKE EXECUTE ON ALL ROUTINES IN SCHEMA alice FROM PUBLIC;
ALTER DEFAULT PRIVILEGES FOR ROLE alice
  REVOKE EXECUTE ON ROUTINES FROM PUBLIC;
ALTER DEFAULT PRIVILEGES FOR ROLE alice
  GRANT EXECUTE ON ROUTINES TO bob;
\ddp
```

Новая `bar()` — bob сразу получает EXECUTE, остальные нет.

---

## Слайд 20 — Итоги

- Роли, атрибуты и привилегии — гибкий механизм: можно «всё всем» или жёстко резать.  
- При создании роли позаботьтесь о **подключении**: LOGIN + pg_hba + (часто) CONNECT.  
- Не забывайте про **PUBLIC** и **default privileges** — иначе hardening «отъезжает» на следующем CREATE.

Чеклист вслух:

1. pg_hba — первое совпадение wins;  
2. USAGE схемы → priv таблицы;  
3. DEFINER + EXECUTE PUBLIC — чинить и текущее, и default.

---

## Слайд 22 — Практика

Задача: writer — полный доступ к таблицам; reader — только чтение.

```sql
CREATE DATABASE access_overview;
CREATE USER writer;
CREATE USER reader;

\c access_overview
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT ALL ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;

ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public
  GRANT SELECT ON TABLES TO reader;

CREATE ROLE w1 LOGIN IN ROLE writer;  -- = CREATE + GRANT writer TO w1
CREATE ROLE r1 LOGIN IN ROLE reader;

\c - writer
CREATE TABLE t(n integer);

-- w1: INSERT ok; r1: SELECT ok, UPDATE denied; w1: DROP ok
```

Напоминание: с PG14 есть `pg_read_all_data` — «читать всё» без ручных GRANT.
`DROP DATABASE` — владелец БД или суперпользователь.

Критерий: reader не пишет; writer владеет (включая DROP).

---

## Слайд 23 — Практика+

Цель: trust только для своих; alice/bob через peer + map.

1. `CREATE ROLE alice LOGIN;` / `bob LOGIN;`  
2. В `pg_hba.conf` перед первой незакомментированной строкой:
   `local all postgres,student trust` — alice/bob получают `no pg_hba.conf entry`.  
3. Добавить `local all alice,bob peer` → ошибка меняется на **Peer authentication failed**
   (нет совпадения имён ОС).  
4. В `pg_ident.conf`: `stmap student alice`, в pg_hba: `peer map=stmap`.  
   Alice заходит, bob — ещё нет.  
5. Добавить `stmap student bob` — оба через одного OS-user `student`.

После правок: `SELECT pg_reload_conf();`.

| Ошибка | Смысл |
|--------|--------|
| `no pg_hba.conf entry` | нет подходящей строки |
| `Peer authentication failed` | нет map / имена не совпали |
| `password authentication failed` | пароль / метод |

В лабе trust ок; в проде trust на широкую сеть — нет.

---

## Финал

Три двери: роль → pg_hba → GRANT.  
`PUBLIC` — не схема. Default privileges — не опциональная мелочь при hardening.
Дальше по курсу бэкапы и реплики; дырявый доступ обесценит оба.
