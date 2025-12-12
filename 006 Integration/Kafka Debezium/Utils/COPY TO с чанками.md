**`COPY TO` с чанками** — это техника экспорта больших таблиц PostgreSQL частями (чанками), чтобы:

1. **Не блокировать** таблицу надолго
2. **Контролировать потребление памяти**
3. **Позволить параллельную загрузку** в целевую систему
4. **Продолжать работу приложения** во время экспорта

## **Базовый синтаксис `COPY TO`**

Обычный `COPY` экспортирует всю таблицу сразу:
```sql
COPY huge_table TO '/tmp/export.csv' WITH CSV HEADER;
```
**Проблема:** Для таблицы с 1 млрд строк это:
- Займет много времени
- Создаст огромный файл (сотни ГБ)
- Может упасть по памяти
- Будет мешать другим операциям

## **`COPY TO` с чанками — решение**

### **Способ 1: По диапазонам первичного ключа**

```sql
-- 1. Находим границы
SELECT MIN(id), MAX(id) FROM huge_table;
-- Результат: 1, 1000000000

-- 2. Экспортируем чанками по 1M записей
DO $$
DECLARE
  chunk_size INTEGER := 1000000;
  start_id BIGINT := 1;
  end_id BIGINT := 1000000000;
  current_start BIGINT;
BEGIN
  FOR current_start IN SELECT generate_series(start_id, end_id, chunk_size) LOOP
    EXECUTE format('
      COPY (
        SELECT * FROM huge_table 
        WHERE id >= %s AND id < %s
        ORDER BY id
      ) TO ''/tmp/chunk_%s.csv'' WITH CSV HEADER',
      current_start,
      current_start + chunk_size,
      current_start
    );
    
    RAISE NOTICE 'Экспортирован чанк % - %', 
      current_start, current_start + chunk_size - 1;
  END LOOP;
END $$;
```

### **Способ 2: Используя `OFFSET` и `LIMIT` (менее эффективно)**

```sql
DO $$
DECLARE
  chunk_size INTEGER := 1000000;
  total_rows BIGINT;
  offset_val INTEGER := 0;
BEGIN
  -- Узнаем общее количество строк
  SELECT COUNT(*) INTO total_rows FROM huge_table;
  
  WHILE offset_val < total_rows LOOP
    EXECUTE format('
      COPY (
        SELECT * FROM huge_table 
        ORDER BY id
        LIMIT %s OFFSET %s
      ) TO ''/tmp/chunk_offset_%s.csv'' WITH CSV',
      chunk_size, offset_val, offset_val
    );
    
    offset_val := offset_val + chunk_size;
    RAISE NOTICE 'Прогресс: %/% строк', 
      LEAST(offset_val, total_rows), total_rows;
  END LOOP;
END $$;
```

### **Способ 3: По диапазонам времени (для временных таблиц)**

```sql
-- Если есть created_at timestamp
DO $$
DECLARE
  start_date TIMESTAMP;
  end_date TIMESTAMP;
  current_date TIMESTAMP;
BEGIN
  SELECT MIN(created_at), MAX(created_at) 
  INTO start_date, end_date 
  FROM huge_table;
  
  current_date := start_date;
  
  WHILE current_date <= end_date LOOP
    EXECUTE format('
      COPY (
        SELECT * FROM huge_table 
        WHERE created_at >= %L 
          AND created_at < %L
      ) TO ''/tmp/chunk_%s.csv'' WITH CSV',
      current_date,
      current_date + INTERVAL '1 day',
      to_char(current_date, 'YYYYMMDD')
    );
    
    current_date := current_date + INTERVAL '1 day';
  END LOOP;
END $$;
```

## **Практический пример: Экспорт с пагинацией и возобновлением**

