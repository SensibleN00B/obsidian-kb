---
tags:
  - postgresql
  - indexes
  - btree
  - gin
  - gist
  - brin
  - performance
  - optimization
aliases:
  - PostgreSQL Індекси
  - PostgreSQL Index Types
  - Типи індексів PostgreSQL
created: 2025-01-17
topic: PostgreSQL Performance
---

# 📑 PostgreSQL - Index Types

> Детальний гайд по типах індексів: B-tree, Hash, GIN, GiST, SP-GiST, BRIN, Bloom

## 🎯 Огляд

PostgreSQL підтримує **7 типів індексів**, кожен оптимізований під специфічні задачі. Правильний вибір індексу може **прискорити запити в 100-1000 разів**.

## 📊 Швидке порівняння

| Тип індексу | Розмір | Швидкість | Кращі Use Cases |
|-------------|--------|-----------|-----------------|
| **B-tree** | Середній | ⚡⚡⚡ | Рівність, діапазони, сортування |
| **Hash** | Малий | ⚡⚡⚡ | Тільки рівність (=) |
| **GIN** | Великий | ⚡⚡ | JSONB, arrays, full-text search |
| **GiST** | Середній | ⚡⚡ | Геодані, nearest-neighbor |
| **SP-GiST** | Малий | ⚡⚡ | Nested data, quad-trees |
| **BRIN** | Дуже малий | ⚡ | Величезні відсортовані таблиці |
| **Bloom** | Малий | ⚡ | Multi-column equality (AND) |

## 1️⃣ B-tree (дефолт)

**Balanced Tree** — найпоширеніший та універсальний індекс.

### Коли використовувати

✅ **Підходить для**:
- Пошук за рівністю: `WHERE id = 123`
- Діапазони: `WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31'`
- Сортування: `ORDER BY name`
- Порівняння: `WHERE age > 18`
- Префіксний пошук: `WHERE name LIKE 'John%'` (але НЕ `LIKE '%John'`)

❌ **НЕ підходить для**:
- Суфіксний пошук: `LIKE '%gmail.com'`
- JSON/Array операції
- Full-text search
- Геопросторові запити

### Приклади

```sql
-- Простий B-tree
CREATE INDEX idx_users_email ON users(email);

-- Використання:
SELECT * FROM users WHERE email = 'user@example.com';

-- Composite index (порядок важливий!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Ефективно:
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2025-01-01';
SELECT * FROM orders WHERE user_id = 123;  -- Перша колонка

-- НЕефективно (пропущена перша колонка):
SELECT * FROM orders WHERE created_at > '2025-01-01';
```

### Partial Index (умовний індекс)

```sql
-- Індекс тільки для активних користувачів
CREATE INDEX idx_active_users 
ON users(email) 
WHERE is_active = true;

-- Індекс тільки для недавніх замовлень
CREATE INDEX idx_recent_orders 
ON orders(created_at) 
WHERE created_at > '2024-01-01';
```

**Переваги**: Менший розмір, швидші запити на підмножині даних.

### Expression Index (обчислювальний індекс)

```sql
-- Індекс на lowercase email
CREATE INDEX idx_users_email_lower 
ON users(LOWER(email));

-- Використання:
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- Індекс на JSON поле
CREATE INDEX idx_users_metadata_age 
ON users((metadata->>'age')::int);
```

## 2️⃣ Hash

**Хеш-таблиця** для швидкого пошуку за рівністю.

### Коли використовувати

✅ **Підходить для**:
- Тільки пошук за рівністю `=`
- Колонки з високою кардинальністю (багато унікальних значень)

❌ **НЕ підходить для**:
- Діапазони (`<`, `>`, `BETWEEN`)
- Сортування
- Префіксний пошук

### Приклади

```sql
-- Hash index
CREATE INDEX idx_users_uuid_hash ON users USING HASH (uuid);

-- Використання:
SELECT * FROM users WHERE uuid = 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11';
```

**⚠️ Важливо**: 
- До PostgreSQL 10 Hash індекси **не були WAL-logged** (втрачалися при краху)
- З PostgreSQL 10+ вони надійні, але **B-tree часто швидший** через кращий caching

