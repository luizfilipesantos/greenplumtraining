# Módulo 5A: Manutenção e VACUUM

**Duração Total:** 60-75 minutos  
**Objetivo:** Dominar técnicas de manutenção de banco de dados, compreender o funcionamento do VACUUM e otimizar operações de limpeza no Greenplum.

---

## Índice
1. [Lab 5A.1: Fundamentos do VACUUM](#lab-5a1-fundamentos-do-vacuum-20-25-min)
2. [Lab 5A.2: VACUUM Avançado e Tuning](#lab-5a2-vacuum-avançado-e-tuning-20-25-min)
3. [Lab 5A.3: Monitoramento e Automação](#lab-5a3-monitoramento-e-automação-20-25-min)

---

## Lab 5A.1: Fundamentos do VACUUM (20-25 min)

### Objetivos
- Entender o modelo MVCC do Greenplum
- Compreender quando e por que usar VACUUM
- Diferenciar VACUUM, VACUUM FULL e ANALYZE
- Identificar problemas de bloat

### Conceitos Abordados
- **MVCC (Multi-Version Concurrency Control):** Controle de versões de linhas
- **Dead Tuples:** Linhas obsoletas após UPDATE/DELETE
- **Bloat:** Espaço desperdiçado por linhas mortas
- **Transaction ID Wraparound:** Proteção contra overflow de XIDs

### MVCC e Dead Tuples

No Greenplum (e PostgreSQL), quando você executa:
- **UPDATE:** Cria nova versão da linha, marca antiga como dead tuple
- **DELETE:** Marca linha como dead tuple
- **INSERT:** Cria nova linha (sem dead tuples)

Dead tuples ocupam espaço até que VACUUM os remova.

---

### Exercício 5A.1.1: Visualizando Dead Tuples

**Objetivo:** Ver o acúmulo de dead tuples

**Cenário:** Tabela com muitos UPDATEs

**Passos:**

1. Crie tabela de teste:
```sql
CREATE TABLE teste_vacuum (
    id INTEGER,
    nome VARCHAR(100),
    valor NUMERIC(10,2),
    data_atualizacao TIMESTAMP
)
DISTRIBUTED BY (id);

-- Insira dados iniciais
INSERT INTO teste_vacuum
SELECT 
    i,
    'Nome_' || i,
    random() * 1000,
    CURRENT_TIMESTAMP
FROM generate_series(1, 100000) i;

ANALYZE teste_vacuum;
```

2. Verifique estatísticas iniciais:
```sql
SELECT 
    schemaname,
    tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE tablename = 'teste_vacuum';
```

3. Execute UPDATEs massivos:
```sql
-- Atualize metade da tabela
UPDATE teste_vacuum
SET valor = valor * 1.1,
    data_atualizacao = CURRENT_TIMESTAMP
WHERE id <= 50000;

-- Atualize novamente
UPDATE teste_vacuum
SET valor = valor * 0.9,
    data_atualizacao = CURRENT_TIMESTAMP
WHERE id <= 25000;
```

4. Verifique acúmulo de dead tuples:
```sql
SELECT 
    schemaname,
    tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) * 100 as percentual_mortas,
    pg_size_pretty(pg_relation_size(relid)) as tamanho_tabela
FROM pg_stat_user_tables
WHERE tablename = 'teste_vacuum';
```

**Análise:**
- `n_dead_tup` deve estar alto (75.000+)
- Percentual de linhas mortas > 50%
- Tabela cresceu além do necessário

---

### Exercício 5A.1.2: VACUUM Básico

**Objetivo:** Executar VACUUM e observar resultados

**Passos:**

1. Execute VACUUM:
```sql
\timing on

VACUUM teste_vacuum;

\timing off
```

2. Verifique resultados:
```sql
SELECT 
    schemaname,
    tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    last_vacuum,
    pg_size_pretty(pg_relation_size(relid)) as tamanho_tabela
FROM pg_stat_user_tables
WHERE tablename = 'teste_vacuum';
```

**Observações:**
- `n_dead_tup` deve estar próximo de 0
- `last_vacuum` atualizado
- ⚠️ **Tamanho da tabela NÃO diminuiu!**

3. Por que o tamanho não diminuiu?
```sql
-- VACUUM recupera espaço mas NÃO o devolve ao SO
-- Espaço fica disponível para REUSo pela mesma tabela
```

---

### Exercício 5A.1.3: VACUUM FULL

**Objetivo:** Recuperar espaço fisicamente

**Passos:**

1. Verifique tamanho antes:
```sql
SELECT pg_size_pretty(pg_total_relation_size('teste_vacuum')) as tamanho_total;
```

2. Execute VACUUM FULL:
```sql
\timing on

VACUUM FULL teste_vacuum;

\timing off
```

3. Verifique tamanho depois:
```sql
SELECT pg_size_pretty(pg_total_relation_size('teste_vacuum')) as tamanho_total;
```

**VACUUM vs VACUUM FULL:**

| Aspecto | VACUUM | VACUUM FULL |
|---------|--------|-------------|
| **Bloqueio** | Minimal (permite leituras/escritas) | Exclusivo (bloqueia tudo) |
| **Velocidade** | Rápido | Lento |
| **Espaço recuperado** | Para reuso interno | Devolvido ao SO |
| **Uso** | Manutenção regular | Excepcional, emergencial |
| **Recomendação** | ✅ Use frequentemente | ❌ Evite em produção |

⚠️ **IMPORTANTE:** VACUUM FULL reescreve tabela inteira e requer espaço em disco igual ao tamanho da tabela!

---

### Exercício 5A.1.4: VACUUM ANALYZE

**Objetivo:** Combinar limpeza com atualização de estatísticas

**Passos:**

1. Crie tabela com dados desatualizados:
```sql
CREATE TABLE produtos_teste (
    produto_id INTEGER,
    categoria VARCHAR(50),
    preco NUMERIC(10,2),
    estoque INTEGER
)
DISTRIBUTED BY (produto_id);

INSERT INTO produtos_teste
SELECT 
    i,
    CASE (i % 5)
        WHEN 0 THEN 'Eletrônicos'
        WHEN 1 THEN 'Livros'
        WHEN 2 THEN 'Roupas'
        WHEN 3 THEN 'Alimentos'
        ELSE 'Diversos'
    END,
    random() * 1000,
    (random() * 100)::INTEGER
FROM generate_series(1, 50000) i;

ANALYZE produtos_teste;
```

2. Faça mudanças significativas:
```sql
-- Delete 30%
DELETE FROM produtos_teste WHERE produto_id % 3 = 0;

-- Update 50% dos remanescentes
UPDATE produtos_teste
SET preco = preco * 1.5,
    estoque = estoque + 10
WHERE produto_id % 2 = 0;
```

3. Execute VACUUM ANALYZE:
```sql
\timing on

VACUUM ANALYZE produtos_teste;

\timing off
```

4. Verifique estatísticas:
```sql
-- Estatísticas atualizadas
SELECT 
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE tablename = 'produtos_teste';

-- Estimativas do planner
EXPLAIN
SELECT categoria, COUNT(*), AVG(preco)
FROM produtos_teste
GROUP BY categoria;
```

**Quando usar VACUUM ANALYZE:**
- ✅ Após cargas massivas
- ✅ Após muitos UPDATEs/DELETEs
- ✅ Antes de queries importantes
- ✅ Como rotina de manutenção regular

---

### Exercício 5A.1.5: Bloat Detection

**Objetivo:** Detectar e quantificar bloat

**Passos:**

1. Query para detectar bloat:
```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as tamanho_total,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as percentual_bloat,
    CASE 
        WHEN n_dead_tup > n_live_tup * 0.2 THEN 'CRÍTICO'
        WHEN n_dead_tup > n_live_tup * 0.1 THEN 'ALERTA'
        ELSE 'OK'
    END as status
FROM pg_stat_user_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY n_dead_tup DESC;
```

2. Bloat detalhado com pg_class:
```sql
SELECT 
    n.nspname as schema,
    c.relname as tabela,
    c.reltuples as linhas_estimadas,
    pg_size_pretty(pg_total_relation_size(c.oid)) as tamanho_total,
    pg_size_pretty(pg_relation_size(c.oid)) as tamanho_tabela,
    pg_size_pretty(pg_total_relation_size(c.oid) - pg_relation_size(c.oid)) as tamanho_indices,
    ROUND(100.0 * s.n_dead_tup / NULLIF(s.n_live_tup + s.n_dead_tup, 0), 2) as pct_bloat
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_stat_user_tables s ON s.relid = c.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
  AND s.n_dead_tup > 1000
ORDER BY s.n_dead_tup DESC
LIMIT 20;
```

3. Function para monitoramento:
```sql
CREATE OR REPLACE FUNCTION check_bloat()
RETURNS TABLE(
    schema_name TEXT,
    table_name TEXT,
    live_tuples BIGINT,
    dead_tuples BIGINT,
    bloat_pct NUMERIC,
    recommendation TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        schemaname::TEXT,
        tablename::TEXT,
        n_live_tup,
        n_dead_tup,
        ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2),
        CASE 
            WHEN n_dead_tup > n_live_tup * 0.5 THEN 'VACUUM FULL necessário'
            WHEN n_dead_tup > n_live_tup * 0.2 THEN 'VACUUM recomendado'
            WHEN n_dead_tup > n_live_tup * 0.1 THEN 'Monitorar'
            ELSE 'OK'
        END
    FROM pg_stat_user_tables
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
      AND n_dead_tup > 100
    ORDER BY n_dead_tup DESC;
END;
$$ LANGUAGE plpgsql;

-- Execute
SELECT * FROM check_bloat();
```

---

## Lab 5A.2: VACUUM Avançado e Tuning (20-25 min)

### Objetivos
- Usar opções avançadas do VACUUM
- Otimizar performance do VACUUM
- Entender VACUUM em tabelas AO/AOCO
- Implementar estratégias de vacuum seletivo

### Conceitos Abordados
- **VACUUM VERBOSE:** Output detalhado
- **VACUUM (PARALLEL):** Paralelização (GP 7+)
- **Configurações:** vacuum_cost_delay, autovacuum
- **AO Tables:** VACUUM vs Compaction

---

### Exercício 5A.2.1: VACUUM VERBOSE

**Objetivo:** Analisar detalhes da operação

**Passos:**

1. Execute VACUUM VERBOSE:
```sql
VACUUM VERBOSE teste_vacuum;
```

**Output esperado:**
```
INFO:  vacuuming "public.teste_vacuum"
INFO:  index "idx_teste" now contains 100000 row versions in 300 pages
INFO:  "teste_vacuum": removed 75000 dead row versions in 1500 pages
INFO:  "teste_vacuum": found 75000 removable, 100000 nonremovable row versions
INFO:  "teste_vacuum": truncated 2000 to 1200 pages
```

2. Interprete o output:
```sql
-- removed X dead row versions: Linhas mortas removidas
-- found X removable: Total de dead tuples encontrados
-- truncated X to Y pages: Páginas devolvidas (só no fim da tabela)
```

---

### Exercício 5A.2.2: VACUUM com Opções

**Objetivo:** Usar variações do comando

**Passos:**

1. VACUUM com opções separadas:
```sql
-- Apenas VACUUM (sem ANALYZE)
VACUUM teste_vacuum;

-- VACUUM + ANALYZE
VACUUM ANALYZE teste_vacuum;

-- VACUUM FULL + ANALYZE
VACUUM FULL ANALYZE teste_vacuum;

-- VACUUM FREEZE (previne wraparound)
VACUUM FREEZE teste_vacuum;
```

2. VACUUM em tabelas específicas vs banco todo:
```sql
-- Apenas uma tabela
VACUUM teste_vacuum;

-- Todas as tabelas do schema
VACUUM;

-- CUIDADO: VACUUM sem argumentos processa TODO o banco!
```

3. VACUUM seletivo por condição:
```sql
-- VACUUM não aceita WHERE, mas você pode usar partições
-- Vacuume apenas partições específicas

-- Liste partições
SELECT 
    schemaname,
    tablename,
    partitiontablename
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
ORDER BY partitionrank DESC
LIMIT 5;

-- Vacuum apenas partições recentes
VACUUM vendas_particionada_1_prt_12;  -- Ajuste nome
```

---

### Exercício 5A.2.3: VACUUM em Tabelas AO/AOCO

**Objetivo:** Entender diferenças em tabelas Append-Optimized

**Passos:**

1. Crie tabela AO:
```sql
CREATE TABLE teste_ao (
    id INTEGER,
    nome VARCHAR(100),
    valor NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=row)
DISTRIBUTED BY (id);

INSERT INTO teste_ao
SELECT i, 'Nome_' || i, random() * 1000
FROM generate_series(1, 100000) i;

ANALYZE teste_ao;
```

2. Execute UPDATEs (não recomendado em AO):
```sql
-- AO tables não são otimizadas para UPDATE/DELETE
UPDATE teste_ao SET valor = valor * 1.1 WHERE id <= 50000;
DELETE FROM teste_ao WHERE id <= 10000;
```

3. Verifique bloat em AO:
```sql
SELECT 
    gp_segment_id,
    COUNT(*) as visible_rows
FROM teste_ao
GROUP BY gp_segment_id;

-- Estatísticas AO específicas
SELECT * FROM gp_toolkit.__gp_aovisimap_name('teste_ao');
```

4. VACUUM em tabela AO:
```sql
-- VACUUM em AO marca blocos como reutilizáveis
VACUUM teste_ao;

-- Para realmente compactar, use:
VACUUM FULL teste_ao;
```

5. Alternativa: Recreate table:
```sql
-- Para AO/AOCO com muito bloat, melhor estratégia:
CREATE TABLE teste_ao_new (LIKE teste_ao INCLUDING ALL)
WITH (appendoptimized=true, orientation=row);

INSERT INTO teste_ao_new SELECT * FROM teste_ao;

-- Depois de validar:
DROP TABLE teste_ao;
ALTER TABLE teste_ao_new RENAME TO teste_ao;
```

**⚠️ Importante para AO/AOCO:**
- UPDATE/DELETE são custosos (criam bloat)
- VACUUM FULL é necessário para compactação real
- Considere INSERT-only workflows
- Use particionamento para DROP de dados antigos

---

### Exercício 5A.2.4: Configurações de VACUUM

**Objetivo:** Otimizar performance do VACUUM

**Passos:**

1. Visualize configurações atuais:
```sql
SELECT 
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name LIKE '%vacuum%'
   OR name LIKE '%autovacuum%'
ORDER BY name;
```

2. Configurações importantes:

```sql
-- Memória para VACUUM
SHOW maintenance_work_mem;  -- Padrão: 64MB
SET maintenance_work_mem = '1GB';  -- Para sessão

-- Custo de I/O (throttling)
SHOW vacuum_cost_delay;  -- 0 = sem delay
SHOW vacuum_cost_limit;  -- 200 = limite antes de delay

-- Autovacuum (GP7+)
SHOW autovacuum;  -- on/off
SHOW autovacuum_naptime;  -- Intervalo entre execuções
```

3. Tune VACUUM para performance:
```sql
-- Para vacuum rápido (fora de horário de pico)
SET maintenance_work_mem = '2GB';
SET vacuum_cost_delay = 0;  -- Sem throttling

VACUUM VERBOSE teste_vacuum;

-- Restaure defaults
RESET maintenance_work_mem;
RESET vacuum_cost_delay;
```

4. VACUUM com menor impacto (durante produção):
```sql
-- Throttle VACUUM para não impactar queries
SET maintenance_work_mem = '256MB';
SET vacuum_cost_delay = 10;  -- 10ms de delay
SET vacuum_cost_limit = 200;

VACUUM teste_vacuum;
```

---

### Exercício 5A.2.5: Transaction ID Wraparound Prevention

**Objetivo:** Prevenir problemas de wraparound

**Passos:**

1. Verifique idade das transações:
```sql
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    2147483647 - age(datfrozenxid) as xids_remaining
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

2. Tabelas com maior idade:
```sql
SELECT 
    n.nspname,
    c.relname,
    age(c.relfrozenxid) as xid_age,
    pg_size_pretty(pg_total_relation_size(c.oid)) as size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;
```

3. VACUUM FREEZE para prevenir wraparound:
```sql
-- Force freeze de XIDs antigos
VACUUM FREEZE teste_vacuum;
```

4. Configure thresholds:
```sql
-- Visualize thresholds
SHOW autovacuum_freeze_max_age;  -- Padrão: 200M transações

-- Tabelas próximas ao limite
SELECT 
    schemaname,
    tablename,
    age(relfrozenxid) as xid_age,
    (SELECT setting FROM pg_settings WHERE name = 'autovacuum_freeze_max_age')::BIGINT as threshold,
    age(relfrozenxid)::FLOAT / (SELECT setting FROM pg_settings WHERE name = 'autovacuum_freeze_max_age')::FLOAT * 100 as pct_threshold
FROM pg_stat_user_tables t
JOIN pg_class c ON c.oid = t.relid
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```

**Thresholds críticos:**
- < 100M XIDs: OK
- 100M - 150M: Monitore
- 150M - 200M: VACUUM urgente
- > 200M: CRÍTICO (autovacuum agressivo)

---

## Lab 5A.3: Monitoramento e Automação (20-25 min)

### Objetivos
- Monitorar execuções de VACUUM
- Automatizar rotinas de manutenção
- Criar alertas para bloat
- Implementar vacuum seletivo inteligente

### Conceitos Abordados
- **pg_stat_progress_vacuum:** Progresso em tempo real
- **Logging:** Registrar execuções
- **Cron:** Agendamento
- **Alertas:** Notificações automáticas

---

### Exercício 5A.3.1: Monitoramento em Tempo Real

**Objetivo:** Acompanhar progresso do VACUUM

**Passos:**

1. Em uma sessão, inicie VACUUM:
```sql
VACUUM VERBOSE teste_vacuum;
```

2. Em outra sessão, monitore progresso:
```sql
-- GP7+ tem pg_stat_progress_vacuum
SELECT 
    pid,
    datname,
    relid::regclass as tabela,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    ROUND(heap_blks_scanned::NUMERIC / NULLIF(heap_blks_total, 0) * 100, 2) as pct_completo
FROM pg_stat_progress_vacuum;
```

3. Histórico de vacuums:
```sql
SELECT 
    schemaname,
    tablename,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    n_dead_tup
FROM pg_stat_user_tables
WHERE last_vacuum IS NOT NULL
   OR last_autovacuum IS NOT NULL
ORDER BY GREATEST(last_vacuum, last_autovacuum) DESC NULLS LAST;
```

---

### Exercício 5A.3.2: Logging de VACUUM

**Objetivo:** Registrar execuções para auditoria

**Passos:**

1. Crie tabela de log:
```sql
CREATE TABLE vacuum_log (
    log_id SERIAL PRIMARY KEY,
    schema_name VARCHAR(100),
    table_name VARCHAR(100),
    vacuum_type VARCHAR(20),
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    duration_seconds NUMERIC(10,2),
    dead_tuples_before BIGINT,
    dead_tuples_after BIGINT,
    size_before TEXT,
    size_after TEXT,
    success BOOLEAN,
    error_message TEXT
)
DISTRIBUTED RANDOMLY;
```

2. Function para VACUUM com logging:
```sql
CREATE OR REPLACE FUNCTION vacuum_with_log(
    p_schema VARCHAR,
    p_table VARCHAR,
    p_full BOOLEAN DEFAULT FALSE
) RETURNS VOID AS $$
DECLARE
    v_start TIMESTAMP;
    v_end TIMESTAMP;
    v_dead_before BIGINT;
    v_dead_after BIGINT;
    v_size_before TEXT;
    v_size_after TEXT;
    v_vacuum_cmd TEXT;
BEGIN
    v_start := CLOCK_TIMESTAMP();
    
    -- Captura estado inicial
    SELECT n_dead_tup, pg_size_pretty(pg_total_relation_size(relid))
    INTO v_dead_before, v_size_before
    FROM pg_stat_user_tables
    WHERE schemaname = p_schema AND tablename = p_table;
    
    -- Executa VACUUM
    v_vacuum_cmd := 'VACUUM';
    IF p_full THEN
        v_vacuum_cmd := v_vacuum_cmd || ' FULL';
    END IF;
    v_vacuum_cmd := v_vacuum_cmd || ' ' || quote_ident(p_schema) || '.' || quote_ident(p_table);
    
    EXECUTE v_vacuum_cmd;
    
    v_end := CLOCK_TIMESTAMP();
    
    -- Captura estado final
    SELECT n_dead_tup, pg_size_pretty(pg_total_relation_size(relid))
    INTO v_dead_after, v_size_after
    FROM pg_stat_user_tables
    WHERE schemaname = p_schema AND tablename = p_table;
    
    -- Log sucesso
    INSERT INTO vacuum_log (
        schema_name, table_name, vacuum_type, start_time, end_time,
        duration_seconds, dead_tuples_before, dead_tuples_after,
        size_before, size_after, success
    )
    VALUES (
        p_schema, p_table,
        CASE WHEN p_full THEN 'FULL' ELSE 'REGULAR' END,
        v_start, v_end,
        EXTRACT(EPOCH FROM (v_end - v_start)),
        v_dead_before, v_dead_after,
        v_size_before, v_size_after,
        TRUE
    );
    
    RAISE NOTICE 'VACUUM % completo em % segundos', p_table, EXTRACT(EPOCH FROM (v_end - v_start));
    
EXCEPTION WHEN OTHERS THEN
    -- Log erro
    INSERT INTO vacuum_log (
        schema_name, table_name, vacuum_type, start_time, end_time,
        success, error_message
    )
    VALUES (
        p_schema, p_table,
        CASE WHEN p_full THEN 'FULL' ELSE 'REGULAR' END,
        v_start, CLOCK_TIMESTAMP(),
        FALSE, SQLERRM
    );
    
    RAISE;
END;
$$ LANGUAGE plpgsql;
```

3. Use a function:
```sql
SELECT vacuum_with_log('public', 'teste_vacuum', FALSE);

-- Verifique log
SELECT * FROM vacuum_log ORDER BY log_id DESC LIMIT 5;
```

---

### Exercício 5A.3.3: Automação com Script

**Objetivo:** Script de manutenção automatizada

**Passos:**

1. Crie script de vacuum inteligente:
```bash
cat > /home/gpadmin/scripts/vacuum_maintenance.sh << 'EOF'
#!/bin/bash

# Configurações
PGDATABASE="seu_database"
PGUSER="gpadmin"
LOGDIR="/home/gpadmin/logs/vacuum"
DATE=$(date +%Y%m%d_%H%M%S)
LOGFILE="$LOGDIR/vacuum_$DATE.log"

mkdir -p $LOGDIR

echo "=== Iniciando manutenção VACUUM: $(date) ===" >> $LOGFILE

# Query para identificar tabelas que precisam vacuum
psql -d $PGDATABASE -U $PGUSER -t -A -F"," << 'SQL' | while IFS=, read schema table dead_pct
SELECT 
    schemaname,
    tablename,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2)
FROM pg_stat_user_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
  AND n_dead_tup > 1000
  AND n_dead_tup > n_live_tup * 0.1
ORDER BY n_dead_tup DESC;
SQL
do
    echo "Processando: $schema.$table (${dead_pct}% dead tuples)" >> $LOGFILE
    
    # VACUUM FULL se muito bloat
    if (( $(echo "$dead_pct > 50" | bc -l) )); then
        echo "  -> VACUUM FULL necessário" >> $LOGFILE
        psql -d $PGDATABASE -U $PGUSER -c "VACUUM FULL ANALYZE $schema.$table;" >> $LOGFILE 2>&1
    else
        echo "  -> VACUUM regular" >> $LOGFILE
        psql -d $PGDATABASE -U $PGUSER -c "VACUUM ANALYZE $schema.$table;" >> $LOGFILE 2>&1
    fi
    
    if [ $? -eq 0 ]; then
        echo "  -> Sucesso" >> $LOGFILE
    else
        echo "  -> ERRO!" >> $LOGFILE
    fi
done

echo "=== Manutenção concluída: $(date) ===" >> $LOGFILE
EOF

chmod +x /home/gpadmin/scripts/vacuum_maintenance.sh
```

2. Agende com cron:
```bash
# Edite crontab
crontab -e

# Execute diariamente às 2h
0 2 * * * /home/gpadmin/scripts/vacuum_maintenance.sh

# Execute aos domingos às 3h (maintenance window)
0 3 * * 0 /home/gpadmin/scripts/vacuum_maintenance.sh
```

---

### Exercício 5A.3.4: Alertas de Bloat

**Objetivo:** Notificações automáticas para problemas

**Passos:**

1. Function de verificação:
```sql
CREATE OR REPLACE FUNCTION check_critical_bloat()
RETURNS TABLE(
    alert_level VARCHAR,
    schema_name VARCHAR,
    table_name VARCHAR,
    dead_tuples BIGINT,
    bloat_pct NUMERIC,
    recommended_action VARCHAR
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        CASE 
            WHEN n_dead_tup > n_live_tup * 0.5 THEN 'CRÍTICO'
            WHEN n_dead_tup > n_live_tup * 0.2 THEN 'ALERTA'
            ELSE 'AVISO'
        END::VARCHAR,
        schemaname::VARCHAR,
        tablename::VARCHAR,
        n_dead_tup,
        ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2),
        CASE 
            WHEN n_dead_tup > n_live_tup * 0.5 THEN 'VACUUM FULL urgente'
            WHEN n_dead_tup > n_live_tup * 0.2 THEN 'VACUUM recomendado'
            ELSE 'Monitorar'
        END::VARCHAR
    FROM pg_stat_user_tables
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
      AND n_dead_tup > 1000
      AND n_dead_tup > n_live_tup * 0.1
    ORDER BY n_dead_tup DESC;
END;
$$ LANGUAGE plpgsql;
```

2. Script de alerta:
```bash
cat > /home/gpadmin/scripts/bloat_alert.sh << 'EOF'
#!/bin/bash

PGDATABASE="seu_database"
PGUSER="gpadmin"
EMAIL="admin@empresa.com"

# Verifica bloat crítico
CRITICAL=$(psql -d $PGDATABASE -U $PGUSER -t -A -c "
SELECT COUNT(*) FROM check_critical_bloat() WHERE alert_level = 'CRÍTICO';
")

if [ "$CRITICAL" -gt 0 ]; then
    # Gera relatório
    REPORT=$(psql -d $PGDATABASE -U $PGUSER -c "
    SELECT * FROM check_critical_bloat() WHERE alert_level IN ('CRÍTICO', 'ALERTA');
    ")
    
    # Envia email
    echo "$REPORT" | mail -s "ALERTA: Bloat crítico detectado no Greenplum" $EMAIL
fi
EOF

chmod +x /home/gpadmin/scripts/bloat_alert.sh

# Agende verificação horária
# 0 * * * * /home/gpadmin/scripts/bloat_alert.sh
```

---

### Exercício 5A.3.5: Dashboard de Manutenção

**Objetivo:** View consolidada para monitoramento

**Passos:**

1. Crie view de dashboard:
```sql
CREATE OR REPLACE VIEW vw_vacuum_dashboard AS
SELECT 
    t.schemaname,
    t.tablename,
    pg_size_pretty(pg_total_relation_size(t.relid)) as tamanho_total,
    t.n_live_tup as linhas_vivas,
    t.n_dead_tup as linhas_mortas,
    ROUND(t.n_dead_tup * 100.0 / NULLIF(t.n_live_tup + t.n_dead_tup, 0), 2) as pct_bloat,
    t.last_vacuum,
    t.last_autovacuum,
    t.vacuum_count,
    age(c.relfrozenxid) as xid_age,
    CASE 
        WHEN t.n_dead_tup > t.n_live_tup * 0.5 THEN 'CRÍTICO - VACUUM FULL'
        WHEN t.n_dead_tup > t.n_live_tup * 0.2 THEN 'ALERTA - VACUUM'
        WHEN t.n_dead_tup > t.n_live_tup * 0.1 THEN 'MONITORAR'
        WHEN age(c.relfrozenxid) > 150000000 THEN 'WRAPAROUND RISK'
        ELSE 'OK'
    END as status,
    CASE 
        WHEN t.n_dead_tup > t.n_live_tup * 0.5 THEN 1
        WHEN t.n_dead_tup > t.n_live_tup * 0.2 THEN 2
        WHEN t.n_dead_tup > t.n_live_tup * 0.1 THEN 3
        ELSE 4
    END as prioridade
FROM pg_stat_user_tables t
JOIN pg_class c ON c.oid = t.relid
WHERE t.schemaname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY prioridade, t.n_dead_tup DESC;
```

2. Consulte dashboard:
```sql
-- Visão geral
SELECT 
    status,
    COUNT(*) as num_tabelas,
    SUM(linhas_mortas) as total_dead_tuples
FROM vw_vacuum_dashboard
GROUP BY status
ORDER BY 
    CASE status
        WHEN 'CRÍTICO - VACUUM FULL' THEN 1
        WHEN 'ALERTA - VACUUM' THEN 2
        WHEN 'MONITORAR' THEN 3
        WHEN 'WRAPAROUND RISK' THEN 4
        ELSE 5
    END;

-- Tabelas críticas
SELECT * FROM vw_vacuum_dashboard
WHERE status LIKE 'CRÍTICO%' OR status LIKE 'ALERTA%'
ORDER BY prioridade, pct_bloat DESC;
```

---

## Resumo do Módulo 5A

### Habilidades Adquiridas
✅ Entender MVCC e dead tuples  
✅ Executar VACUUM, VACUUM FULL e VACUUM ANALYZE  
✅ Detectar e quantificar bloat  
✅ Otimizar configurações de VACUUM  
✅ Trabalhar com tabelas AO/AOCO  
✅ Prevenir transaction wraparound  
✅ Monitorar execuções de VACUUM  
✅ Automatizar manutenção  
✅ Criar alertas de bloat  

### Comandos Principais

```sql
-- VACUUM básico
VACUUM tabela;
VACUUM ANALYZE tabela;
VACUUM FULL tabela;
VACUUM FREEZE tabela;
VACUUM VERBOSE tabela;

-- Monitoramento
SELECT * FROM pg_stat_user_tables WHERE tablename = 'tabela';
SELECT * FROM pg_stat_progress_vacuum;
SELECT * FROM check_bloat();

-- Configurações
SET maintenance_work_mem = '1GB';
SET vacuum_cost_delay = 0;
SHOW autovacuum;
```

### Quando Usar Cada VACUUM

| Situação | Comando | Frequência |
|----------|---------|------------|
| Manutenção regular | VACUUM | Diário/Semanal |
| Após cargas/updates | VACUUM ANALYZE | Imediato |
| Bloat > 50% | VACUUM FULL | Excepcional |
| Prevenção wraparound | VACUUM FREEZE | Mensal |
| Monitoramento | VACUUM VERBOSE | Conforme necessário |

### Troubleshooting

| Problema | Causa | Solução |
|----------|-------|---------|
| VACUUM lento | maintenance_work_mem baixo | Aumente para 1-2GB |
| Bloat não reduz | VACUUM regular não libera espaço | Use VACUUM FULL |
| Impacto em produção | VACUUM sem throttling | Configure vacuum_cost_delay |
| Tabela AO bloated | Muitos UPDATEs/DELETEs | Evite DML, use partições |
| Wraparound warning | Tabela não vacuada há muito tempo | VACUUM FREEZE urgente |

### Boas Práticas

**Rotina Regular:**
1. VACUUM ANALYZE semanal em tabelas grandes
2. VACUUM ANALYZE após cargas massivas
3. Monitore bloat diariamente
4. VACUUM FREEZE mensal preventivo

**Emergencial:**
1. VACUUM FULL apenas em maintenance windows
2. Verifique espaço em disco antes de VACUUM FULL
3. Considere REINDEX após VACUUM FULL
4. Teste em dev/staging primeiro

### Próximos Passos
No **Módulo 5B**, você aprenderá sobre detecção de skew (dados e processamento) e análise avançada com EXPLAIN.

---

**Fim do Módulo 5A**
