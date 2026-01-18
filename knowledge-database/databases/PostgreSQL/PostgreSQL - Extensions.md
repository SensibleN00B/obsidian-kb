---
tags:
  - postgresql
  - extensions
  - postgis
  - timescaledb
  - pgvector
  - full-text-search
aliases:
  - PostgreSQL Розширення
  - PostgreSQL Extensions
  - Розширення PostgreSQL
created: 2025-01-17
topic: PostgreSQL Ecosystem
---

# 🧩 PostgreSQL - Extensions

> [!SUMMARY] TL;DR
> Сила PostgreSQL - в екосистемі **1000+ розширень**, що додають функціональність від геоданих (PostGIS) до time-series (TimescaleDB) та AI векторного пошуку (pgvector). Розширення інтегруються на рівні ядра БД, працюють швидше за зовнішні інструменти.
> **Key idea:** PostgreSQL = платформа, а не просто БД. Одна база може замінити MongoDB (JSONB), ElasticSearch (full-text), Redis (кеш), InfluxDB (time-series), та Pinecone (векторний пошук).

## 1. Що таке Extensions?

**PostgreSQL Extension** - це модуль, який розширює функціональність БД:
- Нові типи даних
- Функції та оператори
- Індекси
- Процедурні мови
- Background workers

### Управління розширеннями

```sql
-- Список доступних розширень
SELECT * FROM pg_available_extensions ORDER BY name;

-- Встановлені розширення
SELECT * FROM pg_extension;

-- Встановити розширення
CREATE EXTENSION IF NOT EXISTS extension_name;

-- Видалити розширення
DROP EXTENSION extension_name;

-- Оновити версію
ALTER EXTENSION extension_name UPDATE TO '2.0';
```

## 2. Категорії розширень

| Категорія | Приклади | Use Cases |
| :--- | :--- | :--- |
| **Геопросторові** | PostGIS | Карти, GIS, геолокація |
| **Time-series** | TimescaleDB | IoT, метрики, логи |
| **AI/ML** | pgvector, MADlib | Векторний пошук, ML моделі |
| **Full-text search** | pg_trgm, dict_xsyn | Пошук по тексту |
| **Security** | pgcrypto, pg_tde | Шифрування |
| **Performance** | pg_stat_statements, pg_hint_plan | Моніторинг, оптимізація |
| **Replication** | Logical replication, pglogical | Репл distribution |
| **Foreign Data** | postgres_fdw, oracle_fdw | Федерація даних |

## 3. PostGIS - Геопросторові дані

### Огляд

**PostGIS** перетворює PostgreSQL на повноцінну **GIS базу даних** для роботи з картами, координатами, маршрутами.

**Використовується**: Uber, Foursquare, OpenStreetMap, Google Maps альтернативи.

### Встановлення

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;  -- Опціонально
```

```sql
-- Перевірка версії
SELECT PostGIS_Version();
-- POSTGIS="3.4.1" ...
```

### Типи даних

```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    geom GEOMETRY(Point, 4326),      -- Point в WGS84 (GPS координати)
    area GEOMETRY(Polygon, 4326)     -- Polygon (межі)
);
```

**Geometry types**:
- `POINT` - точка (lat/lon)
- `LINESTRING` - лінія (маршрут)
- `POLYGON` - багатокутник (межі міста)
- `MULTIPOINT`, `MULTILINESTRING`, `MULTIPOLYGON`

**SRID (Spatial Reference System)**:
- `4326` - WGS84 (GPS координати, lat/lon)
- `3857` - Web Mercator (Google Maps)
- `2154` - Lambert 93 (France)

### Приклади запитів

**Вставка геоданих**:

```sql
-- З WKT (Well-Known Text)
INSERT INTO locations (name, geom) VALUES
('Kyiv', ST_GeomFromText('POINT(30.5234 50.4501)', 4326));

-- З координат
INSERT INTO locations (name, geom) VALUES
('Lviv', ST_SetSRID(ST_MakePoint(24.0297, 49.8397), 4326));
```

**Пошук поблизу (Within distance)**:

```sql
-- Знайти всі точки в радіусі 10 км від Києва
SELECT name, ST_Distance(
    geom,
    ST_GeomFromText('POINT(30.5234 50.4501)', 4326)::geography
) AS distance_meters
FROM locations
WHERE ST_DWithin(
    geom::geography,
    ST_GeomFromText('POINT(30.5234 50.4501)', 4326)::geography,
    10000  -- 10 км
)
ORDER BY distance_meters;
```

**Nearest Neighbor (k найближчих)**:

```sql
-- 5 найближчих ресторанів
SELECT name,
    ST_Distance(geom::geography, $user_location::geography) AS dist