**Рекомендація**: Використовуйте B-tree, якщо сумніваєтесь.

## 3️⃣ GIN (Generalized Inverted Index)

**Інвертований індекс** для складних типів даних.

### Коли використовувати

✅ **Підходить для**:
- JSONB запити: `@>`, `?`, `?|`, `?&`
- Array операції: `&&`, `@>`, `<@`
- Full-text search: `@@`
- Пошук елементів всередині складних структур

❌ **НЕ підходить для**:
- Прості скалярні значення (використовуйте B-tree)

### Приклади для JSONB

```sql
-- JSONB таблиця
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- GIN індекс (дефолтний operator class)
CREATE INDEX idx_products_data_gin 
ON products USING GIN (data);

-- Ефективні запити:
SELECT * FROM products WHERE data @> '{"category": "electronics"}';
SELECT * FROM products WHERE data ? 'brand';
SELECT * FROM products WHERE data ?| array['brand', 'model'];
```

**Два operator classes**:

```sql
-- jsonb_ops (дефолт) — більше операторів, більший індекс
CREATE INDEX idx_data_ops ON products USING GIN (data jsonb_ops);

-- jsonb_path_ops — менший індекс, швидший пошук, тільки @>, @?, @@
CREATE INDEX idx_data_path_ops ON products USING GIN (data jsonb_path_ops);
```

**Рекомендація**: Використовуйте `jsonb_path_ops` для `@>` запитів (containment).

### Приклади для Arrays

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    tags TEXT[]
);

CREATE INDEX idx_articles_tags_gin ON articles USING GIN (tags);

-- Ефективні запити:
SELECT * FROM articles WHERE tags @> ARRAY['postgresql', 'database'];
SELECT * FROM articles WHERE tags && ARRAY['python', 'javascript'];
```

### Full-Text Search

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    content_tsv TSVECTOR
);

-- Генерувати tsvector при вставці
CREATE TRIGGER tsvector_update 
BEFORE INSERT OR UPDATE ON documents
FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(content_tsv, 'pg_catalog.english', content);

-- GIN індекс для full-text
CREATE INDEX idx_documents_fts 
ON documents USING GIN (content_tsv);

-- Пошук:
SELECT * FROM documents 
WHERE content_tsv @@ to_tsquery('postgresql & performance');
```

**Trade-offs GIN**:
- 🐌 **Повільні INSERT/UPDATE** (треба оновлювати інвертований індекс)
- ⚡ **Швидкі SELECT**
- 💾 **Великий розмір індексу**

**Оптимізація**:

```sql
-- fastupdate для batch updates (дефолт ON)
CREATE INDEX idx_data_gin ON products USING GIN (data)
WITH (fastupdate = on, gin_pending_list_limit = 4096);
```

## 4️⃣ GiST (Generalized Search Tree)

**R-tree подібна структура** для багатовимірних даних.

### Коли використовувати

✅ **Підходить для**:
- Геопросторові дані (PostGIS)
- Nearest-neighbor search
- Range types
- Складні типи даних з кастомними операторами

❌ **НЕ підходить для**:
- Прості скалярні значення

### Приклади для геоданих (PostGIS)

```sql
-- Розширення PostGIS
CREATE EXTENSION postgis;

CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    geom GEOMETRY(Point, 4326)
);

-- GiST індекс для геометрії
CREATE INDEX idx_locations_geom 
ON locations USING GiST (geom);

-- Запити:
-- Пошук в межах прямокутника
SELECT * FROM locations 
WHERE ST_Contains(
    ST_MakeEnvelope(-74.0, 40.7, -73.9, 40.8, 4326),
    geom
);

-- Nearest-neighbor (10 найближчих)
SELECT * FROM locations 
ORDER BY geom <-> ST_MakePoint(-73.9857, 40.7484, 4326)
LIMIT 10;
```

### Приклади для Range Types

```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_id INT,
    during TSRANGE
);

CREATE INDEX idx_reservations_during 
ON reservations USING GiST (during);

-- Пошук перекриттів
SELECT * FROM reservations 
WHERE during && tsrange('2025-01-15', '2025-01-20');
```

