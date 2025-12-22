# Referência: Views de Catálogo e Comandos Úteis - Greenplum 7

Este documento contém comandos essenciais para administração e monitoramento do Greenplum 7.

---

## Índice

1. [Views de Particionamento](#1-views-de-particionamento)
2. [Análise de Skew](#2-análise-de-skew)
3. [Tamanho de Objetos](#3-tamanho-de-objetos)
4. [Compressão e Storage](#4-compressão-e-storage)
5. [Distribuição de Dados](#5-distribuição-de-dados)
6. [Monitoramento de Sessões](#6-monitoramento-de-sessões)
7. [Manutenção e VACUUM](#7-manutenção-e-vacuum)
8. [Locks e Bloqueios](#8-locks-e-bloqueios)
9. [Performance e Estatísticas](#9-performance-e-estatísticas)
10. [Informações do Cluster](#10-informações-do-cluster)
11. [Catálogo de Tabelas](#11-catálogo-de-tabelas)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Views de Particionamento

> ⚠️ **NOTA:** A view `pg_catalog.pg_partitions` não existe em Greenplum 7. Use os comandos alternativos baseados em `pg_inherits` e `pg_partitioned_table`.

### Listar todas as partições de uma tabela (com detalhes)

> **Nota:** `pg_catalog.pg_partitions` não existe em GP7. Use este comando alternativo:

```sql
SELECT 
    c.relname AS particao,
    pg_get_expr(c.relpartbound, c.oid) AS limite,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho,
    c.reltuples::BIGINT AS linhas_estimadas
FROM pg_class p
JOIN pg_inherits i ON p.oid = i.inhparent
JOIN pg_class c ON i.inhrelid = c.oid
WHERE p.relname = 'nome_tabela'
ORDER BY c.relname;
```

### Alternativa usando pg_inherits (método nativo PostgreSQL)

```sql
SELECT 
    c.relname AS particao,
    pg_get_expr(c.relpartbound, c.oid) AS limite,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho,
    c.reltuples::BIGINT AS linhas_estimadas
FROM pg_class p
JOIN pg_inherits i ON p.oid = i.inhparent
JOIN pg_class c ON i.inhrelid = c.oid
WHERE p.relname = 'nome_tabela'
ORDER BY c.relname;
```

### Colunas de particionamento

```sql
SELECT 
    c.relname AS tabela,
    a.attname AS coluna_particionamento,
    format_type(a.atttypid, a.atttypmod) AS tipo_dado
FROM pg_class c
JOIN pg_partitioned_table pt ON c.oid = pt.partrelid
JOIN pg_attribute a ON a.attrelid = c.oid AND a.attnum = ANY(pt.partattrs)
WHERE c.relname = 'nome_tabela';
```

---

## 2. Análise de Skew

### Coeficiente de skew por tabela

```sql
SELECT 
    skcnamespace AS schema,
    skcrelname AS tabela,
    ROUND(skccoeff::numeric, 2) AS skew_coefficient,
    CASE 
        WHEN skccoeff > 1.0 THEN 'CRÍTICO'
        WHEN skccoeff > 0.5 THEN 'ALERTA'
        WHEN skccoeff > 0.3 THEN 'MODERADO'
        ELSE 'OK'
    END AS status
FROM gp_toolkit.gp_skew_coefficients
WHERE skcnamespace NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY skccoeff DESC;
```

### Distribuição de linhas por segmento

```sql
SELECT 
    gp_segment_id,
    COUNT(*) AS num_rows
FROM nome_tabela
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

### Estatísticas detalhadas de skew

```sql
WITH segment_counts AS (
    SELECT gp_segment_id, COUNT(*) AS row_count
    FROM nome_tabela
    GROUP BY gp_segment_id
),
stats AS (
    SELECT 
        AVG(row_count) AS avg_rows,
        MAX(row_count) AS max_rows,
        MIN(row_count) AS min_rows,
        STDDEV(row_count) AS stddev_rows
    FROM segment_counts
)
SELECT 
    ROUND(avg_rows) AS media_linhas,
    max_rows AS max_linhas,
    min_rows AS min_linhas,
    ROUND(stddev_rows) AS desvio_padrao,
    ROUND((max_rows - avg_rows) / NULLIF(avg_rows, 0) * 100, 2) AS skew_percentage,
    CASE 
        WHEN (max_rows - avg_rows) / NULLIF(avg_rows, 0) > 0.5 THEN 'CRÍTICO'
        WHEN (max_rows - avg_rows) / NULLIF(avg_rows, 0) > 0.2 THEN 'ALERTA'
        WHEN (max_rows - avg_rows) / NULLIF(avg_rows, 0) > 0.1 THEN 'MODERADO'
        ELSE 'OK'
    END AS status
FROM stats;
```

### Tabelas com maior skew no banco

```sql
SELECT 
    skcnamespace AS schema,
    skcrelname AS tabela,
    ROUND(skccoeff::numeric, 2) AS skew,
    pg_size_pretty(pg_total_relation_size((skcnamespace || '.' || skcrelname)::regclass)) AS tamanho
FROM gp_toolkit.gp_skew_coefficients
WHERE skcnamespace NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
  AND skccoeff > 0.5
ORDER BY skccoeff DESC
LIMIT 20;
```

---

## 3. Tamanho de Objetos

### Tamanho de uma tabela específica

```sql
SELECT 
    pg_size_pretty(pg_relation_size('schema.tabela')) AS tamanho_tabela,
    pg_size_pretty(pg_total_relation_size('schema.tabela')) AS tamanho_total,
    pg_size_pretty(pg_indexes_size('schema.tabela')) AS tamanho_indices;
```

### Top 20 maiores tabelas do banco

```sql
SELECT 
    n.nspname AS schema,
    c.relname AS tabela,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho_total,
    pg_size_pretty(pg_relation_size(c.oid)) AS tamanho_dados,
    pg_size_pretty(pg_indexes_size(c.oid)) AS tamanho_indices,
    c.reltuples::BIGINT AS linhas_estimadas
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 20;
```

### Tamanho por schema

```sql
SELECT 
    n.nspname AS schema,
    pg_size_pretty(SUM(pg_total_relation_size(c.oid))) AS tamanho_total,
    COUNT(*) AS num_tabelas
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
GROUP BY n.nspname
ORDER BY SUM(pg_total_relation_size(c.oid)) DESC;
```

### Tamanho do banco de dados

```sql
SELECT 
    datname AS banco,
    pg_size_pretty(pg_database_size(datname)) AS tamanho
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY pg_database_size(datname) DESC;
```

---

## 4. Compressão e Storage

### Verificar tipo de storage das tabelas

```sql
SELECT 
    n.nspname AS schema,
    c.relname AS tabela,
    CASE 
        WHEN c.relam = 0 THEN 'Heap'
        ELSE am.amname
    END AS storage_type,
    array_to_string(c.reloptions, ', ') AS options
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_am am ON c.relam = am.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY n.nspname, c.relname;
```

### Tabelas AO/AOCO com detalhes de compressão

```sql
SELECT 
    n.nspname AS schema,
    c.relname AS tabela,
    CASE 
        WHEN c.reloptions::text LIKE '%orientation=column%' THEN 'AOCO (Column)'
        WHEN c.reloptions::text LIKE '%appendoptimized=true%' THEN 'AO (Row)'
        ELSE 'Heap'
    END AS tipo,
    CASE 
        WHEN c.reloptions::text LIKE '%compresstype=zstd%' THEN 'ZSTD'
        WHEN c.reloptions::text LIKE '%compresstype=zlib%' THEN 'ZLIB'
        WHEN c.reloptions::text LIKE '%compresstype=lz4%' THEN 'LZ4'
        WHEN c.reloptions::text LIKE '%compresstype=quicklz%' THEN 'QuickLZ'
        WHEN c.reloptions::text LIKE '%compresstype=rle_type%' THEN 'RLE'
        ELSE 'Nenhuma'
    END AS compressao,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY pg_total_relation_size(c.oid) DESC;
```

### Análise de bloat em tabelas AO

```sql
SELECT * FROM gp_toolkit.gp_bloat_diag
ORDER BY bdirelpages DESC
LIMIT 20;
```

---

## 5. Distribuição de Dados

### Verificar chave de distribuição das tabelas

```sql
SELECT 
    n.nspname AS schema,
    c.relname AS tabela,
    CASE d.policytype
        WHEN 'p' THEN 'HASH'
        WHEN 'r' THEN 'REPLICATED'
        ELSE 'RANDOM'
    END AS tipo_distribuicao,
    CASE 
        WHEN d.policytype = 'p' THEN 
            (SELECT string_agg(a.attname, ', ')
             FROM pg_attribute a
             WHERE a.attrelid = c.oid
               AND a.attnum = ANY(d.distkey))
        ELSE NULL
    END AS colunas_distribuicao
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN gp_distribution_policy d ON d.localoid = c.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY n.nspname, c.relname;
```

### Verificar distribuição de uma tabela específica

```sql
SELECT 
    gp_segment_id,
    COUNT(*) AS linhas,
    pg_size_pretty(SUM(pg_column_size(t.*))::BIGINT) AS tamanho_estimado
FROM nome_tabela t
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

---

## 6. Monitoramento de Sessões

### Sessões ativas

```sql
SELECT 
    pid,
    usename AS usuario,
    datname AS banco,
    client_addr AS ip_cliente,
    state AS estado,
    query_start,
    NOW() - query_start AS duracao,
    LEFT(query, 100) AS query_resumida
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

> **Nota:** Se remover a linha `AND pid != pg_backend_pid()`, verá todas as sessões ativas, incluindo a atual. Se manter o filtro, verá apenas as **outras** sessões.

### Queries em execução há mais de X minutos

```sql
SELECT 
    pid,
    usename AS usuario,
    datname AS banco,
    state AS estado,
    NOW() - query_start AS duracao,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND NOW() - query_start > INTERVAL '5 minutes'
  AND pid != pg_backend_pid()
ORDER BY query_start;
```

### Cancelar uma query específica

```sql
-- Cancelar query (graceful)
SELECT pg_cancel_backend(pid);

-- Terminar sessão (forçado)
SELECT pg_terminate_backend(pid);
```

### Sessões por usuário

```sql
SELECT 
    usename AS usuario,
    COUNT(*) AS total_sessoes,
    COUNT(*) FILTER (WHERE state = 'active') AS ativas,
    COUNT(*) FILTER (WHERE state = 'idle') AS ociosas,
    COUNT(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY usename
ORDER BY total_sessoes DESC;
```

---

## 7. Manutenção e VACUUM

### Status de dead tuples por tabela

```sql
SELECT 
    schemaname,
    relname AS tabela,
    n_live_tup AS linhas_vivas,
    n_dead_tup AS linhas_mortas,
    CASE 
        WHEN n_live_tup > 0 
        THEN ROUND(n_dead_tup::NUMERIC / n_live_tup * 100, 2)
        ELSE 0
    END AS pct_mortas,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    vacuum_count,
    analyze_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Tabelas que precisam de VACUUM

```sql
SELECT 
    schemaname || '.' || relname AS tabela,
    n_dead_tup AS dead_tuples,
    pg_size_pretty(pg_total_relation_size(relid)) AS tamanho,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### Tabelas que precisam de ANALYZE

```sql
SELECT 
    schemaname || '.' || relname AS tabela,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze AS modificacoes_desde_analyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 10000
   OR last_analyze IS NULL
ORDER BY n_mod_since_analyze DESC NULLS FIRST;
```

### Executar VACUUM ANALYZE em todas as tabelas de um schema

```sql
-- Gerar comandos (executar o output)
SELECT 'VACUUM ANALYZE ' || schemaname || '.' || relname || ';'
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;
```

---

## 8. Locks e Bloqueios

### Locks ativos no momento

```sql
SELECT 
    l.pid,
    l.locktype,
    l.mode,
    l.granted,
    c.relname AS tabela,
    a.usename AS usuario,
    a.query_start,
    NOW() - a.query_start AS duracao,
    LEFT(a.query, 80) AS query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
LEFT JOIN pg_class c ON l.relation = c.oid
WHERE l.pid != pg_backend_pid()
  AND NOT l.granted
ORDER BY a.query_start;
```

### Sessões bloqueadas e bloqueadores

```sql
SELECT 
    blocked.pid AS pid_bloqueado,
    blocked.usename AS usuario_bloqueado,
    LEFT(blocked.query, 50) AS query_bloqueada,
    blocking.pid AS pid_bloqueador,
    blocking.usename AS usuario_bloqueador,
    LEFT(blocking.query, 50) AS query_bloqueadora
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.relation = blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;
```

---

## 9. Performance e Estatísticas

### Estatísticas de I/O por tabela

```sql
SELECT 
    schemaname,
    relname AS tabela,
    heap_blks_read AS blocos_lidos_disco,
    heap_blks_hit AS blocos_cache,
    CASE 
        WHEN heap_blks_read + heap_blks_hit > 0 
        THEN ROUND(heap_blks_hit::NUMERIC / (heap_blks_read + heap_blks_hit) * 100, 2)
        ELSE 0
    END AS cache_hit_ratio,
    idx_blks_read AS idx_disco,
    idx_blks_hit AS idx_cache
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;
```

### Índices mais utilizados

```sql
SELECT 
    schemaname,
    relname AS tabela,
    indexrelname AS indice,
    idx_scan AS scans,
    idx_tup_read AS tuplas_lidas,
    idx_tup_fetch AS tuplas_retornadas,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;
```

### Índices não utilizados (candidatos a remoção)

```sql
SELECT 
    schemaname || '.' || relname AS tabela,
    indexrelname AS indice,
    idx_scan AS scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Estatísticas de sequências de tabelas

```sql
SELECT 
    schemaname,
    relname AS tabela,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes
FROM pg_stat_user_tables
ORDER BY seq_scan DESC
LIMIT 20;
```

---

## 10. Informações do Cluster

### Versão do Greenplum

```sql
SELECT version();
```

### Informações dos segmentos

```sql
SELECT 
    content AS segment_id,
    role,
    preferred_role,
    mode,
    status,
    hostname,
    port,
    datadir
FROM gp_segment_configuration
ORDER BY content, role;
```

### Segmentos primários ativos

```sql
SELECT 
    content AS segment_id,
    hostname,
    port,
    datadir
FROM gp_segment_configuration
WHERE role = 'p' AND status = 'u'
ORDER BY content;
```

### Verificar se há segmentos down

```sql
SELECT 
    content AS segment_id,
    hostname,
    port,
    status,
    mode
FROM gp_segment_configuration
WHERE status = 'd' OR mode != 's';
```

### Configurações do cluster

```sql
SELECT name, setting, unit, category, short_desc
FROM pg_settings
WHERE name IN (
    'max_connections',
    'shared_buffers', 
    'work_mem',
    'maintenance_work_mem',
    'effective_cache_size',
    'gp_vmem_protect_limit',
    'gp_segment_configuration',
    'gp_max_slices'
)
ORDER BY name;
```

---

## 11. Catálogo de Tabelas

### Listar todas as tabelas com detalhes

```sql
SELECT 
    n.nspname AS schema,
    c.relname AS tabela,
    CASE c.relkind
        WHEN 'r' THEN 'Tabela'
        WHEN 'p' THEN 'Tabela Particionada'
        WHEN 'v' THEN 'View'
        WHEN 'm' THEN 'Materialized View'
        WHEN 'i' THEN 'Índice'
        WHEN 'S' THEN 'Sequence'
    END AS tipo,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho,
    c.reltuples::BIGINT AS linhas_estimadas,
    obj_description(c.oid) AS comentario
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'p', 'v', 'm')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY n.nspname, c.relname;
```

### Colunas de uma tabela

```sql
SELECT 
    a.attname AS coluna,
    format_type(a.atttypid, a.atttypmod) AS tipo,
    a.attnotnull AS not_null,
    COALESCE(pg_get_expr(d.adbin, d.adrelid), '') AS default_value,
    col_description(a.attrelid, a.attnum) AS comentario
FROM pg_attribute a
LEFT JOIN pg_attrdef d ON a.attrelid = d.adrelid AND a.attnum = d.adnum
WHERE a.attrelid = 'schema.tabela'::regclass
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

### Índices de uma tabela

```sql
SELECT 
    i.relname AS indice,
    am.amname AS tipo,
    pg_get_indexdef(i.oid) AS definicao,
    pg_size_pretty(pg_relation_size(i.oid)) AS tamanho
FROM pg_index x
JOIN pg_class i ON i.oid = x.indexrelid
JOIN pg_am am ON i.relam = am.oid
WHERE x.indrelid = 'schema.tabela'::regclass;
```

### Constraints de uma tabela

```sql
SELECT 
    conname AS constraint_name,
    CASE contype
        WHEN 'p' THEN 'PRIMARY KEY'
        WHEN 'u' THEN 'UNIQUE'
        WHEN 'c' THEN 'CHECK'
        WHEN 'f' THEN 'FOREIGN KEY'
    END AS tipo,
    pg_get_constraintdef(oid) AS definicao
FROM pg_constraint
WHERE conrelid = 'schema.tabela'::regclass;
```

---

## 12. Troubleshooting

### Verificar espaço em disco por segmento

```sql
SELECT 
    gp_segment_id,
    pg_size_pretty(SUM(pg_total_relation_size(oid))) AS tamanho_total
FROM gp_dist_random('pg_class')
WHERE relkind = 'r'
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

### Queries mais lentas / em execução (pg_stat_statements ou alternativa)

> **Nota:** `pg_stat_statements` pode não estar instalado por padrão. Use a alternativa abaixo:

**Opção 1: Se pg_stat_statements estiver instalado:**

```sql
SELECT 
    LEFT(query, 100) AS query,
    calls,
    ROUND(total_exec_time::NUMERIC / 1000, 2) AS total_sec,
    ROUND(mean_exec_time::NUMERIC, 2) AS media_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Opção 2: Alternativa usando pg_stat_activity (sempre funciona):**

```sql
SELECT 
    pid,
    usename,
    datname,
    state,
    query_start,
    NOW() - query_start AS duracao,
    LEFT(query, 100) AS query_resumida
FROM pg_stat_activity
WHERE query NOT LIKE '%pg_stat_activity%'
  AND state != 'idle'
ORDER BY query_start ASC;
```

### Verificar configuração de resource groups

```sql
SELECT 
    groupname,
    concurrency,
    cpu_max_percent,
    cpu_weight,
    memory_quota
FROM gp_toolkit.gp_resgroup_config;
```

### Verificar tabelas corrompidas ou com problemas

```sql
SELECT 
    nspname AS schema,
    relname AS tabela,
    relpages,
    reltuples::BIGINT
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
  AND c.relpages = 0
  AND c.reltuples > 0
  AND n.nspname NOT IN ('pg_catalog', 'information_schema');
```

---

## Comandos Administrativos Úteis

### Forçar checkpoint

```sql
CHECKPOINT;
```

### Recarregar configurações sem reiniciar

```sql
SELECT pg_reload_conf();
```

### Ver configuração atual de um parâmetro

```sql
SHOW work_mem;
SHOW shared_buffers;
SHOW gp_vmem_protect_limit;
```

### Alterar configuração para sessão atual

```sql
SET work_mem = '256MB';
SET statement_timeout = '30min';
```

### Verificar se extensão está instalada

```sql
SELECT * FROM pg_extension;
```

### Verificar funções disponíveis

```sql
SELECT 
    n.nspname AS schema,
    p.proname AS funcao,
    pg_get_function_arguments(p.oid) AS argumentos
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
WHERE n.nspname = 'public'
ORDER BY p.proname;
```

---

## Documentação Oficial

- **Greenplum 7 Documentation:** https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/landing-index.html
- **System Catalogs:** https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/ref_guide-system_catalogs-catalog_ref.html
- **gp_toolkit Views:** https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/ref_guide-gp_toolkit.html
- **PostgreSQL 12 Catalogs:** https://www.postgresql.org/docs/12/catalogs.html

---

**📅 Última atualização:** Dezembro 2025 (Greenplum 7.x)
