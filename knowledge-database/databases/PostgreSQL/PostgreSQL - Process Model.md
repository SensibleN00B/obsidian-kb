---
tags:
  - postgresql
  - architecture
  - processes
  - connections
  - pgbouncer
aliases:
  - PostgreSQL Процеси
  - PostgreSQL Process Model
  - Модель процесів PostgreSQL
created: 2025-01-17
topic: PostgreSQL Fundamentals
---

# ⚙️ PostgreSQL - Process Model

> [!SUMMARY] TL;DR
> PostgreSQL використовує **process-based архітектуру**, де кожне з'єднання = окремий процес (~10MB пам'яті). Для масштабування критично важливо використовувати connection pooling (PgBouncer).
> **Key idea:** Розуміння моделі процесів дозволяє правильно налаштувати PostgreSQL під ваш workload та уникнути виснаження ресурсів.

## 1. Архітектура процесів

PostgreSQL **НЕ використовує threads** (на відміну від MySQL). Кожне з'єднання обслуговується окремим процесом операційної системи.

### Основні компоненти

```
┌──────────────────────────────────────────┐
│       PostgreSQL Server Instance         │
├──────────────────────────────────────────┤
│  Postmaster (Parent Process)             │
│    ├─ Прийом з'єднань                    │
│    ├─ Запуск backend процесів            │
│    └─ Моніторинг статусу                 │
│                                          │
│  Backend Processes (User connections)    │
│    ├─ Backend #1 ─── Client #1           │
│    ├─ Backend #2 ─── Client #2           │
│    └─ Backend #N ─── Client #N           │
│                                          │
│  Background Processes (Daemon workers)   │
│    ├─ Background Writer                  │
│    ├─ WAL Writer                         │
│    ├─ Checkpointer                       │
│    ├─ Autovacuum Launcher                │
│    │   └─ Autovacuum Workers (1-N)       │
│    ├─ Stats Collector                    │
│    ├─ Logical Replication Launcher       │
│    └─ Archiver                           │
└──────────────────────────────────────────┘
```

## 2. Типи процесів

### Postmaster (головний процес)

| Характеристика | Опис |
| :--- | :--- |
| **PID** | Зазвичай найменший (перший запущений) |
| **Роль** | Приймає TCP з'єднання на порт 5432 |
| **Функції** | Fork backend процесів, запуск background workers |
| **Критичність** | 🔴 Якщо Postmaster падає, вся БД зупиняється |

**Перевірка**:

```bash
# Знайти Postmaster процес
ps aux | grep postgres | grep -v grep | head -1

# Або через pg_stat_activity
SELECT pid, usename, application_name, state
FROM pg_stat_activity
WHERE pid = pg_backend_pid();
```

### Backend Processes (user connections)

**Властивості**:
- ✅ **1 процес = 1 клієнтське з'єднання**
- ✅ Незалежна пам'ять (~10MB + work_mem)
- ✅ Створюються через fork() Postmaster
- ❌ Overhead при створенні (~1-5ms)

**Memory Layout одного Backend процесу**:

```
Backend Process Memory:
├─ Shared Memory (спільна для всіх)
│  └─ shared_buffers, WAL buffers, locks
├─ Private Memory (окрема для кожного)
│  ├─ work_mem (для сортування, hash joins)
│  ├─ maintenance_work_mem (для VACUUM, CREATE INDEX)
│  └─ temp_buffers (для temp таблиць)
└─ Process overhead (~2-5MB)

Total per connection: ~10MB + work_mem
```

**Приклад розрахунку**:

```sql
-- Параметри
shared_buffers = 4GB
work_mem = 32MB
max_connections = 200

-- Максимальна пам'ять
= shared_buffers
  + (max_connections × work_mem)
  + (max_connections × 10MB overhead)
= 4GB + (200 × 32MB) + (200 × 10MB)
= 4GB + 6.4GB + 2GB
= ~12.4GB RAM potential usage
```

### Background Writer

**Призначення**: Записує **dirty pages** з shared_buffers на диск.

```
Shared Buffers          Background Writer       Disk
┌──────────┐                  │              ┌──────┐
│ Page 1 ✓ │                  │              │      │
│ Page 2 💾│ ────────────────►│─────────────►│ File │
│ Page 3 💾│   (dirty pages)  │              │      │
└──────────┘                  │              └──────┘
```

**Налаштування**:

```sql
-- postgresql.conf
bgwriter_delay = 200ms           -- Частота запуску (дефолт)
bgwriter_lru_maxpages = 100      -- Макс сторінок за раз
bgwriter_lru_multiplier = 2.0    -- Множник для адаптації
```

**Моніторинг**:

```sql
SELECT
    buffers_clean,        -- Скільки сторінок записано
    maxwritten_clean,     -- Скільки разів досягнуто maxpages
    buffers_backend       -- Скільки backend процеси писали самі (погано!)
FROM pg_stat_bgwriter;
```

> [!WARNING] Warning
> Якщо `buffers_backend` високий → Background Writer не встигає → треба збільшити `bgwriter_lru_maxpages`

### WAL Writer

**Призначення**: Записує **WAL буфери** на диск для забезпечення Durability (ACID).

```
Transaction          WAL Buffer        WAL Writer       Disk
    │                   │                  │           ┌─────┐
    ├─ INSERT ─────────►│                  │           │     │
    ├─ UPDATE ─────────►│                  │           │ WAL │
    │                   │                  │           │ Log │
    ├─ COMMIT ─────────►│─────────────────►│──────────►│     │
    │                   │  (fsync)         │           └─────┘
    └─ ACK ◄────────────┴──────────────────┘
```

**Ключовий параметр**: `wal_sync_method`

```sql
-- Перевірка методу fsync
SHOW wal_sync_method;
-- Типові значення: fdatasync (Linux), open_sync (FreeBSD)

-- Інші WAL параметри
SHOW wal_buffers;              -- Дефолт: -1 (auto = 1/32 shared_buffers)
SHOW wal_writer_delay;         -- Дефолт: 200ms
SHOW synchronous_commit;       -- Дефолт: on (гарантія Durability)
```

> [!TIP] Tip
> Для non-critical даних можна встановити `synchronous_commit = off` для 2-3x прискорення INSERT, але з ризиком втрати останніх транзакцій при краху.

### Checkpointer

**Призначення**: Створює **checkpoint** — точку узгодженості БД на диску.

**Що робить Checkpoint**:
1. Записує всі dirty pages з shared_buffers на диск
2. Записує checkpoint запис у WAL
3. Після краху: відновлення починається з останнього checkpoint

```
Timeline:
│───────────────────────────────────────────────────►
    Checkpoint    WAL      WAL      Checkpoint
       #1        writes   writes       #2
       │◄────recovery──────►│

Recovery потребує replay WAL тільки після checkpoint #1
```

**Налаштування**:

```sql
-- postgresql.conf
checkpoint_timeout = 5min               -- Час між checkpoint (дефолт)
max_wal_size = 1GB                      -- Макс WAL перед checkpoint
checkpoint_completion_target = 0.9      -- Розтягнути на 90% інтервалу

-- Моніторинг
SELECT
    checkpoints_timed,      -- Checkpoint за таймером
    checkpoints_req,        -- Checkpoint за вимогою (погано якщо багато!)
    checkpoint_write_time,  -- Час запису (ms)
    checkpoint_sync_time    -- Час sync (ms)
FROM pg_stat_bgwriter;
```

> [!WARNING] Warning
> Якщо `checkpoints_req` >> `checkpoints_timed` → треба збільшити `max_wal_size`

### Autovacuum Launcher та Workers

**Призначення**: Автоматичне очищення мертвих кортежів (dead tuples).

```
Autovacuum Launcher (1 процес)
    │
    ├─ Autovacuum Worker #1 ─── VACUUM table1
    ├─ Autovacuum Worker #2 ─── VACUUM table2
    └─ Autovacuum Worker #N ─── VACUUM tableN
```

**Налаштування**:

```sql
-- postgresql.conf
autovacuum = on                           -- ✅ ОБОВ'ЯЗКОВО
autovacuum_max_workers = 3                -- Кількість workers (дефолт)
autovacuum_naptime = 1min                 -- Інтервал перевірки

-- Моніторинг активних workers
SELECT
    pid,
    datname,
    usename,
    query,
    state,
    now() - query_start AS duration
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%';
```

Детальніше: [[PostgreSQL - VACUUM Guide]]

### Stats Collector

**Призначення**: Збирає статистику про активність БД для планувальника запитів.

```sql
-- Статистика таблиць
SELECT * FROM pg_stat_user_tables WHERE relname = 'users';

-- Статистика індексів
SELECT * FROM pg_stat_user_indexes WHERE relname = 'users';

-- Активні запити
SELECT * FROM pg_stat_activity;
```