**Trade-offs GiST**:
- 🐌 **Повільніший за GIN** для full-text
- ⚡ **Швидкий nearest-neighbor**
- 💾 **Менший індекс** ніж GIN

## 5️⃣ SP-GiST (Space-Partitioned GiST)

**Quad-trees, k-d trees** для nested та partitioned data.

### Коли використовувати

✅ **Підходить для**:
- Nested data structures
- IP адреси
- Телефонні номери
- Quad-trees

❌ **НЕ підходить для**:
- Прості випадки (використовуйте B-tree)

### Приклади

```sql
-- IP адреси
CREATE TABLE ip_logs (
    ip INET,
    accessed_at TIMESTAMP
);

CREATE INDEX idx_ip_spgist ON ip_logs USING SP-GiST (ip);

-- Пошук підмереж
SELECT * FROM ip_logs WHERE ip << inet '192.168.1.0/24';
```

**Trade-offs SP-GiST**:
- 💾 **Компактніший** ніж GiST для деяких структур
- ⚡ **Швидший** для partitioned data

## 6️⃣ BRIN (Block Range Index)

**Мінімалістичний індекс** для величезних відсортованих таблиць.

### Коли використовувати

✅ **Підходить для**:
- Величезні таблиці (TB+)
- Природно відсортовані дані (логи за датою, IoT sensor data)
- Append-only таблиці

❌ **НЕ підходить для**:
- Невідсортовані дані
- Високоселективні запити

### Приклади

```sql
-- Логи (відсортовані за датою)
CREATE TABLE logs (
    id BIGSERIAL,
    created_at TIMESTAMP DEFAULT NOW(),
    message TEXT
);

-- BRIN індекс (1MB на 1TB даних!)
CREATE INDEX idx_logs_created_brin 
ON logs USING BRIN (created_at);

-- Запити:
SELECT * FROM logs 
WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';
```

**Як працює BRIN**:

```
Таблиця розділена на блоки (128 pages за дефолтом)
┌──────────────────────────────────────┐
│ Block 1: min=2025-01-01, max=2025-01-05 │
├──────────────────────────────────────┤
│ Block 2: min=2025-01-06, max=2025-01-10 │
├──────────────────────────────────────┤
│ Block 3: min=2025-01-11, max=2025-01-15 │
└──────────────────────────────────────┘

Запит WHERE created_at > '2025-01-12'
→ Сканує тільки Block 3
```

**Trade-offs BRIN**:
- 💾 **Дуже малий індекс** (1-2 MB на 1 TB!)
- ⚡ **Повільніший за B-tree** для точкового пошуку
- ✅ **Ідеально для time-series data**

**Налаштування**:

```sql
-- Більше pages per range = менший індекс, але менша точність
CREATE INDEX idx_logs_brin 
ON logs USING BRIN (created_at) 
WITH (pages_per_range = 256);
```

## 7️⃣ Bloom

**Bloom filter** для multi-column equality пошуку.

### Коли використовувати

✅ **Підходить для**:
- Багато колонок з AND умовами
- Низька селективність кожної колонки окремо

❌ **НЕ підходить для**:
- Високоселективні запити (використовуйте B-tree)
- OR умови

### Приклади

```sql
-- Розширення
CREATE EXTENSION bloom;

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    color TEXT,
    size TEXT,
    brand TEXT,
    material TEXT
);

-- Bloom індекс на 4 колонки
CREATE INDEX idx_products_bloom 
ON products USING BLOOM (color, size, brand, material);

-- Ефективний запит:
SELECT * FROM products 
WHERE color = 'red' 
  AND size = 'L' 
  AND brand = 'Nike' 
  AND material = 'cotton';
```

**Альтернатива**: Composite B-tree, але порядок колонок критичний:

```sql
-- B-tree потребує правильного порядку
CREATE INDEX idx_products_btree ON products(color, size, brand, material);

-- Ефективно:
WHERE color = 'red' AND size = 'L' AND brand = 'Nike'

-- НЕефективно (пропущена перша колонка):
WHERE size = 'L' AND brand = 'Nike'
```

