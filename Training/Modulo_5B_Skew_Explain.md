# Módulo 5B: Detecção de Skew e EXPLAIN Avançado

**Duração Total:** 90-120 minutos  
**Objetivo:** Dominar técnicas de detecção de skew (dados e processamento), análise avançada de planos de execução e otimização de queries no Greenplum.

---

## Índice
1. [Lab 5B.1: Data Skew - Detecção e Correção](#lab-5b1-data-skew---detecção-e-correção-30-35-min)
2. [Lab 5B.2: Processing Skew - Identificação](#lab-5b2-processing-skew---identificação-25-30-min)
3. [Lab 5B.3: EXPLAIN Avançado](#lab-5b3-explain-avançado-35-40-min)

---

## Lab 5B.1: Data Skew - Detecção e Correção (30-35 min)

### Objetivos
- Entender o que é data skew
- Detectar distribuição desbalanceada
- Quantificar severidade do skew
- Corrigir problemas de distribuição

### Conceitos Abordados
- **Data Skew:** Distribuição desigual de dados entre segmentos
- **Distribution Key:** Chave de distribuição inadequada
- **Skew Coefficient:** Métrica de desbalanceamento
- **Redistribution:** Correção de distribuição

### O que é Data Skew?

Data skew ocorre quando dados não são distribuídos uniformemente entre segmentos:
- **Ideal:** Cada segmento tem ~mesma quantidade de dados
- **Skew:** Alguns segmentos têm muito mais dados que outros
- **Impacto:** Queries lentas, uso desigual de recursos

**Causas comuns:**
- Chave de distribuição com baixa cardinalidade
- Valores null ou repetidos na chave
- Distribuição RANDOM com dados desbalanceados
- Dados categóricos concentrados

---

### Exercício 5B.1.1: Criando Tabela com Skew

**Objetivo:** Simular e visualizar data skew

**Passos:**

1. Crie tabela com distribuição ruim:
```sql
CREATE TABLE vendas_skewed (
    venda_id BIGSERIAL,
    regiao VARCHAR(20),
    loja_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (regiao);  -- Má escolha: baixa cardinalidade!

-- Insira dados com concentração em uma região
INSERT INTO vendas_skewed (regiao, loja_id, produto_id, quantidade, valor)
SELECT 
    CASE 
        WHEN i <= 800000 THEN 'Sudeste'  -- 80% dos dados
        WHEN i <= 900000 THEN 'Sul'       -- 10%
        WHEN i <= 950000 THEN 'Nordeste'  -- 5%
        WHEN i <= 980000 THEN 'Norte'     -- 3%
        ELSE 'Centro-Oeste'               -- 2%
    END,
    (random() * 100)::INTEGER + 1,
    (random() * 1000)::INTEGER + 1,
    (random() * 10)::INTEGER + 1,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 1000000) i;

ANALYZE vendas_skewed;
```

2. Visualize distribuição entre segmentos:
```sql
SELECT 
    gp_segment_id,
    COUNT(*) as num_rows,
    pg_size_pretty(pg_relation_size(gp_segment_id)) as segment_size
FROM vendas_skewed
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

3. Calcule estatísticas de skew:
```sql
WITH segment_counts AS (
    SELECT 
        gp_segment_id,
        COUNT(*) as row_count
    FROM vendas_skewed
    GROUP BY gp_segment_id
),
stats AS (
    SELECT 
        AVG(row_count) as avg_rows,
        MAX(row_count) as max_rows,
        MIN(row_count) as min_rows,
        STDDEV(row_count) as stddev_rows
    FROM segment_counts
)
SELECT 
    ROUND(avg_rows) as media_linhas,
    max_rows as max_linhas,
    min_rows as min_linhas,
    ROUND(stddev_rows) as desvio_padrao,
    ROUND((max_rows - avg_rows) / NULLIF(avg_rows, 0) * 100, 2) as skew_percentage,
    CASE 
        WHEN (max_rows - avg_rows) / NULLIF(avg_rows, 0) > 0.5 THEN 'CRÍTICO'
        WHEN (max_rows - avg_rows) / NULLIF(avg_rows, 0) > 0.2 THEN 'ALERTA'
        WHEN (max_rows - avg_rows) / NULLIF(avg_rows, 0) > 0.1 THEN 'MODERADO'
        ELSE 'OK'
    END as status_skew
FROM stats;
```

---

### Exercício 5B.1.2: Usando gp_toolkit para Skew

**Objetivo:** Ferramentas built-in de detecção

**Passos:**

1. Use gp_toolkit.gp_skew_coefficients:
```sql
SELECT 
    skcnamespace as schema,
    skcrelname as tabela,
    skccoeff as skew_coefficient
FROM gp_toolkit.gp_skew_coefficients
WHERE skcnamespace = 'public'
  AND skcrelname = 'vendas_skewed';
```

**Interpretando skew_coefficient:**
- 0.0 - 0.1: Distribuição excelente
- 0.1 - 0.3: Distribuição boa
- 0.3 - 0.5: Skew moderado
- 0.5 - 1.0: Skew significativo
- > 1.0: Skew crítico

2. View detalhada de skew:
```sql
SELECT * FROM gp_toolkit.gp_skew_details
WHERE skcrelname = 'vendas_skewed';
```

3. Análise de todas as tabelas:
```sql
SELECT 
    skcnamespace,
    skcrelname,
    ROUND(skccoeff::NUMERIC, 3) as skew_coef,
    CASE 
        WHEN skccoeff > 1.0 THEN 'CRÍTICO'
        WHEN skccoeff > 0.5 THEN 'ALERTA'
        WHEN skccoeff > 0.3 THEN 'MODERADO'
        ELSE 'OK'
    END as status,
    pg_size_pretty(pg_total_relation_size((skcnamespace||'.'||skcrelname)::regclass)) as tamanho
FROM gp_toolkit.gp_skew_coefficients
WHERE skcnamespace NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY skccoeff DESC
LIMIT 20;
```

---

### Exercício 5B.1.3: Análise de Distribution Key

**Objetivo:** Avaliar qualidade da chave de distribuição

**Passos:**

1. Analise cardinalidade da chave:
```sql
-- Verifique quantos valores distintos na chave
SELECT 
    COUNT(DISTINCT regiao) as valores_distintos,
    COUNT(*) as total_linhas,
    COUNT(DISTINCT regiao)::FLOAT / COUNT(*) as cardinalidade_ratio
FROM vendas_skewed;
```

**Regra geral:**
- Cardinalidade alta (próxima de 1.0): Boa distribuição
- Cardinalidade baixa (< 0.1): Provável skew

2. Distribuição de valores na chave:
```sql
SELECT 
    regiao,
    COUNT(*) as num_rows,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentual
FROM vendas_skewed
GROUP BY regiao
ORDER BY num_rows DESC;
```

3. Análise de NULLs na chave:
```sql
SELECT 
    COUNT(*) as total,
    COUNT(regiao) as nao_nulls,
    COUNT(*) - COUNT(regiao) as nulls,
    ROUND((COUNT(*) - COUNT(regiao)) * 100.0 / COUNT(*), 2) as pct_null
FROM vendas_skewed;
```

4. Compare com tabela bem distribuída:
```sql
-- Crie versão bem distribuída
CREATE TABLE vendas_balanced (LIKE vendas_skewed)
DISTRIBUTED BY (venda_id);  -- Alta cardinalidade!

INSERT INTO vendas_balanced SELECT * FROM vendas_skewed;
ANALYZE vendas_balanced;

-- Compare skew
SELECT 
    'SKEWED' as tipo,
    skccoeff as coefficient
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'vendas_skewed'
UNION ALL
SELECT 
    'BALANCED',
    skccoeff
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'vendas_balanced';
```

---

### Exercício 5B.1.4: Correção de Data Skew

**Objetivo:** Estratégias para corrigir distribuição

**Passos:**

1. **Estratégia 1: Mudar Distribution Key**
```sql
-- Identifique coluna com alta cardinalidade
SELECT 
    COUNT(DISTINCT venda_id) as distinct_venda_id,
    COUNT(DISTINCT loja_id) as distinct_loja_id,
    COUNT(DISTINCT produto_id) as distinct_produto_id,
    COUNT(*) as total
FROM vendas_skewed;

-- Recrie tabela com melhor chave
CREATE TABLE vendas_fixed AS
SELECT * FROM vendas_skewed
DISTRIBUTED BY (venda_id);  -- ou loja_id, produto_id

ANALYZE vendas_fixed;

-- Compare
SELECT 
    'Antes' as momento,
    skccoeff
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'vendas_skewed'
UNION ALL
SELECT 
    'Depois',
    skccoeff
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'vendas_fixed';
```

2. **Estratégia 2: Chave Composta**
```sql
-- Use múltiplas colunas para distribuição
CREATE TABLE vendas_composite AS
SELECT * FROM vendas_skewed
DISTRIBUTED BY (regiao, loja_id);

ANALYZE vendas_composite;

-- Verifique skew
SELECT skccoeff FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'vendas_composite';
```

3. **Estratégia 3: DISTRIBUTED RANDOMLY**
```sql
-- Para tabelas pequenas ou quando não há boa chave
CREATE TABLE vendas_random AS
SELECT * FROM vendas_skewed
DISTRIBUTED RANDOMLY;

ANALYZE vendas_random;

-- Verifique distribuição
SELECT 
    gp_segment_id,
    COUNT(*) as rows
FROM vendas_random
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

4. **Estratégia 4: Particionamento + Distribuição**
```sql
-- Combine particionamento com boa distribuição
CREATE TABLE vendas_optimized (
    venda_id BIGSERIAL,
    regiao VARCHAR(20),
    loja_id INTEGER,
    produto_id INTEGER,
    data_venda DATE,
    quantidade INTEGER,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id)  -- Alta cardinalidade
PARTITION BY RANGE (data_venda)
(
    START ('2024-01-01'::DATE) END ('2024-12-31'::DATE)
    EVERY (INTERVAL '1 month')
);

-- Carregue dados
INSERT INTO vendas_optimized 
SELECT 
    venda_id, regiao, loja_id, produto_id,
    CURRENT_DATE - (random() * 365)::INTEGER,
    quantidade, valor
FROM vendas_skewed;

ANALYZE vendas_optimized;
```

---

### Exercício 5B.1.5: Monitoramento Contínuo de Skew

**Objetivo:** Criar sistema de monitoramento

**Passos:**

1. View de monitoramento:
```sql
CREATE OR REPLACE VIEW vw_skew_monitor AS
SELECT 
    n.nspname as schema,
    c.relname as tabela,
    COALESCE(s.skccoeff, 0) as skew_coefficient,
    CASE 
        WHEN s.skccoeff IS NULL THEN 'N/A'
        WHEN s.skccoeff > 1.0 THEN 'CRÍTICO'
        WHEN s.skccoeff > 0.5 THEN 'ALERTA'
        WHEN s.skccoeff > 0.3 THEN 'MODERADO'
        ELSE 'OK'
    END as status,
    d.policytype as dist_policy,
    CASE d.policytype
        WHEN 'p' THEN 'HASH'
        WHEN 'r' THEN 'REPLICATED'
        ELSE 'RANDOM'
    END as dist_type,
    pg_size_pretty(pg_total_relation_size(c.oid)) as tamanho,
    c.reltuples::BIGINT as linhas_estimadas
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN gp_distribution_policy d ON d.localoid = c.oid
LEFT JOIN gp_toolkit.gp_skew_coefficients s ON s.skcrelname = c.relname
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY COALESCE(s.skccoeff, 0) DESC;
```

2. Consulte o monitor:
```sql
-- Visão geral
SELECT 
    status,
    COUNT(*) as num_tabelas,
    AVG(skew_coefficient) as avg_skew
FROM vw_skew_monitor
GROUP BY status
ORDER BY 
    CASE status
        WHEN 'CRÍTICO' THEN 1
        WHEN 'ALERTA' THEN 2
        WHEN 'MODERADO' THEN 3
        WHEN 'OK' THEN 4
        ELSE 5
    END;

-- Tabelas problemáticas
SELECT * FROM vw_skew_monitor
WHERE status IN ('CRÍTICO', 'ALERTA')
ORDER BY skew_coefficient DESC;
```

3. Function de análise detalhada:
```sql
CREATE OR REPLACE FUNCTION analyze_table_skew(p_table_name VARCHAR)
RETURNS TABLE(
    metric VARCHAR,
    value TEXT
) AS $$
DECLARE
    v_skew_coef NUMERIC;
    v_min_rows BIGINT;
    v_max_rows BIGINT;
    v_avg_rows NUMERIC;
BEGIN
    -- Skew coefficient
    SELECT skccoeff INTO v_skew_coef
    FROM gp_toolkit.gp_skew_coefficients
    WHERE skcrelname = p_table_name;
    
    -- Distribuição entre segmentos
    EXECUTE format('
        SELECT MIN(count), MAX(count), AVG(count)
        FROM (
            SELECT gp_segment_id, COUNT(*) as count
            FROM %I
            GROUP BY gp_segment_id
        ) t
    ', p_table_name)
    INTO v_min_rows, v_max_rows, v_avg_rows;
    
    RETURN QUERY
    SELECT 'Skew Coefficient'::VARCHAR, ROUND(v_skew_coef, 4)::TEXT
    UNION ALL
    SELECT 'Min Rows/Segment', v_min_rows::TEXT
    UNION ALL
    SELECT 'Max Rows/Segment', v_max_rows::TEXT
    UNION ALL
    SELECT 'Avg Rows/Segment', ROUND(v_avg_rows)::TEXT
    UNION ALL
    SELECT 'Variation %', ROUND((v_max_rows - v_avg_rows) / NULLIF(v_avg_rows, 0) * 100, 2)::TEXT
    UNION ALL
    SELECT 'Status', 
        CASE 
            WHEN v_skew_coef > 1.0 THEN 'CRÍTICO - Redistribuir'
            WHEN v_skew_coef > 0.5 THEN 'ALERTA - Revisar'
            WHEN v_skew_coef > 0.3 THEN 'MODERADO - Monitorar'
            ELSE 'OK'
        END;
END;
$$ LANGUAGE plpgsql;

-- Use
SELECT * FROM analyze_table_skew('vendas_skewed');
```

---

## Lab 5B.2: Processing Skew - Identificação (25-30 min)

### Objetivos
- Entender processing skew vs data skew
- Detectar desbalanceamento de processamento
- Identificar queries problemáticas
- Analisar uso de CPU/memória por segmento

### Conceitos Abordados
- **Processing Skew:** Processamento desigual entre segmentos
- **Broadcast Motion:** Envio de dados para todos segmentos
- **Redistribute Motion:** Redistribuição de dados
- **Segment Timing:** Tempo de execução por segmento

### Processing Skew vs Data Skew

| Tipo | Causa | Sintoma | Detecção |
|------|-------|---------|----------|
| **Data Skew** | Distribuição desigual de dados | Tabela desbalanceada | gp_skew_coefficients |
| **Processing Skew** | Query processa desigualmente | Alguns segmentos lentos | EXPLAIN ANALYZE |

---

### Exercício 5B.2.1: Detectando Processing Skew

**Objetivo:** Identificar queries com processamento desigual

**Passos:**

1. Execute query que causa processing skew:
```sql
-- Query com GROUP BY em coluna de baixa cardinalidade
EXPLAIN ANALYZE
SELECT 
    regiao,
    COUNT(*) as total_vendas,
    SUM(valor) as receita_total,
    AVG(valor) as ticket_medio
FROM vendas_skewed
GROUP BY regiao;
```

2. Analise o output:
```
Gather Motion (slice1; segments: 4)
  ->  HashAggregate
        Group Key: regiao
        ->  Seq Scan on vendas_skewed
              
Slice statistics:
  Slice 0: (seg0) avg/max CPU: 0.5s/0.5s
  Slice 1: (seg0) 5.2s/5.2s   <-- Lento!
           (seg1) 0.3s/0.3s
           (seg2) 0.2s/0.2s
           (seg3) 0.1s/0.1s
```

**Indicadores de Processing Skew:**
- Grande diferença entre avg/max tempo
- Um segmento processa muito mais que outros
- "Rows out" muito desigual entre segmentos

3. Query para análise detalhada:
```sql
-- Execute query e capture query ID
SELECT pg_backend_pid();  -- Anote o PID

-- Em outra sessão, monitore
SELECT 
    gp_segment_id,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE pid = <PID_ANTERIOR>  -- Substitua
ORDER BY gp_segment_id;
```

---

### Exercício 5B.2.2: Skew em JOINs

**Objetivo:** Detectar redistribuição custosa

**Passos:**

1. Crie cenário de JOIN com skew:
```sql
-- Tabela fato com skew
CREATE TABLE fato_vendas (
    venda_id BIGINT,
    cliente_id INTEGER,
    data_venda DATE,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (cliente_id);

-- Concentre dados em poucos clientes
INSERT INTO fato_vendas
SELECT 
    i,
    CASE 
        WHEN i <= 800000 THEN 1  -- 80% para cliente 1
        ELSE (i % 1000) + 2
    END,
    CURRENT_DATE - (i % 365),
    random() * 1000
FROM generate_series(1, 1000000) i;

-- Dimensão clientes
CREATE TABLE dim_clientes_join (
    cliente_id INTEGER,
    nome VARCHAR(100),
    categoria VARCHAR(20)
)
DISTRIBUTED BY (cliente_id);

INSERT INTO dim_clientes_join
SELECT 
    i,
    'Cliente_' || i,
    CASE (i % 3)
        WHEN 0 THEN 'Bronze'
        WHEN 1 THEN 'Silver'
        ELSE 'Gold'
    END
FROM generate_series(1, 10000) i;

ANALYZE fato_vendas;
ANALYZE dim_clientes_join;
```

2. Execute JOIN e analise:
```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT 
    c.categoria,
    COUNT(*) as num_vendas,
    SUM(f.valor) as receita
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
GROUP BY c.categoria;
```

3. Verifique Redistribute Motion:
```
HashAggregate
  ->  Redistribute Motion (hash: categoria)  <-- Redistribuição!
        ->  Hash Join
              Hash Cond: (f.cliente_id = c.cliente_id)
              ->  Seq Scan on fato_vendas f
                    Rows out: Avg 250000 Max 800000  <-- Skew!
              ->  Hash
                    ->  Seq Scan on dim_clientes_join c
```

4. Solução: DISTRIBUTED REPLICATED para dim pequena:
```sql
-- Recrie dimensão como replicada
DROP TABLE dim_clientes_join;

CREATE TABLE dim_clientes_join (
    cliente_id INTEGER,
    nome VARCHAR(100),
    categoria VARCHAR(20)
)
DISTRIBUTED REPLICATED;

INSERT INTO dim_clientes_join
SELECT i, 'Cliente_' || i, CASE (i % 3) WHEN 0 THEN 'Bronze' WHEN 1 THEN 'Silver' ELSE 'Gold' END
FROM generate_series(1, 10000) i;

ANALYZE dim_clientes_join;

-- Repita JOIN
EXPLAIN (ANALYZE, VERBOSE)
SELECT 
    c.categoria,
    COUNT(*) as num_vendas,
    SUM(f.valor) as receita
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
GROUP BY c.categoria;
```

**Resultado:** Sem Redistribute Motion!

---

### Exercício 5B.2.3: Monitoramento de Segmentos em Tempo Real

**Objetivo:** Observar atividade por segmento durante query

**Passos:**

1. View de atividade por segmento:
```sql
CREATE OR REPLACE VIEW vw_segment_activity AS
SELECT 
    gp_segment_id,
    COUNT(*) as active_queries,
    STRING_AGG(DISTINCT state, ', ') as states,
    STRING_AGG(DISTINCT wait_event_type, ', ') as wait_types,
    MAX(EXTRACT(EPOCH FROM (CLOCK_TIMESTAMP() - query_start))) as longest_query_sec
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
  AND backend_type = 'client backend'
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

2. Monitore durante query longa:
```sql
-- Em sessão 1, execute query pesada
SELECT 
    regiao,
    loja_id,
    COUNT(*) as vendas,
    SUM(valor) as receita
FROM vendas_skewed
GROUP BY regiao, loja_id
ORDER BY receita DESC;

-- Em sessão 2, monitore
SELECT * FROM vw_segment_activity;
```

3. Histórico de queries lentas por segmento:
```sql
-- Requer logging habilitado
SELECT 
    gp_segment_id,
    query,
    EXTRACT(EPOCH FROM (state_change - query_start)) as duration_sec,
    state
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration_sec DESC;
```

---

### Exercício 5B.2.4: Identificando Bottlenecks

**Objetivo:** Encontrar segmentos gargalo

**Passos:**

1. Function para identificar bottlenecks:
```sql
CREATE OR REPLACE FUNCTION identify_bottleneck_segments()
RETURNS TABLE(
    segment_id INTEGER,
    issue_type VARCHAR,
    description TEXT,
    severity VARCHAR
) AS $$
BEGIN
    -- Verifica data skew
    RETURN QUERY
    WITH skew_tables AS (
        SELECT 
            skcrelname,
            skccoeff
        FROM gp_toolkit.gp_skew_coefficients
        WHERE skccoeff > 0.5
    )
    SELECT 
        -1,
        'DATA_SKEW'::VARCHAR,
        'Tabela ' || skcrelname || ' tem skew coef ' || ROUND(skccoeff, 2)::TEXT,
        CASE 
            WHEN skccoeff > 1.0 THEN 'CRÍTICO'
            ELSE 'ALERTA'
        END::VARCHAR
    FROM skew_tables;
    
    -- Adicione mais verificações conforme necessário
END;
$$ LANGUAGE plpgsql;

SELECT * FROM identify_bottleneck_segments();
```

2. Análise de disk I/O por segmento:
```sql
-- Verifica uso de disco
SELECT 
    gp_segment_id,
    pg_size_pretty(SUM(pg_total_relation_size(oid))) as total_size,
    COUNT(*) as num_tables
FROM pg_class
WHERE relkind = 'r'
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

---

## Lab 5B.3: EXPLAIN Avançado (35-40 min)

### Objetivos
- Dominar interpretação de EXPLAIN ANALYZE
- Entender operadores complexos
- Analisar Motion Nodes detalhadamente
- Otimizar queries baseado no plano

### Conceitos Abordados
- **EXPLAIN vs EXPLAIN ANALYZE:** Estimativa vs execução real
- **Motion Nodes:** Gather, Redistribute, Broadcast
- **Join Methods:** Nested Loop, Hash Join, Merge Join
- **Scan Methods:** Seq Scan, Index Scan, Bitmap Scan
- **Slices:** Paralelização de execução

---

### Exercício 5B.3.1: EXPLAIN Detalhado - Anatomia

**Objetivo:** Entender cada parte do plano

**Passos:**

1. Query exemplo:
```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, TIMING)
SELECT 
    c.categoria,
    p.nome as produto,
    SUM(f.valor) as receita,
    COUNT(*) as num_vendas
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
JOIN (
    SELECT produto_id, 'Produto_' || produto_id as nome
    FROM generate_series(1, 1000) produto_id
) p ON f.venda_id % 1000 + 1 = p.produto_id
WHERE f.data_venda >= CURRENT_DATE - 90
GROUP BY c.categoria, p.nome
HAVING SUM(f.valor) > 10000
ORDER BY receita DESC
LIMIT 100;
```

2. **Anatomia do plano de execução:**

```
Limit  (cost=X..Y rows=Z width=W)
  ^       ^    ^    ^      ^
  |       |    |    |      |
  |       |    |    |      +-- Largura estimada da linha (bytes)
  |       |    |    +-- Número estimado de linhas
  |       |    +-- Custo final estimado
  |       +-- Custo inicial estimado
  +-- Tipo de operador

  Execution Time: 123.456 ms  <-- Tempo real (ANALYZE)
  
  ->  Gather Motion 1:1  (slice1; segments: 4)
        ^         ^  ^      ^         ^
        |         |  |      |         |
        |         |  |      |         +-- Número de segmentos
        |         |  |      +-- Identificador do slice
        |         |  +-- Destino (coordinator)
        |         +-- Origem (todos segmentos)
        +-- Tipo de motion
        
        Rows out:  Avg 100.0 rows x 4 workers.  Max 105 rows (seg2)
        ^          ^                ^            ^
        |          |                |            |
        |          |                |            +-- Segmento com mais rows
        |          |                +-- Número de workers
        |          +-- Média de rows por worker
        +-- Linhas produzidas
```

3. **Componentes importantes:**

```sql
-- EXPLAIN sem ANALYZE (apenas estimativas)
EXPLAIN
SELECT * FROM vendas_skewed WHERE regiao = 'Sudeste';

-- EXPLAIN ANALYZE (executa e mostra reais)
EXPLAIN ANALYZE
SELECT * FROM vendas_skewed WHERE regiao = 'Sudeste';

-- Opções adicionais
EXPLAIN (
    ANALYZE TRUE,    -- Executa query
    VERBOSE TRUE,    -- Output detalhado
    BUFFERS TRUE,    -- Info de buffers/cache
    TIMING TRUE,     -- Tempo de cada nó
    COSTS TRUE,      -- Custos estimados
    SUMMARY TRUE     -- Resumo final
)
SELECT ...;
```

---

### Exercício 5B.3.2: Motion Nodes - Análise Profunda

**Objetivo:** Entender cada tipo de Motion

**Passos:**

1. **Gather Motion** (N:1 - segmentos para coordinator):
```sql
EXPLAIN ANALYZE
SELECT categoria, COUNT(*)
FROM dim_clientes_join
GROUP BY categoria;
```

Output:
```
Gather Motion 4:1  (slice1; segments: 4)
  ->  HashAggregate
        Group Key: categoria
        ->  Seq Scan on dim_clientes_join
```

**Quando ocorre:**
- Agregações finais
- ORDER BY global
- LIMIT/OFFSET
- Resultado final para cliente

2. **Redistribute Motion** (N:N - redistribuição entre segmentos):
```sql
EXPLAIN ANALYZE
SELECT 
    f.cliente_id,
    SUM(f.valor) as total
FROM fato_vendas f
GROUP BY f.cliente_id
HAVING SUM(f.valor) > 5000;
```

Output:
```
Gather Motion 4:1
  ->  HashAggregate
        Filter: (SUM(valor) > 5000)
        ->  Redistribute Motion 4:4  (hash: cliente_id)  <--
              ->  Seq Scan on fato_vendas f
```

**Quando ocorre:**
- GROUP BY em coluna não-distribution key
- JOIN em colunas diferentes das distribution keys
- Necessidade de reagrupar dados

**Custo:** ALTO - requer movimento de dados pela rede

3. **Broadcast Motion** (1:N - envia para todos segmentos):
```sql
-- Tabela pequena distribuída
CREATE TEMP TABLE filtros (
    categoria VARCHAR(20)
) DISTRIBUTED RANDOMLY;

INSERT INTO filtros VALUES ('Bronze'), ('Silver');

EXPLAIN ANALYZE
SELECT c.*
FROM dim_clientes_join c
JOIN filtros f ON c.categoria = f.categoria;
```

Output:
```
Hash Join
  Hash Cond: (c.categoria = f.categoria)
  ->  Seq Scan on dim_clientes_join c
  ->  Hash
        ->  Broadcast Motion 4:4  <-- Envia para todos
              ->  Seq Scan on filtros f
```

**Quando ocorre:**
- JOIN com tabela pequena
- Tabela pequena não replicada
- Otimizador escolhe broadcast ao invés de redistribute

**Custo:** Médio - aceita para tabelas pequenas

4. **Análise de custos:**
```sql
-- Compare planos
EXPLAIN
SELECT c.categoria, COUNT(*)
FROM dim_clientes_join c
JOIN filtros f ON c.categoria = f.categoria
GROUP BY c.categoria;

-- vs tabela replicada
DROP TABLE filtros;
CREATE TEMP TABLE filtros (categoria VARCHAR(20)) DISTRIBUTED REPLICATED;
INSERT INTO filtros VALUES ('Bronze'), ('Silver');

EXPLAIN
SELECT c.categoria, COUNT(*)
FROM dim_clientes_join c
JOIN filtros f ON c.categoria = f.categoria
GROUP BY c.categoria;
```

---

### Exercício 5B.3.3: Join Methods

**Objetivo:** Entender quando cada método é usado

**Passos:**

1. **Hash Join** (mais comum no GP):
```sql
EXPLAIN ANALYZE
SELECT c.nome, COUNT(*) as vendas
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
GROUP BY c.nome
LIMIT 10;
```

Output:
```
Hash Join  (cost=X..Y rows=Z)
  Hash Cond: (f.cliente_id = c.cliente_id)
  ->  Seq Scan on fato_vendas f
  ->  Hash
        ->  Seq Scan on dim_clientes_join c
              Buckets: 1024  Batches: 1  Memory Usage: 50kB
```

**Características:**
- Constrói hash table da tabela menor
- Probe na tabela maior
- Eficiente para equijoins (=)
- Requer memória (work_mem)

2. **Nested Loop Join** (raro, para tabelas muito pequenas):
```sql
-- Force nested loop
SET enable_hashjoin = OFF;
SET enable_mergejoin = OFF;

EXPLAIN ANALYZE
SELECT *
FROM dim_clientes_join c
WHERE c.cliente_id IN (1, 2, 3);

RESET enable_hashjoin;
RESET enable_mergejoin;
```

**Características:**
- Loop externo e interno
- Bom para pequenos datasets
- Suporta non-equijoins
- Pode usar índices

3. **Merge Join** (para dados pré-ordenados):
```sql
-- Datasets ordenados
EXPLAIN ANALYZE
SELECT *
FROM (SELECT * FROM fato_vendas ORDER BY cliente_id) f
JOIN (SELECT * FROM dim_clientes_join ORDER BY cliente_id) c
  ON f.cliente_id = c.cliente_id;
```

**Características:**
- Requer dados ordenados
- Scan linear em ambas tabelas
- Eficiente para grandes datasets ordenados
- Comum com índices

---

### Exercício 5B.3.4: Scan Methods

**Objetivo:** Diferenciar tipos de scans

**Passos:**

1. **Sequential Scan:**
```sql
EXPLAIN ANALYZE
SELECT * FROM fato_vendas
WHERE valor > 100;
```

**Quando usado:**
- Acessa grande % da tabela
- Sem índice disponível
- Índice mais custoso que seq scan
- Paralelizado entre segmentos

2. **Index Scan:**
```sql
CREATE INDEX idx_fato_cliente ON fato_vendas(cliente_id);
ANALYZE fato_vendas;

EXPLAIN ANALYZE
SELECT * FROM fato_vendas
WHERE cliente_id = 1;
```

**Quando usado:**
- Query muito seletiva (< 1% linhas)
- Índice disponível
- ORDER BY matching índice

3. **Bitmap Scan:**
```sql
EXPLAIN ANALYZE
SELECT * FROM fato_vendas
WHERE cliente_id IN (1, 2, 3, 4, 5);
```

Output:
```
Bitmap Heap Scan
  Recheck Cond: (cliente_id = ANY ('{1,2,3,4,5}'))
  ->  Bitmap Index Scan on idx_fato_cliente
        Index Cond: (cliente_id = ANY ('{1,2,3,4,5}'))
```

**Quando usado:**
- Múltiplos valores (IN, OR)
- Seletividade moderada (1-10% linhas)
- Combina múltiplos índices

---

### Exercício 5B.3.5: Otimização Baseada em EXPLAIN

**Objetivo:** Usar EXPLAIN para otimizar query

**Passos:**

1. Query problemática:
```sql
EXPLAIN ANALYZE
SELECT 
    f.data_venda,
    c.categoria,
    COUNT(*) as vendas,
    SUM(f.valor) as receita
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
WHERE f.data_venda >= CURRENT_DATE - 365
  AND EXTRACT(MONTH FROM f.data_venda) IN (1, 2, 3)
GROUP BY f.data_venda, c.categoria
ORDER BY receita DESC;
```

2. **Problemas identificados:**
```
❌ Redistribute Motion (hash: data_venda, categoria)
   -> Custo alto de redistribuição

❌ Filter: EXTRACT(MONTH FROM data_venda) IN (...)
   -> Função impede uso de índice/partição

❌ Seq Scan on fato_vendas
   -> Tabela grande sem filtro efetivo
```

3. **Otimizações:**

```sql
-- Otimização 1: Remova função do filtro
EXPLAIN ANALYZE
SELECT 
    f.data_venda,
    c.categoria,
    COUNT(*) as vendas,
    SUM(f.valor) as receita
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
WHERE f.data_venda >= '2024-01-01'
  AND f.data_venda < '2024-04-01'  -- Ao invés de EXTRACT
GROUP BY f.data_venda, c.categoria
ORDER BY receita DESC;

-- Otimização 2: Materialize subquery se reusada
CREATE TEMP TABLE vendas_q1 AS
SELECT 
    f.data_venda,
    c.categoria,
    f.cliente_id,
    f.valor
FROM fato_vendas f
JOIN dim_clientes_join c ON f.cliente_id = c.cliente_id
WHERE f.data_venda >= '2024-01-01'
  AND f.data_venda < '2024-04-01'
DISTRIBUTED BY (data_venda, categoria);

-- Queries subsequentes são rápidas
SELECT data_venda, categoria, SUM(valor)
FROM vendas_q1
GROUP BY data_venda, categoria;
```

4. **Checklist de otimização:**
```sql
-- ✅ Estatísticas atualizadas?
ANALYZE fato_vendas;

-- ✅ Partition pruning funcionando?
-- Verifique "Partitions selected" no EXPLAIN

-- ✅ Motion nodes minimizados?
-- Menos Redistribute/Broadcast = melhor

-- ✅ Join method apropriado?
-- Hash Join geralmente melhor no GP

-- ✅ work_mem suficiente?
SET work_mem = '256MB';

-- ✅ Índices apropriados?
-- Só para queries seletivas
```

---

## Exercício Integrador: Diagnóstico Completo

**Objetivo:** Análise end-to-end de performance

**Cenário:** Sistema lento, identifique problemas

**Passos:**

1. **Passo 1: Identificar tabelas com skew**
```sql
SELECT * FROM vw_skew_monitor
WHERE status IN ('CRÍTICO', 'ALERTA')
ORDER BY skew_coefficient DESC
LIMIT 10;
```

2. **Passo 2: Analisar queries lentas**
```sql
-- Top 10 queries lentas (requer pg_stat_statements)
SELECT 
    query,
    calls,
    ROUND(total_exec_time::NUMERIC / 1000, 2) as total_sec,
    ROUND(mean_exec_time::NUMERIC, 2) as avg_ms,
    ROUND(stddev_exec_time::NUMERIC, 2) as stddev_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

3. **Passo 3: EXPLAIN queries identificadas**
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
<query_lenta>;
```

4. **Passo 4: Identifique problemas:**
- ❌ Data skew alto (> 0.5)
- ❌ Redistribute Motion frequente
- ❌ Broadcast de tabelas grandes
- ❌ Nested Loop em datasets grandes
- ❌ Seq Scan sem partition pruning
- ❌ Funções em WHERE impedem índices

5. **Passo 5: Aplique correções**
```sql
-- Redistribua tabelas com skew
ALTER TABLE tabela_problema SET DISTRIBUTED BY (nova_chave);

-- Replique dimensões pequenas
ALTER TABLE dim_pequena SET DISTRIBUTED REPLICATED;

-- Atualize estatísticas
ANALYZE tabela_problema;

-- Crie índices apropriados
CREATE INDEX idx_seletivo ON tabela(coluna_seletiva);

-- Ajuste configurações
SET work_mem = '512MB';
```

---

## Resumo do Módulo 5B

### Habilidades Adquiridas
✅ Detectar e quantificar data skew  
✅ Corrigir problemas de distribuição  
✅ Identificar processing skew  
✅ Otimizar JOINs e motion nodes  
✅ Interpretar EXPLAIN ANALYZE detalhadamente  
✅ Reconhecer operadores e métodos  
✅ Otimizar queries baseado em planos  
✅ Diagnosticar problemas de performance  

### Ferramentas Principais

```sql
-- Skew detection
SELECT * FROM gp_toolkit.gp_skew_coefficients;
SELECT * FROM gp_toolkit.gp_skew_details;

-- Distribuição
SELECT gp_segment_id, COUNT(*) FROM tabela GROUP BY 1;

-- EXPLAIN
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) query;

-- Monitoramento
SELECT * FROM pg_stat_activity;
SELECT * FROM pg_stat_statements;
```

### Thresholds de Skew

| Métrica | Excelente | Bom | Moderado | Crítico |
|---------|-----------|-----|----------|---------|
| Skew Coefficient | 0.0-0.1 | 0.1-0.3 | 0.3-0.5 | > 0.5 |
| Variation % | 0-10% | 10-20% | 20-50% | > 50% |
| Max/Avg Ratio | 1.0-1.1 | 1.1-1.3 | 1.3-1.5 | > 1.5 |

### Padrões de Otimização

**Pattern 1: Corrigir Data Skew**
```sql
-- Antes: DISTRIBUTED BY (baixa_cardinalidade)
-- Depois: DISTRIBUTED BY (alta_cardinalidade)
ALTER TABLE tabela SET DISTRIBUTED BY (chave_melhor);
```

**Pattern 2: Eliminar Motion**
```sql
-- Para dimensões pequenas
ALTER TABLE dim SET DISTRIBUTED REPLICATED;

-- Para tabelas grandes, alinhe distribution keys
-- fato DISTRIBUTED BY (cliente_id)
-- dim DISTRIBUTED BY (cliente_id)
```

**Pattern 3: Otimizar Query**
```sql
-- Evite funções em WHERE
WHERE data >= '2024-01-01'  -- ✅
-- Ao invés de
WHERE EXTRACT(YEAR FROM data) = 2024  -- ❌

-- Materialize subqueries complexas
CREATE TEMP TABLE resultado AS SELECT ...;
```

### Troubleshooting

| Sintoma | Causa Provável | Solução |
|---------|---------------|---------|
| Query lenta em tabela específica | Data skew | Redistribuir tabela |
| Um segmento lento | Processing skew | Revisar distribution key |
| Redistribute Motion custoso | Keys não alinhadas | Alinhar distribution keys |
| Broadcast de tabela grande | Tabela não replicada | SET DISTRIBUTED REPLICATED |
| Estimativas ruins | Estatísticas desatualizadas | ANALYZE |
| Nested Loop em big data | Planner escolheu errado | Revisar JOINs ou force hash |

### Boas Práticas

**Design de Tabelas:**
1. Use distribution keys de alta cardinalidade
2. Alinhe keys entre fato e dimensões
3. Replique dimensões pequenas (< 10MB)
4. Evite DISTRIBUTED RANDOMLY em tabelas grandes

**Queries:**
1. Sempre use EXPLAIN ANALYZE em novas queries
2. Minimize Motion nodes
3. Evite funções em colunas de filtro/join
4. Mantenha estatísticas atualizadas

**Monitoramento:**
1. Monitore skew coefficient regularmente
2. Revise queries lentas semanalmente
3. Ajuste work_mem conforme necessário
4. Log e analise EXPLAIN de queries críticas

---

**Parabéns!** Você concluiu o treinamento completo de Greenplum!

Você agora domina:
- ✅ Preparação de ambiente e navegação
- ✅ DDL, distribuição e compressão
- ✅ Particionamento e índices
- ✅ Carregamento de dados e ETL
- ✅ Manutenção com VACUUM
- ✅ Detecção de skew e otimização

Continue praticando e aplicando esses conceitos em cenários reais!

---

**Fim do Módulo 5B e do Treinamento Completo**
