# Текст лектора — l3

## Базовый инструментарий: конфигурирование сервера

> Меньше воды, больше команд и фактов.
> Не про «какой work_mem поставить», а **как** параметры живут: файлы, reload, SET.

---

## Слайд 1 — Титул

Тема: **конфигурирование сервера PostgreSQL**.

Факт: параметров сотни, но механизм один — файлы → reload/restart → при необходимости SET в сессии.
Сегодня не разбираем *назначение* каждого параметра, только **способы установки**.

Доки (полный список): https://postgrespro.ru/docs/postgresql/16/runtime-config

---

## Слайд 2 — Темы

Три блока:

1. **Параметры конфигурации** — что это и где задаются  
2. **Файлы конфигурации** — `postgresql.conf`, `postgresql.auto.conf`, include  
3. **Уровни экземпляра и сеанса** — reload vs restart, `SET`, `ALTER SYSTEM`

Две демо + практика. К концу должны уметь найти «кто виноват» в `pg_settings` и не убить кластер опечаткой в conf.d.

---

## Слайд 3 — Параметры

**Задача:** управлять поведением и ресурсами СУБД.

| Уровень | Как задаётся | Пример |
|---------|--------------|--------|
| экземпляр | файлы конфигурации | `max_connections = 100` |
| база / роль | `ALTER DATABASE` / `ALTER ROLE` | позже в курсе |
| сеанс | `SET`, `PGOPTIONS`, libpq `options` | `work_mem = 32MB` |

Факты:

- Пример «на весь кластер»: `max_connections` — потолок одновременных backend'ов.
- Приоритет: **позже прочитанное побеждает** (важно для `include_dir` и дублей).
- Уровни базы/роли — спойлер; сегодня файлы + текущая сессия.

---

## Слайд 4 — postgresql.conf

Основной конфиг — **текстовый**, формат `ключ = значение`.

| факт | значение |
|------|----------|
| по умолчанию | в **PGDATA** |
| Ubuntu/Debian | `/etc/postgresql/16/main/postgresql.conf` |
| data отдельно | `data_directory = '/var/lib/postgresql/16/main'` |
| override пути | `-c config_file=...` при старте `postgres` |

**Include-директивы** (обычно в конце файла):

```text
include_dir = 'conf.d'          # все *.conf из каталога
include = 'extra.conf'
include_if_exists = 'maybe.conf'
```

**Применить изменения в файле:**

```bash
pg_ctl reload
pg_ctlcluster 16 main reload
```

```sql
SELECT pg_reload_conf();
```

Факт: часть параметров — только **restart** (`context = postmaster`). Reload их не подхватит; в `pg_settings` будет `pending_restart = true`.

---

## Слайд 5 — Демонстрация

Четыре пункта — показать руками.

### 1. Где лежит конфиг

```sql
SHOW config_file;
\dconfig config_file
```

На Ubuntu conf **не** в PGDATA — это норма пакетного дистрибутива.

### 2. Фрагмент файла

```sql
SELECT pg_read_file('/etc/postgresql/16/main/postgresql.conf', 1516, 861)
\g (tuples_only=on format=unaligned)
```

Смотрим секцию FILE LOCATIONS: `data_directory`, `hba_file`, `ident_file`.

### 3. pg_file_settings — что реально в файлах

```sql
SELECT sourceline, name, setting, applied, error
FROM pg_file_settings
ORDER BY sourcefile, sourceline;
```

| applied | когда false |
|---------|-------------|
| f | нужен restart; есть более поздняя строка с тем же param; **ошибка в значении** |

### 4. pg_settings — «что работает сейчас»

```sql
SELECT name, unit, setting, boot_val, reset_val,
       source, sourcefile, sourceline,
       pending_restart, context
FROM pg_settings
WHERE name = 'work_mem'
\gx
```

**context — шпаргалка:**

| context | как менять |
|---------|------------|
| internal | нельзя |
| postmaster | restart |
| sighup | reload |
| superuser | SET супером в сессии |
| user | SET любому пользователю |

**Демо дублей** в `conf.d/work_mem.conf`:

```bash
echo work_mem=12MB | sudo tee .../conf.d/work_mem.conf
echo work_mem=8MB  | sudo tee -a .../conf.d/work_mem.conf
```

Первая строка `applied = f`, вторая `applied = t` → победит **8MB** после `SELECT pg_reload_conf();`.

---

## Слайд 6 — postgresql.auto.conf

Файл **только в PGDATA**, читается **после** `postgresql.conf` → перебивает его.

| команда | эффект |
|---------|--------|
| `ALTER SYSTEM SET param TO value` | добавить/изменить строку |
| `ALTER SYSTEM RESET param` | удалить строку |
| `ALTER SYSTEM RESET ALL` | очистить файл |

**Не редактировать руками** — заголовок файла прямо об этом кричит.

Применение — как у `postgresql.conf`: reload или restart (зависит от param).

Доки: https://postgrespro.ru/docs/postgresql/16/sql-altersystem

---

## Слайд 7 — Демонстрация

### ALTER SYSTEM + валидация

```sql
ALTER SYSTEM SET work_mem TO '16mb';   -- ERROR: invalid unit
ALTER SYSTEM SET work_mem TO '16MB';   -- OK
```

`ALTER SYSTEM` **проверяет** значение — опечатка в единицах ловится сразу, не при restart.

### Файл vs сессия

```sql
SELECT pg_read_file('postgresql.auto.conf') \g (tuples_only=on format=unaligned)
SHOW work_mem;                    -- ещё старое, пока не reload
SELECT pg_reload_conf();
-- sourcefile → .../postgresql.auto.conf
```

