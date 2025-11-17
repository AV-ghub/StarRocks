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

## max_wal_senders
```bash
права у пользователя абсолютно на все есть, да и данные то в итоге копируются
но, тем не менее, ошибками напоследок (или в процессе) сыпет
полностью примерно так
Caused by: org.apache.flink.cdc.connectors.shaded.org.apache.kafka.connect.errors.ConnectException: An exception occurred in the change event producer. This connector will be stopped.
Caused by: io.debezium.jdbc.JdbcConnectionException: FATAL: number of requested standby connections exceeds max_wal_senders (currently 10)
Caused by: org.postgresql.util.PSQLException: FATAL: number of requested standby connections exceeds max_wal_senders (currently 10)
```
Отлично! Теперь мы видим **конкретную причину** ошибки. Проблема в настройках PostgreSQL:

```
FATAL: number of requested standby connections exceeds max_wal_senders (currently 10)
```

## 🔧 Решение проблемы

### 1. **Что происходит**
Flink CDC для каждой таблицы создает отдельное соединение репликации к PostgreSQL. При 300 таблицах требуется много соединений, а лимит `max_wal_senders = 10` слишком мал.

### 2. **Быстрое решение - увеличить max_wal_senders**

**В postgresql.conf:**
```ini
max_wal_senders = 50
max_replication_slots = 50
wal_sender_timeout = 0
```

### 3. **Перезапуск PostgreSQL**
После изменения конфигурации:
```bash
# Перезапуск PostgreSQL (не reload!)
sudo systemctl restart postgresql
# или
pg_ctl restart -D /var/lib/pgsql/data
```

### 4. **Проверка новых настроек**
```sql
SHOW max_wal_senders;
SHOW max_replication_slots;
```

## ⚡ Альтернативные решения

### **Вариант A: Уменьшить количество одновременных соединений**
В конфиге SMT можно ограничить параллелизм:
```bash
flink.parallelism = 5
```

### **Вариант B: Использовать одну публикацию для всех таблиц**
Вместо отдельных слотов для каждой таблицы можно создать одну публикацию:

```sql
CREATE PUBLICATION flink_publication FOR ALL TABLES;
```

И в конфиге SMT указать:
```bash
flink.cdc.debezium.publication.name = flink_publication
flink.cdc.debezium.publication.autocreate.mode = filtered
```

## 🎯 Рекомендация

1. **Увеличьте `max_wal_senders` до 50-100** (в зависимости от количества таблиц)
2. **Перезапустите PostgreSQL**
3. **Перезапустите Flink задания**

После этого ошибка должна исчезнуть, так как PostgreSQL сможет обслуживать все необходимые соединения репликации.

Попробуйте увеличить `max_wal_senders` - это самое простое и эффективное решение для вашего случая с 300 таблицами.






















