# Текст лектора — l13

## Управление доступом: обзор

> Роли, pg_hba, GRANT — три слоя: «кто ты», «пустят ли на порог», «что можно трогать».
> student до сих пор superuser — пора включить паранойю в учебном режиме.

---

## Слайд 1 — Титул

Тема: **управление доступом** — роли, аутентификация, привилегии.

Postgres не «пользователь ОС = пользователь БД» формально, но psql и peer так **ведут себя**.

---

## Слайд 2 — Темы

1. Роли и **атрибуты**  
2. **Подключение** (pg_hba)  
3. **Пароли**  
4. **Привилегии** на объекты  
5. Группы, **predefined roles**, default privileges  
6. **Подпрограммы** (SECURITY DEFINER — осторожно)

---

## Слайд 3 — Роли и атрибуты

Роль = пользователь **и/или** группа. Циклы в membership запрещены.

| Атрибут | Смысл |
|---------|-------|
| `LOGIN` / `NOLOGIN` | можно подключаться |
| `SUPERUSER` | bypass всего ACL |
| `CREATEDB` / `CREATEROLE` | создавать БД / роли |
| `INHERIT` / `NOINHERIT` | наследовать priv группы |

При init кластера — роль `postgres` (superuser).

Доки: database-roles, role-attributes.

---

## Слайд 4 — Роли и атрибуты (демо)

```sql
CREATE ROLE alice LOGIN PASSWORD 'alice';
\du
CREATE DATABASE access_overview;
\c access_overview
```

Факт: `student` в учебном стенде — **superuser**, поэтому раньше GRANT можно было не замечать.

На слайде — имя роли в prompt (`student=#`), чтобы видеть **от чьего имени** команда.

---

## Слайд 5 — Подключение к серверу

**pg_hba.conf** — host-based authentication. Правила **сверху вниз**, первое совпадение wins.

```text
# TYPE  DATABASE  USER  ADDRESS        METHOD
local   all       postgres              peer
local   all       all                     peer
host    all       all   127.0.0.1/32     scram-sha-256
```

Поля match:

- **TYPE**: `local` (socket) vs `host` (TCP)  
- **DATABASE**, **USER**: `all` или имя  
- **ADDRESS**: для host — IP/CIDR (для local — нет)  

`listen_addresses` — **где слушаем**; `pg_hba` — **кого пускаем**.

Reload: `pg_reload_conf()` или `pg_ctl reload`.

---

## Слайд 6 — Подключение: аутентификация

После match строки — **method**, плюс проверка `LOGIN` и `CONNECT`.

| METHOD | Поведение |
|--------|-----------|
| `trust` | пустить без пароля |
| `reject` | всегда нет |
| `scram-sha-256` | пароль (современный default) |
| `md5` | устаревает |
| `peer` | имя ОС = имя роли (local) |

Порядок правил: **узкие сверху**, `all all` — внизу.

Нет подходящей строки → **FATAL: no pg_hba.conf entry**.

Доки: auth-methods.

---

## Слайд 7 — Парольная аутентификация

Пароль задаётся:

```sql
CREATE ROLE bob LOGIN PASSWORD 'secret';
ALTER ROLE bob PASSWORD 'newsecret';
```

Хранится в **`pg_authid`** (не светите `\du+` посторонним).

На клиенте:

| Способ | Заметка |
|--------|---------|
| интерактив | prompt |
| `PGPASSWORD` | небезопасно в prod |
| `~/.pgpass` | `host:port:db:user:pass`, chmod **600** |

Без пароля при `scram-sha-256` — отказ даже при правильном pg_hba.

---

## Слайд 8 — Подключение (демо)

```sql
SHOW hba_file;
SELECT type, database, user_name, address, auth_method
FROM pg_hba_file_rules();
```

Подключение alice по TCP:

```bash
psql 'host=localhost user=alice dbname=access_overview password=alicepass'
\conninfo
```

`-h localhost` ≠ local socket — **разные** строки pg_hba.

