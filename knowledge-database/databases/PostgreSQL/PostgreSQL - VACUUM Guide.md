---
tags:
  - postgresql
  - vacuum
  - autovacuum
  - maintenance
  - bloat
  - performance
  - mvcc
aliases:
  - PostgreSQL VACUUM
  - PostgreSQL Autovacuum
  - Обслуговування PostgreSQL
created: 2025-01-17
topic: PostgreSQL Maintenance
---

# 🧹 PostgreSQL - VACUUM Guide

> Керування мертвими кортежами та підтримка здоров'я бази даних

## 🎯 Навіщо потрібен VACUUM?

Через MVCC (Multi-Version Concurrency Control), PostgreSQL **не видаляє старі версії рядків** одразу при UPDATE/DELETE. Натомість створюються **мертві кортежі (dead tuples)**, які займають місце доки VACUUM їх не очистить.

### Проблема без VACUUM

```sql
-- Початковий стан
INSERT INTO products (id, name, price) VALUES (1, 'Widget', 100);
-- Row: [xmin=100, xmax=0] ✅ alive

-- Оновлення
UPDATE products SET price = 120 WHERE id = 1;
-- Old row: [xmin=100, xmax=101] ☠️ dead tuple
-- New row: [xmin=101, xmax=0] ✅ alive

-- Ще оновлення
UPDATE products SET price = 150 WHERE id = 1;
-- Old row 1: [xmin=100, xmax=101] ☠️ dead
-- Old row 2: [xmin=101, xmax=102] ☠️ dead
-- New row: [xmin=102, xmax=0] ✅ alive

-- В таблиці: 3 версії рядка (2 мертві!)
```

**Наслідки bloat**:
- 🔴 Таблиця займає більше місця на диску
- 🔴 Повільніші Seq Scans (треба пропускати мертві кортежі)
- 🔴 Індекси роздуті (вказують на мертві кортежі)
- 🔴 Менше ефективного shared_buffers кешу

## 🔄 Що робить VACUUM?

```
┌────────────────────────────────────────┐
│  Функції VACUUM                        │
├────────────────────────────────────────┤
│  1. Позначає мертві кортежі як вільні  │
│  2. Оновлює FSM (Free Space Map)       │
│  3. Оновлює статистики планувальника   │
│  4. Запобігає transaction ID wraparound│
│  5. Оновлює visibility map             │
└────────────────────────────────────────┘
```

**Важливо**: Стандартний VACUUM **НЕ повертає місце ОС**!  
Він просто позначає простір як доступний для повторного використання.

## ⚙️ Типи VACUUM

### 1. VACUUM (стандартний)

**Легке очищення** без блокування таблиці.

```sql
-- Одна таблиця
VACUUM products;

-- Вся база даних
VACUUM;

-- З verbose output
VACUUM VERBOSE products;
```

**Що відбувається**:
1. Сканує таблицю для мертвих кортежів
2. Позначає їх як вільні в FSM
3. Оновлює visibility map
4. **НЕ блокує** читання/запис

**Коли використовувати**: Регулярне обслуговування (автоматично через Autovacuum).

### 2. VACUUM FULL

**Повна перепаковка** таблиці для повернення місця ОС.

```sql
VACUUM FULL products;
```

**Що відбувається**:
1. Створює **нову копію** таблиці без мертвих кортежів
2. Копіює живі рядки
3. Перебудовує всі індекси
4. Видаляє стару таблицю
5. **Блокує таблицю** (EXCLUSIVE LOCK)

**⚠️ Обережно**:
- 🔴 Потребує 2x дискового простору
- 🔴 Блокує всі операції (читання/запис)
- 🔴 Довга операція на великих таблицях
- 🔴 Спричиняє I/O spike

**Коли використовувати**: Рідко! Тільки коли bloat критичний (>50%).

**Альтернатива**: pg_repack (без блокування).

### 3. VACUUM FREEZE

**Freezing старих transaction IDs** для запобігання wraparound.

```sql
VACUUM FREEZE products;
```

**Що відбувається**:
1. Замінює старі XID на `FrozenTransactionId` (2)
2. Запобігає transaction ID wraparound
3. Оновлює `relfrozenxid` в pg_class

**Коли використовувати**: Автоматично через Autovacuum коли `age(relfrozenxid) > vacuum_freeze_min_age`.

### 4. VACUUM ANALYZE

**VACUUM + оновлення статистик** для планувальника.

```sql
VACUUM ANALYZE products;
```

Еквівалентно:
```sql
VACUUM products;
ANALYZE products;
```

**Рекомендація**: Використовуйте після масових INSERT/UPDATE/DELETE.

## 🤖 Autovacuum (найважливіше!)

**Autovacuum daemon** — це фоновий процес, який автоматично запускає VACUUM коли потрібно.

### Як працює Autovacuum

**Умова запуску**:

```
dead_tuples >= threshold + (scale_factor × live_tuples)
```

