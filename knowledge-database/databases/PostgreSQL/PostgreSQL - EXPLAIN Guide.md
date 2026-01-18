---
tags:
  - postgresql
  - explain
  - performance
  - optimization
  - query-tuning
  - indexes
aliases:
  - PostgreSQL EXPLAIN
  - PostgreSQL Query Analysis
  - Аналіз запитів PostgreSQL
created: 2025-01-17
topic: PostgreSQL Performance
---

# 🔍 PostgreSQL - EXPLAIN Guide

> Майстерність аналізу планів виконання для оптимізації запитів

## 🎯 Що таке EXPLAIN?

**EXPLAIN** показує план виконання запиту, який обрав PostgreSQL query planner. Це головний інструмент для діагностики повільних запитів.

**EXPLAIN ANALYZE** — додатково **виконує запит** і показує реальні метрики.

## 🚀 Швидкий старт

### Базовий синтаксис

```sql
-- Показати план БЕЗ виконання
EXPLAIN 
SELECT * FROM users WHERE email = 'user@example.com';

-- Виконати і показати реальні метрики
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- З додатковими деталями
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING)
SELECT * FROM users WHERE email = 'user@example.com';
```

### Опції EXPLAIN

| Опція | Призначення | Рекомендація |
|-------|-------------|--------------|
| **ANALYZE** | Виконує запит, показує реальний час | ✅ Завжди для діагностики |
| **BUFFERS** | Показує використання кешу | ✅ Важливо для I/O аналізу |
| **VERBOSE** | Детальний вивід колонок | ⚠️ Для складних запитів |
| **TIMING** | Детальний час кожного вузла | ⚠️ Додає overhead |
| **COSTS** | Показує оцінки cost (дефолт ON) | ✅ Завжди ON |
| **FORMAT** | JSON / XML / YAML | 📊 Для парсинга скриптами |

## 📊 Розуміння output

### Простий приклад

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE id = 123;
```

```
Index Scan using users_pkey on users  
  (cost=0.29..8.31 rows=1 width=524) 
  (actual time=0.015..0.017 rows=1 loops=1)
  Index Cond: (id = 123)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```

**Розшифровка**:

```
Index Scan using users_pkey on users
│
├─ Тип операції: Index Scan
├─ Використаний індекс: users_pkey
└─ Таблиця: users

(cost=0.29..8.31 rows=1 width=524)
│      │      │      │       │
│      │      │      │       └─ Середній розмір рядка (bytes)
│      │      │      └─ Очікувана кількість рядків
│      │      └─ Total cost (умовні одиниці)
│      └─ Startup cost (до початку виводу)
│
└─ Оцінка планувальника (НЕ реальні дані!)

(actual time=0.015..0.017 rows=1 loops=1)
│            │      │       │      │
│            │      │       │      └─ Кількість виконань вузла
│            │      │       └─ Реальна кількість рядків
│            │      └─ Час до останнього рядка (ms)
│            └─ Час до першого рядка (ms)
│
└─ РЕАЛЬНІ метрики від ANALYZE
```

### ⚠️ Важливі моменти

**Cost** — це НЕ мілісекунди!
- Умовні одиниці для порівняння планів
- Базуються на конфігурації: `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`

**Actual time** — це реальний час у мілісекундах:
- Перше число: час до першого рядка
- Друге число: час до останнього рядка

**Loops** — кількість виконань вузла:
- Для nested loops може бути >1
- **Множте actual time на loops** для загального часу

## 🎭 Типи scan операцій

### 1. Sequential Scan (Seq Scan)

**Повне сканування таблиці** рядок за рядком.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE age > 18;
```

```
Seq Scan on users  (cost=0.00..1725.00 rows=50000 width=524)
  Filter: (age > 18)
  Rows Removed by Filter: 10000
```

**Коли відбувається**:
- Немає індексу
- Запит повертає >10-15% рядків (індекс не ефективний)
- Таблиця дуже мала (<10 сторінок)

