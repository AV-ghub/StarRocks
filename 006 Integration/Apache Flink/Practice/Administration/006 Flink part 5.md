# Flink DDL из схемы PostgreSQL
Для автоматизации создания Flink DDL из схемы PostgreSQL таблиц можно написать скрипт, который генерирует нужный SQL. Вот несколько вариантов:

## 🎯 Основной скрипт для генерации Flink DDL

```sql
WITH table_columns AS (
    -- Получаем колонки из двух таблиц
    SELECT 
        c.table_schema,
        c.table_name,
        c.column_name,
        c.data_type,
        c.is_nullable,
        c.ordinal_position,
        -- Приоритет: сначала PK, потом остальные
        CASE WHEN tc.constraint_type = 'PRIMARY KEY' THEN 1 ELSE 2 END as priority
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
        AND c.table_name IN ('constants', 'enumdescriptor')
        AND c.table_catalog = current_database()
),
unique_columns AS (
    -- Убираем дубликаты (если имена колонок повторяются)
    SELECT DISTINCT ON (lower(column_name))
        column_name,
        data_type,
        is_nullable,
        priority,
        table_name as source_table
    FROM table_columns
    ORDER BY lower(column_name), priority, ordinal_position
),
mapped_types AS (
    -- Маппинг типов PostgreSQL → Flink SQL
    SELECT 
        column_name,
        source_table,
        CASE 
            -- Числовые типы
            WHEN data_type IN ('integer', 'int', 'smallint', 'bigint') THEN 'BIGINT'
            WHEN data_type IN ('numeric', 'decimal') THEN 'DECIMAL'
            WHEN data_type = 'double precision' THEN 'DOUBLE'
            WHEN data_type = 'real' THEN 'FLOAT'
            
            -- Строковые
            WHEN data_type IN ('character varying', 'varchar', 'text', 'char') THEN 'STRING'
            
            -- Бинарные
            WHEN data_type IN ('bytea', 'binary') THEN 'BYTES'
            
            -- Дата/время
            WHEN data_type IN ('timestamp', 'timestamp without time zone') THEN 'TIMESTAMP'
            WHEN data_type = 'timestamp with time zone' THEN 'TIMESTAMP_LTZ'
            WHEN data_type = 'date' THEN 'DATE'
            WHEN data_type = 'time' THEN 'TIME'
            
            -- Логические
            WHEN data_type = 'boolean' THEN 'BOOLEAN'
            
            -- Массивы → STRING (упрощённо)
            WHEN data_type LIKE '%[]' THEN 'STRING'
            
            -- JSON
            WHEN data_type IN ('json', 'jsonb') THEN 'STRING'
            
            -- UUID
            WHEN data_type = 'uuid' THEN 'STRING'
            
            -- По умолчанию
            ELSE 'STRING'
        END as flink_type,
        CASE WHEN is_nullable = 'YES' THEN 'NULL' ELSE 'NOT NULL' END as nullability,
        priority
    FROM unique_columns
),
pk_columns AS (
    -- Получаем первичные ключи
    SELECT DISTINCT kcu.column_name
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu 
        ON tc.constraint_schema = kcu.constraint_schema 
        AND tc.constraint_name = kcu.constraint_name
    WHERE tc.table_schema = 'dbo'
        AND tc.table_name IN ('constants', 'enumdescriptor')
        AND tc.constraint_type = 'PRIMARY KEY'
)
SELECT 
    'CREATE TABLE IF NOT EXISTS `default_catalog`.`cs_28826`.`cdc_source__dbo_dict` (' || E'\n' ||
    string_agg(
        '  `' || column_name || '`' || 
        repeat(' ', greatest(30 - length(column_name), 1)) || 
        flink_type || ' ' || nullability,
        ',' || E'\n' 
        ORDER BY priority, column_name
    ) || ',' || E'\n' ||
    '  `insertutcdate`       TIMESTAMP  NULL' || E'\n' ||
    ') with (' as ddl
FROM mapped_types
UNION ALL
SELECT '  PRIMARY KEY(`' || string_agg(column_name, '`, `') || '`) NOT ENFORCED'
FROM pk_columns;
```

## 🛠️ Улучшенная версия с разделением на логические блоки

