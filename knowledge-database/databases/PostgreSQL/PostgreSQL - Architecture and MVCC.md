---
tags:
  - postgresql
  - mvcc
  - architecture
  - concurrency
  - acid
  - transactions
aliases:
  - PostgreSQL MVCC
  - PostgreSQL Архітектура
  - Multi-Version Concurrency Control
created: 2025-01-17
topic: PostgreSQL Fundamentals
---

# 🏗️ PostgreSQL - Architecture and MVCC

> Multi-Version Concurrency Control: як PostgreSQL досягає високої паралельності без традиційних блокувань

## 🎯 Що таке MVCC?

**MVCC (Multi-Version Concurrency Control)** — це механізм керування паралельним доступом, при якому кожна транзакція бачить "знімок" (snapshot) бази даних на певний момент часу, незалежно від поточного стану даних.

### Ключові принципи

```
┌─────────────────────────────────────┐
│  Традиційні БД з локами             │
├─────────────────────────────────────┤
│  Reader ────┐                       │
│             ├──► WAIT ◄──── Writer  │
│  Reader ────┘                       │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  PostgreSQL з MVCC                  │
├─────────────────────────────────────┤
│  Reader ──► Snapshot v1             │
│  Writer ──► Create v2               │
│  Reader ──► Snapshot v1 (той самий!)│
└─────────────────────────────────────┘
```

**Головна перевага**: 
- ❌ Читання НЕ блокує запис
- ❌ Запис НЕ блокує читання
- ✅ Висока паралельність "з коробки"

## 🏗️ Архітектура процесів

PostgreSQL використовує **process-based модель** (не threads):

```
┌──────────────────────────────────────────┐
│           PostgreSQL Server              │
├──────────────────────────────────────────┤
│  Postmaster (головний процес)            │
│    │                                      │
│    ├──► Backend Process 1 (клієнт 1)     │
│    ├──► Backend Process 2 (клієнт 2)     │
│    ├──► Backend Process N (клієнт N)     │
│    │                                      │
│    ├──► Background Writer                │
│    ├──► WAL Writer                       │
│    ├──► Autovacuum Launcher              │
│    ├──► Stats Collector                  │
│    └──► Checkpointer                     │
└──────────────────────────────────────────┘
```

### Основні компоненти

| Процес | Призначення | Критичність |
|--------|-------------|-------------|
| **Postmaster** | Приймає з'єднання, створює backend процеси | 🔴 Критичний |
| **Backend** | Обробляє запити конкретного клієнта | 🟢 1 на з'єднання |
| **Background Writer** | Записує dirty pages на диск | 🟡 Важливий |
| **WAL Writer** | Записує WAL буфери на диск | 🔴 Критичний |
| **Autovacuum** | Автоматичне очищення мертвих кортежів | 🟡 Важливий |
| **Checkpointer** | Створює checkpoint'и для відновлення | 🟡 Важливий |

### ⚠️ Важливе про з'єднання

**Проблема**: Кожне з'єднання = окремий процес (~10MB пам'яті)

```python
# 🔴 ПОГАНО: Створення з'єднань на кожен запит
def handle_request():
    conn = psycopg2.connect("postgresql://...")  # Новий процес!
    # ... робота ...
    conn.close()  # Процес припиняється

# ✅ ДОБРЕ: Connection pooling
from psycopg2.pool import ThreadedConnectionPool

pool = ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    dsn="postgresql://..."
)

def handle_request():
    conn = pool.getconn()  # Переиспользуем существующий процес
    # ... робота ...
    pool.putconn(conn)
```

**Рішення**: Використовувати connection pooler
- **Application-level**: SQLAlchemy, psycopg2.pool, asyncpg
- **External**: PgBouncer, PgPool-II (рекомендовано для production)

## 🔄 Як працює MVCC: Версіонування рядків

Кожен рядок у PostgreSQL має **системні стовпці**:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT,
    email TEXT
    -- Невидимі системні стовпці:
    -- xmin XMIN,      -- ID транзакції, яка створила рядок
    -- xmax XMAX,      -- ID транзакції, яка видалила рядок (0 = активний)
    -- cmin CMIN,      -- Command ID всередині транзакції
    -- cmax CMAX,      -- Command ID видалення
    -- ctid TID        -- Фізична адреса (page, offset)
);

-- Подивитись системні стовпці:
SELECT xmin, xmax, ctid, * FROM users;
```

### Приклад версіонування

```sql
-- Транзакція 100: Вставка рядка
BEGIN;  -- xid = 100
INSERT INTO users VALUES (1, 'Alice', 'alice@example.com');
-- Створюється рядок: xmin=100, xmax=0
COMMIT;

-- Транзакція 101: Оновлення рядка
BEGIN;  -- xid = 101
UPDATE users SET email = 'alice@newdomain.com' WHERE id = 1;
-- Старий рядок: xmin=100, xmax=101 (позначено як видалений)
-- Новий рядок: xmin=101, xmax=0 (активний)
COMMIT;

