# StarRocks DDL из схемы PostgreSQL
## 🎯 Скрипт генерации StarRocks DDL из PostgreSQL

```sql
WITH table_columns AS (
    SELECT 
        c.table_schema,
        c.table_name,
        c.column_name,
        c.data_type,
        c.character_maximum_length,
        c.numeric_precision,
        c.numeric_scale,
        c.datetime_precision,
        c.is_nullable,
        c.ordinal_position,
        CASE 
            WHEN tc.constraint_type = 'PRIMARY KEY' THEN 1
            ELSE 2
        END as priority
    FROM information_schema.columns c
    LEFT JOIN information_schema.key_column_usage kcu 
        ON c.table_schema = kcu.table_schema 
        AND c.table_name = kcu.table_name 
        AND c.column_name = kcu.column_name
    LEFT JOIN information_schema.table_constraints tc 
        ON kcu.constraint_schema = tc.constraint_schema 
        AND kcu.constraint_name = tc.constraint_name 
        AND tc.constraint_type = 'PRIMARY KEY'
    WHERE c.table_schema = 'dbo'
        AND c.table_name IN ('calls') 
),
unique_columns AS (
    SELECT 
        column_name,
        data_type,
        character_maximum_length,
        numeric_precision,
        numeric_scale,
        datetime_precision,
        is_nullable,
        priority,
        ordinal_position
    FROM table_columns
    ORDER BY priority, ordinal_position
),
starrocks_types AS (
    SELECT 
        column_name,
        data_type,
        -- Точный маппинг PostgreSQL → StarRocks
        CASE 
            WHEN data_type = 'smallint' THEN 'SMALLINT'
            WHEN data_type IN ('integer', 'int') THEN 'INT'
            WHEN data_type = 'bigint' THEN 'BIGINT'
            WHEN data_type = 'serial' THEN 'BIGINT'
            WHEN data_type = 'bigserial' THEN 'BIGINT'
            
            WHEN data_type IN ('decimal', 'numeric') THEN 
                'DECIMAL(' || 
                COALESCE(numeric_precision::text, '27') || ',' || 
                COALESCE(numeric_scale::text, '9') || ')'
            
            WHEN data_type = 'real' THEN 'FLOAT'
            WHEN data_type IN ('double precision', 'float8') THEN 'DOUBLE'
            
            WHEN data_type IN ('character varying', 'varchar') THEN 
                CASE 
                    WHEN character_maximum_length IS NULL OR character_maximum_length <= 65533
                    THEN 'VARCHAR(' || COALESCE(character_maximum_length::text, '65533') || ')'
                    ELSE 'STRING'
                END
            WHEN data_type = 'char' THEN 
                'CHAR(' || COALESCE(character_maximum_length::text, '1') || ')'
            WHEN data_type = 'text' THEN 'STRING'
            
            WHEN data_type = 'date' THEN 'DATE'
            WHEN data_type = 'time' THEN 'TIME'
            WHEN data_type IN ('timestamp', 'timestamp without time zone') THEN 'DATETIME'
            WHEN data_type = 'timestamp with time zone' THEN 'DATETIME'
            
            WHEN data_type = 'boolean' THEN 'BOOLEAN'
            WHEN data_type = 'bytea' THEN 'STRING'
            WHEN data_type IN ('json', 'jsonb') THEN 'JSON'
            WHEN data_type = 'uuid' THEN 'VARCHAR(36)'
            
            WHEN data_type LIKE '%[]' THEN 
                'ARRAY<' || 
                CASE 
                    WHEN data_type = 'integer[]' THEN 'INT'
                    WHEN data_type = 'text[]' THEN 'STRING'
                    WHEN data_type = 'numeric[]' THEN 'DECIMAL(27,9)'
                    ELSE 'STRING'
                END || '>'
            
            ELSE 'STRING'
        END as starrocks_type,
        
        CASE 
            WHEN is_nullable = 'YES' THEN ''
            ELSE 'NOT NULL'
        END as nullability,
        
        priority,
        ROW_NUMBER() OVER (ORDER BY priority, ordinal_position) as col_order
    FROM unique_columns
),
pk_columns AS (
    SELECT DISTINCT kcu.column_name
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu 
        ON tc.constraint_schema = kcu.constraint_schema 
        AND tc.constraint_name = kcu.constraint_name
    WHERE tc.table_schema = 'dbo'
        AND tc.table_name = 'calls'
        AND tc.constraint_type = 'PRIMARY KEY'
),
formatted_columns AS (
    SELECT 
        uc.column_name,
        starrocks_type,
        nullability,
        uc.priority,
        col_order,
        CASE 
            WHEN col_order = MAX(col_order) OVER () THEN ''  -- последняя колонка без запятой
            ELSE ','
        END as trailing_comma,
        '-- PostgreSQL: ' || uc.data_type || 
        CASE 
            WHEN character_maximum_length IS NOT NULL THEN '(' || character_maximum_length || ')'
            WHEN numeric_precision IS NOT NULL THEN '(' || numeric_precision || 
                CASE WHEN numeric_scale > 0 THEN ',' || numeric_scale ELSE '' END || ')'
            WHEN datetime_precision IS NOT NULL AND datetime_precision > 0 
                THEN '(' || datetime_precision || ')'
            ELSE ''
        END as pg_comment
    FROM starrocks_types st
    JOIN unique_columns uc ON st.column_name = uc.column_name
)
SELECT 
    'CREATE TABLE IF NOT EXISTS cs_28826.dbo__calls (' || CHR(10) ||
    STRING_AGG(
        '    `' || column_name || '`' || 
        REPEAT(' ', GREATEST(35 - LENGTH(column_name), 1)) ||
        starrocks_type || ' ' || nullability || 
        ' COMMENT ''' || pg_comment || '''' || trailing_comma,
        CHR(10)
        ORDER BY col_order
    ) || CHR(10) ||
    ')' || CHR(10) ||
    'ENGINE = OLAP' || CHR(10) ||
    CASE WHEN EXISTS (SELECT 1 FROM pk_columns) THEN
        'PRIMARY KEY (' || 
            (SELECT STRING_AGG('`' || column_name || '`', ', ') FROM pk_columns) || 
        ')' || CHR(10)
    ELSE '' END ||
    'DISTRIBUTED BY HASH(' || 
        COALESCE(
            (SELECT column_name FROM pk_columns ORDER BY column_name LIMIT 1),
            (SELECT column_name FROM formatted_columns ORDER BY col_order LIMIT 1)
        ) || 
    ') BUCKETS 1' || CHR(10) ||
    'PROPERTIES (' || CHR(10) ||
    '    "compression" = "LZ4",' || CHR(10) ||
    '    "enable_persistent_index" = "true",' || CHR(10) ||
    '    "fast_schema_evolution" = "true",' || CHR(10) ||
    '    "replicated_storage" = "true",' || CHR(10) ||
    '    "replication_num" = "1"'
    ');'
FROM formatted_columns fc;
```

