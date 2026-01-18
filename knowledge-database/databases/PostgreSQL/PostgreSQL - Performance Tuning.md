---
tags:
  - postgresql
  - performance
  - optimization
  - tuning
  - configuration
aliases:
  - PostgreSQL Оптимізація
  - PostgreSQL Performance Tuning
  - Налаштування PostgreSQL
created: 2025-01-17
topic: PostgreSQL Performance
---

# ⚡ PostgreSQL - Performance Tuning

> [!SUMMARY] TL;DR
> Продуктивність PostgreSQL залежить від **правильних налаштувань, індексів, та розуміння workload**. Дефолтні налаштування розраховані на 1GB RAM - для production треба тюнити. Ключові параметри: shared_buffers, work_mem, effective_cache_size, max_connections + PgBouncer.
> **Key idea:** 80% продуктивності досягається через 20% зусиль - фокус на критичних параметрах та правильних індексах.

## 1. Діагностика: З чого почати?

### Швидкий чек-лист

```sql
-- 1. Поточна конфігурація
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
SHOW max_connections;

-- 2. Повільні запити (потрібно увімкнути log_min_duration_statement)
SELECT
    query,
    calls,
    total_exec_time / 1000 AS total_sec,
    mean_exec_time / 1000 AS mean_sec,
    max_exec_time / 1000 AS max_sec
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 3. Таблиці без індексів з багатьма Seq Scans
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_tup
FROM pg_stat_user_tables
WHERE seq_scan > 1000  -- Багато Seq Scans
  AND idx_scan < 100   -- Мало Index Scans
ORDER BY seq_tup_read DESC;

-- 4. Cache hit ratio (має бути >99%)
SELECT
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

## 2. Критичні параметри пам'яті

### shared_buffers (найважливіший!)

**Що це**: Кеш для даних таблиць та індексів у RAM.

```ini
# postgresql.conf
shared_buffers = 4GB         # Дефолт: 128MB (замало!)
```

**Рекомендації**:

| RAM | shared_buffers | Пояснення |
| :--- | :--- | :--- |
| <1GB | 256MB | Розділяйте з OS |
| 1-4GB | 25% RAM | 1GB для 4GB RAM |
| 4-32GB | 8GB | Не більше 8GB |
| 32GB+ | 8-16GB | Більше немає сенсу |

> [!WARNING] Warning
> **Не ставте > 40% RAM!**
>
> PostgreSQL також покладається на OS cache. Занадто великий shared_buffers = менше для OS cache = гірша performance.

```bash
# Перевірка effective usage
SELECT
    setting::INTEGER * 8 / 1024 AS shared_buffers_mb
FROM pg_settings
WHERE name = 'shared_buffers';

-- Cache hit ratio (має бути >99%)
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

### work_mem

**Що це**: Пам'ять для **кожного** сортування та hash join в запиті.

```ini
work_mem = 64MB              # Дефолт: 4MB (дуже мало!)
```

**Розрахунок**:

```
Потенційне використання = max_connections × work_mem × avg_sorts_per_query

Приклад:
200 connections × 64MB × 2 sorts = 25.6GB (!)
```

**Рекомендації**:
- **OLTP**: 16-32MB (багато коротких запитів)
- **OLAP**: 256MB-1GB (мало складних запитів)
- **Mixed**: 64MB

**Динамічна зміна** для важких запитів:

```sql
-- Тільки для цієї сесії
SET work_mem = '256MB';

-- Виконати важкий запит
SELECT ... ORDER BY ... LIMIT 1000;

-- Повернути дефолт
RESET work_mem;
```

> [!TIP] Tip
> **Моніторте temp files!** Якщо `temp files` > 0 → work_mem замалий:
>
> ```sql
> SELECT
>     datname,
>     temp_files,
>     pg_size_pretty(temp_bytes) AS temp_size
> FROM pg_stat_database
> WHERE temp_files > 0;
> ```

### effective_cache_size

**Що це**: Підказка планувальнику про **скільки RAM доступно для кешу** (shared_buffers + OS cache).

```ini
effective_cache_size = 16GB   # Дефолт: 4GB
```