-- Транзакція 102: Читання (почалась до коміту 101)
BEGIN;  -- xid = 102, snapshot = 100
SELECT * FROM users WHERE id = 1;
-- Бачить старий рядок! (xmin=100, xmax=0 для її snapshot)
COMMIT;
```

**Візуалізація:**

```
Час →
T1: INSERT    │ Row v1: [xmin=100, xmax=0]
T2: UPDATE    │ Row v1: [xmin=100, xmax=101] ← мертвий кортеж
              │ Row v2: [xmin=101, xmax=0]   ← активний
T3: SELECT    │ Snapshot бачить v1 (якщо почалась до коміту T2)
```

## 🎭 Transaction ID (XID) та Wraparound

PostgreSQL використовує **32-bit Transaction ID**:
- Діапазон: 0 до 4,294,967,295 (~4.3 мільярда)
- **Проблема**: Після 2 млрд транзакцій XID "обертається" (wraparound)

```
0 ────────────► 2^31 ────────────► 2^32 ─┐
│                                         │
└─────────────────────────────────────────┘
         Wraparound problem!
```

### Freeze механізм

```sql
-- Перевірка віку найстарішої транзакції
SELECT datname, age(datfrozenxid) 
FROM pg_database 
ORDER BY age(datfrozenxid) DESC;

-- Коли age() досягає 200M, автоматично запускається VACUUM FREEZE
```

**VACUUM FREEZE** замінює старі XID на спеціальне значення `FrozenTransactionId`, яке завжди вважається "в минулому" для будь-якого snapshot.

## 🛡️ ACID властивості

| Властивість | Механізм PostgreSQL |
|-------------|---------------------|
| **Atomicity** | WAL (Write-Ahead Log) — або всі зміни, або нічого |
| **Consistency** | Constraints, triggers, правила цілісності |
| **Isolation** | MVCC + рівні ізоляції (див. [[PostgreSQL - Transaction Isolation]]) |
| **Durability** | WAL + fsync: дані на диску до підтвердження транзакції |

### Write-Ahead Log (WAL)

```
Транзакція                WAL на диску           Таблиця на диску
    │                          │                        │
    ├──► 1. Записати в WAL ───►│                        │
    │         (гарантовано)     │                        │
    │                          │                        │
    ├──► 2. Підтвердити клієнту                         │
    │                          │                        │
    └──► 3. Пізніше записати   │                        │
          в таблицю ────────────┴───────────────────────►│
         (в фоні, background writer)
```

**Чому це важливо?**
- Послідовний запис WAL (швидко) vs випадковий запис таблиці (повільно)
- У разі краху: відновлення з WAL (replay)
- Базис для реплікації: standby сервери "програють" WAL

## 📊 Shared Memory Architecture

```
┌────────────────────────────────────────┐
│        Shared Memory                   │
├────────────────────────────────────────┤
│  Shared Buffers (кеш сторінок)         │
│    ├─ Dirty pages (змінені)            │
│    └─ Clean pages (читання)            │
├────────────────────────────────────────┤
│  WAL Buffers                           │
├────────────────────────────────────────┤
│  Lock Tables                           │
├────────────────────────────────────────┤
│  CLOG (Commit Log)                     │
└────────────────────────────────────────┘
         ▲
         │
    Всі backend процеси
```

**Shared Buffers** — найважливіший параметр:
- Дефолт: 128MB (занадто мало!)
- Рекомендація: 25% RAM (для dedicated сервера)
- Максимум: ~40% RAM (більше — марно, OS кеш теж працює)

Детальніше: [[PostgreSQL - Memory Configuration]]

## 🎯 Trade-offs MVCC

### ✅ Переваги

1. **Висока паралельність**: Читачі не чекають писців
2. **Узгоджене читання**: Кожна транзакція бачить snapshot
3. **Простота використання**: Не треба думати про locking

### ⚠️ Недоліки

1. **Bloat (роздування)**: Старі версії займають місце
   - Рішення: VACUUM (автоматичний/ручний)
   
2. **Додаткова I/O**: Запис нових версій замість update-in-place
   - Рішення: HOT updates, FILLFACTOR налаштування

3. **Overhead на XID**: Потреба в VACUUM FREEZE
   - Рішення: Autovacuum з правильними налаштуваннями

## 🔗 Пов'язані теми

- [[PostgreSQL - Transaction Isolation|Рівні ізоляції транзакцій]]
- [[PostgreSQL - VACUUM Guide|VACUUM та очищення мертвих кортежів]]
- [[PostgreSQL - Memory Configuration|Налаштування пам'яті]]
- [[PostgreSQL - Performance Tuning|Загальна оптимізація]]

## 📚 Додаткові ресурси

- [PostgreSQL MVCC Introduction](https://www.postgresql.org/docs/18/mvcc-intro.html)
- [PostgreSQL Internals](https://www.interdb.jp/pg/)
- [MVCC Unmasked](https://momjian.us/main/writings/pgsql/mvcc.pdf)

## 🏷️ Теги

#postgresql #mvcc #architecture #concurrency #acid #transactions

---

**Останнє оновлення**: 2025-01-17  
**Версія**: PostgreSQL 18