---

## Слайд 9 — Привилегии

На **таблицы/представления**:

| Priv | Действие |
|------|----------|
| SELECT | чтение |
| INSERT / UPDATE / DELETE | DML |
| TRUNCATE | очистка |
| REFERENCES | FK |
| TRIGGER | создавать триггеры |

Можно на **столбец**: `GRANT SELECT(n) ON t TO bob`.

Проверка: `has_table_privilege`, `has_column_privilege`, …

---

## Слайд 10 — Привилегии (объекты)

| Объект | Ключевые priv |
|--------|---------------|
| **DATABASE** | CONNECT, CREATE (схемы), TEMPORARY |
| **SCHEMA** | USAGE, CREATE |
| **TABLESPACE** | CREATE (объекты в ТП) |
| **SEQUENCE** | USAGE, SELECT (currval), UPDATE (nextval/setval) |

TEMP — на уровне **БД** (имя `pg_temp_*` заранее неизвестно).

USAGE схемы ≠ SELECT таблицы — классическая двухшаговая ошибка.

---

## Слайд 11 — Категории ролей

1. **SUPERUSER** — ACL не проверяется  
2. **Владелец объекта** — все priv + DDL (DROP, GRANT/REVOKE), даже если priv отозвали  
3. **Остальные** — только выданное  

Владелец = создатель (или сменённый `ALTER … OWNER TO`). Члены роли-владельца тоже «как владелец».

---

## Слайд 12 — Управление привилегиями

```sql
GRANT SELECT ON TABLE t TO bob;
GRANT SELECT, UPDATE ON alice.t1 TO bob;
REVOKE DELETE ON t FROM bob;
```

Выдавать/отзывать может **владелец** (и superuser).

`\dp`, `\dn+`, `\ddp` — быстрый просмотр ACL в psql.

Кодировка в `\dp`: `r`=select, `a`=insert, `w`=update, `d`=delete, `D`=truncate, …

---

## Слайд 13 — Привилегии (демо)

Alice без `CREATE` на БД:

```sql
CREATE SCHEMA alice;  -- permission denied
-- student:
GRANT CREATE ON DATABASE access_overview TO alice;
```

Bob без USAGE на схему:

```sql
SELECT * FROM alice.t1;  -- denied for schema alice
```

```sql
GRANT CREATE, USAGE ON SCHEMA alice TO bob;
SELECT * FROM alice.t1;  -- denied for table
GRANT SELECT, UPDATE ON alice.t1 TO bob;
GRANT SELECT(n), INSERT ON alice.t2 TO bob;
\dp alice.*
```

---

## Слайд 14 — Привилегии (демо, продолжение)

```sql
ALTER ROLE bob SET search_path = public, alice;
UPDATE t1 SET n = n + 1;   -- ok
DELETE FROM t1;            -- denied
INSERT INTO t2(n) VALUES (100);  -- ok
SELECT * FROM t2;          -- denied — нет SELECT на who
```

Column-level grant: видит только `n`.

Факт: `\dp` пустое поле priv = **только owner** (не «все могут»).

---

## Слайд 15 — Включение роли в роль

```sql
GRANT dba TO bob;
REVOKE dba FROM bob;
```

Отдельного типа «группа» нет — это роль с `NOLOGIN`.

**`public`** — псевдороль, **включает всех**. `GRANT … TO PUBLIC` = всем.

`NOINHERIT` → priv группы только через `SET ROLE group_name`.

`\drg` — кто в какой группе.

---

## Слайд 16 — Предопределённые роли

Примеры (полный список: `\duS`):

| Роль | Доступ |
|------|--------|
| `pg_read_all_data` | SELECT везде |
| `pg_write_all_data` | INSERT/UPDATE/DELETE везде |
| `pg_read_all_stats` | stats views |
| `pg_monitor` | агрегат для мониторинга |
| `pg_read_server_files` | COPY/pg_read_file… |

Зачем: дать мониторинг/backup **без SUPERUSER**.

