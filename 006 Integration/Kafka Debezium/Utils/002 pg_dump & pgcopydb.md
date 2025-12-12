## **1. `pg_dump` с `-j N` (параллельный дамп)**

### **Как работает:**
```bash
pg_dump -F d -j 8 -Z 5 --no-synchronized-snapshots \
  -h source-db -U postgres mydb -f /path/to/dump
```

**Ключевые опции:**
- `-F d` — directory format (создает каталог с файлами)
- `-j 8` — 8 потоков параллельно
- `-Z 5` — сжатие уровня 5
- `--no-synchronized-snapshots` — важно для параллельности!

### **Что происходит внутри:**

```
Шаг 1: Создание снапшота транзакции
    ↓
Шаг 2: Параллельное чтение разных таблиц/чанков
    ├── Поток 1: Читает таблицу users (чанки 1-1000000)
    ├── Поток 2: Читает таблицу users (чанки 1000001-2000000)  
    ├── Поток 3: Читает таблицу orders
    └── Поток 4: Читает индексы
    ↓
Шаг 3: Запись в файлы с сжатием
```

### **Особенности с Debezium:**

```sql
-- ПРОБЛЕМА: pg_dump создает свой снапшот в начале
-- Debezium создает свой снапшот в другое время
-- Получаем рассинхрон!

-- РЕШЕНИЕ: Использовать --no-synchronized-snapshots
-- И фиксировать LSN вручную:

-- Перед дампом:
SELECT pg_export_snapshot() as snapshot_id;  -- получаем: 0000000A-12345678
SELECT pg_current_wal_lsn() as start_lsn;    -- 0/1567D88

-- Запускаем дамп с использованием этого снапшота:
pg_dump -j 8 --snapshot=0000000A-12345678 ...

-- Запускаем Debezium с:
-- "snapshot.mode": "never"
-- (он начнет с start_lsn)
```

### **Преимущества:**
- **Встроен в PostgreSQL** — не требует дополнительных инструментов
- **Консистентный снапшот** всей БД (все таблицы на один момент времени)
- **Сжатие на лету** (`-Z`)
- **Восстановление в один транзакции** (если нужно)

### **Недостатки для CDC:**
- **Один большой файл/директория** — сложно чанковать загрузку
- **Нет встроенной синхронизации с LSN** — нужно делать вручную
- **Может блокировать DDL** во время дампа

## **2. `pgcopydb` — специализированный инструмент**

Это инструмент, созданный специально для **миграции с минимальным простоем**, идеально подходит для CDC!

### **Базовая архитектура:**
```
┌─────────────────────────────────────────────────────────┐
│                     Источник (PostgreSQL)               │
│  • Таблица A ──────────────┐                            │
│  • Таблица B ──────────────├─→ [pgcopydb] → Фикс. LSN   │
│  • Таблица C ──────────────┘                            │
└─────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────┐
│                    Назначение (StarRocks)               │
│  • Прием данных через Kafka Connect                     │
│  • Или прямая загрузка CSV                              │
└─────────────────────────────────────────────────────────┘
```

### **Как работает с CDC:**

**Этап 1: Инициализация и снапшот**
```bash
# Создаем снапшот и сразу фиксируем LSN
pgcopydb snapshot --source "host=source-db dbname=mydb"

# Результат:
# - Создается слот репликации: pgcopydb_slot
# - Фиксируется LSN: 0/1567D88
# - Экспортируется снапшот: 0000000A-12345678
```

**Этап 2: Параллельный копирование данных**
```bash
pgcopydb copy-db \
  --source "host=source-db dbname=mydb" \
  --target "host=target-db dbname=mydb" \
  --table-jobs 4           # 4 потока на таблицу
  --index-jobs 2           # 2 потока на индексы
  --split-tables-larger-than "1 GB"  # автоматическое чанкование
  --resume                 # можно возобновить при падении
  --follow                 # СЛЕДИТЬ ЗА ИЗМЕНЕНИЯМИ ПОСЛЕ СНАПШОТА!
```