FROM restaurants
ORDER BY geom <-> $user_location  -- KNN оператор (потребує GiST індекс)
LIMIT 5;
```

**Contains (точка в полігоні)**:

```sql
-- Чи знаходиться точка в межах міста?
SELECT c.name AS city
FROM cities c
WHERE ST_Contains(c.area, ST_GeomFromText('POINT(30.5234 50.4501)', 4326));
```

**Buffering (зона навколо)**:

```sql
-- Створити зону 1км навколо точки
SELECT ST_Buffer(
    ST_GeomFromText('POINT(30.5234 50.4501)', 4326)::geography,
    1000  -- метри
)::geometry;
```

### Індексування

```sql
-- GiST індекс для геопросторових запитів (обов'язково!)
CREATE INDEX idx_locations_geom ON locations USING GIST (geom);

-- Для geography (точніші дистанції)
CREATE INDEX idx_locations_geom_geog ON locations USING GIST ((geom::geography));
```

### Use Cases

| Use Case | Запит |
| :--- | :--- |
| **Geofencing** | ST_Contains(polygon, point) |
| **Nearest location** | ORDER BY geom <-> point LIMIT N |
| **Route planning** | ST_Length(linestring) |
| **Area calculation** | ST_Area(polygon) |
| **Intersections** | ST_Intersects(geom1, geom2) |

> [!TIP] Tip
> **geography vs geometry**:
> - `geometry` - швидкий, planar (flat earth)
> - `geography` - точний, spherical (врахує кривизну Землі)
>
> Для GPS координат використовуйте `geography`!

## 4. TimescaleDB - Time-Series Database

### Огляд

**TimescaleDB** перетворює PostgreSQL на **time-series БД** для IoT, метрик, логів, фінансових даних.

**Performance**: 10-100x швидше за звичайний PostgreSQL для time-series workloads.

**Використовується**: Comcast, Schneider Electric, IBM.

### Встановлення

```bash
# Ubuntu/Debian
sudo apt install timescaledb-2-postgresql-18

# Увімкнути розширення
sudo timescaledb-tune
```

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

### Hypertables

**Hypertable** - розділена таблиця, оптимізована для time-series.

```sql
-- Створити звичайну таблицю
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INTEGER,
    temperature DOUBLE PRECISION,
    humidity DOUBLE PRECISION
);

-- Перетворити на hypertable (розділити по часу)
SELECT create_hypertable('sensor_data', 'time');
```

**Що відбувається**:
- Дані автоматично розділяються на **chunks** (за днями/тижнями)
- Старі chunks стискаються
- Запити оптимізуються для time ranges

### Continuous Aggregates (pre-computed views)

```sql
-- Матеріалізований вигляд з автоматичним оновленням
CREATE MATERIALIZED VIEW sensor_data_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM sensor_data
GROUP BY hour, sensor_id;

-- Політика оновлення (кожні 15 хвилин)
SELECT add_continuous_aggregate_policy('sensor_data_hourly',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '15 minutes'
);
```

### Data Retention Policies

```sql
-- Автоматично видаляти дані старші 90 днів
SELECT add_retention_policy('sensor_data', INTERVAL '90 days');
```

### Compression

```sql
-- Увімкнути компресію
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Автоматично стискати chunks старші 7 днів
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');

-- Ручна компресія chunk
SELECT compress_chunk('_timescaledb_internal._hyper_1_1_chunk');
```

**Результат**: До **20x економії місця** + швидші запити на історичних даних.

### Приклади запитів

```sql
-- Запити через time_bucket (як GROUP BY для часу)
SELECT
    time_bucket('5 minutes', time) AS bucket,
    sensor_id,
    AVG(temperature) AS avg_temp
FROM sensor_data
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, sensor_id
ORDER BY bucket DESC;

-- Downsample (зменшити resolution)
SELECT
    time_bucket('1 hour', time) AS hour,
    AVG(temperature)
FROM sensor_data
WHERE sensor_id = 123
  AND time > NOW() - INTERVAL '7 days'
GROUP BY hour;

-- Gap filling (заповнити пропуски)
SELECT
    time_bucket_gapfill('5 minutes', time) AS bucket,
    locf(AVG(temperature)) AS temp  -- Last Observation Carried Forward