С PG14+ `pg_read_all_data` — альтернатива ручному SELECT на каждую таблицу.

---

## Слайд 17 — Подпрограммы

Единственная priv — **`EXECUTE`**.

| Security | Поведение |
|----------|-----------|
| **INVOKER** (default) | права **вызывающего** |
| **DEFINER** | права **владельца** функции |

DEFINER = «лазейка с правами автора». Полезно, но дырка, если функция пишет в `search_path` неаккуратно.

С PG15 для `CREATE` в `public` нужен явный `GRANT CREATE ON SCHEMA public`.

---

## Слайд 18 — Привилегии по умолчанию

У **`PUBLIC`** из коробки широко:

- **CONNECT** на любую БД (alice подключилась без явного GRANT)  
- USAGE на системный каталог  
- **EXECUTE** на **новые** функции  

Отозвать `EXECUTE` на существующие — мало: **новая** функция снова public.

Решение — **`ALTER DEFAULT PRIVILEGES`**:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE alice
  REVOKE EXECUTE ON ROUTINES FROM PUBLIC;
ALTER DEFAULT PRIVILEGES FOR ROLE alice
  GRANT EXECUTE ON ROUTINES TO bob;
\ddp
```

Действует на объекты, создаваемые **указанной ролью** в будущем.

---

## Слайд 19 — Подпрограммы и default priv (демо)

```sql
CREATE FUNCTION foo() RETURNS SETOF t2 AS $$
  SELECT * FROM t2;
$$ LANGUAGE sql STABLE;
-- bob: SELECT foo(); — denied for table (invoker)
```

```sql
ALTER FUNCTION foo() SECURITY DEFINER;
-- bob видит alice.t2 через права alice
```

```sql
REVOKE EXECUTE ON ALL ROUTINES IN SCHEMA alice FROM PUBLIC;
-- + ALTER DEFAULT PRIVILEGES …
```

Урок: DEFINER + EXECUTE для public = **почти root**. Чистите default priv в prod.

---

## Слайд 20 — Итоги

- Три слоя: **роль** → **pg_hba** → **GRANT**  
- Owner ≠ superuser, но мощный  
- `public` — не «схема public», а **все роли**  
- Новые объекты наследуют default priv — планируйте заранее  

При создании роли думайте сразу: LOGIN? пароль? CONNECT? pg_hba строка?

---

## Слайд 22 — Практика

### writer / reader

```sql
CREATE DATABASE access_overview;
CREATE USER writer;
CREATE USER reader;

REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT ALL ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;

ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public
  GRANT SELECT ON TABLES TO reader;

CREATE ROLE w1 LOGIN IN ROLE writer;
CREATE ROLE r1 LOGIN IN ROLE reader;

\c - writer
CREATE TABLE t(n int);

-- w1: INSERT ok; r1: SELECT ok, UPDATE denied; w1: DROP ok
```

`IN ROLE` = `CREATE ROLE` + `GRANT group TO member`.

---

## Слайд 23 — Практика+

### trust только postgres/student

```text
local all postgres,student trust
local all alice,bob peer map=stmap
```

```text
# pg_ident.conf
stmap  student  alice
stmap  student  bob
```

Peer без map: ОС-user должен **совпадать** с именем роли.

Один OS-user → **несколько** ролей через map — удобно в лабе, в prod редко.

Проверка:

```bash
psql -U alice -d student   # ok через stmap
psql -U bob -d student     # ok после второй строки map
```

После правок — `pg_reload_conf()`. Ошибка «no pg_hba entry» vs «Peer authentication failed» — разные диагнозы.

---

## Финал

1. pg_hba: **порядок строк** решает  
2. CONNECT у public по умолчанию — не забывайте REVOKE в hardening  
3. SCHEMA USAGE → TABLE priv — два шага  
4. `\dp`, `\dn+`, `\drg`, `\ddp` — ваши друзья  
5. SECURITY DEFINER + PUBLIC EXECUTE = рецепт инцидента