**✅ Добре**: Для малих таблиць або коли потрібна більшість рядків  
**❌ Погано**: Для великих таблиць з високою селективністю

### 2. Index Scan

**Використання індексу** для пошуку рядків.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';
```

```
Index Scan using idx_users_email on users  
  (cost=0.42..8.44 rows=1 width=524)
  Index Cond: (email = 'user@example.com'::text)
```

**Етапи**:
1. Пошук в індексі (B-tree traversal)
2. Читання heap page (фізичне читання рядка з таблиці)

**✅ Добре**: Для високоселективних запитів (<1-5% рядків)  
**❌ Погано**: Для низької селективності (багато random I/O)

### 3. Index Only Scan

**Читання тільки з індексу** без доступу до таблиці.

```sql
CREATE INDEX idx_users_email_name ON users(email, name);

EXPLAIN ANALYZE
SELECT email, name FROM users WHERE email = 'user@example.com';
```

```
Index Only Scan using idx_users_email_name on users  
  (cost=0.42..8.44 rows=1 width=64)
  Index Cond: (email = 'user@example.com'::text)
  Heap Fetches: 0
```

**Умови**:
- Всі колонки в SELECT є в індексі
- Таблиця **VACUUM**'лена (visibility map актуальна)

**Heap Fetches: 0** ← найкраще!  
**Heap Fetches: 500** ← індекс містить дані, але треба перевірити visibility

**✅ Найшвидший тип сканування**

### 4. Bitmap Index Scan / Bitmap Heap Scan

**Двофазний скан**: спочатку індекс → bitmap → сортування → heap.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 123;
```

```
Bitmap Heap Scan on orders  (cost=12.15..856.28 rows=500 width=128)
  Recheck Cond: (user_id = 123)
  Heap Blocks: exact=450
  ->  Bitmap Index Scan on idx_orders_user  (cost=0.00..12.03 rows=500)
        Index Cond: (user_id = 123)
```

**Як працює**:
1. **Bitmap Index Scan**: Створює bitmap matching рядків
2. **Сортування** bitmap за фізичним розташуванням (page number)
3. **Bitmap Heap Scan**: Читає heap pages **послідовно**

**Переваги**:
- Менше random I/O ніж Index Scan
- Може поєднувати декілька індексів (BitmapAnd, BitmapOr)

**✅ Добре**: Для середньої селективності (5-20% рядків)  
**❌ Погано**: Overhead для дуже високої селективності

### 5. BitmapAnd / BitmapOr

**Комбінування індексів** для складних умов.

```sql
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE user_id = 123 AND status = 'pending';
```

```
Bitmap Heap Scan on orders  
  Recheck Cond: ((user_id = 123) AND (status = 'pending'))
  ->  BitmapAnd
        ->  Bitmap Index Scan on idx_orders_user
              Index Cond: (user_id = 123)
        ->  Bitmap Index Scan on idx_orders_status
              Index Cond: (status = 'pending')
```

**BitmapAnd**: Перетин результатів (AND)  
**BitmapOr**: Об'єднання результатів (OR)

## 🔀 Join операції

### 1. Nested Loop

**Найпростіший join**: для кожного рядка A шукає matching в B.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.email = 'user@example.com';
```

```
Nested Loop  (cost=0.71..24.76 rows=1 width=652)
  ->  Index Scan using idx_users_email on users u  
        (cost=0.42..8.44 rows=1)
        Index Cond: (email = 'user@example.com')
  ->  Index Scan using idx_orders_user on orders o  
        (cost=0.29..16.31 rows=5)
        Index Cond: (user_id = u.id)
```

**Складність**: O(N × M) без індексів, O(N × log M) з індексом на inner table

**✅ Добре**: 
- Outer table має мало рядків
- Inner table має індекс на join колонці

**❌ Погано**: 
- Обидві таблиці великі
- Немає індексу на inner table

### 2. Hash Join

**Створення hash table** в пам'яті для швидкого lookup.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN products p ON o.product_id = p.id;
```