**Этап 3: Catch-up изменений (самая крутая фича!)**
```bash
# После копирования снапшота, pgcopydb может:
# 1. Продолжить читать изменения из WAL
# 2. Применять их в целевой БД
# 3. Догнать до текущего состояния

pgcopydb follow \
  --source "host=source-db dbname=mydb" \
  --target "host=target-db dbname=mydb" \
  --endpos "0/FFFFFFFF"  # или до определенного LSN
```

### **Интеграция с Debezium и Kafka:**

```bash
# Сценарий миграции с минимальным простоем
#!/bin/bash

# 1. Фиксируем точку начала
START_LSN=$(psql source-db -t -c "SELECT pg_current_wal_lsn()")

# 2. Запускаем Debezium в фоне (начинает копить дельты)
curl -X POST http://kafka-connect:8083/connectors -H "Content-Type: application/json" -d '{
  "name": "debezium-init",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "source-db",
    "snapshot.mode": "never",
    "slot.name": "migration_slot",
    "publication.autocreate.mode": "filtered",
    "table.include.list": "public.users,public.orders"
  }
}'

# 3. Копируем снапшот через pgcopydb
pgcopydb copy-db \
  --source "host=source-db dbname=mydb" \
  --target-dir "/data/snapshot" \
  --snapshot "0000000A-12345678" \
  --split-tables-larger-than "500 MB"

# 4. Загружаем снапшот в StarRocks параллельно
# Каждый файл CSV загружаем отдельным потоком
for chunk in /data/snapshot/*.csv; do
  starrocks-load --file "$chunk" --table users &
done
wait

# 5. Запускаем Routine Load, но С ОПРЕДЕЛЕННОГО OFFSET!
# Нужно найти offset в Kafka, соответствующий START_LSN

# 6. Останавливаем pgcopydb follow, когда Routine Load догонит
```

### **Автоматическое чанкование больших таблиц:**

```bash
# pgcopydb сам разбивает большие таблицы
pgcopydb copy table \
  --source "host=source-db dbname=mydb" \
  --table "public.huge_table" \
  --split-tables-larger-than "1 GB" \
  --split-tables-chunk-size "1000000 rows"

# Создаст файлы:
# /dump/huge_table/chunk_0001.csv  # строки 1-1,000,000
# /dump/huge_table/chunk_0002.csv  # строки 1,000,001-2,000,000
# ...
```

### **Сравнение подходов:**

| Функция | `pg_dump -j N` | `pgcopydb` | Ручной `COPY TO` чанками |
|---------|----------------|------------|--------------------------|
| **Параллельность** | Да, на уровне таблиц | Да, таблицы + чанки | Только ручное управление |
| **Чанкование таблиц** | Нет | Автоматическое | Ручное |
| **Синхронизация с LSN** | Ручная | Встроенная | Ручная |
| **Catch-up изменений** | Нет | **Встроенный follow режим!** | Нет |
| **Интеграция с CDC** | Средняя | **Отличная** | Средняя |
| **Восстановление при сбое** | С начала | Resume с места падения | Ручное управление |
| **Требует места на диске** | Полный дамп | Можно stream'ить | Промежуточные файлы |

### **Практический пример: Миграция с pgcopydb → Kafka → StarRocks**