## 📋 Таблица соответствия типов PostgreSQL → StarRocks

| PostgreSQL тип | StarRocks тип | Примечания |
|----------------|---------------|------------|
| `smallint` | `SMALLINT` | |
| `integer`, `int` | `INT` | |
| `bigint` | `BIGINT` | |
| `decimal(p,s)`, `numeric(p,s)` | `DECIMAL(p,s)` | StarRocks: до 38 цифр (27,9 по умолчанию) |
| `real` | `FLOAT` | |
| `double precision` | `DOUBLE` | |
| `varchar(n)` | `VARCHAR(n)` | Если n > 65533 → `STRING` |
| `char(n)` | `CHAR(n)` | |
| `text` | `STRING` | StarRocks не имеет TEXT типа |
| `date` | `DATE` | |
| `time` | `TIME` | |
| `timestamp`, `timestamp without time zone` | `DATETIME` | StarRocks не различает TZ |
| `timestamp with time zone` | `DATETIME` | Часовой пояс игнорируется |
| `boolean` | `BOOLEAN` | |
| `bytea` | `STRING` | Base64 кодирование |
| `json`, `jsonb` | `JSON` | Нативный тип JSON |
| `uuid` | `VARCHAR(36)` | |
| `array` | `ARRAY<тип>` | Поддержка массивов |
| `hstore` | `STRING` | |