### RESET

```sql
ALTER SYSTEM RESET work_mem;
SELECT pg_reload_conf();
-- вернётся значение из conf.d/work_mem.conf (если там было)
```

Факт: `pg_file_settings` и `pg_settings` — два разных «рентгена»: файл vs runtime.

---

## Слайд 8 — В текущем сеансе

Клиент ↔ сервер:

| установить | прочитать |
|------------|-----------|
| `SET`, `set_config()` | `SHOW`, `current_setting()` |

Факты:

- Действует до **конца сеанса** или до **конца транзакции** (`SET LOCAL` / `set_config(..., true)`).
- Изменения **транзакционны** — `ROLLBACK` откатывает и параметры.
- **Пользовательские** параметры: в имени **обязательна точка** (`myapp.currency_code`), иначе Postgres думает, что это системный GUC.

Лайфхак для пулов: `set_config(..., true)` — только на транзакцию, чтобы не протекло в следующего клиента на том же backend.

---

## Слайд 9 — Демонстрация

### SET / set_config

```sql
SET work_mem TO '24MB';
SELECT set_config('work_mem', '32MB', false);  -- false = до конца сеанса
```

### Чтение — четыре способа

```sql
SHOW work_mem;
\dconfig work_mem
SELECT current_setting('work_mem');
SELECT name, setting, unit FROM pg_settings WHERE name = 'work_mem';
```

`RESET work_mem;` — к значению **на начало сеанса** (`reset_val`).

### Транзакция

```sql
BEGIN;
SET work_mem TO '64MB';
ROLLBACK;
SHOW work_mem;   -- откатилось

BEGIN;
SET LOCAL work_mem TO '64MB';  -- или set_config('work_mem','64MB',true)
COMMIT;
SHOW work_mem;   -- LOCAL слетел после COMMIT
```

### Пользовательский параметр

```sql
SELECT CASE
  WHEN current_setting('myapp.currency_code', true) IS NULL
  THEN set_config('myapp.currency_code', 'RUB', false)
  ELSE current_setting('myapp.currency_code')
END;
```

Можно задать и в conf-файле — инициализируется во всех новых сессиях.

---

## Слайд 10 — Итоги

Коротко:

- **`postgresql.conf`** — основной файл; include_dir на Ubuntu = `conf.d/`
- **`ALTER SYSTEM`** → **`postgresql.auto.conf`** (SQL-интерфейс, не трогать vim'ом)
- После правок файлов — **`pg_reload_conf()`**, но не для всех параметров
- Много параметров — **`SET`** в сессии (`context = user`)
- **`postmaster`**-параметры — только restart; смотреть `pending_restart`

Три представления на память: `pg_settings`, `pg_file_settings`, `SHOW config_file`.

---

## Слайд 11 — Практика

### 1. Параметры, требующие restart

```sql
SELECT name, setting, unit
FROM pg_settings
WHERE context = 'postmaster'
ORDER BY name;
```

~57 штук: `shared_buffers`, `max_connections`, `wal_level`, `listen_addresses`…

### 2. Ошибка в conf.d — сервер не стартует

```bash
echo max_connections=5O | sudo tee /etc/postgresql/16/main/conf.d/max_connections.conf
```

Проверка **до** restart:

```sql
SELECT * FROM pg_file_settings WHERE name = 'max_connections'\gx
-- error: setting could not be applied, setting = 5O
```

Restart → FATAL в логе:

```text
invalid value for parameter "max_connections": "5O"
configuration file ".../max_connections.conf" contains errors
```

```bash
tail -n 5 /var/log/postgresql/postgresql-16-main.log
echo max_connections=50 | sudo tee .../max_connections.conf
sudo pg_ctlcluster 16 main start
```

Факт: **reload** при битом postmaster-параметре может «прожить»; **restart** — нет. Поэтому практика именно с restart.

---

## Слайд 12 — Практика+

### 1. work_mem при старте psql (libpq)

**Строка подключения** — ключ `options`:

```bash
psql "options='-c work_mem=32MB'" -c 'SHOW work_mem'
```

**Переменная окружения:**

```bash
export PGOPTIONS='-c work_mem=32MB'
psql -c 'SHOW work_mem'
```

Доки: https://postgrespro.ru/docs/postgresql/16/libpq-connect#LIBPQ-CONNSTRING

### 2. Как Ubuntu находит postgresql.conf вне PGDATA

```sql
\dconfig (config_file|data_directory)
```

Ответ: в **командной строке** postmaster:

```bash
sudo cat /var/lib/postgresql/16/main/postmaster.pid | head -n 1   # PID
ps -p <PID> -ho command
# .../postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
```

Факт: **PGDATA** и **config_file** разведены специально — conf в `/etc`, данные в `/var/lib`. Systemd/unit `pg_ctlcluster` прокидывает `-c config_file=...`.

---

## Финал

На выход:

1. **Порядок чтения:** `postgresql.conf` → include → `postgresql.auto.conf` → последняя строка побеждает  
2. **reload vs restart** — смотреть `context` и `pending_restart`  
3. **`pg_file_settings.error`** — ловить опечатки до restart  
4. **Сессия:** `SET` / `SET LOCAL` / `PGOPTIONS` / libpq `options`  
5. **Ubuntu:** config в `/etc`, data в `/var/lib`, связь через `-c config_file`

Конфиг — не магия, а текстовые файлы с понятной иерархией. Главное — не править `postgresql.auto.conf` руками и не писать `5O` вместо `50`.