## 3. Проблема: Обмеження кількості з'єднань

### Чому багато з'єднань = погано

**Проблема 1: Memory overhead**

```
200 connections × 10MB = 2GB RAM тільки на процеси
+ work_mem × active queries
```

**Проблема 2: Context switching overhead**

```
# CPU з 4 ядрами + 200 активних процесів
= ~50 процесів на ядро
= величезний overhead на context switching
```

**Проблема 3: Процеси vs Threads**

| Характеристика | Процеси (PostgreSQL) | Threads (MySQL) |
| :--- | :--- | :--- |
| **Memory isolation** | ✅ Окрема пам'ять | ❌ Спільна |
| **Overhead створення** | 🐌 Повільно (fork) | ⚡ Швидко |
| **Memory overhead** | 🔴 ~10MB на з'єднання | ✅ ~1MB |
| **Crash isolation** | ✅ Один процес не вб'є інші | ❌ Один thread може вбити всі |

### Тест: Скільки з'єднань може PostgreSQL?

```bash
# Максимум з'єднань
SHOW max_connections;  # Дефолт: 100

# Поточна кількість
SELECT count(*) FROM pg_stat_activity;

# Reserved для superuser
SHOW superuser_reserved_connections;  # Дефолт: 3
```

**Типові ліміти**:
- Невелика VM: `max_connections = 100`
- Production сервер: `max_connections = 200-300`
- **Не ставте >500** без connection pooling!

## 4. Рішення: Connection Pooling

### Application-level pooling

**psycopg2 (Python)**:

```python
from psycopg2.pool import ThreadedConnectionPool

# Створити пул
pool = ThreadedConnectionPool(
    minconn=5,      # Мінімум підтримувати
    maxconn=20,     # Максимум створювати
    dsn="postgresql://user:pass@localhost/db"
)

# Використання
def query_database():
    conn = pool.getconn()          # Взяти з пулу
    try:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM users")
            return cur.fetchall()
    finally:
        pool.putconn(conn)         # Повернути в пул
```

**SQLAlchemy**:

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:pass@localhost/db",
    pool_size=10,              # Базовий розмір
    max_overflow=20,           # Додаткові при навантаженні
    pool_pre_ping=True,        # Перевірка з'єднань
    pool_recycle=3600          # Оновлювати кожну годину
)
```

### External pooling: PgBouncer

**Найпопулярніший** connection pooler для PostgreSQL.

```
Application                PgBouncer           PostgreSQL
  │                           │                    │
  ├─ 1000 з'єднань ──────────►│                    │
  ├─ від застосунку           │                    │
  │                           │                    │
  │                           ├─ 50 з'єднань ─────►│
  │                           │  до PostgreSQL     │
```

**Режими роботи**:

| Режим | Як працює | Use Case |
| :--- | :--- | :--- |
| **Session** | 1 client = 1 backend до disconnect | Дефолт, безпечно |
| **Transaction** | Backend повертається після COMMIT | ⚡ High performance, OLTP |
| **Statement** | Backend після кожного запиту | 🔥 Максимальна щільність, обмеження |

**Конфігурація pgbouncer.ini**:

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction          # Найефективніший
max_client_conn = 1000           # Макс з'єднань від клієнтів
default_pool_size = 25           # Скільки backend на БД
reserve_pool_size = 5            # Резерв
server_idle_timeout = 600        # Вбити idle backend через 10min
```

**Запуск**:

```bash
# Встановлення (Ubuntu)
sudo apt install pgbouncer

# Старт
sudo systemctl start pgbouncer

# Підключення до PgBouncer
psql -h localhost -p 6432 -U myuser mydb
```

**Моніторинг PgBouncer**:

```sql
-- Підключитись до admin консолі
psql -h localhost -p 6432 -U pgbouncer pgbouncer

-- Статистика пулів
SHOW POOLS;
-- database | user | cl_active | cl_waiting | sv_active | sv_idle

-- Статистика БД
SHOW STATS;

-- Список клієнтів
SHOW CLIENTS;
```

> [!TIP] Tip
> **Золоте правило**: `default_pool_size = (CPU cores × 2) + effective_spindle_count`
>
> Для SSD: `default_pool_size = CPU cores × 4`

### Порівняння: з PgBouncer vs без

**Без PgBouncer**:

