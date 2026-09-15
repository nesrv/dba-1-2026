# Текст лектора — l11

## Организация данных: низкий уровень

> Файлы, слои, TOAST — то, что обычно прячется за `\d` и `pg_total_relation_size`.
> Полезно, когда «таблица 100 строк, а на диске гиг» или когда копаетесь в PGDATA.

---

## Слайд 1 — Титул

Тема: **низкий уровень хранения** — forks, сегменты, TOAST, функции размера.

Не путаем с логической моделью (схемы, типы). Здесь — что лежит в `base/16386/16388`.

---

## Слайд 2 — Темы

1. Файлы данных и **слои (forks)**  
2. Карты видимости и свободного места  
3. TOAST для «длинных» строк  

Практика — `UNLOGGED` + `_init`, заглянуть в `pg_toast`.

---

## Слайд 3 — Слои объекта

Один объект (таблица, индекс, sequence, matview) = **несколько файлов-слоёв**.

| Имя файла | Слой |
|-----------|------|
| `16388` | main — данные |
| `16388_fsm` | free space map |
| `16388_vm` | visibility map |
| `16388.1` | 2-й сегмент main (>1 ГБ) |

Сегмент = **1 ГБ** (исторически под старые FS). Пересборка: `--with-segsize`.

```sql
SELECT pg_relation_size('t');          -- все слои main
SELECT pg_relation_size('t','main'); -- один fork
```

Факт: в одном каталоге БД может быть **тысячи файлов** — некоторые FS на этом страдают.

---

## Слайд 4 — Слои

| Слой | Кому | Зачем |
|------|------|-------|
| **main** | всем | строки / индексные записи |
| **init** | только UNLOGGED | «пустышка» после crash |
| **vm** | таблицы | all-visible / all-frozen (VACUUM, index-only) |
| **fsm** | таблицы и индексы | где есть свободное место на странице |

**UNLOGGED**: без WAL → быстрее, после crash — **пустая таблица** (init перезаписывает main).

Доки: storage-init, storage-fsm, storage-vm.

---

## Слайд 5 — Расположение файлов

```sql
CREATE DATABASE data_lowlevel;
\c data_lowlevel
CREATE TABLE t(
  id int PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  n numeric
);
INSERT INTO t(n) SELECT id FROM generate_series(1, 10000);
VACUUM t;   -- чтобы появились vm/fsm
```

Путь одной строкой:

```sql
SELECT pg_relation_filepath('t');
-- base/16386/16388
```

Расшифровка: `base` = pg_default, `16386` = oid БД, `16388` = relfilenode.

```bash
sudo -u postgres ls -l .../base/16386/16388*
# 16388, 16388_fsm, 16388_vm
```

Индекс — **main + fsm** (fsm когда есть пустые страницы). Sequence — только main.

Временные таблицы — те же слои, префикс схемы:

```sql
CREATE TEMP TABLE temp AS SELECT * FROM t;
SELECT pg_relation_filepath('temp');  -- .../t4_16397
```

---

## Слайд 6 — Размер слоёв и oid2name

Утилита из поставки:

```bash
/usr/lib/postgresql/16/bin/oid2name
oid2name -d data_lowlevel
oid2name -d data_lowlevel -t t
oid2name -d data_lowlevel -f 16388
```

Размер по fork:

```sql
SELECT pg_relation_size('t','main') main,
       pg_relation_size('t','fsm') fsm,
       pg_relation_size('t','vm') vm;
```

Факт: `relfilenode` может **меняться** (`TRUNCATE`, `VACUUM FULL`, некоторые reindex) — не кешируйте навечно.

---

## Слайд 7 — TOAST

**T**he **O**versized **A**ttribute **S**torage **T**echnique — строка **целиком на одной странице** (~8 КБ).

Если не влезает:

- сжать поле  
- вынести в **toast-таблицу** (`pg_toast.pg_toast_<oid>`)  
- или оба варианта  

Toast-таблица:

- своя версионность — UPDATE без touch «длинного» поля не дублирует toast  
- читается **только при обращении к столбцу**  
- для приложения прозрачна  

Temp → `pg_toast_temp_N`.

---

## Слайд 8 — TOAST (демо и стратегии)

```sql
SELECT length((123456789::numeric ^ 12345)::text);  -- ~99900
INSERT INTO t(n) SELECT 123456789::numeric ^ 12345;
SELECT pg_relation_size('t','main');  -- не вырос — ушло в toast
```