**Розрахунок**:

```
effective_cache_size = (RAM - OS - applications) × 0.75

Для dedicated PostgreSQL сервера з 32GB RAM:
= (32GB - 2GB OS) × 0.75
= 22.5GB
```

> [!INFO] Info
> `effective_cache_size` **НЕ виділяє пам'ять!** Це тільки hint для планувальника.

### maintenance_work_mem

**Що це**: Пам'ять для VACUUM, CREATE INDEX, ALTER TABLE.

```ini
maintenance_work_mem = 512MB  # Дефолт: 64MB
```

**Рекомендації**:
- **Production**: 512MB - 2GB
- **Для великих таблиць**: До 4-8GB

```sql
-- Тимчасово збільшити для важкої операції
SET maintenance_work_mem = '4GB';
CREATE INDEX CONCURRENTLY idx_huge_table ON huge_table(column);
RESET maintenance_work_mem;
```

## 3. Параметри з'єднань

### max_connections

```ini
max_connections = 100         # Дефолт: 100
```

**Проблема**: Кожне з'єднання = ~10MB RAM.

```
200 connections × 10MB = 2GB RAM тільки на процеси
```

**Рішення**: **PgBouncer** (connection pooler)

```
Application: 1000 connections
      ↓
PgBouncer: Pool до 50 connections
      ↓
PostgreSQL: max_connections = 100
```

Детальніше: [[PostgreSQL - Process Model]]

## 4. Налаштування диску та WAL

### WAL параметри

```ini
# Розмір WAL файлів
wal_buffers = -1              # Дефолт: auto (1/32 shared_buffers, min 64KB)

# Checkpoint параметри
checkpoint_timeout = 15min    # Дефолт: 5min
max_wal_size = 4GB            # Дефолт: 1GB
min_wal_size = 1GB            # Дефолт: 80MB

# Розтягнути checkpoint на 90% інтервалу (плавніша I/O)
checkpoint_completion_target = 0.9  # Дефолт: 0.5
```

**Діагностика**:

```sql
-- Занадто часті checkpoint (погано!)
SELECT
    checkpoints_timed,
    checkpoints_req,         -- Має бути <<< checkpoints_timed
    checkpoint_write_time,
    checkpoint_sync_time
FROM pg_stat_bgwriter;

-- Якщо checkpoints_req багато → збільшити max_wal_size
```

### Синхронізація диску

```ini
# Для SSD (швидкий random I/O)
random_page_cost = 1.1        # Дефолт: 4.0 (для HDD)
seq_page_cost = 1.0           # Дефолт: 1.0

# fsync метод (для Linux)
wal_sync_method = fdatasync   # Дефолт: fdatasync

# Commit delay (НЕ рекомендовано для більшості)
synchronous_commit = on       # Дефолт: on (safe)
# = off → 2-3x швидше, але ризик втрати останніх транзакцій
```

> [!WARNING] Warning
> **synchronous_commit = off** → втрата до 1 секунди транзакцій при краху!
>
> Використовуйте тільки для non-critical даних (логи, analytics).

## 5. Query planner параметри

### Statistics

```ini
# Кількість рядків для ANALYZE (більше = точніші статистики)
default_statistics_target = 100  # Дефолт: 100

# Для важливих колонок можна збільшити
ALTER TABLE users ALTER COLUMN email SET STATISTICS 500;
```

```sql
-- Оновити статистики
ANALYZE;

-- Для конкретної таблиці
ANALYZE users;

-- Перевірити свіжість статистик
SELECT
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 10000;  -- Застарілі статистики
```

### Planner costs

```ini
# SSD налаштування
random_page_cost = 1.1        # HDD: 4.0, SSD: 1.1
seq_page_cost = 1.0

# CPU costs (рідко змінюють)
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025
```

## 6. Autovacuum tuning

```ini
# Global settings
autovacuum = on                              # ✅ ОБОВ'ЯЗКОВО!
autovacuum_max_workers = 3                   # Збільшити для high-churn tables
autovacuum_naptime = 1min                    # Частота перевірки

# Aggressiveness
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2         # 20% таблиці

# I/O throttling
autovacuum_vacuum_cost_limit = 200           # Збільшити для швидшого VACUUM
autovacuum_vacuum_cost_delay = 2ms
```

