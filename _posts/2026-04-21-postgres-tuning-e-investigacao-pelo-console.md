---
title: "PostgreSQL: tuning e investigação pelo console SQL"
date: 2026-04-21 14:00:00 -0300
updated: 2026-04-21 14:00:00 -0300
tags: [postgresql, sql, performance, tuning, ops]
excerpt: "EXPLAIN visual com PEV2, ajuste de work_mem, leitura de uso de índices, contagem de linhas de todas as tabelas, busca de FK pelo nome e reset de sequence."
---

> **Série PostgreSQL** — parte 2 de 2
>
> 1. [Instalação no Mac e bugs comuns]({% post_url 2026-04-20-postgres-mac-homebrew-instalacao-e-bugs %})
> 2. **Você está aqui — Tuning e investigação pelo console SQL**

Com o Postgres rodando e o `pg_stat_statements` habilitado (ver [parte 1]({% post_url 2026-04-20-postgres-mac-homebrew-instalacao-e-bugs %})), o passo seguinte é entender o que está lento e por quê. Esse post junta as queries de diagnóstico que eu rodo direto no `psql` quando preciso entender uma instalação que não conheço bem.

## EXPLAIN visual com PEV2

`EXPLAIN ANALYZE` cuspido no terminal é legível, mas árvore com 50 nós vira sopa. Pra entender plano grande eu uso o [PEV2](https://github.com/dalibo/pev2) — basta gerar o plano em JSON e colar na página.

```sql
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON)
SELECT ...;
```

- `ANALYZE` executa a query de fato (cuidado com `UPDATE`/`DELETE`).
- `BUFFERS` mostra cache hit vs leitura de disco — fundamental pra saber se o problema é I/O ou plano.
- `FORMAT JSON` é o formato que o PEV2 entende.

Salve o JSON num arquivo, abra o PEV2 (dá pra rodar offline com o HTML que eles distribuem), cole, e você ganha um diagrama com tempos por nó e gargalos pintados.

## Ajustando `work_mem` pra uma query

Se uma query está fazendo merge/sort em disco em vez de em memória, dá pra dar mais memória só pra ela:

```sql
SET work_mem = '100MB';
COMMIT;
SHOW work_mem;
```

`SET` só vale na sessão atual — sem risco de afetar produção globalmente. Útil pra testar se aumentar memória resolve antes de fazer alteração no `postgresql.conf` (que afeta todas as conexões e pode estourar RAM em paralelo).

## Resetando estatísticas pra começar limpo

```sql
SELECT pg_stat_reset();
```

Zera os contadores de `pg_stat_*`. Útil quando você quer medir só o efeito de uma mudança recente — depois de rodar isso, espera um pouco e vai ver o estado novo do Postgres sem o histórico misturado.

## Uso de índices: quem é útil e quem está lá à toa

Essa é a query que eu mais rodo em banco que herdo:

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

A leitura é direta:

- Índice com `index_scans_count` = 0 e tamanho grande é candidato a `DROP INDEX`. Está ocupando disco e desacelerando todo `INSERT`/`UPDATE` daquela tabela sem ser usado.
- Tabela com `seq_scan` muito maior que `idx_scan` em geral está faltando índice — ou tem cardinalidade tão baixa que `seq_scan` é melhor mesmo (medir antes de criar).
- O filtro `indexdef !~* 'unique'` esconde índices únicos (que existem por garantia de constraint, não por performance) — eles não devem ser candidatos a remoção.

## Contar linhas de todas as tabelas de um schema

`SELECT count(*) FROM cada_tabela` é tedioso. Essa query gera o `count(*)` por tabela usando XML pra evitar SQL dinâmico:

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

Troque `'public'` pelo schema que você quer inspecionar. É lento em base grande (faz `count(*)` sequencial em cada tabela), mas dá um inventário completo numa query só.

## Achar uma foreign key pelo nome

Quando o Rails reclama `PG::ForeignKeyViolation: ERROR: insert or update on table "x" violates foreign key constraint "fk_rails_96f00dec22"`, o nome `fk_rails_<hash>` não diz nada. Pra descobrir o que ela referencia:

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

`pg_get_constraintdef` cospe o `FOREIGN KEY (...) REFERENCES ...` em texto. Junto com `conrelid::regclass` (a tabela onde a FK vive), você fecha a investigação em segundos.

## Resetando uma sequence depois de import manual

Se você importou dados via `INSERT` mantendo os IDs originais, a sequence não foi tocada — o próximo `INSERT` que dependa do `nextval` vai estourar `duplicate key`. Pra ressincronizar:

```sql
SELECT SETVAL('table_name_id_seq', (SELECT MAX(id) + 1 FROM table_name));
```

Convenção do Postgres: a sequence default de `tabela.id` se chama `tabela_id_seq`. Se você renomeou o PK ou usou nome custom, confira no `\d table_name`.

## Bônus: dump pra arquivo

Não é tuning, mas é o comando que sempre uso junto:

```bash
pg_dump -C -h localhost -U user dbname > ~/source/db1.sql
```

`-C` inclui o `CREATE DATABASE` no dump — útil pra restaurar num servidor onde o banco ainda não existe. Sem `-C`, o dump assume que o banco-destino já está criado.

## Próximas anotações

- Receita pra caçar slow query com `pg_stat_statements` (top 20 por `total_time`, top 20 por `calls`).
- Configuração inicial de `shared_buffers`, `effective_cache_size` e `random_page_cost` pra SSD.
- `VACUUM`, `ANALYZE` e quando rodar manualmente além do autovacuum.