**Параметри за дефолтом**:

```sql
-- Глобальні (postgresql.conf)
autovacuum = on  -- Увімкнено (обов'язково!)

autovacuum_vacuum_threshold = 50  -- Мінімум мертвих кортежів
autovacuum_vacuum_scale_factor = 0.2  -- 20% таблиці

autovacuum_vacuum_cost_delay = 2ms  -- Затримка для throttling
autovacuum_vacuum_cost_limit = 200  -- Budget для I/O

autovacuum_max_workers = 3  -- Кількість worker процесів

autovacuum_naptime = 1min  -- Інтервал перевірки
```

**Приклад розрахунку**:

```sql
-- Таблиця: 1,000,000 рядків
-- Dead tuples: 200,000

-- Threshold:
50 + (0.2 × 1,000,000) = 200,050

-- 200,000 < 200,050 → Autovacuum НЕ запуститься
-- Потрібно ще 51 мертвий кортеж
```

### Per-table налаштування

**Override дефолтів** для конкретних таблиць:

```sql
-- Агресивніший VACUUM для активних таблиць
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- 5% замість 20%
    autovacuum_vacuum_threshold = 100
);

-- Рідкісний VACUUM для append-only таблиць
ALTER TABLE logs SET (
    autovacuum_vacuum_scale_factor = 0.5,  -- 50%
    autovacuum_vacuum_threshold = 10000
);

-- Вимкнути Autovacuum (НЕ рекомендується!)
ALTER TABLE temp_table SET (
    autovacuum_enabled = false
);
```

### Моніторинг Autovacuum

```sql
-- Коли остання vacuum/analyze
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_dead_tup,
    n_live_tup,
    n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Вік найстарішої транзакції (wraparound)
SELECT 
    datname,
    age(datfrozenxid) AS age,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY age DESC;
```

**⚠️ Критичні значення**:
- `age(datfrozenxid) > 200M` → Autovacuum запускає aggressive VACUUM
- `age(datfrozenxid) > 2B` → **БД переходить в read-only!** 🔴

### Проблема: Autovacuum не встигає

**Симптоми**:
```sql
SELECT relname, n_dead_tup, n_live_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > n_live_tup * 0.3  -- Bloat >30%
ORDER BY n_dead_tup DESC;
```

**Рішення 1**: Збільшити ресурси Autovacuum

```sql
-- postgresql.conf
autovacuum_max_workers = 6  -- Більше workers (дефолт 3)
autovacuum_vacuum_cost_limit = 2000  -- Більший I/O budget (дефолт 200)
autovacuum_naptime = 30s  -- Частіше перевіряти (дефолт 1min)
```

**Рішення 2**: Агресивніші per-table налаштування

```sql
ALTER TABLE high_churn_table SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- 1%
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_cost_limit = 5000  -- Більший budget для цієї таблиці
);
```

**Рішення 3**: Ручний VACUUM в низький час

```sql
-- Cron job (ночі)
0 2 * * * psql -c "VACUUM ANALYZE high_churn_table;"
```

## 📊 Діагностика Bloat

### Перевірка bloat таблиць

```sql
-- Оцінка bloat через pg_stat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup,
    ROUND(100 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

**Розширення pgstattuple** для точної оцінки:

```sql
CREATE EXTENSION pgstattuple;

-- Детальна статистика bloat
SELECT * FROM pgstattuple('products');

-- Результат:
-- table_len: 10485760 (10 MB)
-- tuple_count: 50000
-- tuple_len: 8000000 (8 MB)
-- dead_tuple_count: 10000
-- dead_tuple_len: 1600000 (1.6 MB)  ← Bloat!
-- free_space: 885760
-- dead_tuple_percent: 15.26  ← 15% bloat
```

### Перевірка bloat індексів

```sql
-- Розмір індексів
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- Bloat через pgstattuple
SELECT * FROM pgstatindex('idx_products_name');

-- Результат:
-- tree_level: 2
-- leaf_pages: 100
-- internal_pages: 1
-- avg_leaf_density: 65.2  ← <90% = bloat
```

**Рішення для bloat індексів**:

```sql
-- Перебудувати індекс БЕЗ блокування
REINDEX INDEX CONCURRENTLY idx_products_name;

-- Або всі індекси таблиці
REINDEX TABLE CONCURRENTLY products;
```

## 🛠️ Best Practices

### 1. Завжди увімкнений Autovacuum

```sql
-- postgresql.conf
autovacuum = on  -- ✅ ОБОВ'ЯЗКОВО!
```

**Ніколи не вимикайте Autovacuum глобально!**

Виключення: можна вимкнути для окремих тимчасових таблиць.

### 2. Налаштування під workload

**OLTP (багато UPDATE/DELETE)**:

```sql
-- Більше workers, агресивніший vacuum
autovacuum_max_workers = 6
autovacuum_vacuum_scale_factor = 0.05  -- 5%
autovacuum_vacuum_cost_limit = 2000
```

**OLAP (більше INSERT, мало UPDATE)**:

```sql
-- Менше workers, рідкісний vacuum
autovacuum_max_workers = 3
autovacuum_vacuum_scale_factor = 0.2  -- 20%
autovacuum_vacuum_cost_limit = 200
```

### 3. HOT Updates оптимізація

**HOT (Heap-Only Tuple)** — оновлення без зміни індексів.

```sql
-- Залишити вільне місце для HOT updates
ALTER TABLE products SET (fillfactor = 80);  -- 20% вільного простору