```python
#!/usr/bin/env python3
"""
Полная миграция с консистентным переключением
"""
import subprocess
import psycopg2
import requests
import json
import time

def migrate_with_pgcopydb():
    # 1. Подготовка
    source_conn = psycopg2.connect("host=source-db dbname=mydb")
    cursor = source_conn.cursor()
    
    # Фиксируем начальную точку
    cursor.execute("SELECT pg_current_wal_lsn(), txid_current()")
    start_lsn, start_txid = cursor.fetchone()
    start_time = int(time.time() * 1000)
    
    print(f"Начальная точка: LSN={start_lsn}, TXID={start_txid}, time={start_time}")
    
    # 2. Запускаем Debezium (начинает копить дельты)
    debezium_config = {
        "name": "migration-cdc",
        "config": {
            "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
            "database.hostname": "source-db",
            "database.dbname": "mydb",
            "snapshot.mode": "never",
            "slot.name": f"migration_slot_{start_txid}",
            "publication.name": f"migration_pub_{start_txid}",
            "topic.prefix": f"migration_{start_txid}",
            "table.include.list": "public.(.*)"
        }
    }
    
    requests.post("http://kafka-connect:8083/connectors", 
                  json=debezium_config)
    
    # 3. Копируем снапшот через pgcopydb
    print("Копирование снапшота...")
    subprocess.run([
        "pgcopydb", "copy-db",
        "--source", "host=source-db dbname=mydb",
        "--target-dir", "/data/migration/snapshot",
        "--snapshot", f"{start_txid}",
        "--table-jobs", "8",
        "--split-tables-larger-than", "500MB",
        "--progress"
    ], check=True)
    
    # 4. Загружаем в StarRocks
    print("Загрузка в StarRocks...")
    # Используем parallel COPY в StarRocks
    load_script = """
    #!/bin/bash
    for file in /data/migration/snapshot/*.csv; do
        table=$(basename $file .csv)
        mysql -h starrocks -P 9030 -u admin \
          -e "COPY INTO mydb.$table FROM '$file' 
              PROPERTIES ('format'='csv', 'max_filter_ratio'='0.1')" &
    done
    wait
    """
    
    # 5. Находим Kafka offset для start_time
    # Используем kafka-admin API
    offset_response = requests.get(
        f"http://kafka:8082/topics/migration_{start_txid}.public.users/partitions/0/offsets",
        params={"timestamp": start_time}
    )
    target_offset = offset_response.json()["offset"]
    
    print(f"Целевой offset в Kafka: {target_offset}")
    
    # 6. Создаем Routine Load с точного offset
    routine_load_sql = f"""
    CREATE ROUTINE LOAD mydb.users_cdc ON users
    PROPERTIES (
        "desired_concurrent_number" = "4",
        "max_error_number" = "1000",
        "format" = "json"
    )
    FROM KAFKA (
        "kafka_broker_list" = "kafka:9092",
        "kafka_topic" = "migration_{start_txid}.public.users",
        "property.kafka_default_offsets" = "{target_offset}"
    );
    """
    
    # 7. Валидация
    print("Валидация данных...")
    # Сравниваем кол-во записей
    # Проверяем контрольные суммы
    
    # 8. Переключение трафика
    print("Миграция завершена!")
```

### **Ключевые преимущества pgcopydb для CDC:**

1. **Встроенная синхронизация со слотом репликации** — не нужно вручную искать LSN
2. **Автоматический follow режим** — может "догнать" изменения после снапшота
3. **Инкрементальное копирование** — при падении продолжает с места останова
4. **Параллелизм на уровне чанков** — большие таблицы автоматически разбиваются
5. **Минимальная блокировка** — использует те же механизмы, что и логическая репликация

### **Когда что использовать:**

- **`pg_dump -j N`** — когда нужен простой одноразовый дамп, консистентность всей БД
- **`pgcopydb`** — для миграций с минимальным простоем, интеграция с CDC
- **Ручной `COPY TO`** — когда нужен полный контроль над процессом, кастомная логика

**Для вашего сценария с Debezium → Kafka → StarRocks я бы рекомендовал:**

1. **Инициализация:** `pgcopydb` для снапшота + фиксации LSN
2. **CDC:** Debezium с `snapshot.mode: never`
3. **Загрузка:** Parallel load чанков в StarRocks
4. **Синхронизация:** Routine Load с точного offset