**Trade-offs Bloom**:
- 💾 **Малий індекс**
- ⚠️ **False positives** (може повернути зайві рядки для перевірки)
- ✅ **Гнучкість** для будь-якої комбінації колонок

## 🎯 Матриця вибору індексу

| Ваш Use Case | Рекомендований індекс |
|-------------|----------------------|
| `WHERE id = 123` | **B-tree** |
| `WHERE created_at BETWEEN ... AND ...` | **B-tree** |
| `WHERE name LIKE 'John%'` | **B-tree** |
| `WHERE data @> '{"key": "value"}'` | **GIN** (jsonb_path_ops) |
| `WHERE tags @> ARRAY['tag1']` | **GIN** |
| `WHERE content_tsv @@ 'search'` | **GIN** |
| `WHERE ST_Contains(geom, point)` | **GiST** |
| `ORDER BY geom <-> point` | **GiST** |
| `WHERE ip << '192.168.0.0/16'` | **SP-GiST** |
| `WHERE created_at > '2025-01-01'` (TB data) | **BRIN** |
| `WHERE c1='a' AND c2='b' AND c3='c'` | **Bloom** або composite B-tree |

## 🛠️ Практичні поради

### 1. Моніторинг використання індексів

```sql
-- Невикористані індекси
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Найбільші індекси
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 10;
```

### 2. CONCURRENTLY для production

```sql
-- БЕЗ блокування таблиці
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Видалення БЕЗ блокування
DROP INDEX CONCURRENTLY idx_old_index;
```

⚠️ **Обережно**: `CONCURRENTLY` не працює всередині транзакції.

### 3. FILLFACTOR для UPDATE-heavy таблиць

```sql
-- Залишити 20% вільного місця для HOT updates
CREATE INDEX idx_products_name ON products(name)
WITH (fillfactor = 80);
```

**HOT (Heap-Only Tuple)** updates — оновлення без зміни індексу.

### 4. Index-only scans

```sql
-- Індекс містить всі потрібні колонки
CREATE INDEX idx_users_email_name ON users(email, name);

-- Index-only scan (швидко!)
SELECT email, name FROM users WHERE email = 'user@example.com';

-- Потребує додаткового читання таблиці
SELECT email, name, address FROM users WHERE email = 'user@example.com';
```

Перевірка через `EXPLAIN`:
```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT email, name FROM users WHERE email = 'user@example.com';
-- Шукайте "Index Only Scan"
```

## 📊 Розмір та продуктивність

### Приклад на 10M рядків

```sql
CREATE TABLE benchmark (
    id SERIAL PRIMARY KEY,
    category TEXT,
    tags TEXT[],
    data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

| Індекс | Розмір | Build Time | SELECT (ms) |
|--------|--------|------------|-------------|
| B-tree (id) | 214 MB | 30s | 0.1 |
| B-tree (created_at) | 214 MB | 32s | 5 |
| GIN (tags) | 450 MB | 120s | 2 |
| GIN (data jsonb_ops) | 850 MB | 180s | 3 |
| GIN (data jsonb_path_ops) | 520 MB | 140s | 2 |
| BRIN (created_at) | 2 MB | 5s | 15 |

**Висновки**:
- BRIN **в 100 разів менший**, але **повільніший**
- GIN `jsonb_path_ops` **на 40% менший** за `jsonb_ops`
- B-tree найшвидший для простих запитів

## 🔗 Пов'язані теми

- [[PostgreSQL - EXPLAIN Guide|Аналіз планів виконання]]
- [[PostgreSQL - Query Optimization|Оптимізація запитів]]
- [[PostgreSQL - JSONB Guide|JSONB індексування]]

## 📚 Додаткові ресурси

- [PostgreSQL Index Types](https://www.postgresql.org/docs/18/indexes-types.html)
- [Index Concurrency](https://www.postgresql.org/docs/18/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)
- [GIN vs GiST](https://www.postgresql.org/docs/18/textsearch-indexes.html)

## 🏷️ Теги

#postgresql #indexes #btree #gin #gist #brin #performance #optimization

---

**Останнє оновлення**: 2025-01-17  
**Версія**: PostgreSQL 18
