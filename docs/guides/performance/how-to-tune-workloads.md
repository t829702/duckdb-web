---
layout: docu
title: Tuning Workloads
---

## Parallelism 

DuckDB parallelizes the workload based on row groups. Parallelism starts at the level of row groups, which have 122,880 rows each. Therefore, for the query to run on _k_ threads, you need at least _k_ * 122,880 rows.

Note that in certain cases DuckDB may launch _too many threads_ (e.g., due to HyperThreading), which can lead to slowdowns. In these cases, it’s worth manually limiting the number of threads using [`PRAGMA threads = X`](../../sql/pragmas#memory_limit-threads).

## Profiling

If your queries are not performing as well as expected, it’s worth studying their query plans:
* Use [`EXPLAIN`](../meta/explain) to print the physical query plan without running the query.
* Use [`EXPLAIN ANALYZE`](../meta/explain_analyze) to run and profile the query. This will show the CPU time that each step in the query takes. Note that due to multi-threading, adding up each individual time will be larger than the total query processing time.

Query plans can point to the root of performance issues. For example:
Avoiding nested loop joins in favor or hash joins.
A scan that does not include a filter pushdown for a filter condition that is later applied.
Bad join orders where the cardinality of an operator explodes to billions of tuples.

## Prepared Statements

[Prepared statements](../../sql/query_syntax/prepared_statements) can improve performance when running the same query many times, but with different parameters. When a statement is prepared, it completes several of the initial portions of the query execution process (parsing, planning, etc.) and caches their output. When it is executed, those steps can be skipped, improving performance. This is beneficial mostly for repeatedly running small queries (with a runtime of < 100ms) with different sets of parameters.

Note that it is not a primary design goal for DuckDB to quickly execute many small queries concurrently. Rather, it is optimized for running larger, less frequent queries.

## Querying Remote Files

DuckDB uses synchronous IO when reading remote files. This means that each DuckDB thread can make at most one HTTP request at a time. If a query must make many small requests over the network, increasing DuckDB’s `threads` setting to larger than the total number of CPU cores (approx. 2-5 times CPU cores) can improve parallelism and performance.

## Best Practices for Using Connections

DuckDB will perform best when reusing the same database connection many times. Disconnecting and reconnecting on every query will incur some overhead, which can reduce performance when running many small queries. DuckDB also caches some data and metadata in memory, and that cache is lost when the last open connection is closed. Frequently, a single connection will work best, but a connection pool may also be used. 

Using multiple connections can parallelize some operations, although it is typically not necessary. DuckDB does attempt to parallelize as much as possible within each individual query, but it is not possible to parallelize in all cases. Making multiple connections can process more operations concurrently. This can be more helpful if DuckDB is not CPU limited, but instead bottlenecked by another resource like network transfer speed.

## The `preserve_insertion_order` Option

When importing or exporting data sets that are much larger than the available memory, out of memory errors may occur. In these cases, it’s worth setting the [`preserve_insertion_order` configuration option](../../sql/configuration) to `false`:

```sql
SET preserve_insertion_order = false;
```

This allows the systems to re-order any results that do not contain `ORDER BY` clauses, potentially reducing memory usage.