FROM sensor_data
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket
ORDER BY bucket;
```

### Use Cases

| Use Case | Features |
| :--- | :--- |
| **IoT sensors** | Hypertables, compression, retention |
| **Application metrics** | Continuous aggregates, downsampling |
| **Financial tick data** | High ingestion rate, time-based queries |
| **Server logs** | Retention policies, compression |

## 5. pgvector - AI Vector Search

### Огляд

**pgvector** додає **векторний пошук** для AI/ML застосунків: embeddings, similarity search, RAG (Retrieval-Augmented Generation).

**Використовується**: ChatGPT plugins, Notion AI, Supabase Vector.

### Встановлення

```bash
# Ubuntu/Debian
sudo apt install postgresql-18-pgvector

# Або з source
git clone https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
```

```sql
CREATE EXTENSION vector;
```

### Vector тип даних

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536)  -- OpenAI ada-002 = 1536 dimensions
);
```

### Вставка векторів

```python
import openai
import psycopg2

# Отримати embedding з OpenAI
response = openai.Embedding.create(
    input="PostgreSQL is awesome",
    model="text-embedding-ada-002"
)
embedding = response['data'][0]['embedding']  # List[float] (1536 dims)

# Вставити в PostgreSQL
conn = psycopg2.connect(...)
cur = conn.cursor()
cur.execute(
    "INSERT INTO documents (content, embedding) VALUES (%s, %s)",
    ("PostgreSQL is awesome", embedding)
)
conn.commit()
```

### Similarity Search

```sql
-- Cosine similarity (найпоширеніший)
SELECT
    content,
    1 - (embedding <=> %s) AS similarity  -- <=> = cosine distance
FROM documents
ORDER BY embedding <=> %s  -- Вектор query
LIMIT 5;
```

**Distance operators**:
- `<->` - L2 distance (Euclidean)
- `<#>` - Inner product
- `<=>` - Cosine distance (рекомендовано для embeddings)

### Індексування

```sql
-- IVFFlat index (approximative nearest neighbor)
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);  -- кількість кластерів (sqrt(rows) зазвичай)

-- HNSW index (швидший, більше пам'яті)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

> [!WARNING] Warning
> Створіть індекс **ПІСЛЯ** заповнення таблиці даними для кращої якості кластеризації.

### RAG (Retrieval-Augmented Generation) pipeline

```python
def rag_query(user_query: str) -> str:
    # 1. Отримати embedding запиту
    query_embedding = openai.Embedding.create(
        input=user_query,
        model="text-embedding-ada-002"
    )['data'][0]['embedding']

    # 2. Знайти схожі документи
    cur.execute("""
        SELECT content
        FROM documents
        ORDER BY embedding <=> %s
        LIMIT 5
    """, (query_embedding,))

    context = "\n".join([row[0] for row in cur.fetchall()])

    # 3. Згенерувати відповідь з контекстом
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": f"Context:\n{context}"},
            {"role": "user", "content": user_query}
        ]
    )

    return response['choices'][0]['message']['content']
```

### Use Cases

| Use Case | Опис |
| :--- | :--- |
| **Semantic search** | Пошук за змістом, не keywords |
| **RAG chatbots** | Chatbots з доступом до документів |
| **Recommendation systems** | Схожі товари, контент |
| **Image search** | CLIP embeddings для зображень |
| **Duplicate detection** | Знайти схожі записи |

## 6. Full-Text Search Extensions

### pg_trgm (Trigram matching)

```sql
CREATE EXTENSION pg_trgm;

-- Fuzzy search (помилки в словах)
CREATE INDEX idx_products_name_trgm ON products
USING GIN (name gin_trgm_ops);

-- Пошук зі схожістю
SELECT name, similarity(name, 'postgre') AS sim
FROM products
WHERE name % 'postgre'  -- Similarity operator
ORDER BY sim DESC;

-- LIKE з індексом
SELECT * FROM products WHERE name ILIKE '%postgre%';
```

### Вбудований Full-Text Search

```sql
-- Створити tsvector колонку
ALTER TABLE articles ADD COLUMN tsv TSVECTOR;

-- Заповнити з існуючих даних
UPDATE articles SET tsv = to_tsvector('english', title || ' ' || content);

-- GIN індекс
CREATE INDEX idx_articles_tsv ON articles USING GIN (tsv);

-- Пошук
SELECT * FROM articles
WHERE tsv @@ to_tsquery('english', 'postgresql & performance');

-- Ranking
SELECT
    title,
    ts_rank(tsv, query) AS rank
