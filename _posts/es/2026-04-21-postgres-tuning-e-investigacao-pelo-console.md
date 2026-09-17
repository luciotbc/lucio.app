---
title: "PostgreSQL: tuning e investigación por la consola SQL"
date: 2026-04-21 14:00:00 -0300
updated: 2026-04-21 14:00:00 -0300
tags: [postgresql, sql, performance, tuning, ops]
excerpt: "EXPLAIN visual con PEV2, ajuste de work_mem, lectura de uso de índices, conteo de filas de todas las tablas, búsqueda de FK por nombre y reset de sequence."
lang: es
ref: postgres-tuning-e-investigacao-pelo-console
---

> **Serie PostgreSQL** — parte 2 de 2
>
> 1. [Instalación en Mac y bugs comunes](/posts/postgres-mac-homebrew-instalacao-e-bugs/)
> 2. **Estás aquí — Tuning e investigación por la consola SQL**

Con Postgres corriendo y `pg_stat_statements` habilitado (ver [parte 1](/posts/postgres-mac-homebrew-instalacao-e-bugs/)), el paso siguiente es entender qué está lento y por qué. Este post reúne las queries de diagnóstico que ejecuto directamente en `psql` cuando necesito entender una instalación que no conozco bien.

## EXPLAIN visual con PEV2

`EXPLAIN ANALYZE` volcado en la terminal es legible, pero un árbol con 50 nodos se convierte en sopa. Para entender planes grandes uso [PEV2](https://github.com/dalibo/pev2) — basta generar el plan en JSON y pegarlo en la página.

```sql
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON)
SELECT ...;
```

- `ANALYZE` ejecuta la query de verdad (cuidado con `UPDATE`/`DELETE`).
- `BUFFERS` muestra cache hit vs lectura de disco — fundamental para saber si el problema es I/O o el plan.
- `FORMAT JSON` es el formato que PEV2 entiende.

Guardá el JSON en un archivo, abrí PEV2 (se puede correr offline con el HTML que distribuyen), pegalo, y obtenés un diagrama con tiempos por nodo y cuellos de botella resaltados.

## Ajustar `work_mem` para una query

Si una query está haciendo merge/sort en disco en lugar de en memoria, podés darle más memoria solo para esa sesión:

```sql
SET work_mem = '100MB';
COMMIT;
SHOW work_mem;
```

`SET` solo vale en la sesión actual — sin riesgo de afectar producción globalmente. Útil para probar si aumentar la memoria resuelve antes de hacer un cambio en `postgresql.conf` (que afecta todas las conexiones y puede reventar la RAM en paralelo).

## Resetear estadísticas para empezar limpio

```sql
SELECT pg_stat_reset();
```

Resetea los contadores de `pg_stat_*`. Útil cuando querés medir solo el efecto de un cambio reciente — después de ejecutar esto, esperá un poco y verás el estado nuevo de Postgres sin el historial mezclado.

## Uso de índices: cuáles son útiles y cuáles están de adorno

Esta es la query que más ejecuto en bases heredadas:

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

La lectura es directa:

- Un índice con `index_scans_count` = 0 y tamaño grande es candidato a `DROP INDEX`. Está ocupando disco y desacelerando cada `INSERT`/`UPDATE` de esa tabla sin ser usado.
- Una tabla con `seq_scan` mucho mayor que `idx_scan` en general está faltándole un índice — o tiene cardinalidad tan baja que `seq_scan` es mejor de todas formas (medí antes de crear).
- El filtro `indexdef !~* 'unique'` oculta índices únicos (que existen por garantía de constraint, no por performance) — no deben ser candidatos a eliminación.

## Contar filas de todas las tablas de un schema

`SELECT count(*) FROM cada_tabla` es tedioso. Esta query genera el `count(*)` por tabla usando XML para evitar SQL dinámico:

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

Reemplazá `'public'` por el schema que querés inspeccionar. Es lento en bases grandes (hace `count(*)` secuencial en cada tabla), pero da un inventario completo en una sola query.

## Encontrar una foreign key por nombre

Cuando Rails se queja con `PG::ForeignKeyViolation: ERROR: insert or update on table "x" violates foreign key constraint "fk_rails_96f00dec22"`, el nombre `fk_rails_<hash>` no dice nada. Para descubrir qué referencia:

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

`pg_get_constraintdef` escupe el `FOREIGN KEY (...) REFERENCES ...` en texto. Junto con `conrelid::regclass` (la tabla donde vive la FK), cerrás la investigación en segundos.

## Resetear una sequence después de una importación manual

Si importaste datos vía `INSERT` manteniendo los IDs originales, la sequence no fue tocada — el próximo `INSERT` que dependa de `nextval` va a tirar `duplicate key`. Para resincronizar:

```sql
SELECT SETVAL('table_name_id_seq', (SELECT MAX(id) + 1 FROM table_name));
```

Convención de Postgres: la sequence por defecto de `tabla.id` se llama `tabla_id_seq`. Si renombraste el PK o usaste un nombre custom, verificá con `\d table_name`.

## Bonus: dump a archivo

No es tuning, pero es el comando que siempre uso junto:

```bash
pg_dump -C -h localhost -U user dbname > ~/source/db1.sql
```

`-C` incluye el `CREATE DATABASE` en el dump — útil para restaurar en un servidor donde la base todavía no existe. Sin `-C`, el dump asume que la base destino ya está creada.

## Próximas anotaciones

- Receta para cazar slow queries con `pg_stat_statements` (top 20 por `total_time`, top 20 por `calls`).
- Configuración inicial de `shared_buffers`, `effective_cache_size` y `random_page_cost` para SSD.
- `VACUUM`, `ANALYZE` y cuándo ejecutar manualmente más allá del autovacuum.