**Per-table tuning**:

```sql
-- Агресивніший VACUUM для активних таблиць
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- 5%
    autovacuum_vacuum_threshold = 100,
    autovacuum_vacuum_cost_limit = 1000
);
```

Детальніше: [[PostgreSQL - VACUUM Guide]]

## 7. Logging та моніторинг

### Повільні запити

```ini
# Логувати запити повільніше 100ms
log_min_duration_statement = 100             # ms

# Детальні логи
log_statement = 'none'                       # 'none', 'ddl', 'mod', 'all'
log_duration = off                           # on → логувати тривалість ВСІХ запитів

# Де логувати
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
```

### pg_stat_statements (must have!)

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

```sql
-- Увімкнути розширення
CREATE EXTENSION pg_stat_statements;

-- Top повільні запити
SELECT
    substring(query, 1, 100) AS short_query,
    calls,
    total_exec_time / 1000 AS total_sec,
    mean_exec_time / 1000 AS mean_sec,
    max_exec_time / 1000 AS max_sec,
    stddev_exec_time / 1000 AS stddev_sec
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Скинути статистику
SELECT pg_stat_statements_reset();
```

## 8. Оптимізація запитів

### 1. EXPLAIN ANALYZE завжди

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM users WHERE email = 'user@example.com';
```

Детальніше: [[PostgreSQL - EXPLAIN Guide]]

### 2. Правильні індекси

```sql
-- ❌ ПОГАНО: Seq Scan
SELECT * FROM users WHERE created_at > '2024-01-01';

-- ✅ ДОБРЕ: Створити індекс
CREATE INDEX idx_users_created ON users(created_at);

-- ✅ ЩЕ КРАЩЕ: Partial index для recent data
CREATE INDEX idx_users_recent
ON users(created_at)
WHERE created_at > '2024-01-01';
```

Детальніше: [[PostgreSQL - Index Types]]

### 3. Index-only scans

```sql
-- Включити ВСІБ потрібні колонки в індекс
CREATE INDEX idx_users_email_name ON users(email, name);

-- Index-only scan (швидко!)
SELECT email, name FROM users WHERE email = 'user@example.com';
```

### 4. Connection pooling

```python
# ❌ ПОГАНО: Створювати з'єднання на кожен request
def handle_request():
    conn = psycopg2.connect(...)
    # ...
    conn.close()

# ✅ ДОБРЕ: Connection pool
from psycopg2.pool import ThreadedConnectionPool

pool = ThreadedConnectionPool(minconn=5, maxconn=20, dsn=...)

def handle_request():
    conn = pool.getconn()
    try:
        # ...
    finally:
        pool.putconn(conn)
```

### 5. LIMIT + OFFSET антипаттерн

```sql
-- ❌ ПОГАНО: OFFSET 10000 → сканує 10100 рядків
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 100 OFFSET 10000;

-- ✅ ДОБРЕ: Keyset pagination
SELECT * FROM products
WHERE created_at < '2025-01-15 12:00:00'
ORDER BY created_at DESC
LIMIT 100;
```

## 9. Hardware рекомендації

### RAM

| Workload | Min RAM | Recommended |
| :--- | :--- | :--- |
| Small (< 10GB DB) | 4GB | 8GB |
| Medium (10-100GB) | 16GB | 32GB |
| Large (100GB - 1TB) | 64GB | 128GB |
| Very Large (> 1TB) | 256GB+ | 512GB+ |

**Правило**: `RAM >= DB size × 1.5` для кешування

### CPU

- **OLTP**: Більше ядер (8-16)
- **OLAP**: Швидкі ядра + багато RAM

```ini
# Використовувати parallel queries (PostgreSQL 9.6+)
max_parallel_workers_per_gather = 4  # Ядра для одного запиту
max_parallel_workers = 8             # Всього workers
```

### Disk

| Type | IOPS | Use Case |
| :--- | :--- | :--- |
| HDD | ~100 | ❌ НЕ рекомендовано |
| SATA SSD | ~10k | ✅ Small-medium workloads |
| NVMe SSD | ~100k+ | ✅ High-performance |

**SSD налаштування**:

```ini
random_page_cost = 1.1   # HDD: 4.0
seq_page_cost = 1.0
effective_io_concurrency = 200  # HDD: 1, SSD: 200
```

## 10. Configuration templates

### Dedicated server (32GB RAM, 8 cores, SSD)

```ini
# Memory
shared_buffers = 8GB
work_mem = 64MB
maintenance_work_mem = 1GB
effective_cache_size = 24GB

