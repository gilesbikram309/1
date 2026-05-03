# PostgreSQL: полное и исчерпывающее руководство для Senior-разработчика (PHP 8.2 + Symfony 6.4)

> Это руководство — практический справочник по PostgreSQL для backend-разработчика уровня Senior. Оно покрывает архитектуру СУБД, типы данных, индексы, планировщик, транзакции и блокировки, MVCC, VACUUM, партиционирование, репликацию, шардирование, отказоустойчивость, оптимизацию, интеграцию в Symfony 6.4 / Doctrine, а также сравнение с MySQL. Все примеры — реальные и работоспособные, решающие конкретные задачи web-приложений.

---

## Оглавление

1. [[#1. Что такое PostgreSQL и зачем его выбирать|Что такое PostgreSQL и зачем его выбирать]]
2. [[#2. Архитектура PostgreSQL|Архитектура PostgreSQL]]
3. [[#3. MVCC, транзакции, уровни изоляции|MVCC, транзакции, уровни изоляции]]
4. [[#4. Блокировки (locking)|Блокировки (locking)]]
5. [[#5. Типы данных|Типы данных]]
6. [[#6. Индексы: все виды и когда какой использовать|Индексы: все виды и когда какой использовать]]
7. [[#7. Планировщик, EXPLAIN, оптимизация запросов|Планировщик, EXPLAIN, оптимизация запросов]]
8. [[#8. VACUUM, autovacuum, bloat|VACUUM, autovacuum, bloat]]
9. [[#9. Расширенные возможности: CTE, Window, JSONB, полнотекстовый поиск|Расширенные возможности: CTE, Window, JSONB, полнотекстовый поиск]]
10. [[#10. Партиционирование|Партиционирование]]
11. [[#11. Репликация, высокая доступность|Репликация, высокая доступность]]
12. [[#12. Шардирование и масштабирование|Шардирование и масштабирование]]
13. [[#13. Бэкапы и восстановление (PITR)|Бэкапы и восстановление (PITR)]]
14. [[#14. Безопасность|Безопасность]]
15. [[#15. Конфигурация и тюнинг под нагрузку|Конфигурация и тюнинг под нагрузку]]
16. [[#16. Интеграция в Symfony 6.4 / Doctrine|Интеграция в Symfony 6.4 / Doctrine]]
17. [[#17. Сравнение PostgreSQL и MySQL|Сравнение PostgreSQL и MySQL]]
18. [[#18. Типичные ошибки и анти-паттерны|Типичные ошибки и анти-паттерны]]
19. [[#19. Zero-downtime миграции схемы|Zero-downtime миграции схемы]]
20. [[#20. Materialized Views, представления, updatable views|Materialized Views, представления, updatable views]]
21. [[#21. Generated columns и IDENTITY|Generated columns и IDENTITY]]
22. [[#22. TOAST и большие значения|TOAST и большие значения]]
23. [[#23. Мониторинг и диагностика live-нагрузки|Мониторинг и диагностика live-нагрузки]]
24. [[#24. Bulk-операции: COPY, batch insert, streaming|Bulk-операции: COPY, batch insert, streaming]]
25. [[#25. Transactional Outbox и idempotency|Transactional Outbox и idempotency]]
26. [[#26. Триггеры, правила, event triggers, audit|Триггеры, правила, event triggers, audit]]
27. [[#27. Unlogged, temp tables, foreign tables, ltree, citext, unaccent|Unlogged, temp tables, foreign tables, ltree, citext, unaccent]]
28. [[#28. Optimistic locking в Doctrine и deferred constraints|Optimistic locking в Doctrine и deferred constraints]]
29. [[#29. Таймауты, read-only транзакции, защита от «зависших» коннектов|Таймауты, read-only транзакции, защита от «зависших» коннектов]]
30. [[#30. SQLSTATE и обработка ошибок в Doctrine|SQLSTATE и обработка ошибок в Doctrine]]
31. [[#31. Тестирование кода, работающего с PostgreSQL|Тестирование кода, работающего с PostgreSQL]]
32. [[#32. PgBouncer: тонкости для PHP/Symfony|PgBouncer: тонкости для PHP/Symfony]]
33. [[#33. Проверочные вопросы с ответами|Проверочные вопросы с ответами]]
34. [[#34. Источники|Источники]]

---

## 1. Что такое PostgreSQL и зачем его выбирать

**PostgreSQL** — объектно-реляционная СУБД с открытым исходным кодом (лицензия PostgreSQL, BSD-like), развивается с 1986 года (проект POSTGRES в Беркли). Ключевые отличия от большинства open-source СУБД:

- Полная поддержка стандарта SQL (SQL:2016).
- Транзакционный DDL (`ALTER TABLE` откатывается внутри транзакции — в MySQL нет).
- MVCC (Multi-Version Concurrency Control) без блокировки читателей.
- Расширяемость: можно создавать свои типы, операторы, индексы, языки функций (PL/pgSQL, PL/Python, PL/V8).
- Богатый набор встроенных типов: `jsonb`, `tsvector`, `uuid`, `range`, `inet`, `cidr`, массивы, геометрия (+ PostGIS).
- Индексы: B-tree, Hash, GIN, GiST, SP-GiST, BRIN, Bloom.
- Высокая надёжность (WAL), логическая и физическая репликация.

**Когда выбрать PostgreSQL:**
- Сложные отчёты/аналитика, оконные функции, CTE.
- Работа с JSON как с документом (JSONB).
- Геоданные (PostGIS).
- Требования к строгой согласованности и корректности (finance, billing).
- Нужны расширения: `pg_trgm`, `pgcrypto`, `pg_stat_statements`, `timescaledb`.

---

## 2. Архитектура PostgreSQL

PostgreSQL — **процессная** модель (а не поточная, как MySQL): на каждое соединение создаётся отдельный процесс-бэкенд (`postgres`). Из-за этого `max_connections` дорогой, и нужен **PgBouncer**.

Основные процессы:
- **Postmaster** — родительский, принимает коннекты.
- **Backend** — обслуживает клиента.
- **WAL writer** — пишет WAL (журнал упреждающей записи).
- **Background writer / Checkpointer** — сбрасывает грязные страницы из shared_buffers на диск.
- **Autovacuum launcher/worker** — поддерживает статистику, убирает dead tuples.
- **WAL sender / receiver** — для репликации.

Память:
- `shared_buffers` — общий кеш страниц (обычно 25% RAM).
- `work_mem` — память на одну сортировку/hash (на операцию!).
- `maintenance_work_mem` — для VACUUM/CREATE INDEX.
- `effective_cache_size` — подсказка планировщику о доступном OS page cache (обычно 50–75% RAM).

> **Как это понять новичку.** `shared_buffers` — это «своя» память PG, `effective_cache_size` — оценка того, сколько данных дополнительно закешировано **самой ОС** (Linux page cache). PG не выделяет `effective_cache_size` физически — это только подсказка планировщику, чтобы он чаще выбирал Index Scan вместо Seq Scan, зная, что данные скорее всего в RAM. `work_mem` выделяется **на каждую операцию сортировки/хэширования в каждом запросе**: запрос с двумя `ORDER BY` в разных подзапросах при `work_mem = 32MB` съест 64 МБ, а 100 одновременных запросов — до 6.4 ГБ. Поэтому ставить `work_mem = 1GB` на сервере с `max_connections = 200` — быстрый путь к OOM-killer.

Файловая структура: `$PGDATA/base/<db_oid>/<relfilenode>` — файлы таблиц и индексов по 1 ГБ сегментами. WAL — в `$PGDATA/pg_wal`.

---

## 3. MVCC, транзакции, уровни изоляции

### MVCC
Каждая строка (tuple) имеет скрытые поля `xmin` (транзакция-создатель) и `xmax` (транзакция-удалитель). При `UPDATE` создаётся **новая версия строки**, старая помечается `xmax`. Поэтому:
- Читатели не блокируют писателей и наоборот.
- `UPDATE` — это фактически `DELETE` + `INSERT` (важно для bloat).
- Нужен **VACUUM** для очистки «мёртвых» версий.

### Уровни изоляции
PostgreSQL поддерживает 4 уровня (по стандарту SQL), но `READ UNCOMMITTED` ведёт себя как `READ COMMITTED` (грязных чтений нет никогда):

> **Мини-шпаргалка по аномалиям.**
> - **Dirty read** — читаем данные, которые другая транзакция ещё не закоммитила.
> - **Non-repeatable read** — внутри транзакции один и тот же `SELECT ... WHERE id = 1` вернул разные значения, потому что другой коммит изменил строку.
> - **Phantom read** — тот же `SELECT ... WHERE status='new'` второй раз вернул **другой набор строк** (кто-то вставил/удалил подходящую).
> - **Serialization anomaly** — две транзакции по отдельности корректны, но их «параллельный» результат не эквивалентен ни одному последовательному выполнению (классика — перевод денег «крест-накрест» при проверке баланса).

| Уровень | Dirty read | Non-repeatable read | Phantom read | Serialization anomaly |
|---|---|---|---|---|
| READ UNCOMMITTED | нет (как RC) | да | да | да |
| **READ COMMITTED** (по умолчанию) | нет | да | да | да |
| **REPEATABLE READ** | нет | нет | нет* | да |
| **SERIALIZABLE** (SSI) | нет | нет | нет | нет |

\* В PostgreSQL `REPEATABLE READ` использует snapshot isolation, поэтому фантомов нет, в отличие от стандарта.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT; -- может упасть с ERROR: could not serialize access — нужна логика retry
```

**Практика в Symfony/Doctrine:**

```php
$em->getConnection()->executeStatement('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
$em->wrapInTransaction(function() use ($em) {
    // бизнес-логика
});
```

Обязательно реализуйте **retry** на коде `40001` (`serialization_failure`) и `40P01` (`deadlock_detected`).

---

## 4. Блокировки (locking)

### Типы блокировок строк
- `FOR UPDATE` — эксклюзивная блокировка строки.
- `FOR NO KEY UPDATE` — слабее, не конфликтует с FK-проверками.
- `FOR SHARE` — разделяемая.
- `FOR KEY SHARE` — самая слабая, позволяет FK-ссылаться.

```sql
-- Пессимистичная блокировка для денежных операций
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### SKIP LOCKED — очереди в БД
Решение задачи «очередь задач» без Redis/RabbitMQ:

```sql
-- Воркер забирает задачу, не блокируясь на тех, что взяли другие
BEGIN;
SELECT id, payload FROM jobs
  WHERE status = 'pending'
  ORDER BY created_at
  LIMIT 1
  FOR UPDATE SKIP LOCKED;
-- обрабатываем...
UPDATE jobs SET status = 'done' WHERE id = :id;
COMMIT;
```

Symfony Messenger имеет транспорт `doctrine://`, внутри использует именно `FOR UPDATE SKIP LOCKED` (см. `DoctrineReceiver`).

### Deadlock
PostgreSQL автоматически детектит deadlock через таймаут `deadlock_timeout` (по умолчанию 1s) и убивает одну из транзакций с SQLSTATE `40P01`. Всегда блокируйте ресурсы **в одном и том же порядке**.

### Advisory locks
Блокировки уровня приложения, не привязанные к строкам:

```sql
SELECT pg_try_advisory_lock(12345); -- true/false
-- критическая секция
SELECT pg_advisory_unlock(12345);
```

Полезно для cron-задач в многосерверной среде (чтобы запустилась только одна копия).

---

## 5. Типы данных

### Числа
- `smallint` (2B), `integer` (4B), `bigint` (8B).
- `numeric(p,s)` — произвольная точность, **обязательно для денег**.
- `real`, `double precision` — НЕ для денег (потеря точности).
- `serial` / `bigserial` — устаревшее, используйте `GENERATED ALWAYS AS IDENTITY` (SQL-стандарт):

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL UNIQUE
);
```

### Строки
- `text` / `varchar(n)` / `char(n)` — физически хранятся одинаково, `text` предпочтителен. Ограничение длины делайте через `CHECK` или валидатор приложения.

### UUID
```sql
-- Начиная с PostgreSQL 13 функция gen_random_uuid() встроена в ядро,
-- расширение pgcrypto больше не требуется именно для UUID.
-- Для UUID v1 используйте uuid-ossp (CREATE EXTENSION "uuid-ossp"; uuid_generate_v1()).
-- Для UUID v7 (временно-упорядоченный, лучше для индексов) используйте приложение
-- (Symfony\Component\Uid\Uuid::v7()) или PostgreSQL 18+, где появилась встроенная
-- функция uuidv7() (коммит принят в июле 2024, релиз PG 18 — сентябрь 2025).
-- В PG 13–17 встроенной uuidv7() нет — либо генерируйте UUID v7 в приложении,
-- либо подключайте сторонние расширения (pg_uuidv7, pg_idkit).
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()   -- UUID v4
);
```

> **Важно для производительности индексов:** UUID v4 распределён случайно, что ухудшает локальность B-tree (каждая вставка пишет в случайное место индекса, растёт WAL и bloat). Для новых сущностей предпочитайте **UUID v7** (timestamp-prefixed) — их порядок близок к хронологическому, как у BIGSERIAL, но без раскрытия счётчика наружу.

### UUID в Doctrine правильно

Чтобы Doctrine создал колонку именно `uuid` (16 байт), а не `varchar(36)` (36 байт + collation), подключите типы из `symfony/uid`:

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        types:
            uuid: Symfony\Bridge\Doctrine\Types\UuidType      # нативный uuid
            # Если хочется привязать generator к v7:
            # uuid: Symfony\Bridge\Doctrine\Types\UuidV7Type
```

```php
use Symfony\Bridge\Doctrine\Types\UuidType;
use Symfony\Component\Uid\Uuid;

#[ORM\Id]
#[ORM\Column(type: UuidType::NAME, unique: true)]
private Uuid $id;

public function __construct()
{
    $this->id = Uuid::v7(); // timestamp-prefixed, дружелюбен к B-tree
}
```

Минусы `varchar(36)`: индекс в 2–3 раза больше, сравнение идёт как строка (медленнее побайтового). Минус случайного UUID v4 в B-tree: каждая вставка — рандомная страница, растут WAL, bloat и I/O.

### Дата/время
- `timestamp` (без TZ) и `timestamptz` (с TZ) — **всегда используйте timestamptz**. Он хранится в UTC, выдается в часовом поясе сессии.
- `interval`, `date`, `time`.

### JSON / JSONB
- `json` — хранит как текст, медленнее.
- `jsonb` — бинарный, индексируемый, почти всегда нужен он.

```sql
CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data JSONB NOT NULL
);

CREATE INDEX idx_events_data ON events USING GIN (data jsonb_path_ops);

SELECT * FROM events WHERE data @> '{"type": "signup"}';
SELECT data->>'email' FROM events WHERE data ? 'email';
```

### Массивы
```sql
CREATE TABLE posts (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tags TEXT[] NOT NULL DEFAULT '{}'
);
CREATE INDEX ON posts USING GIN (tags);
SELECT * FROM posts WHERE tags @> ARRAY['php', 'symfony'];
```

### Enum
```sql
CREATE TYPE order_status AS ENUM ('new', 'paid', 'shipped', 'cancelled');
```
Минус: добавление значения — `ALTER TYPE ... ADD VALUE` (нельзя удалить). Часто лучше `CHECK` + `text` или отдельная таблица-справочник.

### Range types
```sql
CREATE TABLE reservations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INT NOT NULL,
    during TSTZRANGE NOT NULL,
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)  -- гарантия отсутствия пересечений!
);
```
Одной строкой обеспечиваем: «одна комната не может быть забронирована дважды в пересекающееся время».

### Network
`inet`, `cidr`, `macaddr` — для IP-адресов, с индексируемыми операторами `<<`, `>>`.

---

## 6. Индексы: все виды и когда какой использовать

### B-tree (по умолчанию)
Для равенств и диапазонов (`=`, `<`, `>`, `BETWEEN`, `IN`, `ORDER BY`).
```sql
CREATE INDEX idx_users_email ON users (email);
```

### Многоколоночные и порядок колонок
Работает правило **левого префикса**: индекс `(a, b, c)` ускоряет `WHERE a=... AND b=...`, но НЕ `WHERE b=...`.

### Частичные (partial)
```sql
-- Индексируем только «активных» пользователей
CREATE INDEX idx_users_active_email ON users(email) WHERE is_active = true;
```
Маленький, быстрый, эффективен для «горячих» подмножеств.

### Функциональные / expression
```sql
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'ivan@example.com';
```

### Уникальные с условием
```sql
-- email уникален только среди не удалённых
CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL;
```

### Covering (INCLUDE) — PG 11+
```sql
CREATE INDEX idx_orders_user_inc ON orders (user_id) INCLUDE (total, created_at);
```
Позволяет **Index Only Scan** — читать данные только из индекса.

### GIN
Для массивов, `jsonb`, полнотекстового поиска, `pg_trgm`.
```sql
CREATE INDEX ON events USING GIN (data);
CREATE INDEX ON articles USING GIN (to_tsvector('russian', body));
```

### GiST
Геометрия, range, full-text (менее эффективно, чем GIN), exclusion constraints.

### BRIN
Block Range Index — для очень больших таблиц, где данные физически упорядочены (например, логи по времени). Занимает копейки.
```sql
CREATE INDEX ON logs USING BRIN (created_at);
```

### Hash
Только для `=`. С PG 10+ WAL-логируется. Редко нужен (B-tree почти всегда не хуже).

### pg_trgm (LIKE/ILIKE ускорение)
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%иван%';  -- теперь по индексу!
```

### Правила гигиены
- Не создавайте индекс «на всякий случай» — он замедляет запись.
- Удаляйте неиспользуемые: `pg_stat_user_indexes.idx_scan = 0`.
- Создавайте в проде **CONCURRENTLY**, чтобы не блокировать таблицу:
```sql
CREATE INDEX CONCURRENTLY idx_... ON ...;
```

> **Как проверить, используется ли индекс.** Самый быстрый способ — `EXPLAIN (ANALYZE, BUFFERS)` на конкретном запросе: если видите `Seq Scan on big_table` вместо `Index Scan using idx_...`, индекс не используется. Причины обычно три: (1) нет подходящего индекса; (2) запрос не `sargable` — функция поверх колонки (`WHERE LOWER(email) = ...` без функционального индекса); (3) планировщик считает, что Seq Scan дешевле — почти всегда это устаревшая статистика, лечится `ANALYZE table_name;`. Если данных мало (<10k строк), PG **осознанно** выбирает Seq Scan — это нормально.

---

## 7. Планировщик, EXPLAIN, оптимизация запросов

### EXPLAIN / EXPLAIN ANALYZE
- `EXPLAIN` — оценка стоимости.
- `EXPLAIN ANALYZE` — реально выполнить и показать фактическое время.
- `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)` — максимум информации.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.email, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > now() - interval '30 days'
GROUP BY u.email;
```

### Что смотреть
- **Seq Scan vs Index Scan vs Index Only Scan vs Bitmap Index Scan**.
- **rows estimated vs actual** — если оценки сильно расходятся, нужны `ANALYZE` или увеличить `default_statistics_target`.
- **Buffers: shared hit/read** — hit = из памяти, read = с диска.
- **Join type**: Nested Loop (маленькие наборы), Hash Join (большие), Merge Join (упорядоченные).
- **Parallel**: `Gather`, `Parallel Seq Scan` — хорошо для аналитики.

### Типичные проблемы и решения
1. **Seq Scan на большой таблице** → нет индекса или `WHERE` не sargable (например, `WHERE date_trunc('day', created_at) = ...` — лучше `WHERE created_at >= :start AND created_at < :end`).
2. **Плохая оценка строк** → `ANALYZE table;`, или `CREATE STATISTICS` для коррелированных колонок:
```sql
CREATE STATISTICS s_geo (dependencies) ON city, country FROM addresses;
ANALYZE addresses;
```
3. **N+1** — на уровне ORM, не БД. Решается `JOIN FETCH` в Doctrine.
4. **OFFSET большой страницы** — `OFFSET 100000` всё равно читает 100000 строк. Используйте **keyset pagination**:
```sql
SELECT * FROM items WHERE (created_at, id) < (:last_created, :last_id)
ORDER BY created_at DESC, id DESC LIMIT 20;
```
5. **LIMIT + ORDER BY без индекса** → добавьте индекс по колонке сортировки.

### pg_stat_statements
Расширение для поиска медленных запросов:
```sql
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
```

### auto_explain
Автоматически логирует план для медленных запросов:
```
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on
```

---

## 8. VACUUM, autovacuum, bloat

### Что делает VACUUM
- Убирает мёртвые версии строк.
- Обновляет карту видимости (visibility map) — нужно для Index Only Scan.
- `VACUUM FULL` — перестраивает таблицу (эксклюзивная блокировка! — только в окно обслуживания).
- `VACUUM ANALYZE` — плюс обновляет статистику.

### Autovacuum
Запускается фоново. Триггеры — `autovacuum_vacuum_scale_factor` (по умолчанию 0.2 = 20% изменений) + `autovacuum_vacuum_threshold` (50 строк).

На крупных таблицах уменьшайте scale_factor:
```sql
ALTER TABLE big_table SET (autovacuum_vacuum_scale_factor = 0.02);
```

### Bloat
Раздувание таблицы/индекса из-за мёртвых версий. Диагностика — расширение `pgstattuple` или запросы из `check_postgres`. Лечение без даунтайма:
- `VACUUM FULL` — с блокировкой.
- **pg_repack** — утилита, перестраивает таблицу онлайн.
- `REINDEX CONCURRENTLY` (PG 12+) — для индексов.

### TXID wraparound
Счётчик транзакций 32-битный (~2 млрд). Если autovacuum отстаёт — БД остановится в режиме «wraparound protection». **Мониторьте `pg_database.datfrozenxid`** и age:
```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
```

---

## 9. Расширенные возможности: CTE, Window, JSONB, полнотекстовый поиск

### CTE и рекурсивные CTE
```sql
-- Дерево комментариев
WITH RECURSIVE tree AS (
    SELECT id, parent_id, body, 1 AS depth
    FROM comments WHERE id = :root
    UNION ALL
    SELECT c.id, c.parent_id, c.body, t.depth + 1
    FROM comments c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY depth;
```
С PG 12+ CTE inlined (как подзапрос), если не `MATERIALIZED`. Раньше всегда материализовывались — часто тормозило.

### Оконные функции
```sql
-- Топ-3 товара в каждой категории
SELECT * FROM (
  SELECT p.*, ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY sales DESC) AS rn
  FROM products p
) t WHERE rn <= 3;

-- Running total
SELECT date, amount, SUM(amount) OVER (ORDER BY date) AS running
FROM payments;
```

### UPSERT
```sql
INSERT INTO metrics (key, value) VALUES ('views', 1)
ON CONFLICT (key) DO UPDATE SET value = metrics.value + EXCLUDED.value;
```

### RETURNING
```sql
INSERT INTO orders (...) VALUES (...) RETURNING id, created_at;
UPDATE users SET status='active' WHERE id=:id RETURNING *;
```

### JSONB — продвинутое
```sql
-- Индекс для точечных путей
CREATE INDEX ON docs USING GIN ((data->'tags'));

-- Обновление части JSON
UPDATE docs SET data = jsonb_set(data, '{user,name}', '"Ivan"') WHERE id = 1;

-- Удаление ключа
UPDATE docs SET data = data - 'temp_field';
```

### Full-text search
```sql
ALTER TABLE articles ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (to_tsvector('russian', coalesce(title,'') || ' ' || coalesce(body,''))) STORED;
CREATE INDEX ON articles USING GIN (tsv);

SELECT title, ts_rank(tsv, q) AS rank
FROM articles, plainto_tsquery('russian', 'symfony postgres') q
WHERE tsv @@ q ORDER BY rank DESC LIMIT 20;
```

> ⚠️ **Для новичка.** В `GENERATED ALWAYS AS (...) STORED` можно использовать только `IMMUTABLE`-выражения. У `to_tsvector` есть две формы: одноаргументная `to_tsvector(text)` — `STABLE` (зависит от GUC `default_text_search_config`), и двухаргументная `to_tsvector(regconfig, text)` — `IMMUTABLE`. Всегда передавайте конфигурацию явно (`'russian'`, `'simple'`, …), иначе получите `ERROR: generation expression is not immutable`. То же касается `unaccent`: чтобы использовать его в `GENERATED`, нужна двухаргументная форма `unaccent('unaccent'::regdictionary, text)` или обёртка в собственную IMMUTABLE-функцию.

### LISTEN/NOTIFY — pub/sub на уровне БД
```sql
-- в одной сессии
LISTEN order_created;
-- в другой
NOTIFY order_created, '{"id":42}';
```
Полезно для легковесных уведомлений; не замена Kafka.

---

## 10. Партиционирование

С PG 10+ есть **декларативное** партиционирование. Виды: `RANGE`, `LIST`, `HASH`.

### RANGE по дате (логи, события)
```sql
CREATE TABLE events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    created_at  TIMESTAMPTZ NOT NULL,
    data        JSONB,
    PRIMARY KEY (id, created_at)              -- в партиционировании PK ОБЯЗАН включать ключ партиционирования
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_04 PARTITION OF events
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE events_2026_05 PARTITION OF events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX ON events_2026_04 (created_at);
```

Плюсы: **partition pruning** (планировщик отбрасывает ненужные партиции), быстрое удаление `DROP TABLE events_2025_01` вместо `DELETE`.

Для автосоздания партиций используйте расширение **pg_partman**.

### HASH (равномерное распределение)
```sql
CREATE TABLE sessions (...) PARTITION BY HASH (user_id);
CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ...
```

### Ограничения
- Первичный/уникальный ключ должен включать колонку партиционирования.
- FK на партиционированную таблицу поддерживается с PG 12.
- Глобальных индексов нет — индекс создаётся на каждой партиции.

> **Типичный путь внедрения.** Не партиционируйте «на будущее» — это всегда дороже простой таблицы. Правило практики: задумываться о партиционировании, когда таблица подошла к **~50–100 ГБ** или когда вы регулярно `DELETE`-ите большие окна данных (логи старше 90 дней). До этого момента достаточно грамотных индексов, autovacuum и, при необходимости, BRIN. После — партиции по времени (RANGE) + `pg_partman` для авто-создания и детача старых партиций.

---

## 11. Репликация, высокая доступность

### Физическая (streaming) репликация
Основано на WAL. На primary — `wal_level = replica`, на replica — `standby.signal` + `primary_conninfo`.
- **Синхронная** — `synchronous_standby_names`, гарантия, но COMMIT ждёт реплику.
- **Асинхронная** — быстрее, но возможна потеря последних байт WAL при крахе primary.

Реплики **read-only** (hot standby). Репликационное отставание: `pg_stat_replication.replay_lag`.

### Логическая репликация
С PG 10+. На уровне «публикаций» и «подписок», реплицирует DML конкретных таблиц. Нужна для:
- Миграции между версиями без даунтайма.
- Репликации в другую схему/подмножество таблиц.
- ETL.

```sql
-- на source
CREATE PUBLICATION pub_orders FOR TABLE orders, order_items;
-- на target
CREATE SUBSCRIPTION sub_orders
  CONNECTION 'host=... dbname=... user=repl'
  PUBLICATION pub_orders;
```

### Failover
Сам PostgreSQL failover НЕ делает. Нужны:
- **Patroni** (de-facto стандарт) + etcd/Consul + HAProxy/PgBouncer.
- **repmgr**.
- В облаке — managed-решения (RDS, Cloud SQL).

### Split-brain и fencing
Patroni + DCS (etcd) гарантируют, что один primary. Никогда не промоутьте реплику вручную без fencing старой primary.

---

## 12. Шардирование и масштабирование

PostgreSQL **не имеет** встроенного шардирования из коробки. Подходы:

### 1) Read-replicas (вертикальное масштабирование чтений)
Simple. Пишем в primary, читаем с реплик. В Symfony/Doctrine реализуется **PrimaryReadReplicaConnection** (см. раздел 16).

### 2) Функциональный шардинг
Разные БД для разных доменов (users в одной, orders в другой). Минус: теряются JOIN, нужны distributed transactions (saga).

### 3) Горизонтальный шардинг по ключу
Обычно по `tenant_id`/`user_id`. Нужен слой маршрутизации в приложении.
- **Citus** (расширение PG, открытый, куплен Microsoft) — автоматическое распределение таблиц по нодам, распределённые запросы.
- **FDW + postgres_fdw** — можно собрать свой шардинг через «родительскую» партиционированную таблицу и foreign партиции на других серверах.

```sql
-- Citus пример
SELECT create_distributed_table('events', 'tenant_id');
```

### 4) Connection pooling
**Без PgBouncer в проде жить нельзя.** Режимы:
- `session` — соединение на всю сессию клиента.
- `transaction` — чаще всего, но нельзя использовать prepared statements на клиенте и `SET LOCAL` вне транзакции.
- `statement` — редко.

Для PHP (stateless, короткие запросы) — `transaction pooling` + в Doctrine `server_version` фиксируйте + отключайте server-side prepared: в DSN добавляйте `?serverVersion=16&charset=utf8` и в opts `'driverOptions' => [PDO::ATTR_EMULATE_PREPARES => true]` (для PDO есть нюансы — см. доку pdo_pgsql).

### 5) Кеши
Read-through кеш (Redis) перед PG для горячих ключей. Инвалидация — по событиям (Doctrine lifecycle / Outbox).

---

## 13. Бэкапы и восстановление (PITR)

### Логические
- `pg_dump` (одна БД), `pg_dumpall` (кластер целиком, роли).
- Формат `-Fc` (custom) — позволяет `pg_restore -j N` (parallel).
- Подходит для небольших БД и миграций версии.

### Физические + WAL = PITR
`pg_basebackup` + архивирование WAL → можно восстановить на любой момент времени:
```conf
archive_mode = on
# Пример «для обучения» — не использовать в проде: cp не гарантирует fsync,
# при аварии ОС в архиве может оказаться «битый» WAL-файл.
archive_command = 'test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
```
Восстановление до нужного времени:
```
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2026-04-10 14:30:00+00'
```

> ⚠️ **Для прода используйте готовые инструменты** — pgBackRest, WAL-G, Barman. Они делают `fsync` после копирования, сжимают/шифруют WAL, параллелят заливку в S3/GCS, управляют retention и поддерживают **инкрементальные / дифференциальные** base backup'ы (pgBackRest — штатно уже много лет, WAL-G — через `backup-push --delta`). В самом ядре PostgreSQL инкрементальный `pg_basebackup --incremental` + сборка через `pg_combinebackup` появились в **PG 17**. Официальная документация прямо предупреждает, что «простой `cp`» не подходит для production: https://www.postgresql.org/docs/current/continuous-archiving.html#BACKUP-ARCHIVING-WAL

Инструменты: **pgBackRest** (лучший для продакшна), **WAL-G**, **Barman**.

### Правило 3-2-1
3 копии, 2 разных носителя, 1 offsite. **Регулярно проверяйте восстановление** — бэкап без теста не существует.

---

## 14. Безопасность

- **pg_hba.conf** — правила аутентификации. `scram-sha-256` (не `md5`).
- **Row-Level Security (RLS)** — фильтрация на уровне строк:
```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id')::int);
-- в приложении: SET app.tenant_id = '42';
```
- TLS для соединений (`ssl = on`).
- Разделение ролей: отдельный пользователь для миграций (owner) и для приложения (только DML).
- `pgcrypto` для хеширования/шифрования чувствительных полей.
- Не храните секреты в JSONB без шифрования.

---

## 15. Конфигурация и тюнинг под нагрузку

Стартовые значения для сервера 32 ГБ RAM, SSD, OLTP:
```
shared_buffers = 8GB                # ~25% RAM
effective_cache_size = 24GB         # ~75% RAM
work_mem = 32MB                     # *max_connections*parallel_workers — осторожно
maintenance_work_mem = 2GB
wal_buffers = 16MB
checkpoint_timeout = 15min
max_wal_size = 8GB
min_wal_size = 2GB
random_page_cost = 1.1              # для SSD (HDD = 4)
effective_io_concurrency = 200      # SSD
max_connections = 200               # за PgBouncer
default_statistics_target = 200
```
Используйте [pgtune](https://pgtune.leopard.in.ua/) как starting point.

Для аналитики — увеличьте `work_mem`, `max_parallel_workers_per_gather`. Для OLTP — внимание к `synchronous_commit` (можно `off` для некритичных данных — огромный прирост, но риск потерять последние транзакции при крахе).

---

## 16. Интеграция в Symfony 6.4 / Doctrine

### Установка

composer.json:
```json
{
    "require": {
        "php": ">=8.2",
        "symfony/framework-bundle": "^6.4",
        "doctrine/doctrine-bundle": "^2.11",
        "doctrine/orm": "^2.17",
        "doctrine/doctrine-migrations-bundle": "^3.3"
    }
}
```

`.env`:
```
DATABASE_URL="postgresql://app:pass@127.0.0.1:6432/app?serverVersion=16&charset=utf8"
```

`config/packages/doctrine.yaml`:
```yaml
doctrine:
    dbal:
        url: '%env(resolve:DATABASE_URL)%'
        # Для pdo_pgsql специальных driverOptions обычно не требуется.
        # Стандартные атрибуты PDO (ERRMODE_EXCEPTION) Doctrine выставляет сама.
        # Если нужно задать параметры сессии для КАЖДОГО нового соединения
        # (после pgbouncer в режиме transaction SET без LOCAL не переживёт),
        # делайте это через search_path/options в DSN или в ConnectionEventSubscriber.
        server_version: '16'
        charset: 'UTF8'
        types:
            jsonb: App\Doctrine\Type\JsonbType
        mapping_types:
            jsonb: jsonb
            _text: string

    orm:
        auto_generate_proxy_classes: true
        naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware
        auto_mapping: true
        mappings:
            App:
                is_bundle: false
                type: attribute
                dir: '%kernel.project_dir%/src/Entity'
                prefix: 'App\Entity'
```

### Сущность с современными PG-типами
```php
<?php
namespace App\Entity;

use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Uid\Uuid;

#[ORM\Entity]
#[ORM\Table(name: 'orders')]
#[ORM\Index(columns: ['status'], name: 'idx_orders_status')]
class Order
{
    #[ORM\Id]
    #[ORM\Column(type: 'uuid', unique: true)]
    private Uuid $id;

    #[ORM\Column(type: Types::DECIMAL, precision: 12, scale: 2)]
    private string $total;

    #[ORM\Column(type: 'jsonb', options: ['jsonb' => true])]
    private array $meta = [];

    #[ORM\Column(type: Types::DATETIMETZ_IMMUTABLE)]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(length: 20)]
    private string $status = 'new';

    public function __construct()
    {
        $this->id = Uuid::v7();
        $this->createdAt = new \DateTimeImmutable();
    }
    // getters/setters...
}
```

### Кастомный тип JSONB
```php
<?php
declare(strict_types=1);

namespace App\Doctrine\Type;

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\JsonType;

/**
 * Кастомный Doctrine-тип для нативного PostgreSQL JSONB.
 *
 * Зачем: встроенный JsonType кодирует/декодирует JSON, но объявляет колонку как JSON
 * (текст), а нам нужно именно JSONB — бинарный формат с поддержкой GIN-индексов и
 * операторов @>, ?, #>>. Класс меняет только SQL-декларацию, сериализация наследуется.
 *
 * Совместимость: Doctrine DBAL 3.x (Symfony 6.4 / ORM 2.17).
 * В DBAL 4.x метод requiresSQLCommentHint() удалён как устаревший.
 */
final class JsonbType extends JsonType
{
    public const NAME = 'jsonb';

    public function getSQLDeclaration(array $column, AbstractPlatform $platform): string
    {
        return 'JSONB';
    }

    public function getName(): string
    {
        return self::NAME;
    }

    /**
     * Для DBAL 3.x: если не вернуть true, Doctrine не сможет отличить JSON от JSONB
     * в schema-diff и будет генерировать лишние миграции.
     */
    public function requiresSQLCommentHint(AbstractPlatform $platform): bool
    {
        return true;
    }
}
```

### Миграции
```bash
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```
Для индексов `CONCURRENTLY` используйте «ручные» миграции с `$this->addSql('CREATE INDEX CONCURRENTLY ...')` и отключайте транзакцию миграции:
```php
public function isTransactional(): bool { return false; }
```

### Native SQL и сложные запросы
```php
$sql = <<<SQL
    SELECT id, email
    FROM users
    WHERE (created_at, id) < (:lastCreated, :lastId)
    ORDER BY created_at DESC, id DESC
    LIMIT 20
SQL;

$rows = $em->getConnection()->executeQuery($sql, [
    'lastCreated' => $lastCreated->format(DATE_ATOM),
    'lastId' => $lastId,
])->fetchAllAssociative();
```

### Транзакции и retry на serialization failure
```php
<?php
namespace App\Service;

use Doctrine\DBAL\Exception\RetryableException;
use Doctrine\ORM\EntityManagerInterface;

class TransferService
{
    public function __construct(private EntityManagerInterface $em) {}

    public function transfer(int $from, int $to, string $amount): void
    {
        $attempts = 0;
        while (true) {
            try {
                $this->em->getConnection()
                    ->executeStatement('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
                $this->em->wrapInTransaction(function () use ($from, $to, $amount) {
                    $conn = $this->em->getConnection();
                    $conn->executeStatement(
                        'UPDATE accounts SET balance = balance - :a WHERE id = :id',
                        ['a' => $amount, 'id' => $from]
                    );
                    $conn->executeStatement(
                        'UPDATE accounts SET balance = balance + :a WHERE id = :id',
                        ['a' => $amount, 'id' => $to]
                    );
                });
                return;
            } catch (RetryableException $e) {
                if (++$attempts >= 3) {
                    throw $e;
                }
                usleep(100_000 * $attempts); // backoff
            }
        }
    }
}
```

### Primary / Read replica
```yaml
doctrine:
    dbal:
        default_connection: default
        connections:
            default:
                url: '%env(DATABASE_URL)%'
                replicas:
                    replica1:
                        url: '%env(DATABASE_REPLICA_URL)%'
```
Doctrine автоматически отправляет writes на primary, но reads по умолчанию тоже могут идти на primary. Чтобы читать с реплики:
```php
$em->getConnection()->ensureConnectedToReplica();
// ... read-only запросы
```
Любой `executeStatement` / начало транзакции переключит на primary.

### Symfony Messenger через Postgres
```yaml
framework:
    messenger:
        transports:
            async: "doctrine://default?queue_name=default"
```
Под капотом — таблица `messenger_messages` + `SELECT ... FOR UPDATE SKIP LOCKED`. Отлично для MVP / малой-средней нагрузки, не требует отдельного брокера.

### LISTEN/NOTIFY в PHP
`LISTEN/NOTIFY` через PDO ограничен: PDO не отдаёт уведомления напрямую, нужно опрашивать `pg_notifies()` из расширения `pgsql` (нативное соединение), либо использовать вторую связь через `pg_connect`.

```php
// Отдельное соединение через pgsql (НЕ PDO), т.к. нужен pg_get_notify()
$conn = pg_connect('host=127.0.0.1 dbname=app user=app password=pass');
pg_query($conn, 'LISTEN order_created');

while (true) {
    // Блокируемся на сокете до прихода уведомления (нет busy-loop)
    $sockets = [pg_socket($conn)];
    $w = $e = [];
    if (stream_select($sockets, $w, $e, 30) > 0) {
        pg_consume_input($conn);
        while ($notify = pg_get_notify($conn, PGSQL_ASSOC)) {
            // $notify['payload'] — строка до 8000 байт
        }
    }
}
```

> ⚠️ Под **PgBouncer в режиме transaction** `LISTEN/NOTIFY` НЕ РАБОТАЕТ, т.к. подписка живёт только на конкретном backend-процессе. Для листенера подключайтесь к PG напрямую или через PgBouncer в режиме `session`.

---

## 17. Сравнение PostgreSQL и MySQL

| Критерий | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Модель процессов | Процесс на коннект (нужен PgBouncer) | Потоки |
| MVCC | В самих таблицах (bloat → VACUUM) | В undo-логе |
| DDL в транзакции | Да (atomic migrations) | Нет (implicit commit) |
| Типы данных | JSONB, arrays, range, inet, uuid, enum, custom | JSON (без полноценных индексов по пути), ограниченный набор |
| Индексы | B-tree, Hash, GIN, GiST, SP-GiST, BRIN, Bloom, partial, expression, covering | B-tree, Hash (Memory), R-tree (MyISAM), fulltext |
| Full-text | Встроенный, многоязычный, GIN | FULLTEXT (InnoDB, ngram для CJK) |
| CTE/рекурсия | Да, давно | Да с 8.0 |
| Window functions | Да | Да с 8.0 |
| Materialized views | Да | Нет (эмуляция через таблицы) |
| Партиционирование | Декларативное, гибкое | Есть, но ограничено |
| Репликация | Streaming (физ.), логическая (PG10+) | Binlog-based (statement/row/mixed), GTID |
| Кластер/HA | Patroni, repmgr (внешние) | InnoDB Cluster / Group Replication (встроено) |
| Шардинг | Citus, FDW, ручной | Vitess, MySQL Cluster (NDB), ручной |
| CHECK constraints | Полноценные | С 8.0 (раньше игнорировались) |
| Последовательности | Настоящие sequences + IDENTITY | AUTO_INCREMENT (на таблицу) |
| Расширения | Огромная экосистема (PostGIS, TimescaleDB, pg_stat_statements) | Плагины, но меньше |
| Case-sensitivity | Чувствителен по умолчанию (для идентификаторов — lowercase, если без кавычек) | Зависит от collation/FS |
| Производительность на простых OLTP | Очень высокая | Часто чуть быстрее на простом key-value |
| Аналитика | Сильнее (parallel, CTE, window, JIT) | Слабее |

**Когда выбрать MySQL:** простой CRUD, нужна совместимость, команда знает только MySQL, managed-хостинг дешевле.
**Когда PostgreSQL:** сложные схемы, JSON, геоданные, аналитика, строгая целостность, расширяемость.

---

## 18. Типичные ошибки и анти-паттерны

1. **`SELECT *`** в проде — ломается при добавлении колонок, читает лишнее.
2. **`OFFSET` на больших страницах** — используйте keyset pagination.
3. **`timestamp` вместо `timestamptz`** — получите ад с часовыми поясами.
4. **Хранение денег во `float`/`double`** — только `numeric`.
5. **Индексация всего подряд** — замедляет INSERT/UPDATE, раздувает WAL.
6. **Отсутствие PgBouncer** — на 500+ коннектах PG умирает.
7. **Длинные транзакции** — блокируют VACUUM, растёт bloat и xid age.
8. **`DELETE` вместо партиционирования** на логах — медленно, порождает bloat.
9. **`ALTER TABLE ... ADD COLUMN ... DEFAULT ...` на больших таблицах** в старых PG — переписывал всю таблицу. С PG 11+ безопасно для non-volatile default.
10. **Не ставить `NOT NULL` + default** при добавлении колонок — получите NULL в старых строках.
11. **`CREATE INDEX` без `CONCURRENTLY`** в проде — блокирует запись.
12. **Игнорирование `pg_stat_statements` и `auto_explain`** — летите вслепую.
13. **Доверять `LIKE 'foo%'` без `text_pattern_ops`** при не-C collation:
```sql
CREATE INDEX ON t (name text_pattern_ops);  -- чтобы LIKE 'foo%' шёл по индексу
```
14. **Миграции через Doctrine без ручного контроля** для `CONCURRENTLY` / партиционирования.

---

## 19. Zero-downtime миграции схемы

Самая недооценённая тема в продакшне. Любая DDL-операция в PG берёт блокировку того или иного уровня; понимание какая — критично для боевых релизов.

### Таблица безопасности операций

| Операция | Блокировка | Безопасна без даунтайма? |
|---|---|---|
| `CREATE INDEX` | `ShareLock` (блокирует запись) | ❌ — используйте `CONCURRENTLY` |
| `CREATE INDEX CONCURRENTLY` | Не блокирует запись | ✅ |
| `ADD COLUMN` без default | `AccessExclusive`, но мгновенно | ✅ |
| `ADD COLUMN ... DEFAULT <const>` (PG 11+) | Мгновенно, метаданные | ✅ |
| `ADD COLUMN ... DEFAULT now()` (volatile) | Переписывает всю таблицу | ❌ |
| `ADD COLUMN NOT NULL` без default | Перепроверяет все строки | ⚠️ на большой таблице |
| `DROP COLUMN` | `AccessExclusive`, мгновенно (данные остаются до VACUUM) | ✅ |
| `ALTER COLUMN TYPE` | Перезаписывает таблицу | ❌ (почти всегда) |
| `ADD CONSTRAINT ... CHECK` | `AccessExclusive` + скан | ⚠️ используйте `NOT VALID` + `VALIDATE CONSTRAINT` |
| `ADD FOREIGN KEY` | Скан обеих таблиц | ⚠️ аналогично: `NOT VALID` → `VALIDATE` |
| `RENAME COLUMN` | `AccessExclusive`, мгновенно | ⚠️ ломает старый код — нужен **expand/contract** |

### Паттерн Expand / Migrate / Contract

Любое «переименование» или «изменение типа» в 0-downtime делается в три релиза:

1. **Expand** — добавляем новое поле/таблицу, код пишет В ОБА (старое и новое).
2. **Migrate** — бэкфилим данные батчами, читаем из нового.
3. **Contract** — удаляем старое поле в следующем релизе.

### Пример: замена колонки `email VARCHAR(100)` на `email CITEXT`

```sql
-- Релиз 1: добавить новую колонку
ALTER TABLE users ADD COLUMN email_new CITEXT;

-- Релиз 1 (триггер двойной записи на время миграции)
CREATE OR REPLACE FUNCTION sync_email() RETURNS trigger AS $$
BEGIN NEW.email_new := NEW.email; RETURN NEW; END;
$$ LANGUAGE plpgsql;
CREATE TRIGGER trg_sync_email BEFORE INSERT OR UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION sync_email();

-- Фоновый backfill батчами, чтобы не держать длинных транзакций и не раздувать WAL
-- (запускаем из Symfony Command в цикле)
UPDATE users SET email_new = email
WHERE id IN (SELECT id FROM users WHERE email_new IS NULL LIMIT 1000 FOR UPDATE SKIP LOCKED);

-- Релиз 2: приложение читает email_new, продолжает писать оба
-- Релиз 3: удалить триггер, ALTER TABLE ... DROP COLUMN email, RENAME email_new -> email
```

### Добавление NOT NULL без блокировки
```sql
-- 1) добавить CHECK без проверки
ALTER TABLE users ADD CONSTRAINT users_email_nn CHECK (email IS NOT NULL) NOT VALID;
-- 2) проверить постепенно (берёт SHARE UPDATE EXCLUSIVE — не блокирует DML)
ALTER TABLE users VALIDATE CONSTRAINT users_email_nn;
-- 3) с PG 12+: превратить в NOT NULL мгновенно (PG использует уже валидированный CHECK)
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

### Taming Doctrine Migrations

- Не полагайтесь на `auto_generate_diff` — он генерит блокирующие DDL. Всегда читайте SQL и переписывайте под небольшие операции.
- Миграция с `CONCURRENTLY` обязана иметь `isTransactional(): false`.
- Ставьте таймаут блокировки в начале миграции, чтобы упасть быстро, а не уронить сайт:
  ```php
  $this->addSql("SET lock_timeout = '3s'");
  ```

### Инструменты

- **[pgroll](https://github.com/xataio/pgroll)** (Xata, Go) — декларативные миграции по схеме expand / migrate / contract. Поддерживает версионирование через views: приложения старой и новой версии одновременно видят свою «форму» таблицы.
- **[pg-osc](https://github.com/shayonj/pg-osc)** (Ruby) — online schema change по образу `gh-ost`: триггерная двойная запись + копирование в теневую таблицу + swap.
- **[pg_repack](https://github.com/reorg/pg_repack)** — не меняет схему, но переcтраивает таблицу/индексы без длительной эксклюзивной блокировки (альтернатива `VACUUM FULL`).
- Для большинства проектов достаточно дисциплины (expand/contract вручную через Doctrine Migrations) — см. примеры выше.

---

## 20. Materialized Views, представления, updatable views

### Представления (views)
Обычные view — это сохранённый `SELECT`, вычисляется при каждом обращении. Полезны как слой абстракции и для RLS (`CREATE VIEW ... WITH (security_barrier)`).

```sql
CREATE VIEW v_active_users AS
SELECT id, email FROM users WHERE deleted_at IS NULL;
```

### Updatable views
Простые view (один FROM, без агрегации) — updatable автоматически: `INSERT/UPDATE/DELETE` через view работают. Для сложных — используйте `INSTEAD OF` триггеры.

### Materialized views
Физически хранит результат запроса — быстрый селект, но устаревающий.

```sql
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT date_trunc('day', created_at) AS day, sum(total) AS revenue
FROM orders WHERE status = 'paid'
GROUP BY 1;

CREATE UNIQUE INDEX ON mv_daily_revenue (day);  -- нужен для CONCURRENTLY

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;  -- не блокирует читателей
```

Когда применять: тяжёлые агрегаты для дашбордов, OLAP-отчёты, допустимо отставание данных на минуты/часы. Обновляйте из Symfony Scheduler или cron.

---

## 21. Generated columns и IDENTITY

### Generated (STORED) columns
Автоматически вычисляемое поле, физически хранится в строке:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    price_cents INT NOT NULL,
    price_usd NUMERIC(10,2) GENERATED ALWAYS AS (price_cents / 100.0) STORED,
    tsv TSVECTOR GENERATED ALWAYS AS (to_tsvector('simple', name)) STORED
);
```

Плюсы: целостность (нельзя рассогласовать), можно индексировать. Минусы: только `STORED` (VIRTUAL нет, обсуждается в roadmap), выражение должно быть IMMUTABLE.

> ⚠️ **Напоминание про IMMUTABLE.** Для `tsv` используется именно двухаргументная `to_tsvector('simple', name)`, потому что одноаргументная форма `STABLE` и в `GENERATED` не компилируется — см. блок полнотекстового поиска выше.

### IDENTITY vs SERIAL

| | `SERIAL` | `GENERATED ALWAYS AS IDENTITY` |
|---|---|---|
| Стандарт SQL | нет | да |
| Прозрачность (видна sequence) | утечка в DDL | инкапсулирована |
| Можно вставить вручную | да | только `OVERRIDING SYSTEM VALUE` |
| Рекомендовано в PG 10+ | ❌ | ✅ |

---

## 22. TOAST и большие значения

**TOAST** (The Oversized-Attribute Storage Technique) — механизм PG для хранения значений больше ~2KB: столбцы сжимаются и/или выносятся в отдельную «тост-таблицу». Работает **прозрачно**. Нюансы:

- `UPDATE` строки, где меняется только toast-поле, всё равно создаёт новую версию строки (MVCC), но toast-значение может не копироваться — меньше I/O.
- Максимум на строку до TOAST — ~2KB, после TOAST — до 1GB на поле.
- Для больших JSONB / TEXT можно управлять стратегией:
  ```sql
  ALTER TABLE t ALTER COLUMN data SET STORAGE EXTERNAL;  -- не сжимать, хранить внешне
  ALTER TABLE t ALTER COLUMN data SET STORAGE EXTENDED;  -- дефолт: сжимать + внешне
  ```
- `LO` / `bytea` для бинарных файлов: в 99% веб-задач **храните файлы в S3/MinIO**, в БД — только ссылку.

---

## 23. Мониторинг и диагностика live-нагрузки

### Активные соединения и долгие запросы
```sql
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS duration,
       left(query, 200) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

### Кто кого блокирует (blocking tree)
```sql
SELECT blocked.pid       AS blocked_pid,
       blocked.query     AS blocked_query,
       blocking.pid      AS blocking_pid,
       blocking.query    AS blocking_query,
       now() - blocked.query_start AS waiting
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY (pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

### Неиспользуемые индексы
```sql
SELECT schemaname, relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Кеш-hit ratio
```sql
SELECT sum(blks_hit)::float / nullif(sum(blks_hit+blks_read),0) AS cache_hit_ratio
FROM pg_stat_database;
-- < 0.99 — возможно, мал shared_buffers
```

### Отставание реплики
```sql
SELECT application_name, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
       replay_lag
FROM pg_stat_replication;
```

### Прибить зависший запрос
```sql
SELECT pg_cancel_backend(pid);     -- мягко (SIGINT)
SELECT pg_terminate_backend(pid);  -- жёстко (закрыть коннект)
```

### Внешние инструменты
- **pgBadger** — парсер логов, отчёты по медленным запросам.
- **pg_stat_statements** — обязательно в проде.
- **pgwatch2**, **postgres_exporter + Prometheus + Grafana** — для алертов.
- **pgAdmin / DBeaver / DataGrip** — для ручного анализа.

---

## 24. Bulk-операции: COPY, batch insert, streaming

### COPY — самый быстрый способ загрузки
```sql
COPY users(email, name) FROM STDIN WITH (FORMAT csv);
```

В PHP (рекомендуемый путь — расширение `pgsql`, функция `pg_copy_from`, принимающая готовый массив строк в TSV-формате; TSV безопаснее CSV: достаточно экранировать `\t`, `\n`, `\r` и `\\`):

```php
// Нативное соединение через расширение pgsql (НЕ PDO) — PDO не умеет COPY.
$pgsql = pg_connect(getenv('PG_DSN_NATIVE'));

$escape = static function (string $v): string {
    return strtr($v, [
        "\\" => '\\\\',
        "\t" => '\\t',
        "\n" => '\\n',
        "\r" => '\\r',
    ]);
};

$lines = [];
foreach ($rows as $r) {
    $lines[] = $escape($r['email']) . "\t" . $escape($r['name']);
}

// pg_copy_from сама выполняет COPY ... FROM STDIN и передаёт все строки разом.
// 3-й аргумент — разделитель полей (по умолчанию \t), 4-й — NULL-метка.
pg_copy_from($pgsql, 'users (email, name)', $lines, "\t", '\\N');
```

> ⚠️ Функции `pg_put_line()` / `pg_end_copy()` **помечены deprecated с PHP 8.4** ([RFC](https://wiki.php.net/rfc/deprecations_php_8_4)). Используйте `pg_copy_from()` / `pg_copy_to()`. Для очень больших файлов удобнее серверный `COPY ... FROM '/path'` (требует прав `pg_read_server_files`) или клиентский `\copy` в psql.

Производительность: `COPY` ~10–50× быстрее `INSERT`. Используйте для импортов, миграций, ETL.

### Multi-row INSERT
```sql
INSERT INTO logs (ts, msg) VALUES
  ('2026-04-01', 'a'), ('2026-04-02', 'b'), ('2026-04-03', 'c')
ON CONFLICT DO NOTHING;
```

### Streaming больших выборок в Doctrine
Нельзя грузить миллион сущностей в память. Используйте `toIterable()`:

```php
$q = $em->createQuery('SELECT u FROM App\Entity\User u');
foreach ($q->toIterable() as $user) {
    // обработка
    $em->detach($user);  // освободить identity map
}
```

Либо работайте на уровне DBAL и используйте **серверный курсор**:
```sql
BEGIN;
DECLARE mycursor CURSOR FOR SELECT * FROM big_table;
FETCH 1000 FROM mycursor;  -- пачка
-- ...
CLOSE mycursor;
COMMIT;
```

---

## 25. Transactional Outbox и idempotency

Частая задача: после коммита заказа отправить событие в Kafka/RabbitMQ. Прямо слать из кода — рискованно: БД закоммитится, а брокер упадёт → рассинхрон.

**Паттерн Outbox:** в той же транзакции записываем событие в таблицу `outbox`, отдельный воркер читает и публикует.

```sql
CREATE TABLE outbox (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    type TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);
CREATE INDEX ON outbox (published_at) WHERE published_at IS NULL;
```

Воркер:
```sql
WITH picked AS (
  SELECT id FROM outbox
   WHERE published_at IS NULL
   ORDER BY id
   LIMIT 100
   FOR UPDATE SKIP LOCKED
)
UPDATE outbox o SET published_at = now()
 FROM picked WHERE o.id = picked.id
 RETURNING o.*;
```

В Symfony идиоматично — **Symfony Messenger** + `doctrine://` транспорт делает именно это. Для интеграции с Kafka — используйте Messenger+транспорт-бридж или Debezium (CDC) с чтением WAL через logical replication.

### Идемпотентность
На уровне API — `Idempotency-Key` в header + `UNIQUE` индекс на `idempotency_key` в таблице запросов. Важная тонкость: `INSERT ... ON CONFLICT DO NOTHING RETURNING id` **не вернёт строку**, если сработал конфликт (строка реально не вставлена, см. [док PG](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)). Корректные варианты:

```sql
-- Вариант 1: DO UPDATE с "no-op", чтобы всегда получить строку через RETURNING.
-- EXCLUDED.column — ссылка на значения, которые пытались вставить.
INSERT INTO requests (idempotency_key, response)
VALUES (:key, :resp)
ON CONFLICT (idempotency_key) DO UPDATE
SET idempotency_key = EXCLUDED.idempotency_key            -- no-op, но даёт RETURNING
RETURNING id, response, (xmax = 0) AS inserted;            -- inserted=true, если это был реальный INSERT

-- Вариант 2: CTE "сначала пробуем вставить, если не получилось — читаем".
WITH ins AS (
    INSERT INTO requests (idempotency_key, response)
    VALUES (:key, :resp)
    ON CONFLICT DO NOTHING
    RETURNING id, response
)
SELECT id, response, TRUE AS inserted FROM ins
UNION ALL
SELECT id, response, FALSE FROM requests WHERE idempotency_key = :key
LIMIT 1;
```

> **Подсказка новичку.** Трюк `xmax = 0` работает, потому что у только что вставленной строки транзакция-удалитель не задана (`xmax = 0`), а у обновлённой `xmax` содержит id блокирующей транзакции. Это общепринятая идиома в PG-сообществе.

---

## 26. Триггеры, правила, event triggers, audit

### Триггер `BEFORE UPDATE` для `updated_at`
```sql
CREATE OR REPLACE FUNCTION touch_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at := now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_touch
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION touch_updated_at();
```

### Аудит через триггер
```sql
CREATE TABLE audit_log (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name TEXT, op CHAR(1), row_id TEXT,
  old JSONB, new JSONB, changed_by TEXT, at TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit() RETURNS trigger AS $$
BEGIN
  INSERT INTO audit_log(table_name, op, row_id, old, new, changed_by)
  VALUES (TG_TABLE_NAME, LEFT(TG_OP,1), COALESCE(NEW.id, OLD.id)::text,
          to_jsonb(OLD), to_jsonb(NEW), current_setting('app.user_id', true));
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

Можно также использовать готовое расширение **pgaudit** или Doctrine-бандл `DoctrineExtensions Loggable`.

### Event triggers
Срабатывают на DDL (например, чтобы логировать все `ALTER TABLE`). Редко нужны в прикладном коде — полезны для платформ/DBA.

### Почему триггеры опасны
- Скрытая логика — «магия», которую не видно в коде.
- Ломают `COPY ... FREEZE` оптимизации.
- Сложно тестировать.
- Частая замена: Doctrine Lifecycle events / PrePersist listeners. Но triggers гарантируют консистентность даже при прямых SQL-вставках (миграции, сторонние ETL).

---

## 27. Unlogged, temp tables, foreign tables, ltree, citext, unaccent

### UNLOGGED tables
Не пишутся в WAL — в разы быстрее, но **теряются при сбое** и не реплицируются. Идеально для кешей, сессий, staging-импорта.
```sql
CREATE UNLOGGED TABLE sessions (...);
```

### TEMP tables
Живут в сессии, автоматически удаляются. Полезны для сложных многошаговых вычислений.
```sql
CREATE TEMP TABLE tmp AS SELECT ... ;
```

### Foreign tables (postgres_fdw, file_fdw)
Чтение внешних источников как обычных таблиц:
```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER remote FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host '10.0.0.5', dbname 'other');
CREATE USER MAPPING FOR app SERVER remote OPTIONS (user 'app', password 'x');
IMPORT FOREIGN SCHEMA public LIMIT TO (reports) FROM SERVER remote INTO ext;
SELECT * FROM ext.reports WHERE ...;  -- pushdown: предикат уйдёт на удалённый сервер
```

### CITEXT — case-insensitive text
```sql
CREATE EXTENSION citext;
CREATE TABLE users (email CITEXT UNIQUE);
SELECT * FROM users WHERE email = 'Ivan@Example.com';  -- найдёт 'ivan@example.com'
```
Индекс работает корректно. Альтернатива — индекс по `LOWER(email)`.

### unaccent — поиск без диакритики
```sql
CREATE EXTENSION unaccent;
SELECT unaccent('Ëcole');  -- 'Ecole'
-- В FTS: to_tsvector('simple', unaccent(name))
```

### ltree — иерархии
Альтернатива рекурсивным CTE для read-heavy деревьев (категории, комментарии):
```sql
CREATE EXTENSION ltree;
CREATE TABLE catalog (path LTREE);
CREATE INDEX ON catalog USING GIST (path);
-- Все потомки "Electronics.Phones":
SELECT * FROM catalog WHERE path <@ 'Electronics.Phones';
```

### hstore
Key-value в одной колонке. Исторически важен, сейчас в 90% случаев заменяется JSONB.

---

## 28. Optimistic locking в Doctrine и deferred constraints

### Optimistic locking
Вместо `FOR UPDATE` — колонка-версия. При UPDATE проверяется, что версия не изменилась. Подходит для «низкоконфликтных» сценариев.

```php
#[ORM\Entity]
class Product
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private int $id;

    #[ORM\Column]
    #[ORM\Version]
    private int $version = 1;
    // ...
}
```
При конкурентной записи Doctrine бросит `OptimisticLockException` — ловите и показывайте пользователю «данные обновились, перезагрузите».

### Deferred constraints
FK и UNIQUE можно отложить до COMMIT — удобно для циклических ссылок и сложных миграций данных:
```sql
ALTER TABLE a ADD CONSTRAINT fk_b FOREIGN KEY (b_id) REFERENCES b(id) DEFERRABLE INITIALLY DEFERRED;

BEGIN;
SET CONSTRAINTS ALL DEFERRED;
-- ... операции, временно нарушающие FK ...
COMMIT;  -- здесь проверка
```

---

## 29. Таймауты, read-only транзакции, защита от «зависших» коннектов

### Важнейшие параметры (ставьте всегда!)
```sql
-- На роль приложения:
ALTER ROLE app SET statement_timeout = '30s';        -- максимум на запрос
ALTER ROLE app SET lock_timeout = '5s';              -- ждать блокировку
ALTER ROLE app SET idle_in_transaction_session_timeout = '60s';  -- добить «забытые» BEGIN
```
`idle_in_transaction_session_timeout` **спасает от главной причины bloat и wraparound** — транзакций, забытых в коде (например, падение процесса после `BEGIN`).

> **Куда ставить таймауты в Symfony-приложении.** Самый простой и надёжный путь — на уровне роли БД: `ALTER ROLE app SET statement_timeout = '30s'`. Тогда любое подключение этой роли получает лимит автоматически, даже из cron/воркеров, которые программист забыл настроить. Для долгих миграций и отчётов создавайте **отдельную роль** (`reports`) с более щедрым лимитом — не правьте `statement_timeout` в рантайме, это легко забыть вернуть назад.

### Read-only транзакции
```sql
BEGIN READ ONLY;
-- или:
SET TRANSACTION READ ONLY;
```
В Doctrine 2.17+: `$em->getConnection()->setTransactionIsolation(...)` и нативно `SET TRANSACTION READ ONLY`. Включайте для отчётов — PG может лучше оптимизировать, не будет записывать hint-bits.

### keepalives
Для соединений через NAT/LB настройте:
```
tcp_keepalives_idle = 60
tcp_keepalives_interval = 10
tcp_keepalives_count = 3
```

---

## 30. SQLSTATE и обработка ошибок в Doctrine

Doctrine DBAL 3 маппит SQLSTATE коды в типизированные исключения (пакет `doctrine/dbal/src/Exception`):

| SQLSTATE | Что значит | Исключение DBAL |
|---|---|---|
| `23505` | unique_violation | `UniqueConstraintViolationException` |
| `23503` | foreign_key_violation | `ForeignKeyConstraintViolationException` |
| `23502` | not_null_violation | `NotNullConstraintViolationException` |
| `40001` | serialization_failure | `DeadlockException` (наследует `RetryableException`) — имя класса историческое, реального deadlock при 40001 нет |
| `40P01` | deadlock_detected | `DeadlockException` (`RetryableException`) |
| `57014` | query_canceled | `Driver\Exception` |
| `08006` | connection_failure | `ConnectionException` |
| `57P01` | admin_shutdown | `ConnectionLost` |

> **Нюанс Doctrine DBAL 3.** Класс `DeadlockException` исторически покрывает и deadlock (`40P01`), и serialization failure (`40001`), поэтому по классу исключения вы не отличите одну ситуацию от другой. Если это важно для метрик — смотрите `$e->getSQLState()` или `$e->getPrevious()->getCode()`. Оба кейса всегда **ретраятся** одинаково — ловите `RetryableException` и повторяйте транзакцию с backoff.

```php
try {
    $em->persist($user); $em->flush();
} catch (UniqueConstraintViolationException $e) {
    throw new DomainException('Email уже используется');
} catch (RetryableException $e) {
    // retry
}
```

---

## 31. Тестирование кода, работающего с PostgreSQL

### Принципы
1. **Тестируйте на той же СУБД**, что и в проде (не SQLite in-memory). Разные диалекты JSONB, `ON CONFLICT`, array ops.
2. **Изоляция тестов** через транзакционный rollback — пакет `dama/doctrine-test-bundle`:
   ```yaml
   # config/packages/test/framework.yaml
   when@test:
       dama_doctrine_test:
           enable_static_connection: true
           enable_static_meta_data_cache: true
   ```
   Каждый тест — одна транзакция, в конце `ROLLBACK`. Скорость х10.
3. **Фикстуры** — `doctrine/doctrine-fixtures-bundle`.
4. **Testcontainers for PHP** — поднимать PG в Docker на каждый запуск CI.
5. **Схема** — накатывайте миграции, не `schema:create` (тестируются реальные миграции).

### Пример
```php
use DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class OrderRepositoryTest extends KernelTestCase
{
    public function testCreateOrder(): void
    {
        $em = static::getContainer()->get('doctrine')->getManager();
        $order = new Order(/*...*/);
        $em->persist($order); $em->flush();
        self::assertSame(1, $em->getRepository(Order::class)->count([]));
    }
    // после теста — транзакция автоматически откатится
}
```

---

## 32. PgBouncer: тонкости для PHP/Symfony

**Режимы:**
- `session` — соединение клиента = одно серверное на всё время. Ломает пулинг при persistent connections.
- `transaction` — **рекомендуемый для PHP**. Серверное соединение возвращается в пул после COMMIT/ROLLBACK.
- `statement` — после каждого statement; запрещает multi-statement транзакции.

### Ограничения `transaction` mode
- ❌ `LISTEN/NOTIFY`, `WITH HOLD CURSOR`, `SET` (без `LOCAL`) — привязаны к серверному бэкенду.
- ❌ **Prepared statements на стороне клиента** ломаются: PDO по умолчанию использует server-side prepared (`PDO::ATTR_EMULATE_PREPARES = false`). После возврата в пул PG-сессия может уже не иметь этого prepared statement.
- ✅ Решение для PHP: либо включить эмуляцию:
  ```php
  // в Doctrine (дополнительно к DBAL)
  $params['driverOptions'][\PDO::ATTR_EMULATE_PREPARES] = true;
  ```
  либо использовать **PgBouncer 1.21+**, который поддерживает протокольные prepared statements прозрачно.
- ✅ Используйте `SET LOCAL` (действует до конца транзакции) вместо `SET`:
  ```sql
  BEGIN;
  SET LOCAL app.tenant_id = '42';
  -- запросы с RLS
  COMMIT;
  ```

### Конфиг-пример
```
[databases]
app = host=127.0.0.1 port=5432 dbname=app

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 25    # серверных соединений на БД+пользователя
reserve_pool_size = 5
server_reset_query = DISCARD ALL
```

> **Подсказка новичку.** `server_reset_query = DISCARD ALL` нужен, чтобы после возврата серверного соединения в пул на нём не осталось «наведённого» состояния: temp-таблиц, `SET`, prepared statements, LISTEN-подписок.
> - В режиме **`session`** — **обязательно** задавать, иначе следующий клиент получит «грязный» бэкенд.
> - В режиме **`transaction`** — PgBouncer по умолчанию **игнорирует** этот параметр (считается, что COMMIT/ROLLBACK уже сбрасывает большую часть состояния). Чтобы принудительно выполнять `DISCARD ALL` и в `transaction` mode, есть отдельный ключ `server_reset_query_always = 1` — включать только при реальных проблемах, это лишний round-trip на каждую транзакцию.
> См. официальную доку: https://www.pgbouncer.org/config.html#server_reset_query

---

## 33. Проверочные вопросы с ответами

> Ссылки ведут на соответствующие разделы этого же документа (Obsidian-совместимые).

> [!question]- Что такое MVCC и как это влияет на UPDATE в PostgreSQL?
> MVCC — механизм многоверсионности: каждая строка имеет версии с `xmin`/`xmax`. `UPDATE` не меняет строку на месте, а создаёт новую версию; старая становится dead tuple и должна быть убрана `VACUUM`. Следствия: читатели не блокируют писателей; растёт bloat; нужен autovacuum.
>
> 🔗 [[#3. MVCC, транзакции, уровни изоляции]]

> [!question]- Чем `REPEATABLE READ` в PostgreSQL отличается от стандарта SQL?
> В PG `REPEATABLE READ` реализован через snapshot isolation — фантомных чтений нет (хотя стандарт допускает). `SERIALIZABLE` (SSI) дополнительно устраняет serialization anomalies.
>
> 🔗 [[#3. MVCC, транзакции, уровни изоляции]]

> [!question]- Как реализовать очередь задач на PostgreSQL без Redis?
> `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` внутри транзакции. Symfony Messenger с транспортом `doctrine://` делает это из коробки.
>
> 🔗 [[#4. Блокировки (locking)]]

> [!question]- Когда использовать GIN, а когда GiST?
> GIN — «обратный» индекс, быстр для поиска элементов в массивах, jsonb, FTS; медленно обновляется. GiST — сбалансированное дерево, хорошо для геометрии, range, exclusion constraints; быстрее в update, но медленнее в select для FTS.
>
> 🔗 [[#6. Индексы: все виды и когда какой использовать]]

> [!question]- Как ускорить `LIKE '%x%'` на большой таблице?
> Включить `pg_trgm` и создать GIN-индекс `USING GIN (col gin_trgm_ops)`.
>
> 🔗 [[#6. Индексы: все виды и когда какой использовать]]

> [!question]- Что такое Index Only Scan и какие условия для него?
> План читает данные только из индекса, не заходя в таблицу. Требуется: все нужные колонки в индексе (или через `INCLUDE`), и страница таблицы помечена all-visible в visibility map (обновляется VACUUM).
>
> 🔗 [[#6. Индексы: все виды и когда какой использовать]]

> [!question]- Что такое bloat и как его лечить без даунтайма?
> Раздувание из-за dead tuples. Лечение без блокировок — `pg_repack` или `REINDEX CONCURRENTLY` для индексов. `VACUUM FULL` требует эксклюзивной блокировки.
>
> 🔗 [[#8. VACUUM, autovacuum, bloat]]

> [!question]- Как правильно пагинировать большие наборы?
> Keyset pagination: `WHERE (created_at, id) < (:last_created, :last_id) ORDER BY ... DESC LIMIT N`. Это работает по индексу и не деградирует с ростом номера страницы, в отличие от `OFFSET`.
>
> 🔗 [[#7. Планировщик, EXPLAIN, оптимизация запросов]]

> [!question]- Когда партиционировать таблицу?
> Когда она превышает ~десятки-сотни ГБ, есть естественный ключ диапазона (время/tenant), запросы фильтруются по этому ключу, и/или нужно быстрое удаление старых данных (`DROP PARTITION`).
>
> 🔗 [[#10. Партиционирование]]

> [!question]- Разница между физической и логической репликацией?
> Физическая — побайтовая копия кластера через WAL, реплики read-only, версия совпадает, реплицируется всё. Логическая — DML конкретных таблиц через publication/subscription, можно между версиями, можно подмножество; не реплицирует DDL.
>
> 🔗 [[#11. Репликация, высокая доступность]]

> [!question]- Есть ли у PostgreSQL встроенное шардирование?
> Нет. Используют Citus (расширение), `postgres_fdw` + партиционирование, или шардирование на уровне приложения. Для HA — Patroni + etcd.
>
> 🔗 [[#12. Шардирование и масштабирование]]

> [!question]- Зачем нужен PgBouncer?
> PG создаёт процесс на каждое соединение — это дорого. PgBouncer держит пул коротких транзакций поверх небольшого числа серверных коннектов. В PHP (короткие запросы) используют режим `transaction`.
>
> 🔗 [[#32. PgBouncer: тонкости для PHP/Symfony]]

> [!question]- Как сделать PITR?
> Включить `archive_mode` + архивацию WAL, снять base backup (`pg_basebackup` / pgBackRest). Восстановить: развернуть base backup, указать `restore_command` и `recovery_target_time`.
>
> 🔗 [[#13. Бэкапы и восстановление (PITR)]]

> [!question]- Какой тип хранить деньги?
> `numeric(p, s)` (`DECIMAL` в Doctrine). Никогда не `float`/`double` — потеря точности.
>
> 🔗 [[#5. Типы данных]]

> [!question]- `timestamp` vs `timestamptz`?
> Всегда `timestamptz`. Хранится в UTC, корректно работает с часовыми поясами.
>
> 🔗 [[#5. Типы данных]]

> [!question]- Как в Symfony/Doctrine использовать JSONB с индексами?
> Зарегистрировать кастомный `JsonbType`, объявить mapping `jsonb => jsonb`, в миграции создать `CREATE INDEX ... USING GIN (column jsonb_path_ops)`. Запросы через нативный SQL или DQL-функции расширения `scienta/doctrine-jsonb-postgresql`.
>
> 🔗 [[#16. Интеграция в Symfony 6.4 / Doctrine]]

> [!question]- Как безопасно создать индекс в проде?
> `CREATE INDEX CONCURRENTLY ...`. В Doctrine Migrations — `isTransactional(): false` и ручной SQL.
>
> 🔗 [[#6. Индексы: все виды и когда какой использовать]]

> [!question]- Как обеспечить мультитенантность на уровне БД?
> Row-Level Security: политика `USING (tenant_id = current_setting('app.tenant_id')::int)`. Приложение выставляет `SET app.tenant_id` в начале каждой транзакции/запроса.
>
> 🔗 [[#14. Безопасность]]

> [!question]- Как ловить медленные запросы в проде?
> Включить `pg_stat_statements` и `auto_explain` с `log_min_duration`. Анализировать топ по `total_exec_time`.
>
> 🔗 [[#7. Планировщик, EXPLAIN, оптимизация запросов]]

> [!question]- Как защититься от `serialization_failure` (40001)?
> Оборачивать бизнес-транзакции в retry-цикл с экспоненциальной задержкой, ловить `Doctrine\DBAL\Exception\RetryableException`.
>
> 🔗 [[#16. Интеграция в Symfony 6.4 / Doctrine]]

> [!question]- Чем PG сильнее MySQL для аналитики?
> Parallel query, JIT, богатые window/CTE, materialized views, FILTER clause, GROUPING SETS/CUBE/ROLLUP, лучший планировщик на сложных JOIN.
>
> 🔗 [[#17. Сравнение PostgreSQL и MySQL]]

> [!question]- Что такое TXID wraparound и почему это опасно?
> Счётчик транзакций 32-битный; если autovacuum не freezит старые строки, БД останавливается для предотвращения повреждения. Мониторить `age(datfrozenxid)`.
>
> 🔗 [[#8. VACUUM, autovacuum, bloat]]

> [!question]- Как устроен LISTEN/NOTIFY и где применять?
> Легковесный pub/sub внутри БД. Подходит для инвалидации кешей, уведомлений воркерам. Не замена Kafka: нет durability, ограничение размера payload 8000 байт.
>
> 🔗 [[#9. Расширенные возможности: CTE, Window, JSONB, полнотекстовый поиск]]

> [!question]- Как делать UPSERT?
> `INSERT ... ON CONFLICT (...) DO UPDATE SET ... EXCLUDED.*`.
>
> 🔗 [[#9. Расширенные возможности: CTE, Window, JSONB, полнотекстовый поиск]]

> [!question]- Главные анти-паттерны с PG?
> `SELECT *`, `OFFSET` на больших страницах, `timestamp` без TZ, float для денег, отсутствие PgBouncer, длинные транзакции, индекс без `CONCURRENTLY`, игнор `pg_stat_statements`.
>
> 🔗 [[#18. Типичные ошибки и анти-паттерны]]

---

## 34. Источники

### Официальная документация PostgreSQL
- PostgreSQL 16 (LTS-like ветка, используется в примерах): https://www.postgresql.org/docs/16/
- PostgreSQL 17 (актуальная на 2025–2026, `uuidv7()`, incremental backup, improved VACUUM): https://www.postgresql.org/docs/17/
- Release notes 17: https://www.postgresql.org/docs/release/17.0/
- MVCC и уровни изоляции: https://www.postgresql.org/docs/current/mvcc.html
- Индексы: https://www.postgresql.org/docs/current/indexes.html
- EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- Партиционирование: https://www.postgresql.org/docs/current/ddl-partitioning.html
- Логическая репликация: https://www.postgresql.org/docs/current/logical-replication.html
- Row Security: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- `pg_stat_statements`: https://www.postgresql.org/docs/current/pgstatstatements.html
- `auto_explain`: https://www.postgresql.org/docs/current/auto-explain.html
- Непрерывное архивирование (WAL / PITR): https://www.postgresql.org/docs/current/continuous-archiving.html
- TOAST: https://www.postgresql.org/docs/current/storage-toast.html
- Generated columns: https://www.postgresql.org/docs/current/ddl-generated-columns.html

### Инфраструктура
- pgBackRest: https://pgbackrest.org/
- WAL-G: https://github.com/wal-g/wal-g
- Patroni: https://patroni.readthedocs.io/
- Citus: https://www.citusdata.com/
- PgBouncer (конфиг): https://www.pgbouncer.org/config.html
- PgBouncer — поддержка prepared statements (1.21+): https://www.pgbouncer.org/features.html
- pgroll (Xata): https://github.com/xataio/pgroll
- pg-osc (shayonj): https://github.com/shayonj/pg-osc
- pg_repack: https://github.com/reorg/pg_repack

### Symfony / Doctrine
- Symfony Doctrine Bundle 6.4: https://symfony.com/doc/6.4/doctrine.html
- Symfony Messenger (Doctrine transport): https://symfony.com/doc/6.4/messenger.html
- Doctrine DBAL 3 — типы: https://www.doctrine-project.org/projects/doctrine-dbal/en/3.8/reference/types.html
- Doctrine DBAL 3 — исключения / SQLSTATE: https://www.doctrine-project.org/projects/doctrine-dbal/en/3.8/reference/exceptions.html
- Doctrine ORM 2.17: https://www.doctrine-project.org/projects/doctrine-orm/en/2.17/
- Symfony UID (UUID v7): https://symfony.com/doc/current/components/uid.html
- DAMA Doctrine Test Bundle: https://github.com/dmaicher/doctrine-test-bundle

### Сравнения и обзоры
- PostgreSQL «About»: https://www.postgresql.org/about/
- MySQL документация: https://dev.mysql.com/doc/

---

> Этот файл — живой справочник. Дополняйте его заметками из практики: конкретные инциденты, запросы, которые выигрывали от индекса, настройки под вашу нагрузку.

