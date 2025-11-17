# [StarRocks Migration Tool (SMT)](https://docs.starrocks.io/docs/integrations/loading_tools/SMT/)

Flink CDC для чтения изменений данных из PostgreSQL использует Debezium, который, в свою очередь, построен на платформе Kafka Connect.   
Поэтому в стектрейсе ошибки вы видите классы из пакета org.apache.flink.cdc.connectors.shaded.org.apache.kafka.connect.errors.

Когда в конфигурации возникает проблема (например, неверные параметры подключения к PostgreSQL, недостаток прав или сетевые issues), Debezium генерирует исключение через механизм Kafka Connect, что и приводит к такой ошибке.