Найти toast-отношение:

```sql
SELECT relname, relfilenode FROM pg_class
WHERE oid = (SELECT reltoastrelid FROM pg_class WHERE oid = 't'::regclass);
```

Стратегии (`\d+ t`, колонка Storage):

| Storage | Поведение |
|---------|-----------|
| **plain** | TOAST не трогаем (фикс. типы) |
| **extended** | сжатие + вынос (default для text/json…) |
| **external** | только вынос, без сжатия |
| **main** | сначала сжатие, вынос — крайний случай |

```sql
ALTER TABLE t ALTER COLUMN n SET STORAGE external;
```

Меняет только **новые** версии строк.

---

## Слайд 9 — Размер таблицы

Иераархия функций:

```
pg_total_relation_size
├── pg_table_size (+ toast table + toast index)
│   └── pg_relation_size (слои main/fsm/vm/…)
└── pg_indexes_size (все индексы таблицы, кроме toast index)
```

| Функция | Что считает |
|---------|-------------|
| `pg_relation_size(rel, fork)` | один слой |
| `pg_table_size(rel)` | heap + toast (можно передать **индекс**!) |
| `pg_indexes_size(table)` | сумма индексов |
| `pg_total_relation_size(table)` | всё вместе |

```sql
SELECT pg_table_size('t'), pg_indexes_size('t'),
       pg_table_size('t_pkey'), pg_total_relation_size('t');
```

---

## Слайд 10 — Итоги

- Объект на диске = **несколько forks**, main может быть **сегментирован**  
- UNLOGGED → слой `_init`  
- Длинные значения → **TOAST**, стратегии через `STORAGE`  
- Размер — семейство `pg_*_size`, не `du` по одному файлу  

---

## Слайд 11 — Практика

### 1. UNLOGGED + `_init`

```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';
CREATE UNLOGGED TABLE u(n int) TABLESPACE ts;
INSERT INTO u SELECT n FROM generate_series(1,1000);
SELECT pg_relation_filepath('u');
-- ls: .../16388, .../16388_fsm, .../16388_init  (init — нулевой файл)
```

### 2. text + toast

```sql
CREATE TABLE t(s text);           -- Storage: extended
ALTER TABLE t ALTER COLUMN s SET STORAGE external;
INSERT INTO t VALUES ('Короткая строка.');
INSERT INTO t VALUES (repeat('A', 3456));

SELECT chunk_id, chunk_seq, length(chunk_data)
FROM pg_toast.pg_toast_<oid>
ORDER BY 1, 2;
-- только длинная строка, 2 chunk'а
```

Короткая остаётся inline — страница и так вмещает.

---

## Слайд 12 — Практика+

### 1. `pg_database_size` vs сумма таблиц

```sql
SELECT sum(pg_total_relation_size(oid))
FROM pg_class
WHERE NOT relisshared AND relkind = 'r';

SELECT pg_database_size('data_lowlevel');
-- второе чуть больше: pg_filenode.map, pg_internal.init, PG_VERSION
```

Object-level vs directory-level — цифры **не обязаны** совпасть идеально.

### 2. Сжатие TOAST

```sql
SELECT * FROM (
  SELECT string_to_table(setting, ' '' ') FROM pg_config WHERE name = 'CONFIGURE'
) s WHERE setting ~ '(lz|zs)';
-- --with-lz4, --with-zstd

\dconfig *toast*
-- default_toast_compression = pglz; enum {pglz,lz4}
```

### 3. Бенч сжатия (~16 МБ base32-текста)

| Вариант | `pg_table_size`/1024 | COPY time |
|---------|----------------------|-----------|
| EXTERNAL, без сжатия | ~17064 | ~409 ms |
| EXTENDED + pglz | ~10384 | ~1780 ms |
| EXTENDED + lz4 | ~10720 | ~586 ms |

**pglz** — лучше сжимает, **lz4** — быстрее. Выбор под ваш CPU/диск.

---

## Финал

1. Таблица ≈ `NNN` + `_fsm` + `_vm` (+ сегменты `.1`, `.2`…)  
2. `pg_relation_filepath` — ваш GPS в PGDATA  
3. TOAST включается **лениво**, `\d+` показывает Storage  
4. `pg_total_relation_size` — для «сколько жрёт объект целиком»  
5. Размер БД ≠ сумма таблиц — есть служебные файлы каталога