# Connections
max_connections = 100

# WAL
wal_buffers = 16MB
checkpoint_timeout = 15min
max_wal_size = 4GB
checkpoint_completion_target = 0.9

# Disk
random_page_cost = 1.1
effective_io_concurrency = 200

# Planner
default_statistics_target = 100

# Autovacuum
autovacuum_max_workers = 4
autovacuum_vacuum_cost_limit = 500

# Parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8

# Logging
log_min_duration_statement = 100
shared_preload_libraries = 'pg_stat_statements'
```

### OLTP high-concurrency (PgBouncer)

```ini
# PostgreSQL
max_connections = 100
shared_buffers = 8GB
work_mem = 16MB

# PgBouncer
[pgbouncer]
pool_mode = transaction
default_pool_size = 25
max_client_conn = 1000
```

### OLAP analytics

```ini
shared_buffers = 16GB
work_mem = 256MB
maintenance_work_mem = 4GB
effective_cache_size = 48GB

max_parallel_workers_per_gather = 8
max_parallel_workers = 16
```

## 11. Troubleshooting чек-лист

### Повільні запити?

1. ✅ `EXPLAIN ANALYZE` для діагностики
2. ✅ Перевірити індекси
3. ✅ `ANALYZE` для оновлення статистик
4. ✅ Збільшити `work_mem` для складних запитів

### Високий I/O wait?

1. ✅ Перевірити `shared_buffers` (cache hit ratio >99%)
2. ✅ Збільшити `max_wal_size` (менше checkpoint)
3. ✅ SSD замість HDD
4. ✅ Autovacuum не встигає? Збільшити workers

### Out of memory?

1. ✅ Зменшити `max_connections` або використати PgBouncer
2. ✅ Зменшити `work_mem`
3. ✅ Перевірити memory leaks в застосунку

### Bloat?

1. ✅ `VACUUM ANALYZE`
2. ✅ Агресивніший autovacuum
3. ✅ `REINDEX CONCURRENTLY` для роздутих індексів

Детальніше: [[PostgreSQL - VACUUM Guide]]

## 12. Benchmark інструменти

### pgbench (вбудований)

```bash
# Ініціалізація test DB
pgbench -i -s 50 testdb

# Запустити benchmark (10 клієнтів, 1000 транзакцій)
pgbench -c 10 -t 1000 testdb

# Custom script
pgbench -f custom_queries.sql -c 50 -T 60 testdb
```

### pg_stat_statements

```sql
-- Top повільні запити за останній час
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

## 13. Моніторинг tools

- **Prometheus + postgres_exporter** - метрики
- **Grafana** - візуалізація
- **pgAdmin** - GUI управління
- **pgBadger** - аналіз логів

## 14. Пов'язані теми

- [[PostgreSQL - Memory Configuration|Налаштування пам'яті детально]]
- [[PostgreSQL - Index Types|Типи індексів]]
- [[PostgreSQL - EXPLAIN Guide|Аналіз запитів]]
- [[PostgreSQL - VACUUM Guide|Autovacuum tuning]]
- [[PostgreSQL - Process Model|Connection pooling]]

## 15. Додаткові ресурси

- [PostgreSQL Performance Tuning](https://www.postgresql.org/docs/18/performance-tips.html)
- [PGTune](https://pgtune.leopard.in.ua/) - auto-generate config
- [PostgreSQL Wiki: Tuning](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)
- [PostgreSQL Performance Blog](https://www.cybertec-postgresql.com/en/blog/)

---

**Останнє оновлення**: 2025-01-17
**Версія**: PostgreSQL 18