```
Hash Join  (cost=1234.00..5678.90 rows=10000 width=256)
  Hash Cond: (o.product_id = p.id)
  ->  Seq Scan on orders o  (cost=0.00..2345.00 rows=100000)
  ->  Hash  (cost=678.00..678.00 rows=5000 width=128)
        Buckets: 8192  Batches: 1  Memory Usage: 512kB
        ->  Seq Scan on products p  (cost=0.00..678.00 rows=5000)
```

**Етапи**:
1. Сканувати **меншу** таблицю (products)
2. Створити hash table в `work_mem`
3. Сканувати **більшу** таблицю (orders) і пробувати lookup

**Memory Usage** — критично важливий параметр!

**✅ Добре**:
- Обидві таблиці великі
- Одна таблиця вміщається в `work_mem`
- Equality join (=)

**❌ Погано**:
- Hash table не вміщується в пам'ять (**Batches > 1**)
- Non-equality joins (<, >)

### 3. Merge Join

**Сортування обох таблиць** і злиття відсортованих списків.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
ORDER BY u.id;
```

```
Merge Join  (cost=1234.56..5678.90 rows=10000 width=256)
  Merge Cond: (o.user_id = u.id)
  ->  Index Scan using idx_orders_user on orders o  
        (cost=0.29..2345.67 rows=100000)
  ->  Index Scan using users_pkey on users u  
        (cost=0.42..1234.56 rows=50000)
```

**Етапи**:
1. Сортувати обидві таблиці (або використати індекс)
2. Злити відсортовані списки (O(N + M))

**✅ Добре**:
- Дані вже відсортовані (індекс)
- Потрібен відсортований результат
- Equality join

**❌ Погано**:
- Потреба в сортуванні (великий overhead)

## 📈 Аналіз BUFFERS

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'user@example.com';
```

```
Index Scan using idx_users_email on users  
  (cost=0.42..8.44 rows=1 width=524) 
  (actual time=0.025..0.027 rows=1 loops=1)
  Index Cond: (email = 'user@example.com'::text)
  Buffers: shared hit=4
Planning:
  Buffers: shared hit=12
Planning Time: 0.156 ms
Execution Time: 0.052 ms
```

**Метрики Buffers**:

| Метрика | Значення |
|---------|----------|
| **shared hit** | Сторінки знайдені в shared_buffers (кеш) ✅ |
| **shared read** | Сторінки зчитані з диску ⚠️ |
| **shared written** | Сторінки записані на диск 🐌 |
| **temp read/written** | Використання temp файлів (дуже повільно!) 🔴 |

**Золоте правило**: `shared hit >> shared read` ✅

**Приклад проблеми**:

```
Buffers: shared hit=10 read=50000 written=5000
         temp read=10000 temp written=10000
```

**Проблеми**:
- 🔴 Багато `shared read` → потрібно більше `shared_buffers`
- 🔴 `temp read/written` → запит потребує більше `work_mem`

## 🎯 Практичні сценарії оптимізації

### Сценарій 1: Seq Scan замість Index Scan

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE created_at > '2020-01-01';
```

```
Seq Scan on users  (cost=0.00..1725.00 rows=90000 width=524)
  Filter: (created_at > '2020-01-01')
  Rows Removed by Filter: 10000
```

**Проблема**: Планувальник вважає Seq Scan дешевшим

**Причини**:
1. Застарілі статистики
2. Запит повертає >15% рядків
3. Немає індексу

**Рішення**:

```sql
-- 1. Оновити статистику
ANALYZE users;

-- 2. Створити індекс
CREATE INDEX idx_users_created ON users(created_at);

-- 3. Partial index для recent data
CREATE INDEX idx_users_recent 
ON users(created_at) 
WHERE created_at > '2024-01-01';
```

### Сценарій 2: Повільний Nested Loop

```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN products p ON o.product_id = p.id;
```

```
Nested Loop  (cost=0.00..500000.00 rows=100000 width=256)
  ->  Seq Scan on orders o  (cost=0.00..2000.00 rows=100000)
  ->  Seq Scan on products p  (cost=0.00..5.00 rows=1)
        Filter: (p.id = o.product_id)
        Rows Removed by Filter: 499
