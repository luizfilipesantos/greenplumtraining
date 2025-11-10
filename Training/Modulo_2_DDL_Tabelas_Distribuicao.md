# Módulo 2: DDL - Criação de Tabelas e Estratégias de Distribuição

**Duração Total:** 90-120 minutos  
**Objetivo:** Dominar a criação de tabelas no Greenplum, compreendendo distribuição de dados, orientação de armazenamento e compressão para otimizar performance.

---

## Índice
1. [Lab 2.1: Conceitos de Distribuição de Dados](#lab-21-conceitos-de-distribuição-de-dados-20-25-min)
2. [Lab 2.2: Orientação de Armazenamento (Row vs Column)](#lab-22-orientação-de-armazenamento-row-vs-column-25-30-min)
3. [Lab 2.3: Compressão de Dados](#lab-23-compressão-de-dados-20-25-min)
4. [Lab 2.4: Cenários Práticos e Otimização](#lab-24-cenários-práticos-e-otimização-25-30-min)

---

## Lab 2.1: Conceitos de Distribuição de Dados (20-25 min)

### Objetivos
- Compreender os tipos de distribuição no Greenplum
- Escolher a estratégia adequada de distribuição
- Identificar impactos de performance
- Evitar data skew (desbalanceamento de dados)

### Conceitos Abordados
- **DISTRIBUTED BY (colunas):** Distribuição por hash em colunas específicas
- **DISTRIBUTED RANDOMLY:** Distribuição aleatória
- **DISTRIBUTED REPLICATED:** Tabela replicada em todos os segmentos
- **Data Skew:** Desbalanceamento de dados entre segmentos

### Por que a Distribuição Importa?

No Greenplum, os dados são **fisicamente distribuídos** entre múltiplos segmentos. A escolha da estratégia de distribuição afeta diretamente:

- ✅ **Performance de JOINs:** Evita redistribuição de dados
- ✅ **Balanceamento de carga:** Distribui processamento uniformemente
- ✅ **Uso de recursos:** Otimiza CPU, memória e I/O
- ❌ **Data Skew:** Distribuição inadequada causa gargalos

---

### Exercício 2.1.1: Distribuição por Hash (Padrão)

**Objetivo:** Criar tabelas com distribuição por hash e entender o comportamento padrão

**Cenário:** Tabela de vendas com distribuição pela chave primária

**Passos:**

1. Crie uma tabela com distribuição explícita por hash:
```sql
CREATE TABLE vendas_hash (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor_total NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id);
```

2. Verifique a distribuição criada:
```sql
SELECT 
    c.relname as table_name,
    CASE 
        WHEN policytype = 'p' THEN 'HASH DISTRIBUTED'
        WHEN policytype = 'r' THEN 'REPLICATED'
        ELSE 'RANDOM'
    END AS distribution_type,
    array_to_string(distkey, ', ') AS distribution_columns
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
JOIN gp_distribution_policy p ON p.localoid = c.oid
JOIN pg_attribute a ON a.attrelid = c.oid AND a.attnum = ANY(p.distkey)
WHERE c.relname = 'vendas_hash'
  AND n.nspname = 'public';
```

3. Insira dados de teste:
```sql
INSERT INTO vendas_hash 
SELECT 
    i AS venda_id,
    CURRENT_DATE - (i % 365) AS data_venda,
    (i % 1000) + 1 AS cliente_id,
    (i % 100) + 1 AS produto_id,
    (random() * 10)::INTEGER + 1 AS quantidade,
    (random() * 1000)::NUMERIC(10,2) AS valor_total
FROM generate_series(1, 100000) i;
```

4. Verifique a distribuição dos dados entre segmentos:
```sql
SELECT 
    gp_segment_id,
    COUNT(*) as total_rows,
    pg_size_pretty(pg_relation_size('vendas_hash')) as segment_size
FROM vendas_hash
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

**Análise:**
- Os dados estão balanceados entre os segmentos?
- Qual o percentual de variação entre segmentos?
- Por que escolhemos `venda_id` como coluna de distribuição?

---

### Exercício 2.1.2: Distribuição Aleatória

**Objetivo:** Entender quando usar distribuição aleatória

**Cenário:** Tabela de logs sem chave natural óbvia

**Passos:**

1. Crie tabela com distribuição aleatória:
```sql
CREATE TABLE logs_sistema (
    log_id BIGSERIAL,
    timestamp TIMESTAMP,
    nivel VARCHAR(10),
    mensagem TEXT,
    usuario VARCHAR(50)
)
DISTRIBUTED RANDOMLY;
```

2. Insira dados:
```sql
INSERT INTO logs_sistema (timestamp, nivel, mensagem, usuario)
SELECT 
    CURRENT_TIMESTAMP - (i || ' seconds')::INTERVAL,
    CASE (i % 4)
        WHEN 0 THEN 'INFO'
        WHEN 1 THEN 'WARNING'
        WHEN 2 THEN 'ERROR'
        ELSE 'DEBUG'
    END,
    'Log message ' || i,
    'user' || (i % 50)
FROM generate_series(1, 50000) i;
```

3. Compare a distribuição:
```sql
SELECT 
    gp_segment_id,
    COUNT(*) as total_rows
FROM logs_sistema
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

**Quando usar DISTRIBUTED RANDOMLY:**
- ✅ Tabelas staging/temporárias
- ✅ Sem chave natural adequada
- ✅ Tabelas pequenas sem JOINs frequentes
- ❌ Tabelas com JOINs frequentes (causa redistribuição)

---

### Exercício 2.1.3: Tabelas Replicadas

**Objetivo:** Usar replicação para otimizar JOINs com tabelas pequenas

**Cenário:** Tabelas dimensão pequenas (lookup tables)

**Passos:**

1. Crie tabelas dimensão replicadas:
```sql
-- Tabela de produtos (pequena, usada em muitos JOINs)
CREATE TABLE dim_produtos (
    produto_id INTEGER PRIMARY KEY,
    nome_produto VARCHAR(100),
    categoria VARCHAR(50),
    preco_base NUMERIC(10,2)
)
DISTRIBUTED REPLICATED;

-- Tabela de clientes (pequena)
CREATE TABLE dim_clientes (
    cliente_id INTEGER PRIMARY KEY,
    nome_cliente VARCHAR(100),
    cidade VARCHAR(50),
    estado CHAR(2),
    tipo_cliente VARCHAR(20)
)
DISTRIBUTED REPLICATED;
```

2. Insira dados nas dimensões:
```sql
INSERT INTO dim_produtos
SELECT 
    i,
    'Produto ' || i,
    'Categoria ' || (i % 10),
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100) i;

INSERT INTO dim_clientes
SELECT 
    i,
    'Cliente ' || i,
    'Cidade ' || (i % 50),
    'SP',
    CASE (i % 3)
        WHEN 0 THEN 'VIP'
        WHEN 1 THEN 'Regular'
        ELSE 'Novo'
    END
FROM generate_series(1, 1000) i;
```

3. Verifique que as tabelas estão replicadas em todos os segmentos:
```sql
-- Para tabelas replicadas, use a view gp_distribution_policy
SELECT 
    schemaname,
    tablename,
    CASE 
        WHEN policytype = 'r' THEN 'REPLICATED (em todos segmentos)'
        WHEN policytype = 'p' THEN 'HASH DISTRIBUTED'
        ELSE 'RANDOM'
    END AS distribution_type
FROM pg_tables pt
LEFT JOIN gp_distribution_policy gp ON gp.localoid = (schemaname||'.'||tablename)::regclass
WHERE tablename IN ('dim_produtos', 'dim_clientes');

-- Verifique o número de segmentos onde a tabela existe
SELECT 
    'dim_produtos' as tabela,
    COUNT(DISTINCT content) as num_segmentos_com_dados
FROM gp_segment_configuration
WHERE role = 'p' AND content >= 0;

-- Compare tamanho: tabela replicada vs distribuída
SELECT 
    tablename,
    pg_size_pretty(pg_relation_size(tablename::regclass)) as size_per_segment,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) as total_size
FROM (VALUES ('dim_produtos'), ('dim_clientes')) AS t(tablename);
```

**Entendendo Tabelas Replicadas:**
- Cada segmento tem uma cópia completa da tabela
- `pg_relation_size()` mostra tamanho em um segmento
- `pg_total_relation_size()` mostra tamanho total (n × tamanho)
- Não é possível usar `gp_segment_id` diretamente em tabelas replicadas

4. Teste performance de JOIN (sem redistribuição):
```sql
EXPLAIN ANALYZE
SELECT 
    v.venda_id,
    p.nome_produto,
    c.nome_cliente,
    v.quantidade,
    v.valor_total
FROM vendas_hash v
JOIN dim_produtos p ON v.produto_id = p.produto_id
JOIN dim_clientes c ON v.cliente_id = c.cliente_id
WHERE v.data_venda >= CURRENT_DATE - 30
LIMIT 100;
```

**Análise do EXPLAIN:**
- Há "Redistribute Motion" no plano?
- Como a replicação evita movimento de dados?

**Quando usar DISTRIBUTED REPLICATED:**
- ✅ Tabelas pequenas (< 10MB geralmente)
- ✅ Tabelas dimensão em JOINs frequentes
- ✅ Lookup tables (países, categorias, status)
- ❌ Tabelas grandes (desperdício de espaço)

---

### Exercício 2.1.4: Detectando Data Skew

**Objetivo:** Identificar e resolver desbalanceamento de dados

**Cenário:** Distribuição inadequada causando skew

**Passos:**

1. Crie tabela com distribuição problemática:
```sql
CREATE TABLE vendas_skew (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    loja_id INTEGER,  -- Poucas lojas, má escolha para distribuição
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (loja_id);  -- PROBLEMA: poucas lojas diferentes
```

2. Insira dados desbalanceados:
```sql
INSERT INTO vendas_skew
SELECT 
    i,
    CURRENT_DATE - (i % 365),
    (i % 1000) + 1,
    (i % 5) + 1,  -- Apenas 5 lojas!
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100000) i;
```

3. Detecte o skew:
```sql
-- Análise detalhada de distribuição
SELECT 
    gp_segment_id,
    COUNT(*) as num_rows,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage
FROM vendas_skew
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

4. Use a view gp_toolkit para detectar skew:
```sql
SELECT 
    skcnamespace as schema_name,
    skcrelname as table_name,
    skccoeff as skew_coefficient  -- > 0.1 indica problema
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'vendas_skew';
```

5. Corrija a distribuição:
```sql
-- Recrie a tabela com distribuição adequada
CREATE TABLE vendas_corrigida (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    loja_id INTEGER,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id, cliente_id);  -- Múltiplas colunas

-- Migre os dados
INSERT INTO vendas_corrigida SELECT * FROM vendas_skew;

-- Verifique a melhoria
SELECT 
    gp_segment_id,
    COUNT(*) as num_rows,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage
FROM vendas_corrigida
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

**Regras para Escolher Colunas de Distribuição:**
1. ✅ Alta cardinalidade (muitos valores distintos)
2. ✅ Usada em JOINs frequentes
3. ✅ Distribuição uniforme de valores
4. ❌ Evitar colunas com poucos valores distintos
5. ❌ Evitar colunas com valores NULL frequentes

---

## Lab 2.2: Orientação de Armazenamento (Row vs Column) (25-30 min)

### Objetivos
- Compreender armazenamento orientado a linha vs coluna
- Escolher orientação adequada ao caso de uso
- Medir impacto de performance

### Conceitos Abordados
- **Row-Oriented (Heap):** Armazenamento tradicional, linha por linha
- **Column-Oriented (AO/AOCO):** Armazenamento colunar, otimizado para analytics
- **Append-Optimized (AO):** Tabelas de inserção em massa
- **Compressão:** Mais efetiva em storage colunar

---

### Exercício 2.2.1: Tabela Orientada a Linha (Heap - Padrão)

**Objetivo:** Entender o comportamento padrão do armazenamento

**Cenário:** Tabela transacional com UPDATEs frequentes

**Passos:**

1. Crie tabela heap (orientada a linha):
```sql
CREATE TABLE transacoes_heap (
    transacao_id BIGSERIAL,
    conta_id INTEGER,
    tipo_transacao VARCHAR(20),
    valor NUMERIC(15,2),
    data_transacao TIMESTAMP,
    status VARCHAR(20),
    descricao TEXT
)
DISTRIBUTED BY (transacao_id);
-- Sem WITH, o padrão é HEAP (row-oriented)
```

2. Verifique o tipo de armazenamento:
```sql
-- No Greenplum 7.x, usar pg_class.relam para identificar tipo de storage
SELECT 
    c.relname AS table_name,
    CASE 
        WHEN am.amname = 'heap' THEN 'HEAP (Row-Oriented)'
        WHEN am.amname = 'ao_row' THEN 'Append-Optimized Row'
        WHEN am.amname = 'ao_column' THEN 'Append-Optimized Column'
        ELSE 'Other: ' || COALESCE(am.amname, 'unknown')
    END AS storage_type,
    COALESCE(
        (SELECT option_value 
         FROM pg_options_to_table(c.reloptions) 
         WHERE option_name = 'compresstype'), 
        'none'
    ) AS compression_type
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_am am ON c.relam = am.oid
WHERE c.relname = 'transacoes_heap'
  AND n.nspname = 'public';
```

**Entendendo a Query:**
- **pg_am (Access Method):** Define o método de acesso/storage da tabela
  - `heap` = tabela tradicional row-oriented
  - `ao_row` = append-optimized row-oriented
  - `ao_column` = append-optimized column-oriented
- **reloptions:** Opções de armazenamento (compressão, etc)
- **pg_options_to_table:** Função que parseia as opções de reloptions

3. Insira dados e meça o tamanho:
```sql
INSERT INTO transacoes_heap (conta_id, tipo_transacao, valor, data_transacao, status, descricao)
SELECT 
    (random() * 10000)::INTEGER,
    CASE (random() * 4)::INTEGER
        WHEN 0 THEN 'DEPOSITO'
        WHEN 1 THEN 'SAQUE'
        WHEN 2 THEN 'TRANSFERENCIA'
        ELSE 'PAGAMENTO'
    END,
    (random() * 10000)::NUMERIC(15,2),
    CURRENT_TIMESTAMP - (random() * 365)::INTEGER * INTERVAL '1 day',
    CASE (random() * 3)::INTEGER
        WHEN 0 THEN 'CONCLUIDO'
        WHEN 1 THEN 'PENDENTE'
        ELSE 'CANCELADO'
    END,
    'Transacao ID ' || i
FROM generate_series(1, 1000000) i;

-- Tamanho da tabela
SELECT pg_size_pretty(pg_total_relation_size('transacoes_heap'));
```

4. Teste UPDATE (eficiente em HEAP):
```sql
EXPLAIN ANALYZE
UPDATE transacoes_heap 
SET status = 'PROCESSADO' 
WHERE transacao_id BETWEEN 1000 AND 2000;
```

**Quando usar HEAP (Row-Oriented):**
- ✅ Tabelas com UPDATEs e DELETEs frequentes
- ✅ Queries que acessam maioria das colunas
- ✅ Tabelas transacionais (OLTP)
- ✅ Tabelas pequenas
- ❌ Análises que leem poucas colunas de muitas linhas

---

### Exercício 2.2.2: Tabela Append-Optimized Row (AO)

**Objetivo:** Usar tabelas append-optimized para carga em massa

**Cenário:** Tabela de fatos com inserções batch, sem updates

**Passos:**

1. Crie tabela AO row-oriented:
```sql
CREATE TABLE fatos_vendas_ao (
    venda_id BIGINT,
    data_venda DATE,
    loja_id INTEGER,
    produto_id INTEGER,
    cliente_id INTEGER,
    quantidade INTEGER,
    valor_unitario NUMERIC(10,2),
    valor_total NUMERIC(12,2),
    custo_produto NUMERIC(10,2),
    margem NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=row)
DISTRIBUTED BY (venda_id);
```

2. Insira dados e compare tamanho:
```sql
INSERT INTO fatos_vendas_ao
SELECT 
    i,
    CURRENT_DATE - (random() * 365)::INTEGER,
    (random() * 100)::INTEGER + 1,
    (random() * 1000)::INTEGER + 1,
    (random() * 10000)::INTEGER + 1,
    (random() * 10)::INTEGER + 1,
    (random() * 1000)::NUMERIC(10,2),
    0,  -- será calculado
    (random() * 500)::NUMERIC(10,2),
    0   -- será calculado
FROM generate_series(1, 1000000) i;

-- Atualiza valores calculados (LENTO em AO!)
UPDATE fatos_vendas_ao 
SET valor_total = quantidade * valor_unitario,
    margem = (quantidade * valor_unitario) - (quantidade * custo_produto);

-- Tamanho
SELECT pg_size_pretty(pg_total_relation_size('fatos_vendas_ao'));
```

3. Teste SELECT completo (acessa todas colunas):
```sql
EXPLAIN ANALYZE
SELECT * FROM fatos_vendas_ao 
WHERE data_venda = CURRENT_DATE - 30
LIMIT 1000;
```

**Características do AO Row:**
- ✅ Melhor compressão que HEAP
- ✅ Inserções batch eficientes
- ✅ Queries que leem linha completa
- ❌ UPDATEs muito lentos (marca deleção + insere nova linha)
- ❌ VACUUMs necessários após updates/deletes

---

### Exercício 2.2.3: Tabela Append-Optimized Column (AOCO)

**Objetivo:** Otimizar queries analíticas com storage colunar

**Cenário:** Data warehouse, análises agregadas

**Passos:**

1. Crie tabela AOCO (orientada a coluna):
```sql
CREATE TABLE fatos_vendas_aoco (
    venda_id BIGINT,
    data_venda DATE,
    loja_id INTEGER,
    produto_id INTEGER,
    cliente_id INTEGER,
    quantidade INTEGER,
    valor_unitario NUMERIC(10,2),
    valor_total NUMERIC(12,2),
    custo_produto NUMERIC(10,2),
    margem NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (venda_id);
```

2. Insira os mesmos dados:
```sql
INSERT INTO fatos_vendas_aoco
SELECT * FROM fatos_vendas_ao;

-- Compare tamanhos
SELECT 
    'AO Row' as table_type,
    pg_size_pretty(pg_total_relation_size('fatos_vendas_ao')) as size
UNION ALL
SELECT 
    'AOCO Column',
    pg_size_pretty(pg_total_relation_size('fatos_vendas_aoco'));
```

3. Teste query analítica (lê poucas colunas):
```sql
-- Query em tabela AO Row
EXPLAIN ANALYZE
SELECT 
    data_venda,
    SUM(valor_total) as total_vendas,
    AVG(margem) as margem_media,
    COUNT(*) as qtd_vendas
FROM fatos_vendas_ao
WHERE data_venda >= CURRENT_DATE - 90
GROUP BY data_venda
ORDER BY data_venda;

-- Mesma query em tabela AOCO
EXPLAIN ANALYZE
SELECT 
    data_venda,
    SUM(valor_total) as total_vendas,
    AVG(margem) as margem_media,
    COUNT(*) as qtd_vendas
FROM fatos_vendas_aoco
WHERE data_venda >= CURRENT_DATE - 90
GROUP BY data_venda
ORDER BY data_venda;
```

4. Compare I/O nas queries:
```sql
-- Análise de colunas acessadas em AOCO
SELECT 
    a.attname as column_name,
    pg_size_pretty(pg_column_size(a.attname::text)) as column_size
FROM pg_attribute a
JOIN pg_class c ON a.attrelid = c.oid
WHERE c.relname = 'fatos_vendas_aoco'
  AND a.attnum > 0
ORDER BY a.attnum;
```

**Quando usar AOCO (Column-Oriented):**
- ✅ Data Warehouse / Analytics
- ✅ Queries que leem poucas colunas
- ✅ Agregações e scans grandes
- ✅ Alta taxa de compressão
- ✅ Tabelas largas (muitas colunas)
- ❌ Queries que precisam de linha completa
- ❌ UPDATEs frequentes
- ❌ Tabelas transacionais

---

### Exercício 2.2.4: Comparação de Performance

**Objetivo:** Medir diferenças reais de performance

**Passos:**

1. Crie versões das três orientações:
```sql
-- Já temos: transacoes_heap, fatos_vendas_ao, fatos_vendas_aoco

-- Crie versão HEAP da tabela de fatos
CREATE TABLE fatos_vendas_heap AS 
SELECT * FROM fatos_vendas_aoco
DISTRIBUTED BY (venda_id);
```

2. Teste 1 - Query seletiva (poucas colunas):
```sql
-- HEAP
EXPLAIN (ANALYZE, BUFFERS)
SELECT loja_id, SUM(valor_total) 
FROM fatos_vendas_heap 
WHERE data_venda >= CURRENT_DATE - 30
GROUP BY loja_id;

-- AO Row
EXPLAIN (ANALYZE, BUFFERS)
SELECT loja_id, SUM(valor_total) 
FROM fatos_vendas_ao 
WHERE data_venda >= CURRENT_DATE - 30
GROUP BY loja_id;

-- AOCO Column
EXPLAIN (ANALYZE, BUFFERS)
SELECT loja_id, SUM(valor_total) 
FROM fatos_vendas_aoco 
WHERE data_venda >= CURRENT_DATE - 30
GROUP BY loja_id;
```

3. Teste 2 - Query que acessa todas as colunas:
```sql
-- Teste em todas as três tabelas
EXPLAIN ANALYZE
SELECT * FROM fatos_vendas_heap WHERE venda_id < 1000;

EXPLAIN ANALYZE
SELECT * FROM fatos_vendas_ao WHERE venda_id < 1000;

EXPLAIN ANALYZE
SELECT * FROM fatos_vendas_aoco WHERE venda_id < 1000;
```

**Métricas para Comparar:**
- Tempo de execução (Execution Time)
- Quantidade de dados lidos (rows/bytes)
- Uso de memória (Peak Memory)

---

## Lab 2.3: Compressão de Dados (20-25 min)

### Objetivos
- Aplicar diferentes algoritmos de compressão
- Balancear compressão vs performance
- Compressão por coluna em AOCO

### Conceitos Abordados
- **Algoritmos:** zlib, zstd, rle_type, none
- **Compression level:** 1-9 (trade-off espaço/CPU)
- **Column-level compression:** Diferentes algoritmos por coluna

---

### Exercício 2.3.1: Compressão em Tabelas AO

**Objetivo:** Aplicar compressão e medir ganhos

**Passos:**

1. Crie tabelas com diferentes compressões:
```sql
-- Sem compressão
CREATE TABLE vendas_nocomp (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor NUMERIC(12,2),
    descricao TEXT
)
WITH (appendoptimized=true, orientation=row, compresstype=none)
DISTRIBUTED BY (venda_id);

-- Compressão zlib (padrão, balanceado)
CREATE TABLE vendas_zlib (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor NUMERIC(12,2),
    descricao TEXT
)
WITH (appendoptimized=true, orientation=row, compresstype=zlib, compresslevel=5)
DISTRIBUTED BY (venda_id);

-- Compressão zstd (mais recente, melhor em GP7+)
CREATE TABLE vendas_zstd (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor NUMERIC(12,2),
    descricao TEXT
)
WITH (appendoptimized=true, orientation=row, compresstype=zstd, compresslevel=5)
DISTRIBUTED BY (venda_id);
```

2. Insira mesmos dados em todas:
```sql
INSERT INTO vendas_nocomp
SELECT 
    i,
    CURRENT_DATE - (i % 365),
    (random() * 10000)::INTEGER,
    (random() * 1000)::INTEGER,
    (random() * 1000)::NUMERIC(12,2),
    'Venda produto ' || (i % 100) || ' quantidade ' || (i % 10) || ' descricao repetida para aumentar compressao'
FROM generate_series(1, 500000) i;

INSERT INTO vendas_zlib SELECT * FROM vendas_nocomp;
INSERT INTO vendas_zstd SELECT * FROM vendas_nocomp;
```

3. Compare tamanhos e taxa de compressão:
```sql
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) as total_size,
    pg_total_relation_size(tablename::regclass) as size_bytes,
    ROUND(100.0 - (pg_total_relation_size(tablename::regclass) * 100.0 / 
          pg_total_relation_size('vendas_nocomp')), 2) as compression_percent
FROM (
    VALUES ('vendas_nocomp'), ('vendas_zlib'), ('vendas_zstd')
) AS t(tablename)
ORDER BY size_bytes DESC;
```

4. Compare performance de leitura:
```sql
-- Sem compressão
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor) FROM vendas_nocomp WHERE data_venda >= CURRENT_DATE - 30;

-- Com zlib
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor) FROM vendas_zlib WHERE data_venda >= CURRENT_DATE - 30;

-- Com zstd
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor) FROM vendas_zstd WHERE data_venda >= CURRENT_DATE - 30;
```

**Análise:**
- Qual teve melhor taxa de compressão?
- O overhead de CPU afetou significativamente a query?
- Quando vale a pena usar compressão máxima (level 9)?

---

### Exercício 2.3.2: Compressão por Coluna (AOCO)

**Objetivo:** Otimizar compressão escolhendo algoritmo por coluna

**Cenário:** Tabela com colunas de diferentes tipos e padrões

**Passos:**

1. Crie tabela AOCO com compressão por coluna:
```sql
CREATE TABLE vendas_detalhadas (
    venda_id BIGINT ENCODING (compresstype=zstd, compresslevel=7),  -- IDs: alta compressão
    data_venda DATE ENCODING (compresstype=rle_type),  -- Datas: RLE eficiente
    hora_venda TIME ENCODING (compresstype=zlib),
    cliente_id INTEGER ENCODING (compresstype=zstd, compresslevel=5),
    cliente_nome VARCHAR(100) ENCODING (compresstype=zlib, compresslevel=6),
    produto_id INTEGER ENCODING (compresstype=zstd, compresslevel=5),
    produto_nome VARCHAR(200) ENCODING (compresstype=zlib, compresslevel=6),
    categoria VARCHAR(50) ENCODING (compresstype=rle_type),  -- Poucos valores: RLE
    quantidade INTEGER ENCODING (compresstype=zlib),
    valor_unitario NUMERIC(10,2) ENCODING (compresstype=zlib),
    valor_total NUMERIC(12,2) ENCODING (compresstype=zlib),
    status VARCHAR(20) ENCODING (compresstype=rle_type),  -- Status limitados: RLE
    observacoes TEXT ENCODING (compresstype=zlib, compresslevel=7)  -- Texto: alta compressão
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (venda_id);
```

2. Insira dados variados:
```sql
INSERT INTO vendas_detalhadas
SELECT 
    i,
    CURRENT_DATE - (i % 365),
    ('08:00:00'::TIME + (i % 86400 || ' seconds')::INTERVAL)::TIME,
    (i % 5000) + 1,
    'Cliente Nome ' || ((i % 5000) + 1),
    (i % 200) + 1,
    'Produto Nome ' || ((i % 200) + 1),
    'Categoria ' || (i % 10),  -- Apenas 10 categorias
    (random() * 10)::INTEGER + 1,
    (random() * 500)::NUMERIC(10,2),
    0,
    CASE (i % 5)
        WHEN 0 THEN 'CONCLUIDO'
        WHEN 1 THEN 'PENDENTE'
        WHEN 2 THEN 'CANCELADO'
        WHEN 3 THEN 'PROCESSANDO'
        ELSE 'AGUARDANDO'
    END,
    'Observacao da venda numero ' || i || ' com detalhes adicionais para aumentar volume de texto'
FROM generate_series(1, 1000000) i;

UPDATE vendas_detalhadas SET valor_total = quantidade * valor_unitario;
```

3. Analise compressão por coluna:
```sql
SELECT 
    a.attname as column_name,
    t.typname as data_type,
    pg_size_pretty(pg_column_size(a.attname::text)) as column_size,
    co.compresstype as compression_type,
    co.compresslevel as compression_level
FROM pg_attribute a
JOIN pg_class c ON a.attrelid = c.oid
JOIN pg_type t ON a.atttypid = t.oid
LEFT JOIN pg_attribute_encoding co ON co.attrelid = a.attrelid AND co.attnum = a.attnum
WHERE c.relname = 'vendas_detalhadas'
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

4. Crie versão sem otimização para comparar:
```sql
-- Versão com compressão uniforme
CREATE TABLE vendas_detalhadas_simples (
    venda_id BIGINT,
    data_venda DATE,
    hora_venda TIME,
    cliente_id INTEGER,
    cliente_nome VARCHAR(100),
    produto_id INTEGER,
    produto_nome VARCHAR(200),
    categoria VARCHAR(50),
    quantidade INTEGER,
    valor_unitario NUMERIC(10,2),
    valor_total NUMERIC(12,2),
    status VARCHAR(20),
    observacoes TEXT
)
WITH (appendoptimized=true, orientation=column, compresstype=zlib, compresslevel=5)
DISTRIBUTED BY (venda_id);

INSERT INTO vendas_detalhadas_simples SELECT * FROM vendas_detalhadas;

-- Compare tamanhos
SELECT 
    'Otimizada por coluna' as version,
    pg_size_pretty(pg_total_relation_size('vendas_detalhadas')) as size
UNION ALL
SELECT 
    'Compressao uniforme',
    pg_size_pretty(pg_total_relation_size('vendas_detalhadas_simples'));
```

**Estratégias de Compressão por Tipo de Coluna:**
- **IDs numéricos:** zstd level 5-7
- **Datas/Timestamps:** rle_type (valores repetidos)
- **Status/Categorias:** rle_type (cardinalidade baixa)
- **Texto/VARCHAR:** zlib level 5-7
- **Números/decimais:** zlib level 3-5
- **Colunas únicas:** compresstype=none (não comprimem bem)

---

### Exercício 2.3.3: Impact Trade-offs

**Objetivo:** Entender trade-offs entre compressão e performance

**Passos:**

1. Crie tabelas com diferentes níveis de compressão:
```sql
-- Compressão baixa (rápida)
CREATE TABLE metrics_comp1 (
    id BIGINT,
    timestamp TIMESTAMP,
    metric_value DOUBLE PRECISION,
    metadata TEXT
)
WITH (appendoptimized=true, orientation=column, compresstype=zlib, compresslevel=1)
DISTRIBUTED BY (id);

-- Compressão alta (máxima)
CREATE TABLE metrics_comp9 (
    id BIGINT,
    timestamp TIMESTAMP,
    metric_value DOUBLE PRECISION,
    metadata TEXT
)
WITH (appendoptimized=true, orientation=column, compresstype=zlib, compresslevel=9)
DISTRIBUTED BY (id);
```

2. Meça tempo de INSERT:
```sql
-- Level 1
\timing on
INSERT INTO metrics_comp1
SELECT 
    i,
    CURRENT_TIMESTAMP - (i || ' seconds')::INTERVAL,
    random() * 1000,
    'Metadata texto repetido ' || (i % 100)
FROM generate_series(1, 1000000) i;
\timing off

-- Level 9
\timing on
INSERT INTO metrics_comp9
SELECT * FROM metrics_comp1;
\timing off
```

3. Compare tamanho vs tempo:
```sql
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) as size,
    'Check insert time from output above' as insert_time
FROM (
    VALUES ('metrics_comp1'), ('metrics_comp9')
) AS t(tablename);
```

**Recomendações:**
- **Level 1-3:** Dados temporários, alta taxa de inserção
- **Level 5-6:** Balanceamento geral (recomendado)
- **Level 7-9:** Dados arquivados, leitura < escrita

---

## Lab 2.4: Cenários Práticos e Otimização (25-30 min)

### Objetivos
- Aplicar conhecimentos em cenários reais
- Tomar decisões de design
- Otimizar tabelas existentes

---

### Exercício 2.4.1: Cenário 1 - E-commerce (Fatos de Vendas)

**Contexto:**
- 10 milhões de vendas/mês
- Queries analíticas: vendas por período, produto, região
- Inserção batch diária
- Sem updates após inserção
- Retenção: 5 anos

**Tarefa:** Projete a tabela ideal

**Solução Proposta:**
```sql
CREATE TABLE fato_vendas_ecommerce (
    venda_id BIGINT,
    data_venda DATE,
    data_entrega DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    loja_id SMALLINT,
    regiao_id SMALLINT,
    quantidade SMALLINT,
    valor_unitario NUMERIC(10,2),
    valor_total NUMERIC(12,2),
    desconto NUMERIC(10,2),
    frete NUMERIC(10,2),
    imposto NUMERIC(10,2),
    custo_produto NUMERIC(10,2),
    margem_lucro NUMERIC(12,2),
    forma_pagamento VARCHAR(30),
    status_pedido VARCHAR(20),
    canal_venda VARCHAR(20)
)
WITH (
    appendoptimized=true,
    orientation=column,
    compresstype=zstd,
    compresslevel=5
)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda)
(
    START ('2020-01-01'::DATE) END ('2025-12-31'::DATE) EVERY (INTERVAL '1 month')
);

-- Otimizações específicas de coluna
ALTER TABLE fato_vendas_ecommerce 
    ALTER COLUMN data_venda SET ENCODING (compresstype=rle_type),
    ALTER COLUMN regiao_id SET ENCODING (compresstype=rle_type),
    ALTER COLUMN loja_id SET ENCODING (compresstype=rle_type),
    ALTER COLUMN forma_pagamento SET ENCODING (compresstype=rle_type),
    ALTER COLUMN status_pedido SET ENCODING (compresstype=rle_type),
    ALTER COLUMN canal_venda SET ENCODING (compresstype=rle_type);
```

**Justificativas:**
- ✅ **AOCO:** Queries analíticas, lê poucas colunas
- ✅ **DISTRIBUTED BY (venda_id):** Alta cardinalidade, única
- ✅ **zstd:** Boa compressão, performance GP7+
- ✅ **RLE em colunas repetitivas:** Status, categorias, datas
- ✅ **PARTITION:** Gestão eficiente, drop de partições antigas

**Teste a solução:**
```sql
-- Insira dados de exemplo
INSERT INTO fato_vendas_ecommerce
SELECT 
    i,
    CURRENT_DATE - (random() * 1825)::INTEGER,
    CURRENT_DATE - (random() * 1800)::INTEGER,
    (random() * 100000)::INTEGER,
    (random() * 5000)::INTEGER,
    (random() * 50)::SMALLINT + 1,
    (random() * 10)::SMALLINT + 1,
    (random() * 5)::SMALLINT + 1,
    (random() * 500)::NUMERIC(10,2),
    0,
    (random() * 50)::NUMERIC(10,2),
    (random() * 30)::NUMERIC(10,2),
    0,
    (random() * 200)::NUMERIC(10,2),
    0,
    CASE (random() * 4)::INTEGER
        WHEN 0 THEN 'CARTAO_CREDITO'
        WHEN 1 THEN 'CARTAO_DEBITO'
        WHEN 2 THEN 'PIX'
        ELSE 'BOLETO'
    END,
    CASE (random() * 5)::INTEGER
        WHEN 0 THEN 'ENTREGUE'
        WHEN 1 THEN 'EM_TRANSITO'
        WHEN 2 THEN 'PROCESSANDO'
        WHEN 3 THEN 'CANCELADO'
        ELSE 'PENDENTE'
    END,
    CASE (random() * 3)::INTEGER
        WHEN 0 THEN 'WEBSITE'
        WHEN 1 THEN 'MOBILE_APP'
        ELSE 'LOJA_FISICA'
    END
FROM generate_series(1, 5000000) i;

-- Teste query analítica
EXPLAIN ANALYZE
SELECT 
    DATE_TRUNC('month', data_venda) as mes,
    regiao_id,
    COUNT(*) as total_vendas,
    SUM(valor_total) as receita,
    AVG(margem_lucro) as margem_media
FROM fato_vendas_ecommerce
WHERE data_venda >= CURRENT_DATE - 365
GROUP BY 1, 2
ORDER BY 1, 2;
```

---

### Exercício 2.4.2: Cenário 2 - Sistema Transacional (Pedidos Ativos)

**Contexto:**
- Tabela de pedidos ativos
- UPDATEs frequentes (status, valores)
- Queries buscam pedido completo
- Volume: 50k pedidos ativos simultâneos
- Pedidos arquivados após conclusão

**Tarefa:** Projete a tabela ideal

**Solução Proposta:**
```sql
CREATE TABLE pedidos_ativos (
    pedido_id BIGSERIAL PRIMARY KEY,
    cliente_id INTEGER NOT NULL,
    data_pedido TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    data_atualizacao TIMESTAMP,
    status VARCHAR(30) NOT NULL,
    valor_total NUMERIC(12,2),
    observacoes TEXT,
    vendedor_id INTEGER,
    prioridade SMALLINT,
    CONSTRAINT ck_status CHECK (status IN ('NOVO', 'PROCESSANDO', 'AGUARDANDO_PAGAMENTO', 'PAGO', 'EM_SEPARACAO'))
)
DISTRIBUTED BY (pedido_id);
-- HEAP (padrão), ideal para UPDATEs frequentes

-- Índices para queries comuns
CREATE INDEX idx_pedidos_cliente ON pedidos_ativos(cliente_id);
CREATE INDEX idx_pedidos_status ON pedidos_ativos(status);
CREATE INDEX idx_pedidos_data ON pedidos_ativos(data_pedido);

-- Tabela para itens (1:N)
CREATE TABLE itens_pedido (
    item_id BIGSERIAL,
    pedido_id BIGINT NOT NULL,
    produto_id INTEGER NOT NULL,
    quantidade SMALLINT NOT NULL,
    valor_unitario NUMERIC(10,2) NOT NULL,
    desconto NUMERIC(10,2) DEFAULT 0,
    PRIMARY KEY (pedido_id, item_id)
)
DISTRIBUTED BY (pedido_id);  -- Co-located com pedidos
```

**Justificativas:**
- ✅ **HEAP:** UPDATEs eficientes
- ✅ **Tabela pequena:** 50k linhas, HEAP é adequado
- ✅ **DISTRIBUTED BY (pedido_id):** Co-location de pedido + itens
- ✅ **Índices:** Aceleram queries por cliente/status
- ❌ **Sem compressão:** HEAP não comprime, mas é pequena

**Arquivamento periódico:**
```sql
-- Tabela de arquivo (AOCO, alta compressão)
CREATE TABLE pedidos_historico (
    pedido_id BIGINT,
    cliente_id INTEGER,
    data_pedido TIMESTAMP,
    data_conclusao TIMESTAMP,
    status_final VARCHAR(30),
    valor_total NUMERIC(12,2),
    observacoes TEXT,
    vendedor_id INTEGER
)
WITH (appendoptimized=true, orientation=column, compresstype=zstd, compresslevel=7)
DISTRIBUTED BY (pedido_id)
PARTITION BY RANGE (data_conclusao)
(
    START ('2020-01-01'::DATE) END ('2025-12-31'::DATE) EVERY (INTERVAL '1 quarter')
);

-- Processo de arquivamento (executar periodicamente)
INSERT INTO pedidos_historico
SELECT 
    pedido_id,
    cliente_id,
    data_pedido,
    data_atualizacao as data_conclusao,
    status as status_final,
    valor_total,
    observacoes,
    vendedor_id
FROM pedidos_ativos
WHERE status IN ('CONCLUIDO', 'CANCELADO')
  AND data_atualizacao < CURRENT_DATE - 30;

DELETE FROM pedidos_ativos
WHERE status IN ('CONCLUIDO', 'CANCELADO')
  AND data_atualizacao < CURRENT_DATE - 30;
```

---

### Exercício 2.4.3: Cenário 3 - IoT / Telemetria

**Contexto:**
- 1000 sensores enviando dados a cada 10 segundos
- 8.6 milhões de leituras/dia
- Queries: agregações por sensor, período, tipo
- Sem updates, apenas inserções
- Retenção detalhada: 90 dias, agregada: 2 anos

**Tarefa:** Projete o modelo completo

**Solução Proposta:**
```sql
-- Tabela de leituras brutas (detalhada, curto prazo)
CREATE TABLE iot_leituras_raw (
    sensor_id INTEGER,
    timestamp TIMESTAMP,
    tipo_metrica VARCHAR(50),
    valor DOUBLE PRECISION,
    unidade VARCHAR(20),
    qualidade SMALLINT,  -- 0-100
    metadata JSONB
)
WITH (
    appendoptimized=true,
    orientation=column,
    compresstype=zstd,
    compresslevel=3  -- Baixo: alta taxa inserção
)
DISTRIBUTED RANDOMLY  -- Alta taxa inserção, distribuição uniforme
PARTITION BY RANGE (timestamp)
(
    START (CURRENT_DATE - 90) END (CURRENT_DATE + 1) EVERY (INTERVAL '1 day')
);

-- Encoding otimizado
ALTER TABLE iot_leituras_raw
    ALTER COLUMN timestamp SET ENCODING (compresstype=rle_type),
    ALTER COLUMN tipo_metrica SET ENCODING (compresstype=rle_type),
    ALTER COLUMN unidade SET ENCODING (compresstype=rle_type);

-- Tabela agregada (longo prazo)
CREATE TABLE iot_leituras_agregadas (
    sensor_id INTEGER,
    data DATE,
    hora INTEGER,  -- 0-23
    tipo_metrica VARCHAR(50),
    media DOUBLE PRECISION,
    minimo DOUBLE PRECISION,
    maximo DOUBLE PRECISION,
    desvio_padrao DOUBLE PRECISION,
    total_leituras INTEGER,
    leituras_validas INTEGER
)
WITH (
    appendoptimized=true,
    orientation=column,
    compresstype=zstd,
    compresslevel=7  -- Alto: dados históricos
)
DISTRIBUTED BY (sensor_id, data)
PARTITION BY RANGE (data)
(
    START ('2023-01-01'::DATE) END ('2025-12-31'::DATE) EVERY (INTERVAL '1 month')
);

-- Processo de agregação (executar hourly via cron)
CREATE OR REPLACE FUNCTION agregar_iot_leituras()
RETURNS void AS $$
BEGIN
    INSERT INTO iot_leituras_agregadas
    SELECT 
        sensor_id,
        timestamp::DATE as data,
        EXTRACT(HOUR FROM timestamp) as hora,
        tipo_metrica,
        AVG(valor) as media,
        MIN(valor) as minimo,
        MAX(valor) as maximo,
        STDDEV(valor) as desvio_padrao,
        COUNT(*) as total_leituras,
        COUNT(*) FILTER (WHERE qualidade >= 80) as leituras_validas
    FROM iot_leituras_raw
    WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '2 hours'
      AND timestamp < CURRENT_TIMESTAMP - INTERVAL '1 hour'
    GROUP BY sensor_id, data, hora, tipo_metrica
    ON CONFLICT DO NOTHING;
END;
$$ LANGUAGE plpgsql;
```

**Estratégia de retenção:**
```sql
-- Drop partições antigas (executar diariamente)
-- Raw: manter últimos 90 dias
ALTER TABLE iot_leituras_raw 
DROP PARTITION FOR (CURRENT_DATE - 91);

-- Agregada: manter 2 anos (drop após 25 meses)
ALTER TABLE iot_leituras_agregadas
DROP PARTITION FOR (CURRENT_DATE - INTERVAL '25 months');
```

---

### Exercício 2.4.4: Dimensões vs Fatos - Colocação

**Objetivo:** Otimizar JOINs com co-location

**Cenário:** Fato de vendas + dimensões

**Solução:**
```sql
-- Dimensão pequena: REPLICATED
CREATE TABLE dim_produto_replicated (
    produto_id INTEGER PRIMARY KEY,
    nome VARCHAR(200),
    categoria VARCHAR(100),
    marca VARCHAR(100),
    preco_lista NUMERIC(10,2)
)
DISTRIBUTED REPLICATED;

-- Dimensão média: mesma distribuição do fato
CREATE TABLE dim_cliente_colocated (
    cliente_id INTEGER PRIMARY KEY,
    nome VARCHAR(200),
    cpf VARCHAR(14),
    data_nascimento DATE,
    cidade VARCHAR(100),
    estado CHAR(2)
)
DISTRIBUTED BY (cliente_id);  -- Mesmo que será usado em fato_vendas

-- Fato distribuído por chave que faz JOIN
CREATE TABLE fato_vendas_otimizado (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,  -- JOIN com dim_cliente_colocated
    produto_id INTEGER,  -- JOIN com dim_produto_replicated
    quantidade INTEGER,
    valor_total NUMERIC(12,2)
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (cliente_id);  -- Co-located com dim_cliente!

-- Teste JOIN otimizado
EXPLAIN
SELECT 
    v.venda_id,
    c.nome as cliente,
    p.nome as produto,
    v.valor_total
FROM fato_vendas_otimizado v
JOIN dim_cliente_colocated c ON v.cliente_id = c.cliente_id  -- Co-located: SEM motion
JOIN dim_produto_replicated p ON v.produto_id = p.produto_id  -- Replicated: SEM motion
WHERE v.data_venda >= CURRENT_DATE - 30;
```

**Análise do plano:**
- ✅ Sem "Redistribute Motion" (co-location funcionou)
- ✅ Processamento paralelo em todos segmentos
- ✅ Menor uso de rede e memória

---

## Exercício Integrador: Redesign de Tabela

**Objetivo:** Aplicar todos os conceitos aprendidos

**Cenário:** Você recebeu esta tabela mal projetada:

```sql
-- Tabela problemática
CREATE TABLE vendas_problematica (
    id SERIAL,
    data VARCHAR(20),  -- ❌ Deveria ser DATE
    cliente TEXT,
    cpf VARCHAR(14),
    produto TEXT,
    qtd INTEGER,
    preco TEXT,  -- ❌ Deveria ser NUMERIC
    total TEXT,  -- ❌ Deveria ser NUMERIC
    vendedor VARCHAR(100),
    loja VARCHAR(100),
    obs TEXT
)
DISTRIBUTED BY (id);  -- ❌ SERIAL causa skew!
-- ❌ HEAP com 100M+ linhas
-- ❌ Sem compressão
-- ❌ Tipos inadequados
```

**Tarefa:** Redesenhe completamente:

**Solução Completa:**
```sql
-- 1. Normalizar: separar dimensões
CREATE TABLE dim_clientes_novo (
    cliente_id SERIAL PRIMARY KEY,
    nome VARCHAR(200),
    cpf VARCHAR(14) UNIQUE,
    data_cadastro TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
DISTRIBUTED REPLICATED;  -- Pequena, muito usada em JOINs

CREATE TABLE dim_produtos_novo (
    produto_id SERIAL PRIMARY KEY,
    nome VARCHAR(200),
    categoria VARCHAR(100),
    preco_base NUMERIC(10,2)
)
DISTRIBUTED REPLICATED;

CREATE TABLE dim_lojas_novo (
    loja_id SERIAL PRIMARY KEY,
    nome VARCHAR(100),
    cidade VARCHAR(100),
    estado CHAR(2)
)
DISTRIBUTED REPLICATED;

CREATE TABLE dim_vendedores_novo (
    vendedor_id SERIAL PRIMARY KEY,
    nome VARCHAR(100),
    loja_id INTEGER
)
DISTRIBUTED REPLICATED;

-- 2. Tabela fato otimizada
CREATE TABLE fato_vendas_novo (
    venda_id BIGINT,  -- Gerado externamente, não SERIAL
    data_venda DATE NOT NULL,
    cliente_id INTEGER NOT NULL,
    produto_id INTEGER NOT NULL,
    vendedor_id INTEGER NOT NULL,
    loja_id INTEGER NOT NULL,
    quantidade INTEGER NOT NULL CHECK (quantidade > 0),
    valor_unitario NUMERIC(10,2) NOT NULL,
    valor_total NUMERIC(12,2) NOT NULL,
    observacoes TEXT
)
WITH (
    appendoptimized=true,
    orientation=column,
    compresstype=zstd,
    compresslevel=5
)
DISTRIBUTED BY (venda_id)  -- Alta cardinalidade
PARTITION BY RANGE (data_venda)
(
    START ('2020-01-01'::DATE) END ('2026-01-01'::DATE) EVERY (INTERVAL '1 month')
);

-- 3. Otimizar encoding por coluna
ALTER TABLE fato_vendas_novo
    ALTER COLUMN data_venda SET ENCODING (compresstype=rle_type),
    ALTER COLUMN produto_id SET ENCODING (compresstype=zstd, compresslevel=7),
    ALTER COLUMN vendedor_id SET ENCODING (compresstype=rle_type),
    ALTER COLUMN loja_id SET ENCODING (compresstype=rle_type);

-- 4. Migração de dados (com limpeza)
INSERT INTO fato_vendas_novo
SELECT 
    id::BIGINT as venda_id,
    data::DATE as data_venda,
    -- Lookup IDs das dimensões...
    -- (simplificado, na prática precisaria de JOINs/lookups)
    1 as cliente_id,
    1 as produto_id,
    1 as vendedor_id,
    1 as loja_id,
    qtd as quantidade,
    preco::NUMERIC(10,2) as valor_unitario,
    total::NUMERIC(12,2) as valor_total,
    obs as observacoes
FROM vendas_problematica
WHERE data IS NOT NULL 
  AND qtd > 0;

-- 5. Compare resultados
SELECT 
    'Antes (problematica)' as versao,
    pg_size_pretty(pg_total_relation_size('vendas_problematica')) as tamanho,
    COUNT(*) as linhas
FROM vendas_problematica
UNION ALL
SELECT 
    'Depois (fato + dims)',
    pg_size_pretty(
        pg_total_relation_size('fato_vendas_novo') +
        pg_total_relation_size('dim_clientes_novo') +
        pg_total_relation_size('dim_produtos_novo') +
        pg_total_relation_size('dim_lojas_novo') +
        pg_total_relation_size('dim_vendedores_novo')
    ),
    COUNT(*)
FROM fato_vendas_novo;
```

**Melhorias Implementadas:**
- ✅ Normalização (dimensões separadas)
- ✅ Tipos de dados corretos
- ✅ Storage colunar (AOCO)
- ✅ Compressão otimizada por coluna
- ✅ Distribuição adequada (venda_id único)
- ✅ Dimensões replicadas (evita motion)
- ✅ Particionamento por data
- ✅ Constraints de integridade

---

## Resumo do Módulo 2

### Habilidades Adquiridas
✅ Escolher estratégia de distribuição adequada  
✅ Evitar e corrigir data skew  
✅ Decidir entre row-oriented e column-oriented  
✅ Aplicar compressão eficazmente  
✅ Otimizar queries através de co-location  
✅ Projetar schemas para data warehouse  
✅ Migrar tabelas mal projetadas  

### Decisões Chave de Design

| Aspecto | OLTP / Transacional | OLAP / Analítico |
|---------|-------------------|------------------|
| **Storage** | HEAP (row) | AOCO (column) |
| **Distribuição** | Chave primária | Chave de JOIN |
| **Compressão** | Nenhuma/baixa | Média/alta |
| **Updates** | Frequentes | Raros/nenhum |
| **Particionamento** | Opcional | Recomendado |
| **Dimensões** | Normalizadas | Replicadas se pequenas |

### Checklist de Design de Tabela

**Para cada tabela, pergunte:**

1. **Padrão de acesso:**
   - [ ] Queries leem linha completa ou poucas colunas?
   - [ ] Há UPDATEs/DELETEs frequentes?

2. **Distribuição:**
   - [ ] Coluna tem alta cardinalidade?
   - [ ] Usada em JOINs frequentes?
   - [ ] Dados estão balanceados?

3. **Storage:**
   - [ ] Tabela será grande (> 1GB)?
   - [ ] Dados são append-only?
   - [ ] Queries são analíticas?

4. **Compressão:**
   - [ ] Espaço é preocupação?
   - [ ] Colunas têm padrões repetitivos?
   - [ ] Performance de escrita é crítica?

5. **Particionamento:**
   - [ ] Tabela tem milhões de linhas?
   - [ ] Queries filtram por data/período?
   - [ ] Precisa dropar dados antigos?

### Comandos Principais

```sql
-- Distribuição
DISTRIBUTED BY (coluna)
DISTRIBUTED RANDOMLY
DISTRIBUTED REPLICATED

-- Storage
WITH (appendoptimized=true, orientation=row)
WITH (appendoptimized=true, orientation=column)

-- Compressão
WITH (compresstype=zstd, compresslevel=5)
ALTER TABLE t ALTER COLUMN c SET ENCODING (compresstype=zlib)

-- Verificações
SELECT * FROM gp_toolkit.gp_skew_coefficients WHERE skcrelname = 'tabela';
SELECT gp_segment_id, COUNT(*) FROM tabela GROUP BY gp_segment_id;
```

### Próximos Passos
No **Módulo 3**, você aprenderá sobre particionamento avançado, índices e otimização de queries no Greenplum.

---

**Fim do Módulo 2**