```sql
-- Создаем функцию для удобства
CREATE OR REPLACE FUNCTION generate_flink_ddl(
    schema_name text,
    table1 text,
    table2 text,
    output_table text
) RETURNS text AS $$
DECLARE
    ddl_text text;
BEGIN
    WITH table_info AS (
        -- Полная информация о колонках
        SELECT DISTINCT ON (lower(c.column_name))
            c.column_name,
            CASE 
                WHEN c.data_type IN ('integer','int','bigint') THEN 'BIGINT'
                WHEN c.data_type LIKE '%char%' OR c.data_type = 'text' THEN 'STRING'
                WHEN c.data_type IN ('timestamp','timestamp without time zone') THEN 'TIMESTAMP'
                WHEN c.data_type = 'timestamp with time zone' THEN 'TIMESTAMP_LTZ'
                WHEN c.data_type = 'date' THEN 'DATE'
                WHEN c.data_type IN ('numeric','decimal') THEN 'DECIMAL'
                WHEN c.data_type = 'boolean' THEN 'BOOLEAN'
                WHEN c.data_type IN ('json','jsonb') THEN 'STRING'
                WHEN c.data_type = 'uuid' THEN 'STRING'
                ELSE 'STRING'
            END as flink_type,
            CASE WHEN c.is_nullable = 'YES' THEN 'NULL' ELSE 'NOT NULL' END as nullable,
            CASE 
                WHEN EXISTS (
                    SELECT 1 FROM information_schema.key_column_usage kcu
                    JOIN information_schema.table_constraints tc 
                        ON kcu.constraint_name = tc.constraint_name
                    WHERE kcu.table_schema = c.table_schema
                        AND kcu.table_name = c.table_name
                        AND kcu.column_name = c.column_name
                        AND tc.constraint_type = 'PRIMARY KEY'
                ) THEN 1
                ELSE 2
            END as sort_order
        FROM information_schema.columns c
        WHERE c.table_schema = schema_name
            AND c.table_name IN (table1, table2)
        ORDER BY lower(c.column_name), 
                 CASE WHEN c.table_name = table1 THEN 1 ELSE 2 END
    ),
    pk_info AS (
        -- Первичные ключи
        SELECT string_agg(kcu.column_name, '`, `') as pk_columns
        FROM information_schema.table_constraints tc
        JOIN information_schema.key_column_usage kcu 
            ON tc.constraint_schema = kcu.constraint_schema 
            AND tc.constraint_name = kcu.constraint_name
        WHERE tc.table_schema = schema_name
            AND tc.table_name IN (table1, table2)
            AND tc.constraint_type = 'PRIMARY KEY'
    )
    SELECT 
        'CREATE TABLE IF NOT EXISTS `default_catalog`.`cs_28826`.`' || output_table || '` (' || E'\n' ||
        string_agg(
            '  `' || column_name || '`' || 
            repeat(' ', greatest(30 - length(column_name), 1)) || 
            flink_type || ' ' || nullable,
            ',' || E'\n'
            ORDER BY sort_order, column_name
        ) || ',' || E'\n' ||
        '  `insertutcdate`       TIMESTAMP  NULL' || E'\n' ||
        ') with (' || E'\n' ||
        '  PRIMARY KEY(`' || (SELECT pk_columns FROM pk_info) || '`) NOT ENFORCED' || E'\n' ||
        ')' || E'\n' ||
        ') with ('
    INTO ddl_text
    FROM table_info;
    
    RETURN ddl_text;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT generate_flink_ddl('dbo', 'constants', 'enumdescriptor', 'cdc_source__dbo_dict');
```

## 📝 Скрипт для сохранения в файл

```bash
#!/bin/bash
# generate_flink_ddl.sh

DB_NAME="ваша_база"
SCHEMA="dbo"
TABLE1="constants"
TABLE2="enumdescriptor"
OUTPUT_FILE="flink_ddl_generated.sql"

# Генерируем DDL и сохраняем в файл
psql -d "$DB_NAME" -t -A -c "
WITH cols AS (
    SELECT DISTINCT ON (lower(column_name))
        column_name,
        CASE 
            WHEN data_type IN ('integer','bigint') THEN 'BIGINT'
            WHEN data_type LIKE '%char%' THEN 'STRING'
            WHEN data_type = 'timestamp without time zone' THEN 'TIMESTAMP'
            WHEN data_type = 'boolean' THEN 'BOOLEAN'
            ELSE 'STRING'
        END as flink_type
    FROM information_schema.columns
    WHERE table_schema = '$SCHEMA'
        AND table_name IN ('$TABLE1', '$TABLE2')
    ORDER BY lower(column_name)
)
SELECT 'CREATE TABLE IF NOT EXISTS \`default_catalog\`.\`cs_28826\`.\`cdc_source__dbo_dict\` ('
UNION ALL
SELECT '  \`' || column_name || '\`' || 
       repeat(' ', greatest(30 - length(column_name), 1)) || 
       flink_type || ' NULL,'
FROM cols
UNION ALL
SELECT '  \`insertutcdate\`       TIMESTAMP  NULL'
UNION ALL
SELECT ') with ('
UNION ALL
SELECT '  PRIMARY KEY(\`constantid\`, \`enumdescriptorid\`) NOT ENFORCED'
UNION ALL
SELECT ') with (';
" > "$OUTPUT_FILE"

echo "DDL сгенерирован в $OUTPUT_FILE"
```

## 💡 Дополнительные возможности

### 1. **Добавить комментарии с исходными типами PostgreSQL:**
```sql
SELECT '  `' || column_name || '`' || 
       repeat(' ', 30 - length(column_name)) ||
       'STRING NULL, -- PostgreSQL: ' || data_type || 
       CASE WHEN character_maximum_length IS NOT NULL 
            THEN '(' || character_maximum_length || ')' 
            ELSE '' END
FROM information_schema.columns
WHERE table_schema = 'dbo'
    AND table_name IN ('constants', 'enumdescriptor')
ORDER BY ordinal_position;
```

### 2. **Автоматически определять PK из обеих таблиц:**
```sql
SELECT 'PRIMARY KEY(`' || 
       string_agg(column_name, '`, `' ORDER BY ordinal_position) || 
       '`) NOT ENFORCED'
FROM (
    SELECT DISTINCT kcu.column_name, kcu.ordinal_position
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu 
        ON tc.constraint_schema = kcu.constraint_schema 
        AND tc.constraint_name = kcu.constraint_name
    WHERE tc.table_schema = 'dbo'
        AND tc.table_name IN ('constants', 'enumdescriptor')
        AND tc.constraint_type = 'PRIMARY KEY'
) pks;
```

## ✅ Рекомендация:

Используйте **первый основной скрипт** — он наиболее полный:
1. Автоматически определяет типы данных
2. Убирает дубликаты колонок
3. Правильно форматирует вывод
4. Добавляет техническое поле `insertutcdate`