```

**Проблема**: Seq Scan на inner table → O(N × M)

**Рішення**:

```sql
-- Створити індекс на join колонці
CREATE INDEX idx_products_id ON products(id);

-- Після:
Nested Loop  (cost=0.29..1234.56 rows=100000 width=256)
  ->  Seq Scan on orders o
  ->  Index Scan using idx_products_id on products p
        Index Cond: (id = o.product_id)
```

### Сценарій 3: Hash Join з Batches

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table1 l1
JOIN large_table2 l2 ON l1.key = l2.key;
```

```
Hash Join  (cost=50000.00..200000.00 rows=1000000 width=256)
  Hash Cond: (l1.key = l2.key)
  Buffers: shared hit=10000 read=5000, temp read=50000 temp written=50000
  ->  Seq Scan on large_table1 l1
  ->  Hash  (cost=25000.00..25000.00 rows=500000 width=128)
        Buckets: 65536  Batches: 8  Memory Usage: 32MB  🔴
```

**Проблема**: `Batches: 8` + temp read/written ← hash table не вміщається в `work_mem`

**Рішення**:

```sql
-- Збільшити work_mem для сесії
SET work_mem = '256MB';

-- Або глобально (postgresql.conf)
work_mem = 64MB
```

**Після**:

```
Hash  (cost=25000.00..25000.00 rows=500000 width=128)
  Buckets: 65536  Batches: 1  Memory Usage: 128MB  ✅
```

### Сценарій 4: Index Not Used

```sql
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

```
Seq Scan on users  (cost=0.00..1725.00 rows=50 width=524)
  Filter: (lower(email) = 'user@example.com')
```

**Проблема**: Функція `LOWER()` не може використати звичайний індекс

**Рішення**:

```sql
-- Expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Після:
Index Scan using idx_users_email_lower on users
  Index Cond: (lower(email) = 'user@example.com')
```

## 🛠️ Корисні команди

### Статистика планувальника

```sql
-- Оновити статистику
ANALYZE users;
ANALYZE;  -- Вся БД

-- Перевірити свіжість статистики
SELECT 
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 1000;
```

### Параметри планувальника

```sql
-- Показати поточні налаштування
SHOW seq_page_cost;
SHOW random_page_cost;
SHOW cpu_tuple_cost;
SHOW work_mem;

-- Тимчасово змінити (для експериментів)
SET random_page_cost = 1.1;  -- SSD (дефолт 4.0 для HDD)
SET work_mem = '256MB';
```

### Forced plan для тестування

```sql
-- Вимкнути Nested Loop
SET enable_nestloop = off;

-- Вимкнути Seq Scan
SET enable_seqscan = off;

-- Не робіть це в production! Тільки для діагностики.
```

## 📊 Візуалізація планів

**Онлайн інструменти**:
- [explain.depesz.com](https://explain.depesz.com/) ← найпопулярніший
- [explain.dalibo.com](https://explain.dalibo.com/)
- [tatiyants.com/pev](https://tatiyants.com/pev/)

**JSON формат для парсинга**:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM users WHERE email = 'user@example.com';
```

## 🔗 Пов'язані теми

- [[PostgreSQL - Index Types|Типи індексів]]
- [[PostgreSQL - Query Optimization|Оптимізація запитів]]
- [[PostgreSQL - Memory Configuration|Налаштування пам'яті]]

## 📚 Додаткові ресурси

- [EXPLAIN Documentation](https://www.postgresql.org/docs/18/using-explain.html)
- [EXPLAIN Depesz](https://www.depesz.com/tag/unexplainable/)
- [Postgres EXPLAIN Visualizer](https://tatiyants.com/pev/)

## 🏷️ Теги

#postgresql #explain #performance #optimization #query-tuning #indexes

---

**Останнє оновлення**: 2025-01-17  
**Версія**: PostgreSQL 18
