# [StarRocks Migration Tool (SMT)](https://docs.starrocks.io/docs/integrations/loading_tools/SMT/)

## Debezium vs Kafka
Flink CDC для чтения изменений данных из PostgreSQL использует Debezium, который, в свою очередь, построен на платформе Kafka Connect.   
Поэтому в стектрейсе ошибки вы видите классы из пакета org.apache.flink.cdc.connectors.shaded.org.apache.kafka.connect.errors.

Когда в конфигурации возникает проблема (например, неверные параметры подключения к PostgreSQL, недостаток прав или сетевые issues), Debezium генерирует исключение через механизм Kafka Connect, что и приводит к такой ошибке.

Вы правы - пользователь `postgres` по умолчанию должен иметь все права, но давайте это проверим и при необходимости настроим.

## 🔍 Проверка прав пользователя PostgreSQL

### 1. **Проверка прав репликации**
```sql
SELECT rolname, rolreplication, rolcanlogin 
FROM pg_roles 
WHERE rolname = 'postgres';
```
**Ожидаемый результат:** `rolreplication = true`

### 2. **Проверка прав на базу данных**
```sql
SELECT 
    datname,
    has_database_privilege('postgres', datname, 'CONNECT') as can_connect,
    has_database_privilege('postgres', datname, 'CREATE') as can_create
FROM pg_database 
WHERE datname = 'your_database_name';
```

### 3. **Проверка прав на таблицы**
```sql
SELECT 
    schemaname,
    tablename,
    has_table_privilege('postgres', schemaname||'.'||tablename, 'SELECT') as can_select
FROM pg_tables 
WHERE schemaname = 'public' 
LIMIT 10;
```

## 🔧 Если прав не хватает - как добавить

### 1. **Добавление прав репликации (если отсутствует)**
```sql
ALTER ROLE postgres WITH REPLICATION;
```

### 2. **Проверка/настройка pg_hba.conf**
Убедитесь, что в `pg_hba.conf` есть строка, разрешающая подключение:
```
# Для IPv4
host    replication     postgres        samenet                 md5
# Или для всех хостов
host    all             postgres        0.0.0.0/0               md5
```

### 3. **Перезагрузка конфигурации PostgreSQL**
```sql
SELECT pg_reload_conf();
```
Или через командную строку:
```bash
pg_ctl reload -D /path/to/data/directory
```

## ⚙️ Дополнительные настройки PostgreSQL

### Проверка параметров конфигурации:
```sql
-- Должно быть 'logical'
SHOW wal_level;

-- Должно быть достаточное количество (минимум 5-10)
SHOW max_replication_slots;

-- Должно быть достаточное количество
SHOW max_wal_senders;
```

### Если нужно изменить в postgresql.conf:
```ini
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

## 🎯 Практическая проверка

Создайте тестовый слот репликации вручную:
```sql
SELECT * FROM pg_create_logical_replication_slot('test_slot', 'pgoutput');

-- Проверьте, что слот создался
SELECT * FROM pg_replication_slots WHERE slot_name = 'test_slot';

-- Удалите тестовый слот
SELECT pg_drop_replication_slot('test_slot');
```

## 🔄 После настройки прав

После выполнения всех настроек:
1. **Перезапустите PostgreSQL** (если меняли wal_level)
2. **Перезапустите Flink задания**
3. **Проверьте логи** - ошибка должна исчезнуть