FROM articles, to_tsquery('english', 'postgresql') query
WHERE tsv @@ query
ORDER BY rank DESC;
```

Детальніше: [[PostgreSQL - Full Text Search]]

## 7. Performance Extensions

### pg_stat_statements

```sql
CREATE EXTENSION pg_stat_statements;

-- Top повільні запити
SELECT
    substring(query, 1, 100) AS short_query,
    calls,
    total_exec_time / 1000 AS total_sec,
    mean_exec_time AS mean_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### pg_hint_plan

```sql
CREATE EXTENSION pg_hint_plan;

-- Підказка планувальнику
/*+ SeqScan(users) */
SELECT * FROM users WHERE age > 18;

/*+ IndexScan(users idx_users_email) */
SELECT * FROM users WHERE email = 'user@example.com';
```

## 8. Security Extensions

### pgcrypto

```sql
CREATE EXTENSION pgcrypto;

-- Хешування паролів
INSERT INTO users (email, password_hash)
VALUES ('user@example.com', crypt('my_password', gen_salt('bf')));

-- Перевірка паролю
SELECT * FROM users
WHERE email = 'user@example.com'
  AND password_hash = crypt('my_password', password_hash);

-- UUID generation
SELECT gen_random_uuid();
```

## 9. Foreign Data Wrappers

### postgres_fdw (Федерація PostgreSQL)

```sql
CREATE EXTENSION postgres_fdw;

-- Підключення до remote БД
CREATE SERVER foreign_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '192.168.1.100', dbname 'remote_db', port '5432');

-- User mapping
CREATE USER MAPPING FOR local_user
SERVER foreign_server
OPTIONS (user 'remote_user', password 'password');

-- Імпорт схеми
IMPORT FOREIGN SCHEMA public
FROM SERVER foreign_server
INTO local_schema;

-- Запити до remote таблиць
SELECT * FROM local_schema.remote_table;
```

### Інші FDW

- **oracle_fdw** - Oracle
- **mysql_fdw** - MySQL
- **mongo_fdw** - MongoDB
- **file_fdw** - CSV файли
- **s3_fdw** - Amazon S3

## 10. Інші корисні розширення

| Extension | Призначення |
| :--- | :--- |
| **uuid-ossp** | UUID generation |
| **hstore** | Key-value store (legacy, використовуйте JSONB) |
| **ltree** | Hierarchical data (categories, org charts) |
| **pg_repack** | VACUUM FULL без блокування |
| **pg_cron** | Cron jobs всередині БД |
| **hypopg** | Hypothetical indexes (тестувати індекси без створення) |
| **pgAudit** | Audit logging |
| **pg_partman** | Auto partition management |

## 11. Створення custom extension

```sql
-- Простий extension (функція)
CREATE EXTENSION IF NOT EXISTS plpgsql;

CREATE OR REPLACE FUNCTION hello(name TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN 'Hello, ' || name || '!';
END;
$$ LANGUAGE plpgsql;

-- Використання
SELECT hello('PostgreSQL');  -- 'Hello, PostgreSQL!'
```

## 12. Best Practices

### ✅ Рекомендації

1. **Встановлюйте лише потрібні розширення**
   - Кожне розширення додає overhead

2. **Читайте документацію**
   - Різні версії можуть мати breaking changes

3. **Тестуйте на staging**
   - Особливо для performance extensions

4. **Моніторте версії**
   ```sql
   SELECT * FROM pg_extension;
   ```

5. **Резервне копіювання**
   - `pg_dump` включає команди CREATE EXTENSION

### ❌ Уникайте

1. ❌ Встановлювати невідомі розширення з інтернету
2. ❌ Використовувати deprecated розширення (hstore → JSONB)
3. ❌ Оновлювати розширення в production без тестування

## 13. Пов'язані теми

- [[PostgreSQL - JSONB Guide|JSONB для NoSQL функціональності]]
- [[PostgreSQL - Full Text Search|Повнотекстовий пошук]]
- [[PostgreSQL - Performance Tuning|Оптимізація з pg_stat_statements]]
- [[PostgreSQL - Index Types|Індекси для extensions]]

## 14. Додаткові ресурси

- [PostgreSQL Extensions](https://www.postgresql.org/docs/18/external-extensions.html)
- [PostGIS Documentation](https://postgis.net/documentation/)
- [TimescaleDB Docs](https://docs.timescale.com/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [PGXN (PostgreSQL Extension Network)](https://pgxn.org/)

---

**Останнє оновлення**: 2025-01-17
**Версія**: PostgreSQL 18