## 🛠️ Упрощенная функция для StarRocks

```sql
CREATE OR REPLACE FUNCTION generate_starrocks_ddl(
    schema_name TEXT DEFAULT 'dbo',
    table1 TEXT DEFAULT 'constants',
    table2 TEXT DEFAULT 'enumdescriptor',
    output_db TEXT DEFAULT 'cs_28826',
    output_table TEXT DEFAULT 'cdc_source__dbo_dict'
) RETURNS TEXT AS $$
DECLARE
    ddl_text TEXT;
    pk_list TEXT;
    hash_key TEXT;
BEGIN
    -- Получаем первичные ключи
    SELECT STRING_AGG(DISTINCT '`' || kcu.column_name || '`', ', ')
    INTO pk_list
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu 
        ON tc.constraint_schema = kcu.constraint_schema 
        AND tc.constraint_name = kcu.constraint_name
    WHERE tc.table_schema = schema_name
        AND tc.table_name IN (table1, table2)
        AND tc.constraint_type = 'PRIMARY KEY';
    
    -- Определяем ключ для DISTRIBUTED BY HASH
    IF pk_list IS NOT NULL THEN
        SELECT SPLIT_PART(pk_list, ',', 1) INTO hash_key;
    ELSE
        hash_key := '`insertutcdate`';
    END IF;
    
    -- Генерируем DDL
    WITH cols AS (
        SELECT DISTINCT ON (LOWER(c.column_name))
            c.column_name,
            CASE 
                -- Числовые
                WHEN c.data_type IN ('integer','int') THEN 'INT'
                WHEN c.data_type = 'bigint' THEN 'BIGINT'
                WHEN c.data_type IN ('decimal','numeric') THEN 
                    'DECIMAL(' || COALESCE(c.numeric_precision::TEXT, '27') || ',' || 
                    COALESCE(c.numeric_scale::TEXT, '9') || ')'
                
                -- Строковые
                WHEN c.data_type IN ('varchar','character varying') THEN 
                    CASE 
                        WHEN c.character_maximum_length IS NULL OR c.character_maximum_length <= 65533
                        THEN 'VARCHAR(' || COALESCE(c.character_maximum_length::TEXT, '65533') || ')'
                        ELSE 'STRING'
                    END
                WHEN c.data_type = 'text' THEN 'STRING'
                
                -- Дата/время
                WHEN c.data_type = 'timestamp' THEN 'DATETIME'
                WHEN c.data_type = 'date' THEN 'DATE'
                
                -- Специальные
                WHEN c.data_type = 'boolean' THEN 'BOOLEAN'
                WHEN c.data_type = 'jsonb' THEN 'JSON'
                
                ELSE 'STRING'
            END as starrocks_type,
            
            CASE WHEN c.is_nullable = 'YES' THEN '' ELSE 'NOT NULL' END as nullable,
            
            -- Сортируем PK первыми
            CASE 
                WHEN EXISTS (
                    SELECT 1 FROM information_schema.key_column_usage kcu2
                    JOIN information_schema.table_constraints tc2 
                        ON kcu2.constraint_name = tc2.constraint_name
                    WHERE kcu2.table_schema = c.table_schema
                        AND kcu2.table_name = c.table_name
                        AND kcu2.column_name = c.column_name
                        AND tc2.constraint_type = 'PRIMARY KEY'
                ) THEN 1
                ELSE 2
            END as sort_order
        FROM information_schema.columns c
        WHERE c.table_schema = schema_name
            AND c.table_name IN (table1, table2)
        ORDER BY LOWER(c.column_name)
    )
    SELECT 
        'CREATE TABLE IF NOT EXISTS ' || output_db || '.' || output_table || ' (' || CHR(10) ||
        STRING_AGG(
            '    `' || column_name || '`' || 
            REPEAT(' ', GREATEST(35 - LENGTH(column_name), 1)) ||
            starrocks_type || ' ' || nullable || ',',
            CHR(10)
            ORDER BY sort_order, column_name
        ) || CHR(10) ||
        '    `insertutcdate` DATETIME NULL' || CHR(10) ||
        ')' || CHR(10) ||
        'ENGINE = OLAP' || CHR(10) ||
        CASE WHEN pk_list IS NOT NULL THEN
            'PRIMARY KEY (' || pk_list || ')' || CHR(10)
        ELSE '' END ||
        'DISTRIBUTED BY HASH(' || hash_key || ') BUCKETS 10' || CHR(10) ||
        'PROPERTIES (' || CHR(10) ||
        '    "replication_num" = "3"' || CHR(10) ||
        ');'
    INTO ddl_text
    FROM cols;
    
    RETURN ddl_text;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT generate_starrocks_ddl();
```

