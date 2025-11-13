# Guia Completo de Compressão no Greenplum

**Documento Auxiliar - Referência Técnica**  
**Greenplum Database 7.5.4**

---

## 📑 Sumário

- [Visão Geral](#visão-geral)
- [Tabela Comparativa de Algoritmos](#tabela-comparativa-de-algoritmos)
- [Tipos de Compressão Detalhados](#tipos-de-compressão-detalhados)
- [Níveis de Compressão](#níveis-de-compressão)
- [Matriz de Decisão](#matriz-de-decisão)
- [Configuração por Tipo de Dados](#configuração-por-tipo-de-dados)
- [Boas Práticas](#boas-práticas)
- [Troubleshooting](#troubleshooting)

---

## Visão Geral

A compressão no Greenplum está disponível **apenas para tabelas Append-Optimized (AO)**, seja row-oriented ou column-oriented (AOCO). Tabelas HEAP não suportam compressão.

**Benefícios da Compressão:**
- ✅ Redução de 50-95% no espaço em disco
- ✅ Menor I/O de disco (mais dados em cache)
- ✅ Queries mais rápidas em scans sequenciais
- ✅ Redução de custos de storage
- ✅ Melhor uso de memória (buffer pool)

**Trade-offs:**
- ❌ CPU adicional para compressão/descompressão
- ❌ Inserções ligeiramente mais lentas
- ❌ Não adequado para tabelas com updates frequentes

---

## Tabela Comparativa de Algoritmos

| Algoritmo | Tipo | Ratio Típico | Velocidade Compressão | Velocidade Descompressão | CPU Usage | I/O Savings | Uso Recomendado |
|-----------|------|--------------|----------------------|--------------------------|-----------|-------------|-----------------|
| **zstd** | General | 3-5x | ⚡⚡⚡ Muito Rápido | ⚡⚡⚡ Muito Rápido | Baixo-Médio | Alto | **Recomendado padrão** - Melhor custo-benefício |
| **zlib** | General | 2-4x | ⚡⚡ Médio | ⚡⚡ Médio | Médio | Médio-Alto | Compatibilidade com versões antigas |
| **RLE_TYPE** | Column-specific | 2-50x* | ⚡⚡⚡⚡ Extremo | ⚡⚡⚡⚡ Extremo | Muito Baixo | Extremo* | Colunas com valores repetidos |
| **none** | - | 1x | N/A | N/A | Zero | Zero | Dados já comprimidos, alta variação |

**Legenda de Performance:**
- ⚡⚡⚡⚡ = Extremamente Rápido (< 10% overhead)
- ⚡⚡⚡ = Muito Rápido (10-20% overhead)
- ⚡⚡ = Médio (20-40% overhead)
- ⚡ = Lento (> 40% overhead)

*RLE pode ter ratio extremo (até 50x) em colunas com alta repetição, mas 1x em colunas únicas.

---

## Tipos de Compressão Detalhados

### 1. ZSTD (Zstandard) - **RECOMENDADO**

**Introduzido:** Greenplum 6.x  
**Tipo:** Compressão general-purpose moderna  
**Desenvolvedor:** Facebook/Meta

#### Características
- ✅ **Melhor custo-benefício** geral
- ✅ Velocidade excepcional (mais rápido que zlib)
- ✅ Ratio competitivo (3-5x típico)
- ✅ Suporta níveis 1-19
- ✅ Excelente para workloads analíticos

#### Quando Usar
- ✅ **Padrão recomendado** para novas tabelas AO/AOCO
- ✅ Tabelas de fatos grandes (> 10 GB)
- ✅ Queries de leitura pesada
- ✅ Data warehousing moderno
- ✅ Dados mistos (texto, numérico, datas)

#### Quando NÃO Usar
- ❌ Workload de inserção ultra-rápida (streaming)
- ❌ Dados já comprimidos (JPEG, PNG, ZIP)
- ❌ Compatibilidade com Greenplum < 6.0

#### CPU
- ⚠️ Toda compressão utiliza CPU. Monitorar uso da CPU para balancear algoritmos e níveis de compressão.

#### Sintaxe
```sql
-- Tabela inteira
CREATE TABLE tabela (...) 
WITH (appendoptimized=true, compresstype=zstd, compresslevel=1);

-- Por coluna (AOCO)
CREATE TABLE tabela (
    col1 INTEGER,
    col2 TEXT ENCODING (compresstype=zstd, compresslevel=3),
    col3 NUMERIC
) WITH (appendoptimized=true, orientation=column);
```

#### Níveis Recomendados
- **Nível 1:** Padrão - Ótimo equilíbrio (recomendado)
- **Nível 3:** Melhor compressão com overhead aceitável
- **Nível 6:** Compressão alta para dados frios
- **Nível 9+:** Raramente necessário (overhead alto)

---

### 2. ZLIB (gzip)

**Tipo:** Compressão general-purpose clássica  
**Baseado:** Algoritmo DEFLATE

#### Características
- ✅ Amplamente testado e estável
- ✅ Compatível com todas versões GP
- ✅ Ratio decente (2-4x)
- ⚡⚡ Velocidade média
- ⚠️ Mais lento que zstd
- ⚠️ Maior uso de CPU

#### Quando Usar
- ✅ Compatibilidade com GP 4.x/5.x
- ✅ Ambientes conservadores (produção crítica)
- ✅ Quando zstd não está disponível
- ✅ Migração de sistemas legados
- ✅ Políticas corporativas exigem gzip

#### Quando NÃO Usar
- ❌ Novos projetos em GP 6.x+
- ❌ Performance é prioridade máxima
- ❌ CPU já é gargalo

#### Sintaxe
```sql
-- Tabela inteira
CREATE TABLE tabela (...) 
WITH (appendoptimized=true, compresstype=zlib, compresslevel=5);

-- Por coluna (AOCO)
CREATE TABLE tabela (
    col1 INTEGER,
    col2 TEXT ENCODING (compresstype=zlib, compresslevel=6),
    col3 NUMERIC
) WITH (appendoptimized=true, orientation=column);
```

#### Níveis Recomendados
- **Nível 1:** Compressão mínima, velocidade máxima
- **Nível 5-6:** Padrão - Bom equilíbrio
- **Nível 9:** Máxima compressão (lento)

---

### 3. RLE_TYPE (Run-Length Encoding)

**Tipo:** Compressão especializada por coluna  
**Disponível:** Apenas AOCO (column-oriented)

#### Características
- ✅ **Extremamente eficiente** para dados repetitivos
- ✅ Overhead CPU mínimo
- ✅ Pode alcançar 10-50x em casos ideais
- ✅ Velocidade excepcional
- ⚠️ Ineficaz para dados únicos (ratio ~1x)
- ⚠️ Apenas para tabelas column-oriented

#### Como Funciona
RLE armazena sequências de valores repetidos como:
```
Original: A A A A B B B C C C C C
RLE:      4A 3B 5C
```

#### Quando Usar (IDEAL)
- ✅ **Colunas de status:** 'ATIVO', 'INATIVO', 'PENDENTE'
- ✅ **Flags booleanos:** TRUE, FALSE
- ✅ **Categorias:** 'Bronze', 'Prata', 'Ouro'
- ✅ **Códigos repetidos:** UF, país, departamento
- ✅ **Valores NULL frequentes**
- ✅ **Timestamps arredondados:** data sem hora
- ✅ **Dados ordenados:** IDs sequenciais, datas

#### Quando NÃO Usar
- ❌ Colunas com alta cardinalidade (IDs únicos)
- ❌ Dados aleatórios ou variados
- ❌ Números de ponto flutuante com muita precisão
- ❌ Texto livre (descrições, comentários)
- ❌ Hashes, UUIDs

#### Sintaxe
```sql
-- Apenas por coluna em AOCO
CREATE TABLE pedidos (
    pedido_id BIGINT,
    cliente_id INTEGER,
    status VARCHAR(20) ENCODING (compresstype=rle_type),  -- Ideal!
    uf CHAR(2) ENCODING (compresstype=rle_type),          -- Ideal!
    ativo BOOLEAN ENCODING (compresstype=rle_type),       -- Ideal!
    categoria VARCHAR(20) ENCODING (compresstype=rle_type), -- Ideal!
    valor NUMERIC(15,2),
    descricao TEXT ENCODING (compresstype=zstd)           -- Não RLE!
) WITH (appendoptimized=true, orientation=column);
```

#### Exemplos de Ratio RLE

| Tipo de Coluna | Valores Distintos | Ratio Típico | Exemplo |
|----------------|-------------------|--------------|---------|
| Status (5 valores) | 5 | 15-30x | 'NOVO', 'PROCESSANDO', 'CONCLUÍDO', 'CANCELADO', 'ERRO' |
| Boolean | 2 | 20-40x | TRUE, FALSE |
| UF (27 valores) | 27 | 10-20x | 'SP', 'RJ', 'MG'... |
| Categoria | 10-50 | 8-15x | 'BRONZE', 'PRATA', 'OURO' |
| ID Sequencial | Milhões | 1-2x | 1, 2, 3, 4, 5... (baixa repetição) |
| UUID | Milhões | ~1x | Cada valor único |

---

### 4. NONE (Sem Compressão)

#### Quando Usar
- ✅ Dados já comprimidos (ZIP, JPEG, PNG, AVRO comprimido)
- ✅ Dados criptografados (alta entropia)
- ✅ Teste/desenvolvimento (debugging) ⚠️
- ✅ Tabelas temporárias de curta duração (Stages que serão apagadas ao final da carga)
- ✅ Dados com altíssima variação (hashes, tokens)
- ✅ CPU é gargalo crítico

#### Sintaxe
```sql
CREATE TABLE tabela (...) 
WITH (appendoptimized=true, compresstype=none);
```

---

## Níveis de Compressão

### ZSTD Levels (1-19)

| Nível | Ratio | Velocidade Comp. | Velocidade Decomp. | CPU | Uso Recomendado |
|-------|-------|------------------|--------------------| ----|-----------------|
| **1** | 3.0x | ⚡⚡⚡⚡ | ⚡⚡⚡⚡ | Baixo | **Padrão geral** - Tabelas quentes |
| **3** | 3.5x | ⚡⚡⚡ | ⚡⚡⚡⚡ | Baixo | Bom equilíbrio - Uso geral |
| **5** | 4.0x | ⚡⚡⚡ | ⚡⚡⚡ | Médio | Tabelas de tamanho médio |
| **6** | 4.2x | ⚡⚡ | ⚡⚡⚡ | Médio | Dados frios (acesso mensal) |
| **9** | 4.5x | ⚡⚡ | ⚡⚡⚡ | Alto | Arquivamento (acesso raro) |
| **12+** | 5.0x+ | ⚡ | ⚡⚡⚡ | Muito Alto | Arquivamento extremo (não recomendado) |

**Recomendação:** Use nível 1 ou 3 para 95% dos casos.

### ZLIB Levels (1-9)

| Nível | Ratio | Velocidade Comp. | Velocidade Decomp. | CPU | Uso Recomendado |
|-------|-------|------------------|--------------------| ----|-----------------|
| **1** | 2.0x | ⚡⚡⚡ | ⚡⚡⚡ | Baixo | Compressão leve |
| **5** | 3.0x | ⚡⚡ | ⚡⚡ | Médio | **Padrão zlib** |
| **6** | 3.2x | ⚡⚡ | ⚡⚡ | Médio | Recomendado |
| **9** | 3.5x | ⚡ | ⚡⚡ | Alto | Máxima compressão |

**Recomendação:** Use nível 5 ou 6 se precisar usar zlib.

---

## Matriz de Decisão

### Por Tipo de Workload

| Workload | Storage Type | Compressão Recomendada | Nível | Justificativa |
|----------|--------------|------------------------|-------|---------------|
| **OLAP / DW** | AOCO | zstd | 1-3 | Scans grandes, CPU disponível |
| **Fatos grandes** | AOCO | zstd + RLE | 1 | RLE em categorias, zstd no resto |
| **Streaming ETL** | AO Row | zstd | 1 | Inserções rápidas, boa compressão |
| **Archive** | AOCO | zstd | 6-9 | Acesso raro, maximize espaço |
| **Tabelas pequenas (<1GB)** | AO Row | zstd | 1 | Overhead mínimo |
| **OLTP híbrido** | HEAP | none | - | HEAP não suporta compressão |
| **Staging** | AO Row | none | - | Temporário, velocidade máxima |

### Por Tipo de Dados

| Tipo de Dado | AOCO Column Type | Compressão | Ratio Esperado | Exemplo |
|--------------|------------------|------------|----------------|---------|
| **IDs Únicos** | BIGINT | zstd, level 1 | 2-3x | customer_id, order_id |
| **IDs Sequenciais** | BIGINT | RLE + zstd | 5-10x | auto_increment |
| **Status/Flags** | VARCHAR/CHAR | **RLE_TYPE** | 15-40x | 'ATIVO', 'INATIVO' |
| **Categorias** | VARCHAR | **RLE_TYPE** | 10-25x | 'BRONZE', 'PRATA', 'OURO' |
| **Datas** | DATE | zstd, level 1 | 3-5x | order_date |
| **Timestamps** | TIMESTAMP | zstd, level 3 | 4-6x | created_at |
| **Valores NULL** | ANY | **RLE_TYPE** | 30-50x | Colunas esparsas |
| **Texto Curto** | VARCHAR(50) | zstd, level 3 | 3-5x | nome, email |
| **Texto Longo** | TEXT | zstd, level 5 | 5-10x | descrição, comentários |
| **JSON** | JSONB/TEXT | zstd, level 3 | 4-8x | Dados semi-estruturados |
| **Números Decimais** | NUMERIC | zstd, level 1 | 2-4x | valor, preco |
| **Floats** | REAL/DOUBLE | zstd, level 1 | 1.5-3x | coordenadas, medições |
| **Booleanos** | BOOLEAN | **RLE_TYPE** | 20-40x | is_active, has_permission |
| **UF/Códigos** | CHAR(2) | **RLE_TYPE** | 15-30x | 'SP', 'RJ', 'MG' |
| **Imagens/Blobs** | BYTEA | **none** | ~1x | Já comprimidos |

---

## Configuração por Tipo de Dados

### Template: Tabela de Vendas Otimizada

```sql
CREATE TABLE vendas_otimizada (
    -- IDs: Alta cardinalidade -> zstd leve
    venda_id BIGINT ENCODING (compresstype=zstd, compresslevel=1),
    cliente_id INTEGER ENCODING (compresstype=zstd, compresslevel=1),
    produto_id INTEGER ENCODING (compresstype=zstd, compresslevel=1),
    
    -- Datas: zstd moderado
    data_venda DATE ENCODING (compresstype=zstd, compresslevel=1),
    data_entrega TIMESTAMP ENCODING (compresstype=zstd, compresslevel=3),
    
    -- Categorias: RLE ideal
    loja_id INTEGER ENCODING (compresstype=rle_type),
    uf CHAR(2) ENCODING (compresstype=rle_type),
    categoria VARCHAR(30) ENCODING (compresstype=rle_type),
    status VARCHAR(20) ENCODING (compresstype=rle_type),
    ativo BOOLEAN ENCODING (compresstype=rle_type),
    
    -- Numéricos: zstd leve
    quantidade INTEGER ENCODING (compresstype=zstd, compresslevel=1),
    valor_unitario NUMERIC(10,2) ENCODING (compresstype=zstd, compresslevel=1),
    valor_total NUMERIC(15,2) ENCODING (compresstype=zstd, compresslevel=1),
    
    -- Textos: zstd moderado/alto
    nome_produto VARCHAR(100) ENCODING (compresstype=zstd, compresslevel=3),
    descricao TEXT ENCODING (compresstype=zstd, compresslevel=5),
    observacoes TEXT ENCODING (compresstype=zstd, compresslevel=5)
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (venda_id);
```

**Ratio esperado:** 5-8x de compressão geral.

---

## Boas Práticas

### ✅ DO's (Faça)

1. **Use ZSTD level 1 como padrão**
   ```sql
   WITH (appendoptimized=true, compresstype=zstd, compresslevel=1)
   ```

2. **Use RLE_TYPE em colunas categóricas (AOCO)**
   - Status, flags, códigos, UF, categorias
   - Verifica com: `SELECT COUNT(DISTINCT coluna) FROM tabela;`
   - Se < 100 valores distintos, considere RLE

3. **Teste antes de aplicar em produção**
   ```sql
   -- Crie tabela teste com sample
   CREATE TABLE teste_compressao AS 
   SELECT * FROM tabela_producao LIMIT 1000000
   WITH (appendoptimized=true, compresstype=zstd);
   
   -- Compare tamanhos
   SELECT pg_size_pretty(pg_total_relation_size('teste_compressao'));
   ```

4. **Configure por coluna em AOCO**
   - Diferentes tipos de dados = diferentes compressões
   - Maximize eficiência

5. **Monitore CPU após habilitar compressão**
   ```sql
   -- Durante queries
   SELECT * FROM pg_stat_activity WHERE state = 'active';
   ```

6. **Use compressão maior para dados frios**
   - Partições antigas: zstd level 6-9
   - Dados acessados raramente

7. **Documente escolhas de compressão**
   ```sql
   COMMENT ON TABLE vendas IS 
   'AOCO com zstd(1) geral, RLE_TYPE em categorias. Ratio 6.2x. Última otimização: 2025-11-10';
   ```

### ❌ DON'Ts (Não Faça)

1. **Não comprima HEAP tables**
   ```sql
   -- ❌ Erro!
   CREATE TABLE heap_tabela (...) 
   WITH (compresstype=zstd);  -- Ignora silenciosamente
   ```

2. **Não use níveis altos sem testar**
   ```sql
   -- ❌ Overhead excessivo
   compresslevel=19  -- Raramente vale a pena
   ```

3. **Não use RLE em colunas únicas**
   ```sql
   -- ❌ Ineficaz
   id SERIAL ENCODING (compresstype=rle_type)  -- Ratio ~1x
   ```

4. **Não comprima dados já comprimidos**
   ```sql
   -- ❌ Desperdício de CPU
   arquivo_zip BYTEA ENCODING (compresstype=zstd)
   ```

5. **Não ignore CPU overhead**
   - Monitore uso de CPU
   - Se CPU > 80%, considere compressão menor ou none

6. **Não use zlib em novos projetos**
   - Use zstd (mais rápido, melhor ratio)
   - zlib apenas para compatibilidade

7. **Não comprima staging tables**
   ```sql
   -- Staging = velocidade
   CREATE TABLE staging (...) 
   WITH (appendoptimized=true, compresstype=none);
   ```

---

## Troubleshooting

### Problema 1: Compressão Não Funcionando

**Sintomas:** Tabela não comprime (ratio ~1x)

**Causas:**
- Tabela HEAP (não suporta compressão)
- Dados já comprimidos
- RLE em coluna de alta cardinalidade

**Diagnóstico:**
```sql
-- Verifique storage type
SELECT 
    c.relname,
    am.amname as storage_type,
    (SELECT option_value FROM pg_options_to_table(c.reloptions) 
     WHERE option_name = 'compresstype') as compression
FROM pg_class c
LEFT JOIN pg_am am ON c.relam = am.oid
WHERE c.relname = 'sua_tabela';

-- Se am.amname = 'heap', compressão não funciona!
```

**Solução:**
```sql
-- Recrie como AO
CREATE TABLE tabela_nova (...) 
WITH (appendoptimized=true, compresstype=zstd, compresslevel=1);

INSERT INTO tabela_nova SELECT * FROM tabela_antiga;
```

### Problema 2: CPU Alto Após Compressão

**Sintomas:** CPU usage 80-100% constante

**Causas:**
- Nível de compressão muito alto
- Queries descomprimem muitos dados
- Hardware subdimensionado

**Diagnóstico:**
```sql
-- Monitore queries ativas
SELECT 
    pid, 
    usename,
    state,
    query_start,
    SUBSTRING(query, 1, 100) as query_preview
FROM pg_stat_activity
WHERE state = 'active' AND pid != pg_backend_pid();
```

**Solução:**
```sql
-- Reduza nível de compressão
ALTER TABLE tabela SET (compresslevel=1);

-- Ou desabilite temporariamente
ALTER TABLE tabela SET (compresstype=none);

-- Recarregue dados
VACUUM FULL tabela;
```

### Problema 3: Inserts Lentos

**Sintomas:** INSERT muito mais lento que esperado

**Causas:**
- Nível de compressão alto (> 5)
- Muitas colunas com compressão individual
- CPU gargalo

**Diagnóstico:**
```sql
-- Teste sem compressão
CREATE TEMP TABLE teste_none AS SELECT * FROM origem
WITH (appendoptimized=true, compresstype=none);

CREATE TEMP TABLE teste_zstd AS SELECT * FROM origem
WITH (appendoptimized=true, compresstype=zstd, compresslevel=1);

-- Compare tempos
```

**Solução:**
- Use zstd level 1 (overhead mínimo)
- Considere none para staging/ETL
- Batch inserts maiores (reduz overhead por linha)

### Problema 4: RLE Não Comprime

**Sintomas:** RLE_TYPE tem ratio ~1x

**Causas:**
- Coluna tem alta cardinalidade
- Dados não ordenados
- Coluna errada para RLE

**Diagnóstico:**
```sql
-- Verifique cardinalidade
SELECT 
    COUNT(*) as total_rows,
    COUNT(DISTINCT coluna) as distinct_values,
    ROUND(COUNT(DISTINCT coluna)::NUMERIC / COUNT(*) * 100, 2) as pct_unique
FROM tabela;

-- Se pct_unique > 10%, RLE não é ideal
```

**Solução:**
```sql
-- Mude para zstd
ALTER TABLE tabela 
ALTER COLUMN coluna SET ENCODING (compresstype=zstd, compresslevel=3);

-- Reorganize
VACUUM FULL tabela;
```

### Problema 5: Queries Lentas Após Compressão

**Sintomas:** SELECT mais lento que antes

**Causas:**
- CPU não acompanha descompressão
- Compressão alta (> 5)
- Queries acessam muitas colunas comprimidas

**Diagnóstico:**
```sql
-- Compare com/sem compressão
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

**Solução:**
- Reduza compresslevel
- Use AOCO (column store) para queries seletivas
- Considere não comprimir colunas mais acessadas

---

## Comandos de Verificação

### Verificar Compressão Atual

```sql
-- Tabela inteira
SELECT 
    c.relname as table_name,
    am.amname as storage_type,
    COALESCE(
        (SELECT option_value FROM pg_options_to_table(c.reloptions) 
         WHERE option_name = 'compresstype'), 
        'none'
    ) AS compression_type,
    COALESCE(
        (SELECT option_value FROM pg_options_to_table(c.reloptions) 
         WHERE option_name = 'compresslevel'), 
        'N/A'
    ) AS compression_level,
    pg_size_pretty(pg_total_relation_size(c.oid)) as total_size,
    pg_size_pretty(pg_relation_size(c.oid)) as table_size
FROM pg_class c
LEFT JOIN pg_am am ON c.relam = am.oid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r' 
  AND n.nspname = 'public'
  AND c.relname = 'sua_tabela';
```

### Verificar Compressão por Coluna (AOCO)

```sql
SELECT 
    a.attname as column_name,
    pg_catalog.format_type(a.atttypid, a.atttypmod) as data_type,
    e.attoptions as encoding_options,
    CASE 
        WHEN e.attoptions IS NULL THEN 'Herda da tabela'
        ELSE 'Específico da coluna'
    END as compression_source
FROM pg_attribute a
LEFT JOIN pg_attribute_encoding e ON e.attrelid = a.attrelid AND e.attnum = a.attnum
WHERE a.attrelid = 'sua_tabela_aoco'::regclass
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

### Comparar Tamanhos

```sql
-- Ratio de compressão estimado
WITH table_info AS (
    SELECT 
        schemaname,
        tablename,
        pg_total_relation_size(schemaname||'.'||tablename) as compressed_size,
        (SELECT SUM(pg_column_size(t.*)) FROM (SELECT * FROM tabela LIMIT 10000) t) * 
        (SELECT COUNT(*) FROM tabela) / 10000 as estimated_uncompressed
    FROM pg_tables
    WHERE tablename = 'sua_tabela'
)
SELECT 
    tablename,
    pg_size_pretty(compressed_size) as compressed,
    pg_size_pretty(estimated_uncompressed) as estimated_uncompressed,
    ROUND(estimated_uncompressed::NUMERIC / NULLIF(compressed_size, 0), 2) as compression_ratio
FROM table_info;
```

---

## Referências Rápidas

### Decisão Rápida: Qual Compressão Usar?

**Fluxograma:**
```
1. Tabela é HEAP?
   └─ SIM: Sem compressão (HEAP não suporta)
   └─ NÃO: Continue...

2. Tabela > 1 GB?
   └─ NÃO: compresstype=none ou zstd(1)
   └─ SIM: Continue...

3. Tabela tem updates frequentes?
   └─ SIM: HEAP sem compressão
   └─ NÃO: Continue...

4. Queries acessam poucas colunas?
   └─ SIM: AOCO + compressão mista
   └─ NÃO: AO Row + zstd(1)

5. Compressão mista (AOCO):
   - Categorias/Status/Flags: RLE_TYPE
   - IDs/Datas/Números: zstd(1)
   - Textos longos: zstd(3-5)
```

### Sintaxe Rápida

```sql
-- AO Row + zstd
CREATE TABLE t (...) 
WITH (appendoptimized=true, compresstype=zstd, compresslevel=1);

-- AOCO + zstd
CREATE TABLE t (...) 
WITH (appendoptimized=true, orientation=column, compresstype=zstd);

-- AOCO + compressão mista
CREATE TABLE t (
    id BIGINT ENCODING (compresstype=zstd, compresslevel=1),
    status VARCHAR(20) ENCODING (compresstype=rle_type),
    descricao TEXT ENCODING (compresstype=zstd, compresslevel=5)
) WITH (appendoptimized=true, orientation=column);

-- Mudar compressão existente
ALTER TABLE t SET (compresstype=zstd, compresslevel=1);
VACUUM FULL t;  -- Necessário para aplicar!

-- Desabilitar compressão
ALTER TABLE t SET (compresstype=none);
VACUUM FULL t;
```

---

## Conclusão

**Recomendações finais:**

1. **Default:** AOCO + zstd level 1 para 90% dos casos
2. **Otimização:** Use RLE_TYPE em colunas categóricas (< 100 valores distintos)
3. **Performance:** Priorize zstd(1) - melhor custo-benefício
4. **Teste sempre:** Valide em ambiente não-produção primeiro
5. **Monitore:** Acompanhe CPU e tamanhos

**Economia esperada:**
- Tabelas OLAP: 4-8x de compressão
- Com RLE otimizado: 6-12x de compressão
- Redução de 70-90% no storage

---

**Última atualização:** Novembro 2025  
**Versão:** Greenplum 7.5.4  
**Documento:** Guia Completo de Compressão