```sql
-- Таблица для контроля прогресса
CREATE TABLE export_progress (
  table_name TEXT PRIMARY KEY,
  last_exported_id BIGINT,
  chunk_size INTEGER,
  total_exported BIGINT DEFAULT 0,
  status TEXT DEFAULT 'running'
);

-- Функция для чанкового экспорта с возможностью паузы/возобновления
CREATE OR REPLACE FUNCTION export_table_chunked(
  table_name TEXT,
  chunk_size INTEGER DEFAULT 100000,
  max_chunks INTEGER DEFAULT NULL
) RETURNS VOID AS $$
DECLARE
  min_id BIGINT;
  max_id BIGINT;
  current_start BIGINT;
  chunks_processed INTEGER := 0;
  last_id_in_chunk BIGINT;
BEGIN
  -- Получаем границы
  EXECUTE format('SELECT MIN(id), MAX(id) FROM %I', table_name)
  INTO min_id, max_id;
  
  -- Инициализируем или продолжаем прогресс
  INSERT INTO export_progress 
    (table_name, last_exported_id, chunk_size)
  VALUES (table_name, min_id - 1, chunk_size)
  ON CONFLICT (table_name) DO UPDATE
    SET status = 'running'
  RETURNING last_exported_id INTO current_start;
  
  current_start := current_start + 1;
  
  WHILE current_start <= max_id LOOP
    -- Экспортируем чанк
    EXECUTE format('
      COPY (
        SELECT * FROM %I 
        WHERE id >= %s 
        ORDER BY id 
        LIMIT %s
      ) TO ''/tmp/%s_chunk_%s.csv'' WITH CSV',
      table_name, current_start, chunk_size,
      table_name, to_char(current_start, 'FM0000000000')
    );
    
    -- Находим последний ID в этом чанке
    EXECUTE format('
      SELECT MAX(id) FROM (
        SELECT id FROM %I 
        WHERE id >= %s 
        ORDER BY id 
        LIMIT %s
      ) t',
      table_name, current_start, chunk_size
    ) INTO last_id_in_chunk;
    
    -- Обновляем прогресс
    UPDATE export_progress 
    SET 
      last_exported_id = COALESCE(last_id_in_chunk, current_start),
      total_exported = total_exported + chunk_size
    WHERE table_name = export_table_chunked.table_name;
    
    -- Подготовка следующего чанка
    IF last_id_in_chunk IS NOT NULL THEN
      current_start := last_id_in_chunk + 1;
    ELSE
      current_start := current_start + chunk_size;
    END IF;
    
    chunks_processed := chunks_processed + 1;
    
    -- Пауза между чанками (чтобы не нагружать систему)
    PERFORM pg_sleep(0.1);
    
    -- Проверяем лимит чанков
    IF max_chunks IS NOT NULL AND chunks_processed >= max_chunks THEN
      UPDATE export_progress 
      SET status = 'paused'
      WHERE table_name = export_table_chunked.table_name;
      RAISE NOTICE 'Экспорт приостановлен после % чанков', max_chunks;
      RETURN;
    END IF;
  END LOOP;
  
  -- Завершаем
  UPDATE export_progress 
  SET status = 'completed'
  WHERE table_name = export_table_chunked.table_name;
  
  RAISE NOTICE 'Экспорт таблицы % завершен', table_name;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT export_table_chunked('huge_table', 100000);

-- Продолжить после паузы
SELECT export_table_chunked('huge_table', 100000);
```

## **Параллельный экспорт несколькими процессами**

```bash
#!/bin/bash
# export_parallel.sh

TABLE="huge_table"
CHUNK_SIZE=1000000
WORKERS=4

# Получаем границы
MIN_MAX=$(psql -t -c "SELECT MIN(id), MAX(id) FROM $TABLE")
MIN_ID=$(echo $MIN_MAX | cut -d'|' -f1)
MAX_ID=$(echo $MIN_MAX | cut -d'|' -f2)

# Разбиваем на диапазоны по воркерам
TOTAL_ROWS=$((MAX_ID - MIN_ID + 1))
ROWS_PER_WORKER=$((TOTAL_ROWS / WORKERS))

for i in $(seq 0 $((WORKERS-1))); do
  START_ID=$((MIN_ID + i * ROWS_PER_WORKER))
  END_ID=$((START_ID + ROWS_PER_WORKER - 1))
  
  # Последний воркер берет остаток
  if [ $i -eq $((WORKERS-1)) ]; then
    END_ID=$MAX_ID
  fi
  
  # Запускаем экспорт в фоне
  psql -c "
    DO \$$
    DECLARE
      cur_start BIGINT := $START_ID;
    BEGIN
      WHILE cur_start <= $END_ID LOOP
        EXECUTE format('
          COPY (
            SELECT * FROM $TABLE 
            WHERE id >= %s AND id < %s
          ) TO ''/tmp/chunk_worker${i}_%s.csv'' WITH CSV',
          cur_start,
          cur_start + $CHUNK_SIZE,
          cur_start
        );
        cur_start := cur_start + $CHUNK_SIZE;
      END LOOP;
    END
    \$$" &
done

# Ждем завершения всех воркеров
wait
echo "Все воркеры завершили экспорт"
```

## **Интеграция с Debezium CDC**

Когда используете `COPY TO` с чанками для инициализации + Debezium для CDC:

```sql
-- 1. Фиксируем LSN перед началом
SELECT pg_current_wal_lsn() AS start_lsn \gset

-- 2. Экспортируем чанками
SELECT export_table_chunked('huge_table', 100000);

-- 3. Запускаем Debezium (уже должен работать в фоне с snapshot.mode=never)
-- 4. Загружаем чанки в StarRocks параллельно

-- 5. Настраиваем Routine Load с offset, соответствующим :start_lsn
```

## **Плюсы `COPY TO` с чанками:**

1. **Нет долгих блокировок** — только на время чтения каждого чанка
2. **Управление памятью** — каждый чанк обрабатывается отдельно
3. **Возможность паузы/возобновления**
4. **Параллельная обработка** — можно загружать чанки в StarRocks параллельно
5. **Мониторинг прогресса** — видно, сколько уже экспортировано
6. **Отказоустойчивость** — при падении можно продолжить с последнего чанка

## **Минусы:**

1. **Сложнее реализовать** чем простой `COPY`
2. **Нужен подходящий ключ для чанков** (желательно sequential ID)
3. **Возможны пропуски/дубли**, если данные меняются во время экспорта

## **Альтернативы для очень больших таблиц:**

1. **pg_dump с `-j N`** (параллельный дамп)
2. **pgcopydb** — специализированный инструмент
3. **Встроенный механизм экспорта Debezium** (инкрементальный снапшот)
4. **Логическая репликация + snapshot** (создать реплику, сделать снимок с неё)

**Рекомендация:** Для инициализации млрдных таблиц с Debezium используйте:
```
1. snapshot.mode: never
2. COPY TO с чанками для начального снимка  
3. Debezium ловит изменения с момента начала COPY
4. Загружаем чанки + применяем дельты с соответствующего LSN
```