-- Перебудувати для застосування
VACUUM FULL products;  -- або pg_repack
```

**Коли працює HOT**:
- ✅ UPDATE не змінює індексовані колонки
- ✅ Є вільне місце на heap page (fillfactor)

**Моніторинг**:

```sql
SELECT 
    relname,
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    ROUND(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 1000
ORDER BY n_tup_upd DESC;
```

**Мета**: HOT % > 80-90%

### 4. Wroaround захист

```sql
-- Перевірка віку
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    CASE 
        WHEN age(datfrozenxid) > 1500000000 THEN 'CRITICAL' 🔴
        WHEN age(datfrozenxid) > 1000000000 THEN 'WARNING' ⚠️
        ELSE 'OK' ✅
    END AS status
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

**Параметри**:

```sql
-- postgresql.conf
vacuum_freeze_min_age = 50000000  -- 50M транзакцій (дефолт)
vacuum_freeze_table_age = 150000000  -- 150M (aggressive vacuum)
autovacuum_freeze_max_age = 200000000  -- 200M (forced vacuum)
```

### 5. Maintenance window strategy

```sql
-- Щоденний cron (низький час)
-- 02:00 - VACUUM активних таблиць
0 2 * * * psql -c "VACUUM ANALYZE orders;"
0 2 * * * psql -c "VACUUM ANALYZE products;"

-- Щотижня - REINDEX
0 3 * * 0 psql -c "REINDEX TABLE CONCURRENTLY orders;"

-- Щомісяця - VACUUM FULL для історичних таблиць
0 4 1 * * psql -c "VACUUM FULL historical_data;"
```

## 🚨 Troubleshooting

### Проблема: Autovacuum працює занадто довго

```sql
-- Знайти довгі autovacuum процеси
SELECT 
    pid,
    usename,
    datname,
    state,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
  AND state = 'active'
ORDER BY xact_start;
```

**Рішення**: Збільшити `autovacuum_vacuum_cost_limit`.

### Проблема: Autovacuum блокується

```sql
-- Перевірка блокувань
SELECT 
    a.pid AS blocked_pid,
    a.query AS blocked_query,
    b.pid AS blocking_pid,
    b.query AS blocking_query
FROM pg_stat_activity a
JOIN pg_locks bl ON a.pid = bl.pid
JOIN pg_locks kl ON bl.transactionid = kl.transactionid
JOIN pg_stat_activity b ON b.pid = kl.pid
WHERE a.query LIKE 'autovacuum:%'
  AND bl.pid != kl.pid;
```

**Рішення**: Завершити довгі транзакції.

### Проблема: Transaction ID wraparound warning

```
WARNING: database "mydb" must be vacuumed within 10000000 transactions
```

**Рішення**:

```sql
-- Агресивний VACUUM FREEZE вручну
VACUUM FREEZE;

-- Або для конкретної БД
\c mydb
VACUUM FREEZE;
```

## 📊 Корисні запити

### Dashboard моніторингу

```sql
-- Статус Autovacuum
SELECT 
    COUNT(*) FILTER (WHERE query LIKE 'autovacuum:%') AS autovacuum_workers,
    MAX(now() - xact_start) FILTER (WHERE query LIKE 'autovacuum:%') AS longest_vacuum
FROM pg_stat_activity;

-- Top bloat таблиць
SELECT 
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS size,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 1) AS bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 10;

-- Wraparound статус
SELECT 
    datname,
    age(datfrozenxid) AS age,
    ROUND(100.0 * age(datfrozenxid) / 2000000000, 2) AS wraparound_pct
FROM pg_database
WHERE datname NOT IN ('template0', 'template1');
```

## 🔗 Пов'язані теми

- [[PostgreSQL - Architecture and MVCC|MVCC механізм]]
- [[PostgreSQL - Performance Tuning|Загальна оптимізація]]
- [[PostgreSQL - Memory Configuration|Налаштування пам'яті]]

## 📚 Додаткові ресурси

- [Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- [Autovacuum Tuning](https://www.postgresql.org/docs/18/runtime-config-autovacuum.html)
- [pg_repack](https://github.com/reorg/pg_repack) — альтернатива VACUUM FULL

## 🏷️ Теги

#postgresql #vacuum #autovacuum #maintenance #bloat #performance #mvcc

---

**Останнє оновлення**: 2025-01-17  
**Версія**: PostgreSQL 18