```
100 web workers × 5 connections each = 500 PostgreSQL processes
500 × 10MB = 5GB RAM
max_connections = 500 (high overhead)
```

**З PgBouncer**:

```
100 web workers × 5 connections = 500 PgBouncer clients
PgBouncer → PostgreSQL: 50 connections
50 × 10MB = 500MB RAM
max_connections = 100 (low overhead)
```

**Результат**: **10x менше пам'яті** + швидший context switching

## 5. Моніторинг процесів

### Перевірка активних з'єднань

```sql
-- Кількість з'єднань по БД
SELECT
    datname,
    count(*) AS connections,
    max(now() - backend_start) AS longest_conn
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;

-- Top активні запити
SELECT
    pid,
    usename,
    datname,
    state,
    now() - query_start AS duration,
    left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY duration DESC;
```

### Завершення процесів

```sql
-- Graceful termination (завершує після поточного запиту)
SELECT pg_terminate_backend(12345);

-- Force kill (одразу)
SELECT pg_cancel_backend(12345);

-- Вбити всі idle з'єднання старші 1 години
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '1 hour'
  AND pid != pg_backend_pid();
```

### Системний моніторинг

```bash
# Всі PostgreSQL процеси
ps aux | grep postgres

# Кількість процесів
ps aux | grep postgres | wc -l

# Memory usage
ps aux | grep postgres | awk '{sum+=$6} END {print sum/1024 " MB"}'

# Top processes
top -u postgres
```

## 6. Best Practices

### ✅ Рекомендації

1. **Завжди використовуйте connection pooling** для web застосунків
   - Application-level: SQLAlchemy, psycopg2.pool
   - External: PgBouncer (найкраще для OLTP)

2. **Налаштуйте max_connections відповідно до RAM**
   ```
   max_connections = (RAM - shared_buffers - OS) / 10MB
   ```

3. **Моніторте idle connections**
   ```sql
   -- Автоматичне закриття idle з'єднань
   idle_in_transaction_session_timeout = 10min
   ```

4. **Використовуйте PgBouncer transaction mode** для OLTP
   - 10-20x більше клієнтів на той же backend pool

5. **Обмежуйте max_connections на рівні користувача**
   ```sql
   ALTER ROLE webuser CONNECTION LIMIT 50;
   ```

### ❌ Анти-паттерни

1. ❌ **Створювати з'єднання на кожен HTTP request**
   ```python
   # ПОГАНО
   def handle_request():
       conn = psycopg2.connect(...)
       # ...
       conn.close()
   ```

2. ❌ **Ставити max_connections = 1000 без PgBouncer**
   - Context switching overhead
   - Виснаження RAM

3. ❌ **Тримати idle транзакції відкритими**
   ```sql
   BEGIN;
   SELECT * FROM users WHERE id = 1;
   -- Забули зробити COMMIT/ROLLBACK...
   ```

4. ❌ **Ігнорувати background workers**
   - Autovacuum потребує workers
   - Резервуйте 3-5 з'єднань для maintenance

## 7. Порівняння з іншими СУБД

| Характеристика | PostgreSQL | MySQL | Oracle |
| :--- | :--- | :--- | :--- |
| **Модель** | Process-based | Thread-based (5.7+) | Thread-based |
| **Overhead на з'єднання** | ~10MB | ~1MB | ~2-3MB |
| **Context switching** | Більший | Менший | Менший |
| **Crash isolation** | ✅ Відмінна | ⚠️ Середня | ✅ Відмінна |
| **Потреба в pooling** | 🔴 Критична | ⚠️ Рекомендована | ⚠️ Рекомендована |

## 8. Пов'язані теми

- [[PostgreSQL - Architecture and MVCC|Архітектура та MVCC]]
- [[PostgreSQL - Memory Configuration|Налаштування пам'яті]]
- [[PostgreSQL - Performance Tuning|Загальна оптимізація]]
- [[PostgreSQL - VACUUM Guide|Autovacuum процеси]]

## 9. Додаткові ресурси

- [PostgreSQL Architecture](https://www.postgresql.org/docs/18/tutorial-arch.html)
- [PgBouncer Documentation](https://www.pgbouncer.org/)
- [PostgreSQL Connection Pooling](https://wiki.postgresql.org/wiki/Number_Of_Database_Connections)
- [Process vs Thread Model](https://www.interdb.jp/pg/pgsql01.html)

---

**Останнє оновлення**: 2025-01-17
**Версія**: PostgreSQL 18
