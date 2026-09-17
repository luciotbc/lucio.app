---
title: "PostgreSQL: tuning and investigation via the SQL console"
date: 2026-04-21 14:00:00 -0300
updated: 2026-04-21 14:00:00 -0300
tags: [postgresql, sql, performance, tuning, ops]
excerpt: "Visual EXPLAIN with PEV2, work_mem adjustment, reading index usage, counting rows across all tables, finding FKs by name and resetting sequences."
lang: en
ref: postgres-tuning-e-investigacao-pelo-console
---

> **PostgreSQL series** — part 2 of 2
>
> 1. [Installation on Mac and common bugs](/posts/postgres-mac-homebrew-instalacao-e-bugs/)
> 2. **You are here — Tuning and investigation via the SQL console**

With Postgres running and `pg_stat_statements` enabled (see [part 1](/posts/postgres-mac-homebrew-instalacao-e-bugs/)), the next step is understanding what's slow and why. This post collects the diagnostic queries I run directly in `psql` when I need to understand an installation I'm not familiar with.

## Visual EXPLAIN with PEV2

`EXPLAIN ANALYZE` output in the terminal is readable, but a tree with 50 nodes becomes a soup. To understand large plans I use [PEV2](https://github.com/dalibo/pev2) — just generate the plan in JSON and paste it into the page.

```sql
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON)
SELECT ...;
```

- `ANALYZE` actually executes the query (be careful with `UPDATE`/`DELETE`).
- `BUFFERS` shows cache hit vs disk reads — essential for knowing whether the problem is I/O or the plan.
- `FORMAT JSON` is the format PEV2 understands.

Save the JSON to a file, open PEV2 (you can run it offline with the HTML they distribute), paste it in, and you get a diagram with timings per node and bottlenecks highlighted.

## Adjusting `work_mem` for a query

If a query is doing merge/sort on disk instead of in memory, you can give it more memory just for that session:

```sql
SET work_mem = '100MB';
COMMIT;
SHOW work_mem;
```

`SET` only applies to the current session — no risk of globally affecting production. Useful for testing whether increasing memory solves the problem before making a change to `postgresql.conf` (which affects all connections and can blow up RAM in parallel).

## Resetting statistics to start fresh

```sql
SELECT pg_stat_reset();
```

Resets the `pg_stat_*` counters. Useful when you want to measure only the effect of a recent change — after running this, wait a bit and you'll see the new state of Postgres without the mixed history.

## Index usage: which ones are useful and which are just there

This is the query I run most often in inherited databases:

```sql
SELECT
    idstat.relname              AS table_name,
    indexrelname                AS index_name,
    idstat.idx_scan             AS index_scans_count,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    tabstat.idx_scan            AS table_reads_index_count,
    tabstat.seq_scan            AS table_reads_seq_count,
    tabstat.seq_scan + tabstat.idx_scan AS table_reads_count,
    n_tup_upd + n_tup_ins + n_tup_del   AS table_writes_count,
    pg_size_pretty(pg_relation_size(idstat.relid)) AS table_size
FROM
    pg_stat_user_indexes AS idstat
JOIN
    pg_indexes
    ON  indexrelname = indexname
    AND idstat.schemaname = pg_indexes.schemaname
JOIN
    pg_stat_user_tables AS tabstat
    ON  idstat.relid = tabstat.relid
WHERE
    indexdef !~* 'unique'
ORDER BY
    idstat.idx_scan DESC,
    pg_relation_size(indexrelid) DESC;
```

Reading this is straightforward:

- An index with `index_scans_count` = 0 and large size is a candidate for `DROP INDEX`. It's taking up disk space and slowing down every `INSERT`/`UPDATE` on that table without being used.
- A table with `seq_scan` much higher than `idx_scan` is generally missing an index — or has such low cardinality that `seq_scan` is actually better (measure before creating one).
- The filter `indexdef !~* 'unique'` hides unique indexes (which exist for constraint guarantees, not performance) — they should not be candidates for removal.

## Count rows for all tables in a schema

`SELECT count(*) FROM each_table` is tedious. This query generates `count(*)` per table using XML to avoid dynamic SQL:

```sql
SELECT table_schema,
       table_name,
       (xpath('/row/cnt/text()', xml_count))[1]::text::int AS row_count
FROM (
  SELECT table_name,
         table_schema,
         query_to_xml(
           format('SELECT count(*) AS cnt FROM %I.%I', table_schema, table_name),
           false, true, ''
         ) AS xml_count
  FROM information_schema.tables
  WHERE table_schema = 'public'
) t
ORDER BY table_name;
```

Replace `'public'` with the schema you want to inspect. It's slow on large databases (does a sequential `count(*)` on each table), but gives a complete inventory in a single query.

## Find a foreign key by name

When Rails complains `PG::ForeignKeyViolation: ERROR: insert or update on table "x" violates foreign key constraint "fk_rails_96f00dec22"`, the name `fk_rails_<hash>` tells you nothing. To find out what it references:

```sql
SELECT conrelid::regclass AS table_name,
       conname            AS foreign_key,
       pg_get_constraintdef(oid)
FROM   pg_constraint
WHERE  contype = 'f'
  AND  conname = 'fk_rails_96f00dec22'
  AND  connamespace = 'public'::regnamespace
ORDER  BY conrelid::regclass::text, contype DESC;
```

`pg_get_constraintdef` outputs the `FOREIGN KEY (...) REFERENCES ...` as text. Together with `conrelid::regclass` (the table where the FK lives), you close the investigation in seconds.

## Resetting a sequence after a manual import

If you imported data via `INSERT` keeping the original IDs, the sequence wasn't touched — the next `INSERT` that depends on `nextval` will throw `duplicate key`. To resync:

```sql
SELECT SETVAL('table_name_id_seq', (SELECT MAX(id) + 1 FROM table_name));
```

Postgres convention: the default sequence for `table.id` is named `table_id_seq`. If you renamed the PK or used a custom name, check with `\d table_name`.

## Bonus: dump to file

Not tuning, but the command I always use alongside:

```bash
pg_dump -C -h localhost -U user dbname > ~/source/db1.sql
```

`-C` includes the `CREATE DATABASE` statement in the dump — useful for restoring to a server where the database doesn't exist yet. Without `-C`, the dump assumes the target database is already created.

## Next notes

- Recipe for hunting slow queries with `pg_stat_statements` (top 20 by `total_time`, top 20 by `calls`).
- Initial configuration of `shared_buffers`, `effective_cache_size` and `random_page_cost` for SSD.
- `VACUUM`, `ANALYZE` and when to run manually beyond autovacuum.
