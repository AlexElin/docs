---
sourcePath: ru/ydb/ydb-docs-core/ru/core/reference/ydb-sdk/example/_includes/steps/03_write_queries.md
---
## Запись данных {#write-queries}

Выполняется запись данных в созданные таблицы с использованием команды [`UPSERT`](../../../../../yql/reference/syntax/upsert_into.md) языка запросов [YQL](../../../../../yql/reference/index.md). Применяется режим передачи запроса на изменение данных с автоматическим подтверждением транзакции в одном запросе к серверу.
