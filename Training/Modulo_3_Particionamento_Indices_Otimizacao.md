# Módulo 3: Particionamento, Índices e Otimização de Queries

**Duração Total:** 120-150 minutos  
**Objetivo:** Dominar técnicas de particionamento, uso efetivo de índices e otimização de queries para maximizar performance no Greenplum.

---

## Índice
1. [Lab 3.1: Fundamentos de Particionamento](#lab-31-fundamentos-de-particionamento-30-35-min)
2. [Lab 3.2: Particionamento Avançado](#lab-32-particionamento-avançado-25-30-min)
3. [Lab 3.3: Índices no Greenplum](#lab-33-índices-no-greenplum-30-35-min)
4. [Lab 3.4: Análise e Otimização de Queries](#lab-34-análise-e-otimização-de-queries-35-40-min)

---

## Lab 3.1: Fundamentos de Particionamento (30-35 min)

### Objetivos
- Compreender conceitos básicos de particionamento
- Criar tabelas particionadas por RANGE e LIST
- Gerenciar partições (adicionar, remover, consultar)
- Entender benefícios de performance

### Conceitos Abordados
- **Partition Pruning:** Eliminação de partições desnecessárias
- **RANGE Partitioning:** Particionamento por faixas de valores (datas, números)
- **LIST Partitioning:** Particionamento por valores específicos (região, status)
- **Gestão de Partições:** ADD, DROP, SPLIT, EXCHANGE

### Por que Particionar?

O particionamento divide uma tabela grande em pedaços menores (partições) com base em uma coluna chave:

- ✅ **Performance:** Queries acessam apenas partições relevantes
- ✅ **Manutenção:** VACUUM, ANALYZE mais rápidos em partições individuais
- ✅ **Gestão de dados:** DROP de partições antigas é instantâneo
- ✅ **Backup/Restore:** Pode ser feito por partição
- ✅ **Carregamento:** Inserção paralela em diferentes partições

---

### Exercício 3.1.1: Particionamento por RANGE - Conceitos Básicos

**Objetivo:** Criar primeira tabela particionada por data

**Cenário:** Tabela de vendas particionada por mês

**Passos:**

1. Crie tabela particionada por data (mensal):
```sql
CREATE TABLE vendas_particionada (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor_total NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda)
(
    START ('2024-01-01'::DATE) END ('2024-12-31'::DATE) 
    EVERY (INTERVAL '1 month'),
    DEFAULT PARTITION outros
);
```

**Entendendo a sintaxe:**
- **PARTITION BY RANGE (data_venda):** Coluna de particionamento
- **START/END:** Faixa inicial e final
- **EVERY (INTERVAL '1 month'):** Cria partições mensais automaticamente
- **DEFAULT PARTITION:** Captura dados fora do range definido

2. Visualize as partições criadas:
```sql
-- Lista todas as partições
SELECT 
    schemaname,
    tablename,
    partitiontablename,
    partitionname,
    partitionrank,
    partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
ORDER BY partitionrank;
```

3. Veja informações detalhadas das partições:
```sql
-- Tamanho de cada partição
SELECT 
    pt.partitiontablename,
    pt.partitionname,
    pt.partitionboundary,
    pg_size_pretty(pg_relation_size(pt.partitiontablename::regclass)) as partition_size
FROM pg_partitions pt
WHERE pt.tablename = 'vendas_particionada'
ORDER BY pt.partitionrank;
```

4. Insira dados de teste:
```sql
INSERT INTO vendas_particionada
SELECT 
    i,
    DATE '2024-01-01' + (i % 365),  -- Distribui ao longo de 2024
    (random() * 1000)::INTEGER + 1,
    (random() * 100)::INTEGER + 1,
    (random() * 10)::INTEGER + 1,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100000) i;
```

5. Verifique distribuição dos dados entre partições:
```sql
SELECT 
    pt.partitionname,
    pt.partitionboundary,
    COUNT(*) as num_rows
FROM vendas_particionada vp
JOIN pg_partitions pt ON pt.tablename = 'vendas_particionada'
WHERE vp.data_venda >= pt.partitionboundary::text::date
  AND vp.data_venda < (pt.partitionboundary::text::date + INTERVAL '1 month')
GROUP BY pt.partitionname, pt.partitionboundary
ORDER BY pt.partitionboundary;
```

Alternativa mais simples:
```sql
-- Contagem por partição usando tableoid
SELECT 
    pt.partitiontablename,
    COUNT(*) as num_rows,
    MIN(data_venda) as min_data,
    MAX(data_venda) as max_data
FROM vendas_particionada vp
JOIN pg_partition_rule pr ON vp.tableoid = pr.parchildrelid
JOIN pg_partitions pt ON pt.partitiontablename = pr.parname::regclass::text
GROUP BY pt.partitiontablename
ORDER BY pt.partitiontablename;
```

6. Teste partition pruning (eliminação de partições):
```sql
-- Query que acessa apenas uma partição
EXPLAIN
SELECT COUNT(*), SUM(valor_total)
FROM vendas_particionada
WHERE data_venda BETWEEN '2024-06-01' AND '2024-06-30';
```

**Análise do EXPLAIN:**
- Procure por "Partition Selector" no plano
- Quantas partições foram selecionadas?
- As outras partições foram ignoradas? (pruned)

---

### Exercício 3.1.2: Particionamento por LIST

**Objetivo:** Particionar por valores categóricos

**Cenário:** Tabela de vendas particionada por região

**Passos:**

1. Crie tabela particionada por lista:
```sql
CREATE TABLE vendas_por_regiao (
    venda_id BIGINT,
    data_venda DATE,
    regiao VARCHAR(20),
    loja_id INTEGER,
    valor_total NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id)
PARTITION BY LIST (regiao)
(
    PARTITION norte VALUES ('Norte', 'N'),
    PARTITION nordeste VALUES ('Nordeste', 'NE'),
    PARTITION centro_oeste VALUES ('Centro-Oeste', 'CO'),
    PARTITION sudeste VALUES ('Sudeste', 'SE', 'Sul'),
    PARTITION sul VALUES ('S'),
    DEFAULT PARTITION outras_regioes
);
```

2. Visualize as partições:
```sql
SELECT 
    partitiontablename,
    partitionname,
    partitionlistvalues
FROM pg_partitions
WHERE tablename = 'vendas_por_regiao'
ORDER BY partitionrank;
```

3. Insira dados:
```sql
INSERT INTO vendas_por_regiao
SELECT 
    i,
    CURRENT_DATE - (i % 365),
    CASE (i % 5)
        WHEN 0 THEN 'Norte'
        WHEN 1 THEN 'Nordeste'
        WHEN 2 THEN 'Centro-Oeste'
        WHEN 3 THEN 'Sudeste'
        ELSE 'Sul'
    END,
    (i % 50) + 1,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 50000) i;
```

4. Verifique distribuição:
```sql
SELECT 
    regiao,
    COUNT(*) as total_vendas,
    SUM(valor_total) as receita_total,
    AVG(valor_total) as ticket_medio
FROM vendas_por_regiao
GROUP BY regiao
ORDER BY total_vendas DESC;
```

5. Teste partition pruning:
```sql
EXPLAIN ANALYZE
SELECT COUNT(*), AVG(valor_total)
FROM vendas_por_regiao
WHERE regiao IN ('Sudeste', 'Sul');
```

**Quando usar LIST Partitioning:**
- ✅ Colunas com valores categóricos fixos
- ✅ Número limitado de valores distintos
- ✅ Regiões, estados, status, tipos
- ❌ Colunas com muitos valores únicos

---

### Exercício 3.1.3: Gerenciamento de Partições - Adicionar

**Objetivo:** Adicionar novas partições dinamicamente

**Cenário:** Adicionar partições para anos futuros

**Passos:**

1. Verifique partições existentes:
```sql
SELECT 
    partitionname,
    partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
ORDER BY partitionrank DESC
LIMIT 5;
```

2. Adicione partições para 2025:
```sql
-- Adicionar todo o ano 2025 (mensalmente)
ALTER TABLE vendas_particionada 
ADD PARTITION 
    START ('2025-01-01'::DATE) END ('2025-12-31'::DATE)
    EVERY (INTERVAL '1 month');
```

3. Adicione partição específica:
```sql
-- Adicionar apenas janeiro de 2026
ALTER TABLE vendas_particionada 
ADD PARTITION jan_2026 
    START ('2026-01-01'::DATE) END ('2026-02-01'::DATE);
```

4. Verifique as novas partições:
```sql
SELECT 
    partitionname,
    partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
  AND partitionboundary::text >= '2025-01-01'
ORDER BY partitionrank;
```

5. Teste inserção nas novas partições:
```sql
INSERT INTO vendas_particionada VALUES
    (999991, '2025-03-15', 1001, 201, 5, 500.00),
    (999992, '2025-06-20', 1002, 202, 3, 350.00),
    (999993, '2026-01-10', 1003, 203, 2, 200.00);

-- Verifique onde foram inseridos
SELECT 
    pt.partitionname,
    vp.*
FROM vendas_particionada vp
JOIN pg_partition_rule pr ON vp.tableoid = pr.parchildrelid
JOIN pg_partitions pt ON pt.partitiontablename = pr.parname::regclass::text
WHERE vp.venda_id >= 999991;
```

---

### Exercício 3.1.4: Gerenciamento de Partições - Remover

**Objetivo:** Remover partições antigas (arquivamento/purge)

**Cenário:** Remover dados antigos para liberar espaço

**Passos:**

1. Identifique partições candidatas à remoção:
```sql
SELECT 
    partitionname,
    partitionboundary,
    pg_size_pretty(pg_relation_size(partitiontablename::regclass)) as size
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
  AND partitionboundary::text < '2024-04-01'  -- Partições antigas
ORDER BY partitionrank;
```

2. **Opção 1:** Dropar partição diretamente (PERDE DADOS):
```sql
-- CUIDADO: Isso deleta permanentemente os dados!
ALTER TABLE vendas_particionada 
DROP PARTITION FOR ('2024-01-15'::DATE);

-- Ou pelo nome
ALTER TABLE vendas_particionada 
DROP PARTITION jan_2024;
```

3. **Opção 2:** Trocar partição (PRESERVA DADOS):
```sql
-- Criar tabela externa para arquivamento
CREATE TABLE vendas_arquivo_jan2024 (LIKE vendas_particionada)
DISTRIBUTED BY (venda_id);

-- Trocar partição (move dados para tabela externa)
ALTER TABLE vendas_particionada 
EXCHANGE PARTITION FOR ('2024-01-15'::DATE)
WITH TABLE vendas_arquivo_jan2024;

-- Agora a partição está vazia e pode ser dropada
ALTER TABLE vendas_particionada 
DROP PARTITION FOR ('2024-01-15'::DATE);

-- Dados preservados em vendas_arquivo_jan2024
SELECT COUNT(*) FROM vendas_arquivo_jan2024;
```

4. **Opção 3:** Truncar partição (mantém estrutura):
```sql
-- Esvazia partição mas mantém ela na tabela
ALTER TABLE vendas_particionada 
TRUNCATE PARTITION FOR ('2024-02-15'::DATE);
```

5. Verifique remoções:
```sql
SELECT 
    partitionname,
    partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
ORDER BY partitionrank
LIMIT 10;
```

**Estratégias de Retenção:**
- **DROP:** Dados não são necessários (compliance, LGPD)
- **EXCHANGE:** Arquivar em tabela externa ou external table
- **TRUNCATE:** Manter estrutura para reuso futuro

---

### Exercício 3.1.5: Partition Pruning - Medindo Benefícios

**Objetivo:** Quantificar ganhos de performance

**Cenário:** Comparar tabela particionada vs não particionada

**Passos:**

1. Crie versão não particionada:
```sql
CREATE TABLE vendas_nao_particionada (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor_total NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id);

-- Copie os mesmos dados
INSERT INTO vendas_nao_particionada 
SELECT * FROM vendas_particionada;
```

2. Execute ANALYZE em ambas:
```sql
ANALYZE vendas_particionada;
ANALYZE vendas_nao_particionada;
```

3. Compare query em período específico:
```sql
-- Não particionada
EXPLAIN ANALYZE
SELECT 
    DATE_TRUNC('day', data_venda) as dia,
    COUNT(*) as vendas,
    SUM(valor_total) as receita
FROM vendas_nao_particionada
WHERE data_venda BETWEEN '2024-06-01' AND '2024-06-30'
GROUP BY 1
ORDER BY 1;

-- Particionada
EXPLAIN ANALYZE
SELECT 
    DATE_TRUNC('day', data_venda) as dia,
    COUNT(*) as vendas,
    SUM(valor_total) as receita
FROM vendas_particionada
WHERE data_venda BETWEEN '2024-06-01' AND '2024-06-30'
GROUP BY 1
ORDER BY 1;
```

4. Compare scan completo (sem filtro):
```sql
-- Não particionada
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor_total)
FROM vendas_nao_particionada;

-- Particionada
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor_total)
FROM vendas_particionada;
```

5. Analise os resultados:
```sql
SELECT 
    'Particionada' as tipo,
    COUNT(*) as total_rows,
    pg_size_pretty(pg_total_relation_size('vendas_particionada')) as total_size
FROM vendas_particionada
UNION ALL
SELECT 
    'Não Particionada',
    COUNT(*),
    pg_size_pretty(pg_total_relation_size('vendas_nao_particionada'))
FROM vendas_nao_particionada;
```

**Métricas para Comparar:**
- ✅ Execution Time (tempo total)
- ✅ Rows scanned (linhas lidas)
- ✅ Presença de "Partition Selector" no plano
- ✅ Número de partições acessadas vs total

---

## Lab 3.2: Particionamento Avançado (25-30 min)

### Objetivos
- Criar particionamento multi-nível (hierárquico)
- Trabalhar com subpartições
- Implementar estratégias híbridas
- Otimizar gestão de partições

### Conceitos Avançados
- **Multi-level Partitioning:** RANGE + LIST combinados
- **Subpartições:** Particionar partições
- **Partition Templates:** Definir padrões para subpartições
- **ALTER PARTITION:** Operações em partições específicas

---

### Exercício 3.2.1: Particionamento Multi-Nível

**Objetivo:** Criar tabela com particionamento hierárquico

**Cenário:** Vendas particionadas por ANO e subparticionadas por REGIÃO

**Passos:**

1. Crie tabela com particionamento em dois níveis:
```sql
CREATE TABLE vendas_multinivel (
    venda_id BIGINT,
    data_venda DATE,
    regiao VARCHAR(20),
    loja_id INTEGER,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor_total NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda)
SUBPARTITION BY LIST (regiao)
SUBPARTITION TEMPLATE
(
    SUBPARTITION norte VALUES ('Norte'),
    SUBPARTITION nordeste VALUES ('Nordeste'),
    SUBPARTITION centro_oeste VALUES ('Centro-Oeste'),
    SUBPARTITION sudeste VALUES ('Sudeste'),
    SUBPARTITION sul VALUES ('Sul'),
    DEFAULT SUBPARTITION outras
)
(
    START ('2023-01-01'::DATE) END ('2025-01-01'::DATE) 
    EVERY (INTERVAL '1 year')
);
```

**Entendendo a estrutura:**
- **Nível 1 (RANGE):** Particiona por ano
- **Nível 2 (LIST):** Cada partição anual é subparticionada por região
- **SUBPARTITION TEMPLATE:** Define padrão aplicado a todas partições

2. Visualize a hierarquia:
```sql
-- Partições de nível 1 (anos)
SELECT 
    partitionlevel,
    partitionname,
    partitionrank,
    partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_multinivel'
  AND partitionlevel = 0
ORDER BY partitionrank;

-- Subpartições de nível 2 (regiões dentro de cada ano)
SELECT 
    partitionlevel,
    partitionname,
    partitionrank,
    partitionlistvalues
FROM pg_partitions
WHERE tablename = 'vendas_multinivel'
  AND partitionlevel = 1
ORDER BY partitionrank;
```

3. Insira dados:
```sql
INSERT INTO vendas_multinivel
SELECT 
    i,
    DATE '2023-01-01' + (random() * 730)::INTEGER,  -- 2023-2024
    CASE (i % 5)
        WHEN 0 THEN 'Norte'
        WHEN 1 THEN 'Nordeste'
        WHEN 2 THEN 'Centro-Oeste'
        WHEN 3 THEN 'Sudeste'
        ELSE 'Sul'
    END,
    (i % 100) + 1,
    (i % 10000) + 1,
    (i % 1000) + 1,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 200000) i;
```

4. Analise distribuição multi-nível:
```sql
SELECT 
    EXTRACT(YEAR FROM data_venda) as ano,
    regiao,
    COUNT(*) as vendas,
    SUM(valor_total) as receita
FROM vendas_multinivel
GROUP BY ROLLUP(ano, regiao)
ORDER BY ano, regiao NULLS LAST;
```

5. Teste partition pruning em múltiplos níveis:
```sql
EXPLAIN ANALYZE
SELECT 
    COUNT(*),
    AVG(valor_total)
FROM vendas_multinivel
WHERE data_venda BETWEEN '2024-01-01' AND '2024-12-31'
  AND regiao IN ('Sudeste', 'Sul');
```

**Análise:**
- Apenas 2 subpartições acessadas (sudeste e sul de 2024)
- Outras 10+ subpartições foram pruned
- Redução massiva de I/O

---

### Exercício 3.2.2: Particionamento com Granularidades Diferentes

**Objetivo:** Criar partições com períodos variados

**Cenário:** Dados recentes mensais, históricos trimestrais

**Passos:**

1. Crie tabela com partições de tamanhos diferentes:
```sql
CREATE TABLE vendas_granularidade_variada (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    valor_total NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda)
(
    -- Dados históricos: trimestrais (2022-2023)
    PARTITION q1_2022 START ('2022-01-01'::DATE) END ('2022-04-01'::DATE),
    PARTITION q2_2022 START ('2022-04-01'::DATE) END ('2022-07-01'::DATE),
    PARTITION q3_2022 START ('2022-07-01'::DATE) END ('2022-10-01'::DATE),
    PARTITION q4_2022 START ('2022-10-01'::DATE) END ('2023-01-01'::DATE),
    
    PARTITION q1_2023 START ('2023-01-01'::DATE) END ('2023-04-01'::DATE),
    PARTITION q2_2023 START ('2023-04-01'::DATE) END ('2023-07-01'::DATE),
    PARTITION q3_2023 START ('2023-07-01'::DATE) END ('2023-10-01'::DATE),
    PARTITION q4_2023 START ('2023-10-01'::DATE) END ('2024-01-01'::DATE),
    
    -- Dados recentes: mensais (2024)
    START ('2024-01-01'::DATE) END ('2024-12-31'::DATE) 
    EVERY (INTERVAL '1 month'),
    
    -- Dados futuros: semanais (2025)
    START ('2025-01-01'::DATE) END ('2025-02-01'::DATE)
    EVERY (INTERVAL '1 week'),
    
    DEFAULT PARTITION outros
);
```

2. Visualize a estrutura:
```sql
SELECT 
    partitionname,
    partitionboundary,
    CASE 
        WHEN partitionname LIKE 'q%' THEN 'Trimestral'
        WHEN partitionboundary::text LIKE '2024%' THEN 'Mensal'
        WHEN partitionboundary::text LIKE '2025%' THEN 'Semanal'
        ELSE 'Outro'
    END as granularidade
FROM pg_partitions
WHERE tablename = 'vendas_granularidade_variada'
ORDER BY partitionrank;
```

3. Insira dados em diferentes períodos:
```sql
INSERT INTO vendas_granularidade_variada
SELECT 
    i,
    DATE '2022-01-01' + (random() * 1095)::INTEGER,  -- 2022-2024
    (random() * 10000)::INTEGER + 1,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100000) i;
```

**Quando usar granularidades variadas:**
- ✅ Dados recentes acessados frequentemente (granular)
- ✅ Dados antigos acessados raramente (agregado)
- ✅ Reduz número total de partições
- ✅ Otimiza armazenamento e manutenção

---

### Exercício 3.2.3: SPLIT Partition - Dividir Partições

**Objetivo:** Dividir partição existente em múltiplas

**Cenário:** Partição DEFAULT cresceu demais, precisa subdividir

**Passos:**

1. Verifique a partição DEFAULT:
```sql
-- Insira dados fora do range
INSERT INTO vendas_particionada VALUES
    (888881, '2023-06-15', 1001, 201, 5, 500.00),
    (888882, '2023-12-20', 1002, 202, 3, 350.00),
    (888883, '2027-03-10', 1003, 203, 2, 200.00);

-- Verifique partição DEFAULT
SELECT 
    pt.partitionname,
    COUNT(*) as num_rows
FROM vendas_particionada vp
JOIN pg_partition_rule pr ON vp.tableoid = pr.parchildrelid
JOIN pg_partitions pt ON pt.partitiontablename = pr.parname::regclass::text
WHERE pt.partitionname = 'outros'
GROUP BY pt.partitionname;
```

2. Divida a partição DEFAULT:
```sql
-- Criar partições específicas para 2023
ALTER TABLE vendas_particionada 
SPLIT DEFAULT PARTITION
START ('2023-01-01'::DATE) END ('2023-12-31'::DATE)
EVERY (INTERVAL '1 month')
INTO (PARTITION ano_2023, DEFAULT PARTITION outros);
```

3. Verifique o resultado:
```sql
SELECT 
    partitionname,
    partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_particionada'
  AND (partitionboundary::text LIKE '2023%' OR partitionname = 'outros')
ORDER BY partitionrank;
```

4. SPLIT de partição específica:
```sql
-- Criar tabela exemplo com partições trimestrais
CREATE TABLE vendas_trimestre (
    venda_id BIGINT,
    data_venda DATE,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda)
(
    PARTITION q1 START ('2024-01-01'::DATE) END ('2024-04-01'::DATE),
    PARTITION q2 START ('2024-04-01'::DATE) END ('2024-07-01'::DATE)
);

-- Dividir Q1 em meses
ALTER TABLE vendas_trimestre
SPLIT PARTITION q1 AT ('2024-02-01'::DATE, '2024-03-01'::DATE)
INTO (PARTITION jan, PARTITION fev, PARTITION mar);

-- Verifique
SELECT partitionname, partitionboundary
FROM pg_partitions
WHERE tablename = 'vendas_trimestre'
ORDER BY partitionrank;
```

---

### Exercício 3.2.4: Otimizações em Partições Específicas

**Objetivo:** Aplicar diferentes estratégias por partição

**Cenário:** Partições antigas comprimidas, recentes descomprimidas

**Passos:**

1. Crie tabela com estrutura base:
```sql
CREATE TABLE logs_otimizados (
    log_id BIGSERIAL,
    timestamp TIMESTAMP,
    nivel VARCHAR(10),
    mensagem TEXT,
    usuario VARCHAR(50)
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (log_id)
PARTITION BY RANGE (timestamp)
(
    START ('2024-01-01'::TIMESTAMP) END ('2024-12-31'::TIMESTAMP)
    EVERY (INTERVAL '1 month')
);
```

2. Modifique compressão de partições antigas:
```sql
-- Comprimir partições antigas (janeiro a junho)
ALTER TABLE logs_otimizados 
ALTER PARTITION FOR ('2024-01-15'::TIMESTAMP)
SET WITH (compresstype=zstd, compresslevel=9);

ALTER TABLE logs_otimizados 
ALTER PARTITION FOR ('2024-02-15'::TIMESTAMP)
SET WITH (compresstype=zstd, compresslevel=9);

-- Partições recentes: sem compressão ou compressão leve
ALTER TABLE logs_otimizados 
ALTER PARTITION FOR ('2024-11-15'::TIMESTAMP)
SET WITH (compresstype=zstd, compresslevel=1);
```

3. Visualize configurações por partição:
```sql
SELECT 
    pt.partitionname,
    pt.partitionboundary,
    pg_size_pretty(pg_relation_size(pt.partitiontablename::regclass)) as size,
    (SELECT option_value 
     FROM pg_options_to_table(c.reloptions) 
     WHERE option_name = 'compresstype') as compress_type,
    (SELECT option_value 
     FROM pg_options_to_table(c.reloptions) 
     WHERE option_name = 'compresslevel') as compress_level
FROM pg_partitions pt
JOIN pg_class c ON c.oid = pt.partitiontablename::regclass
WHERE pt.tablename = 'logs_otimizados'
ORDER BY pt.partitionrank;
```

**Estratégias por Partição:**
- **Partições antigas:** Alta compressão, read-only
- **Partições recentes:** Baixa/sem compressão, escrita ativa
- **Partições futuras:** Pré-criadas com configuração otimizada

---

## Lab 3.3: Índices no Greenplum (30-35 min)

### Objetivos
- Compreender tipos de índices no Greenplum
- Decidir quando criar índices
- Evitar índices desnecessários
- Medir impacto de índices

### Conceitos Abordados
- **B-tree Index:** Índice tradicional (único tipo no GP)
- **Quando NÃO usar índices:** Scan paralelo é mais eficiente
- **Índices em tabelas AO/AOCO:** Restrições e considerações
- **Maintenance:** REINDEX, estatísticas

### Índices no Greenplum vs PostgreSQL

⚠️ **IMPORTANTE:** No Greenplum, índices funcionam diferente:

- ❌ **Sem GIN, GiST, BRIN:** Apenas B-tree disponível
- ❌ **Menos úteis que no PostgreSQL:** Scan paralelo é muito rápido
- ✅ **Úteis para:** Queries seletivas, JOINs em tabelas pequenas
- ✅ **Melhores em:** Tabelas HEAP pequenas/médias

---

### Exercício 3.3.1: Quando Índices SÃO Úteis

**Objetivo:** Identificar casos onde índices melhoram performance

**Cenário:** Tabela pequena com queries seletivas

**Passos:**

1. Crie tabela pequena sem índice:
```sql
CREATE TABLE clientes (
    cliente_id SERIAL PRIMARY KEY,
    cpf VARCHAR(14) UNIQUE,
    nome VARCHAR(200),
    email VARCHAR(100),
    data_nascimento DATE,
    cidade VARCHAR(100),
    estado CHAR(2),
    ativo BOOLEAN DEFAULT TRUE
)
DISTRIBUTED BY (cliente_id);

-- Insira 100k clientes
INSERT INTO clientes (cpf, nome, email, data_nascimento, cidade, estado, ativo)
SELECT 
    LPAD((random() * 99999999999)::BIGINT::TEXT, 11, '0'),
    'Cliente ' || i,
    'cliente' || i || '@email.com',
    DATE '1950-01-01' + (random() * 25000)::INTEGER,
    'Cidade ' || (i % 100),
    CASE (i % 27)
        WHEN 0 THEN 'SP' WHEN 1 THEN 'RJ' WHEN 2 THEN 'MG'
        WHEN 3 THEN 'RS' WHEN 4 THEN 'PR' WHEN 5 THEN 'SC'
        ELSE 'SP'
    END,
    (random() < 0.9)  -- 90% ativos
FROM generate_series(1, 100000) i;

ANALYZE clientes;
```

2. Teste query seletiva sem índice:
```sql
-- Query por CPF (muito seletiva - retorna 1 linha)
EXPLAIN ANALYZE
SELECT * FROM clientes WHERE cpf = '12345678901';

-- Query por email
EXPLAIN ANALYZE
SELECT * FROM clientes WHERE email = 'cliente5000@email.com';
```

3. Crie índices em colunas de busca:
```sql
CREATE INDEX idx_clientes_cpf ON clientes(cpf);
CREATE INDEX idx_clientes_email ON clientes(email);
CREATE INDEX idx_clientes_estado_cidade ON clientes(estado, cidade);

ANALYZE clientes;
```

4. Repita as mesmas queries:
```sql
-- Deve usar índice agora
EXPLAIN ANALYZE
SELECT * FROM clientes WHERE cpf = '12345678901';

EXPLAIN ANALYZE
SELECT * FROM clientes WHERE email = 'cliente5000@email.com';

-- Índice composto
EXPLAIN ANALYZE
SELECT * FROM clientes 
WHERE estado = 'SP' AND cidade = 'Cidade 42';
```

5. Compare tempos de execução:
```sql
-- Tabela de comparação
SELECT 
    'Sem índice' as cenario,
    'Veja output EXPLAIN acima' as execution_time
UNION ALL
SELECT 
    'Com índice',
    'Veja output EXPLAIN acima';
```

**Quando índices SÃO úteis:**
- ✅ Tabelas pequenas/médias (< 1M linhas)
- ✅ Queries muito seletivas (< 1% das linhas)
- ✅ Colunas de JOIN frequentes
- ✅ Chaves estrangeiras
- ✅ Constraints UNIQUE

---

### Exercício 3.3.2: Quando Índices NÃO São Úteis

**Objetivo:** Entender limitações de índices no Greenplum

**Cenário:** Tabela grande particionada com scans paralelos

**Passos:**

1. Use tabela grande existente:
```sql
-- Tabela de vendas (já criada anteriormente)
SELECT 
    COUNT(*) as total_rows,
    pg_size_pretty(pg_total_relation_size('vendas_particionada')) as size
FROM vendas_particionada;
```

2. Teste query agregada sem índice:
```sql
EXPLAIN ANALYZE
SELECT 
    DATE_TRUNC('month', data_venda) as mes,
    COUNT(*) as vendas,
    SUM(valor_total) as receita
FROM vendas_particionada
WHERE data_venda >= '2024-01-01'
GROUP BY 1
ORDER BY 1;
```

3. Crie índice na coluna de filtro:
```sql
CREATE INDEX idx_vendas_data ON vendas_particionada(data_venda);
ANALYZE vendas_particionada;
```

4. Repita a mesma query:
```sql
EXPLAIN ANALYZE
SELECT 
    DATE_TRUNC('month', data_venda) as mes,
    COUNT(*) as vendas,
    SUM(valor_total) as receita
FROM vendas_particionada
WHERE data_venda >= '2024-01-01'
GROUP BY 1
ORDER BY 1;
```

**Análise:**
- O índice foi usado? Provavelmente NÃO!
- Por quê? Seq Scan paralelo é mais eficiente
- Query acessa muitas linhas (agregação)
- Partition pruning já otimizou o acesso

5. Remova o índice desnecessário:
```sql
DROP INDEX idx_vendas_data;
```

**Quando índices NÃO são úteis:**
- ❌ Tabelas grandes com queries agregadas
- ❌ Queries que retornam muitas linhas (> 5%)
- ❌ Tabelas AO/AOCO com scan completo
- ❌ Colunas de baixa cardinalidade (poucos valores distintos)
- ❌ Tabelas particionadas (partition pruning já otimiza)

---

### Exercício 3.3.3: Índices em Tabelas Particionadas

**Objetivo:** Gerenciar índices em estruturas particionadas

**Passos:**

1. Crie índice em tabela particionada:
```sql
-- Índice será criado em TODAS as partições
CREATE INDEX idx_vendas_part_cliente ON vendas_particionada(cliente_id);
```

2. Verifique índices criados:
```sql
-- Lista índices em cada partição
SELECT 
    pt.partitiontablename,
    i.indexname,
    pg_size_pretty(pg_relation_size(i.indexname::regclass)) as index_size
FROM pg_partitions pt
JOIN pg_indexes i ON i.tablename = pt.partitiontablename::text
WHERE pt.tablename = 'vendas_particionada'
  AND i.indexname LIKE '%cliente%'
ORDER BY pt.partitionrank;
```

3. Crie índice apenas em partição específica:
```sql
-- Índice apenas na partição mais recente
CREATE INDEX idx_vendas_dez_cliente 
ON vendas_particionada_1_prt_12(cliente_id);  -- Ajuste nome da partição
```

4. DROP de índices em tabela particionada:
```sql
-- Drop em todas as partições
DROP INDEX idx_vendas_part_cliente;

-- Ou drop em partição específica
-- DROP INDEX idx_vendas_dez_cliente;
```

**Considerações:**
- Índices consomem espaço em CADA partição
- REINDEX pode ser custoso em tabelas grandes
- Considere índices apenas em partições ativas

---

### Exercício 3.3.4: Manutenção de Índices

**Objetivo:** REINDEX e atualização de estatísticas

**Passos:**

1. Verifique índices fragmentados:
```sql
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    idx_scan as num_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND tablename = 'clientes'
ORDER BY pg_relation_size(indexrelid) DESC;
```

2. REINDEX de índice específico:
```sql
REINDEX INDEX idx_clientes_cpf;
```

3. REINDEX de tabela completa:
```sql
REINDEX TABLE clientes;
```

4. Atualizar estatísticas após mudanças:
```sql
ANALYZE clientes;
```

5. Verificar uso de índices:
```sql
-- Índices não utilizados (candidatos a remoção)
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Boas Práticas:**
- Execute ANALYZE após cargas grandes
- REINDEX após muitos DELETEs/UPDATEs
- Monitore uso de índices periodicamente
- Remova índices não utilizados

---

## Lab 3.4: Análise e Otimização de Queries (35-40 min)

### Objetivos
- Ler e interpretar EXPLAIN ANALYZE
- Identificar gargalos de performance
- Otimizar queries problemáticas
- Usar hints e configurações

### Conceitos Abordados
- **EXPLAIN vs EXPLAIN ANALYZE:** Plano vs execução real
- **Motion Nodes:** Redistribuição de dados entre segmentos
- **Query Optimizer:** Como o GP escolhe planos
- **Statistics:** Importância do ANALYZE

---

### Exercício 3.4.1: Lendo EXPLAIN ANALYZE - Básico

**Objetivo:** Compreender estrutura do plano de execução

**Passos:**

1. Execute query simples:
```sql
EXPLAIN ANALYZE
SELECT 
    cliente_id,
    COUNT(*) as num_vendas,
    SUM(valor_total) as receita_total
FROM vendas_particionada
WHERE data_venda >= '2024-06-01'
GROUP BY cliente_id
HAVING COUNT(*) > 5
ORDER BY receita_total DESC
LIMIT 100;
```

2. **Entenda os componentes principais:**

```
Componentes do EXPLAIN:
├── Limit                          <- Operação de topo
│   ├── Execution Time: X ms       <- Tempo real
│   └── Rows out: 100              <- Linhas retornadas
│
├── Gather Motion                  <- Coleta dados dos segmentos
│   ├── Slice: X                   <- Identificador de fatia paralela
│   └── Segments: 4                <- Número de segmentos
│
├── Sort                           <- Ordenação
│   ├── Sort Key: receita_total    <- Coluna de ordenação
│   └── Sort Method: top-N         <- Método usado
│
├── HashAggregate                  <- Agregação (GROUP BY)
│   ├── Group Key: cliente_id      <- Chave de agrupamento
│   └── Filter: COUNT(*) > 5       <- HAVING
│
└── Seq Scan on vendas_...        <- Varredura sequencial
    ├── Partition Selector         <- Seleção de partições
    ├── Partitions selected: 7     <- Partições acessadas
    └── Filter: data_venda >= ...  <- WHERE
```

3. **Identifique métricas importantes:**
```sql
-- Mesma query com opções detalhadas
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT cliente_id, COUNT(*), SUM(valor_total)
FROM vendas_particionada
WHERE data_venda >= '2024-06-01'
GROUP BY cliente_id
LIMIT 10;
```

**Métricas-chave:**
- **Total Execution Time:** Tempo total (milissegundos)
- **Rows out:** Linhas retornadas em cada etapa
- **Partitions selected:** Quantas partições foram acessadas
- **Motion nodes:** Movimento de dados (caro!)
- **Buffers:** Leituras de disco vs cache

---

### Exercício 3.4.2: Identificando Motion Nodes

**Objetivo:** Entender custo de redistribuição de dados

**Cenário:** JOINs que causam movimento de dados

**Passos:**

1. Crie tabelas para teste:
```sql
-- Tabela fato distribuída por venda_id
CREATE TABLE vendas_teste (
    venda_id BIGINT,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id);

-- Tabela dimensão distribuída diferente
CREATE TABLE produtos_teste (
    produto_id INTEGER,
    nome VARCHAR(100),
    categoria VARCHAR(50)
)
DISTRIBUTED BY (produto_id);

-- Inserir dados
INSERT INTO vendas_teste
SELECT i, (i % 1000) + 1, (i % 100) + 1, (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100000) i;

INSERT INTO produtos_teste
SELECT i, 'Produto ' || i, 'Categoria ' || (i % 10)
FROM generate_series(1, 100) i;

ANALYZE vendas_teste;
ANALYZE produtos_teste;
```

2. Execute JOIN com redistribuição:
```sql
EXPLAIN ANALYZE
SELECT 
    p.categoria,
    COUNT(*) as vendas,
    SUM(v.valor) as receita
FROM vendas_teste v
JOIN produtos_teste p ON v.produto_id = p.produto_id
GROUP BY p.categoria;
```

**Identifique no plano:**
- **Redistribute Motion:** Dados redistribuídos entre segmentos
- **Broadcast Motion:** Tabela pequena enviada para todos segmentos
- Qual tabela foi redistribuída?

3. Otimize com DISTRIBUTED REPLICATED:
```sql
-- Recrie tabela produtos como replicada
DROP TABLE produtos_teste;

CREATE TABLE produtos_teste (
    produto_id INTEGER,
    nome VARCHAR(100),
    categoria VARCHAR(50)
)
DISTRIBUTED REPLICATED;

INSERT INTO produtos_teste
SELECT i, 'Produto ' || i, 'Categoria ' || (i % 10)
FROM generate_series(1, 100) i;

ANALYZE produtos_teste;
```

4. Repita o JOIN:
```sql
EXPLAIN ANALYZE
SELECT 
    p.categoria,
    COUNT(*) as vendas,
    SUM(v.valor) as receita
FROM vendas_teste v
JOIN produtos_teste p ON v.produto_id = p.produto_id
GROUP BY p.categoria;
```

**Análise:**
- Ainda há Motion? Provavelmente NÃO!
- Tempo de execução melhorou?
- Menos "Rows redistributed"

---

### Exercício 3.4.3: Otimizando Queries com Problemas

**Objetivo:** Diagnosticar e corrigir queries lentas

**Cenário 1: Função em WHERE impede uso de partição**

```sql
-- Query problemática
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor_total)
FROM vendas_particionada
WHERE EXTRACT(YEAR FROM data_venda) = 2024;
```

**Problema:** EXTRACT impede partition pruning

**Solução:**
```sql
-- Query otimizada
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor_total)
FROM vendas_particionada
WHERE data_venda >= '2024-01-01'
  AND data_venda < '2025-01-01';
```

**Cenário 2: Estatísticas desatualizadas**

```sql
-- Simule dados desatualizados
INSERT INTO vendas_particionada
SELECT 
    i,
    CURRENT_DATE,
    (i % 1000) + 1,
    (i % 100) + 1,
    (random() * 10)::INTEGER + 1,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(900000, 1000000) i;

-- Query sem ANALYZE
EXPLAIN
SELECT * FROM vendas_particionada WHERE venda_id = 950000;
-- Veja o "rows" estimado

-- Atualize estatísticas
ANALYZE vendas_particionada;

-- Query após ANALYZE
EXPLAIN
SELECT * FROM vendas_particionada WHERE venda_id = 950000;
-- Estimativa deve estar melhor
```

**Cenário 3: SELECT * desnecessário**

```sql
-- Ineficiente (lê todas colunas)
EXPLAIN ANALYZE
SELECT *
FROM vendas_particionada
WHERE data_venda >= '2024-06-01';

-- Otimizado (lê apenas necessárias)
EXPLAIN ANALYZE
SELECT cliente_id, valor_total
FROM vendas_particionada
WHERE data_venda >= '2024-06-01';
```

**Benefício:** Em tabelas AOCO, reduz I/O drasticamente

---

### Exercício 3.4.4: Hints e Configurações

**Objetivo:** Forçar comportamentos específicos do otimizador

**Passos:**

1. **Desabilitar Seq Scan para forçar uso de índice:**
```sql
SET enable_seqscan = OFF;

EXPLAIN ANALYZE
SELECT * FROM clientes WHERE cpf = '12345678901';

SET enable_seqscan = ON;  -- Voltar ao padrão
```

2. **Aumentar work_mem para sorts maiores:**
```sql
-- Padrão pode ser pequeno
SHOW work_mem;

-- Aumentar temporariamente
SET work_mem = '256MB';

EXPLAIN ANALYZE
SELECT * 
FROM vendas_particionada
ORDER BY valor_total DESC
LIMIT 1000;

RESET work_mem;
```

3. **Forçar redistribuição de tabela específica:**
```sql
-- Use hints através de comentários especiais
EXPLAIN ANALYZE
SELECT /*+ Leading((v p)) */ 
    p.categoria,
    SUM(v.valor) as receita
FROM vendas_teste v
JOIN produtos_teste p ON v.produto_id = p.produto_id
GROUP BY p.categoria;
```

4. **Verificar configurações importantes:**
```sql
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name IN (
    'work_mem',
    'shared_buffers',
    'effective_cache_size',
    'random_page_cost',
    'gp_enable_minmax_optimization',
    'optimizer'
)
ORDER BY name;
```

**Configurações importantes:**
- **work_mem:** Memória para sorts e hashes
- **gp_enable_minmax_optimization:** Otimização de MIN/MAX
- **optimizer:** ORCA (novo) vs Planner (antigo)

---

### Exercício 3.4.5: Checklist de Otimização

**Objetivo:** Processo sistemático para otimizar queries

**Checklist:**

```sql
-- 1. Execute EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT ... FROM ... WHERE ...;

-- 2. Verifique estatísticas atualizadas
SELECT 
    schemaname,
    tablename,
    last_analyze,
    n_live_tup as estimated_rows
FROM pg_stat_user_tables
WHERE tablename IN ('sua_tabela');

-- Se desatualizado:
ANALYZE sua_tabela;

-- 3. Verifique partition pruning
-- No EXPLAIN, procure por "Partition Selector"
-- Partitions selected deve ser < total de partições

-- 4. Identifique Motion nodes
-- Procure por "Redistribute Motion" ou "Broadcast Motion"
-- Considere DISTRIBUTED REPLICATED para tabelas pequenas

-- 5. Verifique uso de índices
SELECT 
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename = 'sua_tabela';

-- 6. Analise colunas projetadas
-- Em AOCO, SELECT * é muito mais caro que colunas específicas

-- 7. Revise filtros (WHERE)
-- Evite funções em colunas: WHERE EXTRACT(YEAR FROM data) = 2024
-- Prefira: WHERE data >= '2024-01-01' AND data < '2025-01-01'

-- 8. Verifique distribuição de dados
SELECT gp_segment_id, COUNT(*)
FROM sua_tabela
GROUP BY gp_segment_id
ORDER BY gp_segment_id;

-- 9. Considere materializar subqueries
-- CREATE TEMP TABLE resultado AS SELECT ...
-- Útil para queries complexas executadas múltiplas vezes

-- 10. Use VACUUM após DELETEs/UPDATEs
VACUUM ANALYZE sua_tabela;
```

---

## Exercício Integrador: Otimização Completa

**Objetivo:** Aplicar todos os conceitos do módulo

**Cenário:** Sistema de relatórios lento

```sql
-- Query problemática original
EXPLAIN ANALYZE
SELECT 
    EXTRACT(MONTH FROM v.data_venda) as mes,
    p.categoria,
    c.estado,
    COUNT(DISTINCT v.cliente_id) as clientes_unicos,
    COUNT(*) as total_vendas,
    SUM(v.valor_total) as receita,
    AVG(v.valor_total) as ticket_medio
FROM vendas_particionada v
JOIN dim_produtos p ON v.produto_id = p.produto_id
JOIN dim_clientes c ON v.cliente_id = c.cliente_id
WHERE EXTRACT(YEAR FROM v.data_venda) = 2024
GROUP BY 1, 2, 3
ORDER BY mes, receita DESC;
```

**Problemas identificados:**
1. ❌ EXTRACT impede partition pruning
2. ❌ JOINs podem causar redistribuição
3. ❌ Estatísticas podem estar desatualizadas
4. ❌ COUNT DISTINCT pode ser custoso

**Solução otimizada:**

```sql
-- 1. Garanta que dimensões são replicadas
-- (já feito nos exercícios anteriores)

-- 2. Atualize estatísticas
ANALYZE vendas_particionada;
ANALYZE dim_produtos;
ANALYZE dim_clientes;

-- 3. Query otimizada
EXPLAIN ANALYZE
SELECT 
    DATE_TRUNC('month', v.data_venda)::DATE as mes,
    p.categoria,
    c.estado,
    COUNT(DISTINCT v.cliente_id) as clientes_unicos,
    COUNT(*) as total_vendas,
    SUM(v.valor_total) as receita,
    AVG(v.valor_total) as ticket_medio
FROM vendas_particionada v
JOIN dim_produtos p ON v.produto_id = p.produto_id
JOIN dim_clientes c ON v.cliente_id = c.cliente_id
WHERE v.data_venda >= '2024-01-01'
  AND v.data_venda < '2025-01-01'
GROUP BY 1, 2, 3
ORDER BY mes, receita DESC;

-- 4. Alternativa: Materializar para uso repetido
CREATE TEMP TABLE report_mensal AS
SELECT 
    DATE_TRUNC('month', v.data_venda)::DATE as mes,
    p.categoria,
    c.estado,
    COUNT(DISTINCT v.cliente_id) as clientes_unicos,
    COUNT(*) as total_vendas,
    SUM(v.valor_total) as receita,
    AVG(v.valor_total) as ticket_medio
FROM vendas_particionada v
JOIN dim_produtos p ON v.produto_id = p.produto_id
JOIN dim_clientes c ON v.cliente_id = c.cliente_id
WHERE v.data_venda >= '2024-01-01'
  AND v.data_venda < '2025-01-01'
GROUP BY 1, 2, 3
DISTRIBUTED BY (mes);

-- Agora queries no resultado são instantâneas
SELECT * FROM report_mensal
WHERE categoria = 'Eletrônicos'
ORDER BY receita DESC;
```

**Melhorias alcançadas:**
- ✅ Partition pruning funcionando (12 de 12 partições acessadas apenas)
- ✅ Sem redistribuição (dimensões replicadas)
- ✅ Estatísticas atualizadas (estimativas precisas)
- ✅ Filtro otimizado (range de datas)
- ✅ Opção de materialização para reuso

---

## Resumo do Módulo 3

### Habilidades Adquiridas
✅ Criar e gerenciar tabelas particionadas  
✅ Implementar particionamento multi-nível  
✅ Adicionar, remover e dividir partições  
✅ Usar índices apropriadamente no Greenplum  
✅ Ler e interpretar EXPLAIN ANALYZE  
✅ Identificar e resolver gargalos de performance  
✅ Otimizar queries complexas  
✅ Aplicar configurações e hints do otimizador  

### Decisões-Chave

| Aspecto | Recomendação | Quando Usar |
|---------|-------------|-------------|
| **Particionamento** | RANGE por data | Tabelas > 10M linhas, queries filtram por data |
| **Granularidade** | Mensal/Trimestral | Depende do volume e padrão de acesso |
| **Multi-nível** | RANGE + LIST | Múltiplas dimensões de filtro |
| **Índices** | Usar com cautela | Tabelas pequenas, queries muito seletivas |
| **Estatísticas** | ANALYZE frequente | Após cargas, antes de queries importantes |
| **Motion** | Evitar | REPLICATED para dimensões pequenas |

### Comandos Principais

```sql
-- Particionamento
CREATE TABLE ... PARTITION BY RANGE (col) (START ... END ... EVERY ...);
ALTER TABLE ... ADD PARTITION ...;
ALTER TABLE ... DROP PARTITION ...;
ALTER TABLE ... SPLIT PARTITION ...;

-- Índices
CREATE INDEX ... ON ... (...);
DROP INDEX ...;
REINDEX TABLE ...;

-- Análise
EXPLAIN ANALYZE ...;
ANALYZE ...;
VACUUM ANALYZE ...;

-- Informações
SELECT * FROM pg_partitions WHERE tablename = '...';
SELECT * FROM pg_stat_user_indexes WHERE tablename = '...';
SELECT * FROM gp_toolkit.gp_skew_coefficients;
```

### Padrões de Otimização

**Pattern 1: Tabela Fato Grande**
```sql
CREATE TABLE fato (...) 
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (id_unico)
PARTITION BY RANGE (data) (EVERY INTERVAL '1 month');
```

**Pattern 2: Dimensão Pequena**
```sql
CREATE TABLE dim (...)
DISTRIBUTED REPLICATED;
```

**Pattern 3: Query Analítica**
```sql
SELECT 
    colunas_especificas,  -- Não use SELECT *
    AGG(col)
FROM tabela_particionada
WHERE data >= '2024-01-01'  -- Use range, não funções
  AND data < '2025-01-01'
GROUP BY colunas_especificas;
```

### Troubleshooting Rápido

| Problema | Causa Provável | Solução |
|----------|---------------|---------|
| Query lenta em tabela particionada | Partition pruning não funciona | Reescreva filtros sem funções |
| JOIN lento | Redistribute Motion | Use DISTRIBUTED REPLICATED |
| Estimativas erradas no EXPLAIN | Estatísticas desatualizadas | Execute ANALYZE |
| Índice não usado | Query não é seletiva | Remova índice, use seq scan |
| Muitas partições | Granularidade muito fina | Consolide partições antigas |

### Próximos Passos
No **Módulo 4**, você aprenderá sobre carregamento de dados (COPY, External Tables), ETL patterns e integração com ferramentas externas.

---

**Fim do Módulo 3**
