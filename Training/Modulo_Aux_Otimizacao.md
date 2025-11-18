# Módulo Auxiliar: Otimização de Queries no Greenplum 7

**Duração:** 60 minutos  
**Objetivo:** Aprender a analisar, diagnosticar e otimizar queries lentas no Greenplum 7.

---

## 📚 Índice
1. [Fundamentos do EXPLAIN](#1-fundamentos-do-explain)
2. [Identificando Gargalos](#2-identificando-gargalos)
3. [Otimizações Comuns](#3-otimizações-comuns)
4. [Configurações do Greenplum](#4-configurações-do-greenplum)
5. [Troubleshooting](#5-troubleshooting)

---

## 1. Fundamentos do EXPLAIN

### O que é EXPLAIN?

**EXPLAIN** mostra o **plano de execução** que o otimizador do Greenplum escolheu para sua query.

```
┌─────────────────────────────────────────────────┐
│ SUA QUERY                                       │
│ SELECT * FROM vendas WHERE data >= '2024-01-01'│
└─────────────────────────────────────────────────┘
         ↓
    OTIMIZADOR (ORCA ou Planner)
         ↓
┌─────────────────────────────────────────────────┐
│ PLANO DE EXECUÇÃO                               │
│ 1. Scan tabela vendas (apenas partições 2024)  │
│ 2. Filter data >= '2024-01-01'                  │
│ 3. Gather dados dos segmentos                  │
└─────────────────────────────────────────────────┘
```

### Variações do EXPLAIN

```sql
-- 1. EXPLAIN simples (sem executar)
EXPLAIN
SELECT * FROM vendas WHERE data_venda >= '2024-01-01';
```

**💡 Mostra:** Plano **estimado**, não executa query

```sql
-- 2. EXPLAIN ANALYZE (executa e mostra tempo real)
EXPLAIN ANALYZE
SELECT * FROM vendas WHERE data_venda >= '2024-01-01';
```

**💡 Mostra:** Plano **real** + tempos de execução + linhas reais

```sql
-- 3. EXPLAIN com opções detalhadas
EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true, COSTS true)
SELECT * FROM vendas WHERE data_venda >= '2024-01-01';
```

**💡 Mostra:** Máximo de detalhes (colunas, I/O, custos)

---

### Exercício 1.1: Lendo um EXPLAIN Básico

**Passo 1:** Criar tabela de exemplo

```sql
-- Tabela de vendas (não particionada, para exemplo)
CREATE TABLE vendas_exemplo (
    venda_id BIGSERIAL,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (venda_id);
```

```sql
-- Inserir dados
INSERT INTO vendas_exemplo (data_venda, cliente_id, produto_id, valor)
SELECT 
    CURRENT_DATE - (random() * 365)::INTEGER,
    (random() * 10000)::INTEGER,
    (random() * 500)::INTEGER,
    (random() * 5000)::NUMERIC(10,2)
FROM generate_series(1, 100000);
```

```sql
-- ANALYZE para estatísticas
ANALYZE vendas_exemplo;
```

**Passo 2:** EXPLAIN simples

```sql
EXPLAIN
SELECT COUNT(*), SUM(valor)
FROM vendas_exemplo
WHERE data_venda >= '2024-01-01';
```

**📊 Resultado típico:**

```
Finalize Aggregate  (cost=0.00..431.00 rows=1 width=40)
  ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=0.00..431.00 rows=1 width=40)
        ->  Partial Aggregate  (cost=0.00..431.00 rows=1 width=40)
              ->  Seq Scan on vendas_exemplo  (cost=0.00..431.00 rows=25000 width=14)
                    Filter: (data_venda >= '2024-01-01'::date)
```

**🔍 Leitura (de baixo para cima):**

1. **Seq Scan:** Varredura sequencial da tabela
   - `Filter: (data_venda >= '2024-01-01')`: Filtro aplicado
   - `rows=25000`: Estimativa de linhas que passam filtro
   - `cost=0.00..431.00`: Custo estimado (início..fim)

2. **Partial Aggregate:** Agregação local em cada segmento
   - `SUM(valor), COUNT(*)` calculados localmente

3. **Gather Motion 2:1:** Coleta dados de 2 segmentos para 1 (master)
   - `slice1; segments: 2`: Executa em paralelo em 2 segmentos

4. **Finalize Aggregate:** Agregação final no master
   - Soma resultados parciais dos segmentos

**Passo 3:** EXPLAIN ANALYZE (executa query)

```sql
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor)
FROM vendas_exemplo
WHERE data_venda >= '2024-01-01';
```

**📊 Resultado típico:**

```
Finalize Aggregate  (cost=0.00..431.00 rows=1 width=40) 
                    (actual time=89.123..89.125 rows=1 loops=1)
  ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=0.00..431.00 rows=1 width=40)
                         (actual time=85.456..89.078 rows=2 loops=1)
        ->  Partial Aggregate  (cost=0.00..431.00 rows=1 width=40)
                               (actual time=82.345..82.347 rows=1 loops=1)
              ->  Seq Scan on vendas_exemplo  (cost=0.00..431.00 rows=25000 width=14)
                                              (actual time=1.234..75.678 rows=27345 loops=1)
                    Filter: (data_venda >= '2024-01-01'::date)
                    Rows Removed by Filter: 72655
Planning Time: 2.345 ms
Execution Time: 89.567 ms
```

**🔍 Novos dados (ANALYZE):**

- `actual time=1.234..75.678`: Tempo **real** (início..fim em ms)
- `rows=27345`: Linhas **reais** retornadas (vs estimado 25000)
- `loops=1`: Quantas vezes o nó foi executado
- `Rows Removed by Filter: 72655`: Linhas descartadas pelo filtro
- `Planning Time: 2.345 ms`: Tempo para planejar query
- `Execution Time: 89.567 ms`: Tempo **total** de execução

**💡 Análise:**
- Estimativa (25k) vs Real (27k): **Próximas** → Estatísticas OK
- 72% das linhas removidas por filtro → Query não muito seletiva

---

### Exercício 1.2: Componentes do Plano GP7

**Slices (Fatias de Execução Paralela)**

```
Gather Motion 2:1  (slice1; segments: 2)
                    ^^^^^^
                    Slice 1 = Executa em 2 segmentos em paralelo
                    Slice 0 = Master (após Gather)
```

**Motion Nodes (Movimentação de Dados)**

```sql
EXPLAIN ANALYZE
SELECT c.estado, COUNT(*), SUM(v.valor)
FROM vendas_exemplo v
JOIN clientes c ON v.cliente_id = c.cliente_id
GROUP BY c.estado;
```

**📊 Resultado mostrará:**

```
-> Redistribute Motion 2:2  (slice2)  ← Redistribui dados entre segmentos
     Hash Key: c.estado

-> Broadcast Motion 1:2  (slice3)  ← Copia tabela para todos segmentos
     
-> Gather Motion 2:1  (slice1)  ← Coleta resultados finais
```

**Tipos de Motion:**

| Motion | Padrão | Quando Ocorre | Custo |
|--------|--------|---------------|-------|
| **Gather Motion N:1** | N segmentos → Master | Resultado final | Baixo (só resultado final) |
| **Redistribute Motion N:N** | Redistribui por hash | JOIN/GROUP BY sem co-location | **Alto** (move muitos dados) |
| **Broadcast Motion 1:N** | Copia para todos | Dimensão pequena em JOIN | Médio (se tabela pequena) |

**Partition Selector (Pruning)**

```sql
-- Tabela particionada
CREATE TABLE vendas_part (...) PARTITION BY RANGE (data_venda) ...;

EXPLAIN ANALYZE
SELECT * FROM vendas_part WHERE data_venda = '2024-06-15';
```

**📊 Resultado:**

```
-> Dynamic Seq Scan on vendas_part
     Partition Selector: $0
     Partitions selected: 1 (out of 12)  ← ✅ Partition Pruning!
     Filter: (data_venda = '2024-06-15'::date)
```

**💡 Ideal:** `Partitions selected` deve ser **mínimo** possível!

---

## 2. Identificando Gargalos

### 2.1: Tempo Total vs Distribuído

**Análise de onde o tempo está sendo gasto:**

```sql
EXPLAIN ANALYZE
SELECT cliente_id, SUM(valor)
FROM vendas_exemplo
GROUP BY cliente_id
ORDER BY SUM(valor) DESC
LIMIT 100;
```

**📊 Resultado:**

```
Limit  (actual time=245.123..245.234 rows=100 loops=1)
  ->  Gather Motion 2:1  (actual time=240.456..245.189 rows=100 loops=1)
        Merge Key: (sum(valor))
        ->  Limit  (actual time=235.789..235.890 rows=50 loops=1)
              ->  Sort  (actual time=230.123..232.456 rows=5000 loops=1)
                    Sort Method: top-N heapsort  Memory: 512kB
                    ->  HashAggregate  (actual time=180.234..185.567 rows=5000 loops=1)
                          Group Key: cliente_id
                          ->  Seq Scan  (actual time=1.234..145.678 rows=50000 loops=1)

Execution Time: 245.678 ms
```

**🔍 Análise:**

| Operação | Tempo | % do Total | Gargalo? |
|----------|-------|------------|----------|
| Seq Scan | 145ms | 59% | ⚠️ Maior custo |
| HashAggregate | 5ms | 2% | ✅ OK |
| Sort | 50ms | 20% | ⚠️ Moderado |
| Gather Motion | 5ms | 2% | ✅ OK |
| **TOTAL** | **245ms** | 100% | |

**💡 Conclusão:** Seq Scan é o gargalo (59% do tempo)

**Ações possíveis:**
1. Particionar por data (se queries filtram por data)
2. Adicionar filtro WHERE para reduzir Seq Scan
3. Verificar se ANALYZE está atualizado

---

### 2.2: Rows Estimado vs Real

**Discrepância entre estimativa e realidade indica problemas:**

```sql
EXPLAIN ANALYZE
SELECT * FROM vendas_exemplo v
JOIN produtos p ON v.produto_id = p.produto_id
WHERE v.data_venda >= '2024-01-01';
```

**📊 Cenário problemático:**

```
Hash Join  (cost=... rows=1000 width=...)  ← Estimado: 1000
           (actual time=... rows=500000 loops=1)  ← Real: 500k!
```

**🚨 Problema:** Estimativa **500x menor** que realidade!

**Consequências:**
- Otimizador escolhe plano errado (Nested Loop em vez de Hash Join)
- Aloca memória insuficiente (spill to disk)
- Performance degradada

**Solução:**

```sql
-- 1. ANALYZE atualizado?
ANALYZE vendas_exemplo;
ANALYZE produtos;

-- 2. Testar novamente
EXPLAIN ANALYZE SELECT ...;

-- 3. Se persistir, aumentar statistics target
ALTER TABLE vendas_exemplo ALTER COLUMN data_venda SET STATISTICS 1000;
ANALYZE vendas_exemplo;
```

---

### 2.3: Spill to Disk (Work Memory Insuficiente)

**Operação precisa escrever temporário em disco (lento!):**

```sql
EXPLAIN ANALYZE
SELECT cliente_id, COUNT(*), SUM(valor)
FROM vendas_exemplo
GROUP BY cliente_id;
```

**📊 Problema (spill to disk):**

```
HashAggregate  (actual time=5678.123..5890.456 rows=10000 loops=1)
  Group Key: cliente_id
  Batches: 5  ← 🚨 Múltiplos batches = spill to disk!
  Memory Usage: 4096kB
  Disk Usage: 153600kB  ← 🚨 Usou disco (150MB)!
```

**💡 Comparação:**

| Situação | Memory | Disk | Tempo |
|----------|--------|------|-------|
| **Fit in memory** | 50MB | 0 | 120ms ✅ |
| **Spill to disk** | 4MB | 150MB | 5890ms ❌ |

**Solução:**

```sql
-- Aumentar work_mem (por sessão)
SET work_mem = '256MB';

-- Executar query novamente
EXPLAIN ANALYZE SELECT ...;

-- Se melhorou, considere aumentar globalmente (DBA)
-- ALTER SYSTEM SET work_mem = '128MB';
```

---

### 2.4: Motion Excessivo

**Motion move dados pela rede entre segmentos (caro!):**

```sql
EXPLAIN ANALYZE
SELECT r.nome_regiao, COUNT(*), SUM(v.valor)
FROM vendas_exemplo v
JOIN regioes r ON v.regiao_id = r.regiao_id
GROUP BY r.nome_regiao;
```

**📊 Problema (muitos motions):**

```
HashAggregate
  ->  Redistribute Motion 2:2  (actual time=890.123..1234.567 rows=50000 loops=1)
        Hash Key: r.nome_regiao
        ->  Hash Join
              Hash Cond: (v.regiao_id = r.regiao_id)
              ->  Redistribute Motion 2:2  (actual time=456.789..678.901 rows=100000)  ← Motion 1
              ->  Broadcast Motion 1:2  (actual time=123.456..125.678 rows=10)  ← Motion 2
```

**🚨 Problema:** 2 motions movendo 100k + 10 linhas

**Tempo gasto em motions:** ~1500ms de 2500ms total (60%!)

**Soluções:**

1. **Replicate dimensão pequena (regioes)**
   ```sql
   CREATE TABLE regioes (...) DISTRIBUTED REPLICATED;
   -- Elimina Broadcast Motion!
   ```

2. **Co-locate fato e dimensão**
   ```sql
   CREATE TABLE vendas (...) DISTRIBUTED BY (regiao_id);
   CREATE TABLE regioes (...) DISTRIBUTED BY (regiao_id);
   -- Elimina Redistribute Motion!
   ```

---

## 3. Otimizações Comuns

### 3.1: Filtrar Antes de JOIN

**❌ ERRADO (JOIN depois filtra):**

```sql
SELECT v.*, c.nome
FROM vendas v
JOIN clientes c ON v.cliente_id = c.cliente_id
WHERE v.data_venda >= '2024-01-01'
  AND c.ativo = true;
```

**✅ CORRETO (filtra antes do JOIN):**

```sql
SELECT v.*, c.nome
FROM (
    SELECT * FROM vendas 
    WHERE data_venda >= '2024-01-01'
) v
JOIN (
    SELECT cliente_id, nome FROM clientes 
    WHERE ativo = true
) c ON v.cliente_id = c.cliente_id;
```

**💡 Benefício:** Menos linhas no JOIN → Mais rápido!

**Exemplo prático:**

```sql
-- Teste 1: SEM filtro prévio
EXPLAIN ANALYZE
SELECT COUNT(*)
FROM vendas_exemplo v
JOIN clientes c ON v.cliente_id = c.cliente_id
WHERE v.data_venda >= '2024-06-01';
```

```sql
-- Teste 2: COM filtro prévio (CTE)
EXPLAIN ANALYZE
WITH vendas_filtradas AS (
    SELECT * FROM vendas_exemplo 
    WHERE data_venda >= '2024-06-01'
)
SELECT COUNT(*)
FROM vendas_filtradas v
JOIN clientes c ON v.cliente_id = c.cliente_id;
```

**📊 Comparação:**
- Teste 1: 450ms (JOIN com 100k linhas)
- Teste 2: 180ms (JOIN com 8k linhas) ← **2.5x mais rápido!**

---

### 3.2: Usar Subqueries Correlatas com Cuidado

**❌ ERRADO (subquery correlata lenta):**

```sql
-- Nested Loop executado 100k vezes!
SELECT v.venda_id, v.valor,
    (SELECT MAX(valor) FROM vendas_exemplo WHERE cliente_id = v.cliente_id) AS max_cliente
FROM vendas_exemplo v
WHERE v.data_venda >= '2024-01-01';
```

**✅ CORRETO (JOIN ou window function):**

```sql
-- Opção 1: JOIN
SELECT v.venda_id, v.valor, m.max_valor
FROM vendas_exemplo v
JOIN (
    SELECT cliente_id, MAX(valor) AS max_valor
    FROM vendas_exemplo
    GROUP BY cliente_id
) m ON v.cliente_id = m.cliente_id
WHERE v.data_venda >= '2024-01-01';
```

```sql
-- Opção 2: Window Function (mais elegante)
SELECT 
    venda_id, 
    valor,
    MAX(valor) OVER (PARTITION BY cliente_id) AS max_cliente
FROM vendas_exemplo
WHERE data_venda >= '2024-01-01';
```

---

### 3.3: Evitar SELECT *

**❌ ERRADO (lê colunas desnecessárias):**

```sql
SELECT * 
FROM vendas_exemplo
WHERE data_venda >= '2024-01-01';
```

**✅ CORRETO (apenas colunas necessárias):**

```sql
SELECT venda_id, data_venda, valor
FROM vendas_exemplo
WHERE data_venda >= '2024-01-01';
```

**💡 Benefício em tabelas AOCO (column-oriented):**
- Lê apenas 3 colunas (não todas as 20)
- **I/O reduzido em ~85%!**

**Teste prático:**

```sql
-- Cria tabela AOCO
CREATE TABLE vendas_colunar (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor NUMERIC(10,2),
    desconto NUMERIC(10,2),
    imposto NUMERIC(10,2),
    -- ... 12 colunas mais
    observacoes TEXT
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (venda_id);
```

```sql
-- Teste 1: SELECT *
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM vendas_colunar WHERE data_venda >= '2024-01-01';
-- Buffers: 500 MB lidos

-- Teste 2: SELECT específico
EXPLAIN (ANALYZE, BUFFERS)
SELECT venda_id, valor FROM vendas_colunar WHERE data_venda >= '2024-01-01';
-- Buffers: 75 MB lidos  ← 6.6x menos I/O!
```

---

### 3.4: LIMIT com ORDER BY

**⚠️ CUIDADO: LIMIT sem índice apropriado pode ser lento:**

```sql
-- Precisa ordenar TUDO antes de pegar TOP 10
SELECT * FROM vendas_exemplo
ORDER BY valor DESC
LIMIT 10;
```

**EXPLAIN mostra:**

```
Limit
  ->  Gather Motion 2:1
        Merge Key: valor DESC
        ->  Limit
              ->  Sort  ← Ordena localmente em cada segmento
                    ->  Seq Scan  ← Lê TODA a tabela!
```

**✅ Otimização:** Adicionar índice ou particionar

```sql
-- Índice descendente
CREATE INDEX idx_vendas_valor_desc ON vendas_exemplo (valor DESC);
ANALYZE vendas_exemplo;

-- Agora usa Index Scan (mais rápido)
EXPLAIN ANALYZE
SELECT * FROM vendas_exemplo
ORDER BY valor DESC
LIMIT 10;
```

---

## 4. Configurações do Greenplum

### 4.1: work_mem (Memória por Operação)

**O que é:** Memória alocada para cada operação (sort, hash, aggregate).

**Padrão GP7:** 64MB

**Quando aumentar:**
- Spill to disk frequente
- Queries com grandes agregações
- Sorts complexos

```sql
-- Ver configuração atual
SHOW work_mem;

-- Aumentar para sessão atual (teste)
SET work_mem = '256MB';

-- Testar query
EXPLAIN ANALYZE SELECT ...;

-- Se melhorou, considere aumentar globalmente (DBA)
-- Cálculo: (RAM total / max_connections / num_operacoes_paralelas)
```

**⚠️ CUIDADO:** `work_mem` muito alto pode causar OOM!

**Exemplo:**
- 100 conexões simultâneas
- Cada query com 3 operações que usam work_mem
- work_mem = 1GB
- **Uso máximo:** 100 × 3 × 1GB = **300GB RAM!**

---

### 4.2: gp_workfile_limit_per_query

**Limite de quanto uma query pode spill para disco:**

```sql
-- Ver limite atual (0 = ilimitado)
SHOW gp_workfile_limit_per_query;

-- Definir limite (previne queries descontroladas)
SET gp_workfile_limit_per_query = '10GB';
```

**💡 Uso:** Previne query descontrolada de encher disco.

---

### 4.3: optimizer (ORCA vs Planner)

**GP7 tem 2 otimizadores:**

| Otimizador | Quando Usar | Vantagens | Limitações |
|------------|-------------|-----------|------------|
| **ORCA** (padrão) | Queries complexas OLAP | Planos melhores para JOINs múltiplos | Algumas features SQL não suportadas |
| **Planner** (legacy) | Queries simples, compatibilidade | Suporta todo SQL | Planos sub-ótimos em queries complexas |

```sql
-- Verificar otimizador atual
SHOW optimizer;

-- Forçar Planner para uma query
SET optimizer = off;
EXPLAIN ANALYZE SELECT ...;

-- Voltar para ORCA
SET optimizer = on;
```

**💡 Dica:** Se EXPLAIN mostra plano estranho, teste o outro otimizador!

---

### 4.4: enable_* (Controle Fino de Operações)

**Desabilitar operações específicas para testes:**

```sql
-- Força Seq Scan (desabilita Index Scan)
SET enable_indexscan = off;

-- Força Hash Join (desabilita Nested Loop)
SET enable_nestloop = off;

-- Desabilita Merge Join
SET enable_mergejoin = off;
```

**💡 Uso:** Debug de planos de execução (qual operação é mais rápida).

---

## 5. Troubleshooting

### Problema 1: Query Lenta sem Motivo Aparente

**Checklist:**

```sql
-- 1. ANALYZE atualizado?
SELECT last_analyze FROM pg_stat_user_tables WHERE tablename = 'vendas';
-- Se > 1 semana: ANALYZE vendas;

-- 2. Partition pruning funcionando?
EXPLAIN SELECT ... WHERE data_venda = '2024-06-01';
-- Deve mostrar: Partitions selected: 1 (out of N)

-- 3. work_mem suficiente?
EXPLAIN ANALYZE SELECT ...;
-- Procure por: "Disk Usage: XXX kB"  ← Spill!

-- 4. Índices sendo usados?
EXPLAIN ANALYZE SELECT ... WHERE coluna = valor;
-- Deve mostrar: Index Scan (não Seq Scan)

-- 5. Motion excessivo?
EXPLAIN ANALYZE SELECT ...;
-- Conte Redistribute/Broadcast Motions (< 2 ideal)
```

---

### Problema 2: Plano Diferente em Produção vs Dev

**Causas:**
- Dados em prod são diferentes (volume, distribuição)
- Estatísticas desatualizadas
- Configurações diferentes (work_mem, optimizer)

**Solução:**

```sql
-- 1. Copie estatísticas de prod para dev
pg_dump --schema-only --table=vendas prod > vendas_schema.sql
pg_dump --data-only --table=pg_statistic prod > vendas_stats.sql
psql dev < vendas_schema.sql
psql dev < vendas_stats.sql

-- 2. Igualar configurações
-- Em dev:
SET work_mem = '128MB';  -- Igual a prod
SET optimizer = on;      -- Igual a prod

-- 3. Testar plano
EXPLAIN ANALYZE SELECT ...;
```

---

### Problema 3: Query Matou Segmento (OOM)

**Sintoma:**

```
ERROR:  Interconnect error: Could not connect to segment 2
DETAIL:  System was unable to allocate memory (seg2 host:port pid=12345)
```

**Causas:**
- `work_mem` muito alto
- Query com cartesian product (JOIN sem ON)
- Spill to disk excessivo

**Solução imediata:**

```sql
-- Reduzir work_mem
SET work_mem = '64MB';

-- Adicionar limite
SET statement_mem = '2GB';  -- Máximo por query
```

**Prevenção:**

```sql
-- Configurar limites globais (DBA)
ALTER DATABASE mydb SET statement_mem = '2GB';
ALTER DATABASE mydb SET gp_workfile_limit_per_query = '10GB';
```

---

## 📝 Resumo do Módulo

### Comandos de Diagnóstico

```sql
-- Plano de execução
EXPLAIN ANALYZE SELECT ...;

-- Estatísticas de índices
SELECT * FROM pg_stat_user_indexes WHERE tablename = 'tabela';

-- Estatísticas de tabelas
SELECT * FROM pg_stat_user_tables WHERE tablename = 'tabela';

-- Configurações atuais
SHOW work_mem;
SHOW optimizer;

-- Sessão atual
SET work_mem = '256MB';
SET optimizer = off;
```

### Checklist de Otimização

**Antes de otimizar:**
- [ ] ANALYZE está atualizado? (< 1 semana)
- [ ] Query usa índices apropriados?
- [ ] Partition pruning está funcionando?
- [ ] Estimativas (rows) próximas da realidade?

**Durante otimização:**
- [ ] Identificou gargalo principal (EXPLAIN ANALYZE)?
- [ ] Spill to disk? (aumentar work_mem)
- [ ] Motion excessivo? (revisar distribuição/replicação)
- [ ] SELECT * desnecessário? (em AOCO)

**Após otimização:**
- [ ] Tempo melhorou significativamente (> 30%)?
- [ ] Query está estável (não varia muito)?
- [ ] Não causou regressão em outras queries?

### Ordem de Otimização

```
1. SCHEMA DESIGN (distribuição, particionamento)
   ↓
2. ANALYZE (estatísticas atualizadas)
   ↓
3. INDEXES (apenas se necessário)
   ↓
4. QUERY REWRITE (filtros, joins, subqueries)
   ↓
5. CONFIGURAÇÕES (work_mem, optimizer)
```

### Quando Pedir Ajuda de DBA

- [ ] Query usa > 50% CPU/memória do cluster
- [ ] Precisa alterar configurações globais (ALTER SYSTEM)
- [ ] Precisa adicionar recursos (RAM, segmentos)
- [ ] Query mata segmentos (OOM)
- [ ] Problema persiste após todas as otimizações

---

**🎯 Fim dos Módulos de Particionamento, Índices e Otimização!**

**Próximos passos:**
- Praticar com dados reais do seu sistema
- Monitorar queries lentas (pg_stat_statements)
- Estabelecer rotinas de manutenção (VACUUM, ANALYZE)
- Revisar schema design periodicamente