## 🔧 Bash-скрипт для экспорта

```bash
#!/bin/bash
# generate_starrocks_ddl.sh

PSQL_CMD="psql -d ваша_база -t -A -q"
OUTPUT_FILE="starrocks_ddl_$(date +%Y%m%d_%H%M%S).sql"

cat > generate_ddl.sql << 'EOF'
-- Генерация StarRocks DDL
WITH cols AS (
    SELECT DISTINCT ON (LOWER(column_name))
        column_name,
        CASE 
            WHEN data_type = 'integer' THEN 'INT'
            WHEN data_type = 'bigint' THEN 'BIGINT'
            WHEN data_type = 'numeric' THEN 'DECIMAL(27,9)'
            WHEN data_type IN ('varchar','text') THEN 'STRING'
            WHEN data_type = 'timestamp' THEN 'DATETIME'
            WHEN data_type = 'boolean' THEN 'BOOLEAN'
            WHEN data_type = 'jsonb' THEN 'JSON'
            ELSE 'STRING'
        END as sr_type
    FROM information_schema.columns
    WHERE table_schema = 'dbo'
        AND table_name IN ('constants', 'enumdescriptor')
    ORDER BY LOWER(column_name)
)
SELECT 'CREATE TABLE IF NOT EXISTS cs_28826.cdc_source__dbo_dict ('
UNION ALL
SELECT '    `' || column_name || '`' || 
       REPEAT(' ', GREATEST(35 - LENGTH(column_name), 1)) ||
       sr_type || ' NULL,'
FROM cols
UNION ALL
SELECT '    `insertutcdate` DATETIME NULL'
UNION ALL
SELECT ')'
UNION ALL
SELECT 'ENGINE = OLAP'
UNION ALL
SELECT 'PRIMARY KEY (`constantid`, `enumdescriptorid`)'
UNION ALL
SELECT 'DISTRIBUTED BY HASH(`constantid`) BUCKETS 10'
UNION ALL
SELECT 'PROPERTIES ('
UNION ALL
SELECT '    "replication_num" = "3"'
UNION ALL
SELECT ');';
EOF

# Выполняем и сохраняем
$PSQL_CMD -f generate_ddl.sql > "$OUTPUT_FILE"

echo "StarRocks DDL создан: $OUTPUT_FILE"
```

## ⚡ Ключевые особенности StarRocks:

1. **Типы данных**:
   - `STRING` вместо `VARCHAR(max)`
   - `DATETIME` без поддержки часовых поясов
   - Нативный `JSON` тип

2. **DDL структура**:
   ```sql
   ENGINE = OLAP  -- Обязательно для StarRocks
   PRIMARY KEY (...)  -- Создает первичный индекс
   DISTRIBUTED BY HASH(...) BUCKETS 10  -- Распределение
   PROPERTIES ("replication_num" = "3")  -- Репликация
   ```

3. **Для CDC** в StarRocks лучше добавить:
   ```sql
   `_op` VARCHAR(10) COMMENT 'I/U/D',
   `_source_ts` BIGINT COMMENT 'Unix timestamp',
   `_batch_id` BIGINT COMMENT 'Batch identifier'
   ```

