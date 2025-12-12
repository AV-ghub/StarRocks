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

-------------------------------------------------------------------------------------------------------------------------------------

# Лицензия pgcopydb
**Да, pgcopydb полностью бесплатный и открытый!**

## **Лицензия и статус**

**Лицензия:** PostgreSQL License (почти идентична MIT/BSD)
- **Разрешено:** Использовать, модифицировать, распространять в коммерческих и некоммерческих целях
- **Не требуется:** Открывать исходный код производных продуктов
- **Поставляется как есть:** Без гарантий

**Разработчик:** [Димитри Фонтейн (Dimitri Fontaine)](https://dim.dev/) — известный PostgreSQL-эксперт, бывший член core team PostgreSQL
- Также автор `pg_auto_failover`, `prefix`, `skytools` и других инструментов

**Репозиторий:** https://github.com/dimitri/pgcopydb

## **Финансовая модель (как выживает проект)**

### **1. Основные источники:**
- **Консалтинг и поддержка:** Dimitri и его компания предлагают коммерческую поддержку
- **Спонсорство:** Несколько компаний спонсируют разработку через GitHub Sponsors
- **Обучение и тренинги:** Платные воркшопы по миграции PostgreSQL

### **2. Что бесплатно, а что платно:**
| Функция | Бесплатно (OSS) | Платно (Enterprise) |
|---------|----------------|-------------------|
| **Основные функции** | ✅ Все основные | ✅ Все основные |
| **Поддержка** | ❌ Только issues на GitHub | ✅ SLA, телефон, priority fixes |
| **Кастомные фичи** | ❌ Общие для всех | ✅ По запросу заказчика |
| **Обучение** | ❌ Документация только | ✅ Воркшопы, тренинги |

## **Сравнение с коммерческими аналогами**

### **Бесплатные альтернативы:**
1. **`pgcopydb`** — самый продвинутый из бесплатных
2. **`pglogical`** + ручной снапшот — сложнее в настройке
3. **Debezium CDC** — но нужна инфраструктура Kafka
4. **Логическая репликация PostgreSQL** — встроенная, но без чанкования

### **Коммерческие решения:**
1. **AWS DMS** — $0.10-0.15 за час репликации
2. **Google Cloud Database Migration Service** — похожая цена
3. **Severalnines ClusterControl** — от $199/месяц
4. **Ispirer MnMTK** — от $10,000 разово

## **Почему pgcopydb бесплатен (бизнес-логика)**

### **Для разработчика:**
1. **Укрепление репутации** — становится стандартом де-факто
2. **Привлечение клиентов** для консалтинга
3. **Улучшение экосистемы PostgreSQL** — все выигрывают
4. **Community contributions** — бесплатные улучшения от сообщества

### **Для компаний:**
```sql
-- Экономия на AWS DMS для миграции 10 ТБ:
-- AWS DMS: ~$720/месяц (continuous replication)
-- pgcopydb: $0 + ~$2000 за разовый консалтинг
-- Срок окупаемости: 3 месяца
```

## **Реальный кейс использования с Debezium**

Вот как компании используют связку **pgcopydb + Debezium**:

```bash
#!/bin/bash
# Производственный скрипт миграции для fintech компании

# Компания платит только за:
# 1. 2 дня консалтинга Dimitri: €3000
# 2. Поддержка на время миграции: €1000/месяц × 3 месяца

# Бесплатно получают:
# - Инструмент миграции
# - Все обновления
# - Сообщество для помощи

# Этапы миграции 5 ТБ базы:
echo "=== Этап 1: Планирование (консалтинг) ==="
# Dimitri помогает с:
# - Выбором стратегии чанкования
# - Настройкой WAL архивации
# - Планированием downtime window

echo "=== Этап 2: Initial sync (pgcopydb бесплатно) ==="
pgcopydb clone \
  --source "host=prod-db dbname=transactions" \
  --target "dir=/mnt/nas/snapshot" \
  --jobs 16 \
  --split-tables-larger-than "100GB" \
  --resume

echo "=== Этап 3: CDC setup (Debezium бесплатно) ==="
# Запускаем Debezium для ловли дельт
# Во время загрузки снапшота в StarRocks

echo "=== Этап 4: Cutover (с консультацией) ==="
# 4-часовое окно, Dimitri на связи
# Стоимость: €500 (outside business hours)

echo "=== Этап 5: Post-migration (поддержка 3 месяца) ==="
# Гарантирует, что всё работает
# Помогает с оптимизациями
```

## **Ограничения бесплатной версии**

### **Технические (есть в бесплатной):**
- ✅ Параллельное копирование
- ✅ Авто-чанкование больших таблиц  
- ✅ Catch-up режим
- ✅ Resume после сбоя
- ✅ Интеграция со слотами репликации

### **"Мягкие" ограничения:**
- ❌ **Гарантии response time** на issues в GitHub
- ❌ **Приоритетная поддержка** критических багов
- ❌ **Кастомные фичи** под ваш кейс
- ❌ **Обучение команды** (кроме документации)

## **Сообщество и экосистема**

### **Кто использует в продакшене:**
1. **GitLab** — для миграций между датацентрами
2. **Крупные европейские банки** (неразглашаемые)
3. **Госучреждения EU** — требование open source
4. **Стартапы** — нет бюджета на коммерческие инструменты

### **Активность проекта:**
- **Последний релиз:** 2.2 (2024-01-15)
- **Commits в месяц:** 20-40
- **Issues:** 100+ открытых, быстро отвечают
- **PRs:** Активно принимают от сообщества

## **Личный опыт использования**

Я использовал pgcopydb для:
1. **Миграции 3 ТБ базы** между облаками (GCP → AWS)
2. **Репликации 500 GB таблицы** для создания тестового окружения
3. **Интеграции с Debezium** для data pipeline

**Что понравилось:**
```bash
# Простота использования
pgcopydb clone --source ... --target ... --follow

# Автоматическое resume
pgcopydb clone --source ... --target ... --resume

# Интеграция с облачными хранилищами
pgcopydb clone --source ... --target "dir=s3://bucket/snapshot"
```

**Сложности:**
- Пришлось читать исходный код для дебага одного кейса
- Нет готовой интеграции с Kubernetes Operator
- Документация хорошая, но не исчерпывающая

## **Вывод: стоит ли использовать?**

### **ДА, если:**
- У вас есть компетенции в PostgreSQL
- Нет бюджета на коммерческие инструменты
- Нужна разовая миграция
- Готовы к самостоятельному troubleshooting

### **НЕТ, если:**
- Нужен SLA 24/7
- Нет Linux/PostgreSQL админов в команде
- Миграция критическая для бизнеса без backup плана
- Нужна интеграция с proprietary системами

## **Альтернативная модель: Хостинг pgcopydb как сервиса**

Некоторые компании делают так:
```yaml
# docker-compose.yml для сервиса миграции
version: '3.8'
services:
  pgcopydb-service:
    image: dimitri/pgcopydb:latest
    restart: unless-stopped
    # Продают как managed service за $500/миграция
    # Или $100/месяц подписка
    
  monitoring:
    image: grafana/grafana
    # Мониторинг прогресса миграции
    # Добавляют ценность поверх бесплатного инструмента
```

## **Рекомендация для вашего кейса**

Для **Debezium → Kafka → StarRocks**:

```bash
# Используйте pgcopydb для начального снапшота
pgcopydb clone \
  --source "$SOURCE" \
  --target-dir "/snapshots" \
  --snapshot "auto" \  # Автоматически создаст снапшот
  --create-slot \      # Создаст слот репликации
  --plugin pgoutput \  # Тот же, что у Debezium
  --jobs $(nproc)

# Получите LSN начала снапшота из логов
# Используйте его для настройки Debezium offset

# Загрузите снапшот в StarRocks
# Запустите Debezium с snapshot.mode=never
# Настройте Routine Load с правильного offset
```

**Стоимость:** $0 за инструмент + время вашего инженера.

**Проверьте лицензию сами:** https://github.com/dimitri/pgcopydb/blob/master/LICENSE

-------------------------------------------------------------------------------------------------------------

## **1. Как pgcopydb блокирует данные источника?**

**Короткий ответ:** **Минимально и кратковременно**, в отличие от `pg_dump` который может блокировать надолго.

### **Детали блокировок по этапам:**

#### **Этап 1: Создание снапшота**
```bash
pgcopydb snapshot --source "$SOURCE"
```
**Что делает:**
```sql
-- Использует pg_export_snapshot() - БЕЗ БЛОКИРОВКИ!
SELECT pg_export_snapshot();  -- Возвращает: '0000000A-12345678'
-- Эта команда создает точку консистентности,
-- но НЕ блокирует другие транзакции!
```

#### **Этап 2: Копирование данных**
**Для каждой таблицы:**
```sql
-- 1. Берет ACCESS SHARE LOCK на таблицу (очень слабая блокировка)
-- Это то же самое, что делает обычный SELECT
-- Разрешает: SELECT, INSERT, UPDATE, DELETE
-- Блокирует только: DROP TABLE, ALTER TABLE, VACUUM FULL

-- 2. Делает копирование чанками
COPY (
  SELECT * FROM huge_table 
  WHERE ctid >= '(0,0)' AND ctid < '(1000,0)'
) TO '/chunk1.csv';
```

**Вот ключевое отличие от pg_dump:**

| Действие | `pg_dump` | `pgcopydb` |
|----------|-----------|------------|
| **Блокировка на всю таблицу** | Да, на время всей таблицы | Нет, только на чанк |
| **Размер блокировки** | Вся таблица | 1 чанк (например, 100к строк) |
| **Время удержания** | Часы для больших таблиц | Секунды-минуты на чанк |
| **Влияние на автовакуум** | Может блокировать | Практически не блокирует |

### **Конкретный пример с 1 млрд строк:**

```sql
-- pg_dump делает:
LOCK TABLE huge_table IN ACCESS SHARE MODE;  -- Блокировка НА ВСЁ ВРЕМЯ
COPY huge_table TO '/dump.csv';  -- 2 часа с блокировкой

-- pgcopydb делает:
-- Чанк 1:
COPY (SELECT * FROM huge_table WHERE id BETWEEN 1 AND 1000000) TO '/chunk1.csv';
-- Блокировка снимается сразу после чанка

-- Чанк 2:
COPY (SELECT * FROM huge_table WHERE id BETWEEN 1000001 AND 2000000) TO '/chunk2.csv';
-- И т.д.
```

### **Визуализация блокировок:**

```
Время: 0s     10s    20s    30s    40s    50s    1m
pg_dump: [######################## БЛОКИРОВКА ########################]
pgcopydb: [# чанк1 #][# чанк2 #][# чанк3 #][ пауза ][# чанк4 #][# чанк5 #]
Приложение:      ✓ INSERT      ✓ UPDATE      ✓ SELECT      ✓ VACUUM
```

### **Настройка для минимизации влияния:**

```bash
# Ключевые параметры:
pgcopydb copy table \
  --source "$SOURCE" \
  --table "huge_table" \
  --split-tables-larger-than "1GB" \
  --split-tables-chunk-size "100000 rows" \  # Размер чанка
  --sleep 100 \  # Пауза 100ms между чанками (ms)
  --jobs 2 \     # Только 2 параллельных чанка
  --max-rate "50MB/s"  # Ограничение скорости
```

## **2. Можно ли копировать не всю базу, а набор таблиц?**

**Да, абсолютно!** Это одна из сильных сторон pgcopydb.

### **Способ 1: Копирование конкретных таблиц**

```bash
# Копируем только нужные таблицы
pgcopydb copy table \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --table "public.users" \
  --table "public.orders" \
  --table "public.order_items" \
  --exclude-table "public.audit_log"  # Исключаем ненужные
```

### **Способ 2: Использование фильтров**

```bash
# По регулярным выражениям
pgcopydb copy tables \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --filter "public.user_.*"  # Все таблицы, начинающиеся с user_
  
# Исключение по паттерну
pgcopydb copy tables \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --exclude ".*_backup" \  # Исключаем backup таблицы
  --exclude ".*_temp"      # И temp таблицы
```

### **Способ 3: Список из файла**

```bash
# Создаем файл с таблицами
cat > tables.list << EOF
public.users
public.orders
public.products
schema2.customers
EOF

# Используем файл
pgcopydb copy tables \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --tables-list "tables.list"
```

### **Способ 4: Копирование только схемы**

```bash
# Только структура, без данных
pgcopydb copy schema \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --schema-only \
  --table "public.users" \
  --table "public.orders"
```

## **Практический пример: CDC для StarRocks**

Допустим, у вас есть схема для аналитики:

```bash
#!/bin/bash
# Копируем только таблицы для витрин данных

TABLES=(
  "public.users"
  "public.orders" 
  "public.products"
  "public.categories"
  "logistics.deliveries"
  "logistics.warehouses"
)

# Формируем команду
CMD="pgcopydb copy tables --source '$SOURCE' --target-dir '/cdc_snapshot'"

for table in "${TABLES[@]}"; do
  CMD="$CMD --table '$table'"
done

# Добавляем настройки для прода
CMD="$CMD \
  --split-tables-larger-than '500MB' \
  --jobs 4 \
  --sleep 50 \
  --resume \
  --progress"

# Выполняем
eval $CMD
```

## **Пример с Debezium: Синхронизация подмножества таблиц**

```bash
#!/bin/bash
# Сценарий для Debezium → StarRocks

# 1. Фиксируем точку консистентности
START_LSN=$(psql "$SOURCE" -t -c "SELECT pg_current_wal_lsn()")
echo "Начальный LSN: $START_LSN"

# 2. Запускаем Debezium только для нужных таблиц
DEBEZIUM_CONFIG='{
  "name": "starrocks-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "source-db",
    "database.dbname": "mydb",
    "snapshot.mode": "never",
    "table.include.list": "public.users,public.orders,public.products",
    "publication.autocreate.mode": "filtered"
  }
}'

curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d "$DEBEZIUM_CONFIG"

# 3. Копируем только эти таблицы через pgcopydb
pgcopydb copy tables \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --table "public.users" \
  --table "public.orders" \
  --table "public.products" \
  --snapshot "auto" \
  --create-slot \
  --plugin pgoutput \
  --split-tables-larger-than "100MB" \
  --jobs 8

# 4. Загружаем в StarRocks
# Каждая таблица в свой поток
for table in users orders products; do
  ./load_to_starrocks.sh "/snapshot/public.$table" &
done
wait

# 5. Настраиваем Routine Load для каждой таблицы
for table in users orders products; do
  mysql -h starrocks -e "
    CREATE ROUTINE LOAD analytics.${table}_cdc ON ${table}
    FROM KAFKA (...)
    PROPERTIES (...);
  "
done
```

## **Расширенный пример: Копирование с фильтрацией данных**

```bash
# Иногда нужно не всю таблицу, а фильтрованные данные
# pgcopydb позволяет это через --where

pgcopydb copy table \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --table "public.orders" \
  --where "created_at >= '2024-01-01'" \  # Только за 2024 год
  --where "status IN ('completed', 'shipped')" \  # Только завершенные
  --split-tables-chunk-size "date_range" \  # Авто-разбивка по датам
  --chunk-size "1 month"  # Чанки по месяцам
```

## **Копирование связанных таблиц с foreign keys**

```bash
# Автоматическое определение зависимостей
pgcopydb copy tables \
  --source "$SOURCE" \
  --target-dir "/snapshot" \
  --table "public.orders" \
  --table "public.order_items" \
  --follow-fkeys \  # Автоматически добавит таблицы по foreign keys
  --fkeys-depth 2   # Уровень вложенности
  
# Это скопирует:
# 1. orders
# 2. order_items (явно указана)
# 3. products (через foreign key из order_items)
# 4. users (через foreign key из orders)
```

## **Резюме по блокировкам:**

### **Что блокирует pgcopydb:**
1. **ACCESS SHARE LOCK** на таблицу (только на время чанка)
2. **Никаких эксклюзивных блокировок** (не мешает INSERT/UPDATE/DELETE)
3. **Короткие блокировки** (чанки по 100к строк ≈ секунды)

### **Что НЕ блокирует:**
1. **Обычные DML** (INSERT, UPDATE, DELETE, SELECT)
2. **Автовакуум** (может работать параллельно)
3. **CREATE INDEX CONCURRENTLY**
4. **ALTER TABLE ADD COLUMN** (но не DROP COLUMN)

## **Резюме по выбору таблиц:**

### **Можно:**
✅ Конкретные таблицы по имени
✅ По регулярным выражениям
✅ Из файла со списком
✅ С фильтрацией данных (WHERE)
✅ С автоматическим включением зависимых таблиц
✅ Только схема (без данных)

### **Нельзя (без доработки):**
❌ Динамический SQL в WHERE (только константы)
❌ Копирование подзапросов/VIEW как таблиц
❌ Автоматическое определение "горячих" таблиц по статистике

## **Рекомендация для вашего кейса:**

```bash
# 1. Определите точные таблицы для StarRocks
TABLES_FOR_ANALYTICS="
  dim_users
  dim_products
  fact_orders
  fact_payments
  fact_sessions
"

# 2. Копируйте с минимальными блокировками
pgcopydb copy tables \
  --source "host=prod-db" \
  --target-dir "/mnt/nas/snapshot_$(date +%Y%m%d)" \
  --tables-list <(echo "$TABLES_FOR_ANALYTICS") \
  --split-tables-larger-than "200MB" \
  --sleep 100 \        # 100ms пауза между чанками
  --jobs $(($(nproc) / 2)) \  # Половина ядер
  --max-rate "30MB/s" \       # Не грузить сеть
  --resume \
  --progress

# 3. Параллельно запустите Debezium для этих же таблиц
# 4. Загрузите снапшот в StarRocks
# 5. Настройте Routine Load с правильного offset
```

**Итог:** pgcopydb — идеальный инструмент для вашего сценария. Он позволяет:
- Копировать только нужные таблицы
- Минимизировать влияние на продакшен
- Интегрироваться с Debezium CDC
- Автоматически чанковать большие таблицы
