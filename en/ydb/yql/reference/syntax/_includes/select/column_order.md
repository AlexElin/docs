---
sourcePath: en/ydb/ydb-docs-core/en/core/yql/reference/yql-docs-core-2/syntax/_includes/select/column_order.md
sourcePath: en/ydb/yql/reference/yql-docs-core-2/syntax/_includes/select/column_order.md
---

## Column order in YQL {#orderedcolumns}

The standard SQL is sensitive to the order of columns in projections (that is, in `SELECT`). While the order of columns must be preserved in the query results or when writing data to a new table, some SQL constructs use this order.
This applies, for example, to [UNION ALL](#unionall) and positional [ORDER BY](#orderby) (ORDER BY ordinal).

The column order is ignored in YQL by default:
* The order of columns in the output tables and query results is undefined
* The data scheme of the `UNION ALL` result is output by column names rather than positions

If you enable `PRAGMA OrderedColumns;`, the order of columns is preserved in the query results and is derived from the order of columns in the input tables using the following rules:
* `SELECT`: an explicit column enumeration dictates the result order.
* `SELECT` with an asterisk (`SELECT * FROM ...`) inherits the order from its input;
* The order of columns after [JOIN](../../join.md): First output the left-hand columns, then the right-hand ones. If the column order in any of the sides in the `JOIN` output is undefined, the column order in the result is also undefined.
* The order in `UNION ALL` depends on the [UNION ALL](#unionall) execution mode.
* The column order for [AS_TABLE](#as_table) is undefined.

{% note warning %}

In the YT table schema, key columns always precede non-key columns. The order of key columns is determined by the order of the composite key.
When `PRAGMA OrderedColumns;` is enabled, non-key columns preserve their output order.

{% endnote %}
