# Módulo 3-1: Particionamento no Greenplum 7

**Duração:** 90 minutos  
**Objetivo:** Dominar técnicas de particionamento para otimizar performance e gerenciamento de dados no Greenplum 7.

**📚 Baseado na documentação oficial:** [Changes to Table Partitioning in Greenplum 7](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/admin_guide-partitions-changes.html)

---

## 🆕 Mudanças Importantes no GP7

**Greenplum 7 suporta DUAS sintaxes de particionamento:**

1. **Sintaxe Clássica (GP6):** `START/END/EVERY` - Compatibilidade com código legado
2. **Sintaxe Moderna (PostgreSQL 12):** `FOR VALUES FROM/TO` - Novos recursos

**⚠️ Importante:** GP7 usa novas estruturas internas. A sintaxe clássica é **mapeada internamente** para a moderna.


**✅ Recursos GP7:**
- Função: `pg_partition_tree()`, `pg_partition_ancestors()`, `pg_partition_root()`
- Partições são **tabelas de primeira classe** (não aliases)
- Suporte a particionamento HASH
- DEFAULT PARTITION melhorado
- ATTACH PARTITION com menos locks

---

## 📚 Índice
1. [Conceitos Fundamentais](#1-conceitos-fundamentais)
2. [Particionamento RANGE](#2-particionamento-range)
3. [Particionamento LIST](#3-particionamento-list)
4. [Gerenciamento de Partições](#4-gerenciamento-de-partições)
5. [Manutenção e Boas Práticas](#5-manutenção-e-boas-práticas)

---

## 1. Conceitos Fundamentais

### O que é Particionamento?

**Particionamento** divide uma tabela grande em **pedaços menores** fisicamente separados.

```
SEM PARTICIONAMENTO:
┌────────────────────────────────────┐
│ vendas (100M linhas, 500GB)        │
│ 2020|2021|2022|2023|2024           │
└────────────────────────────────────┘
Query 2024? Escaneia TUDO ❌

COM PARTICIONAMENTO:
┌────┐┌────┐┌────┐┌────┐┌────┐
│2020││2021││2022││2023││2024│
│20GB││25GB││30GB││35GB││40GB│
└────┘└────┘└────┘└────┘└────┘
Query 2024? Acessa APENAS 2024 ✅
```

### Benefícios

| Benefício | Exemplo |
|-----------|---------|
| **🚀 Performance** | Partition pruning elimina 11 de 12 partições |
| **🔧 Manutenção** | VACUUM 1 partição (10GB) vs tabela inteira (500GB) |
| **🗑️ Data Retention** | `DROP TABLE jan_2020` remove mês em < 1s |
| **💾 Backup** | Backup incremental por partição |
| **🗄️ Compressão** | Dados antigos: zstd 9, recentes: zstd 3 |

#### Partition Pruning
Partition Pruning é uma técnica de otimização de consultas que elimina automaticamente partições desnecessárias durante a execução de queries em tabelas particionadas.


### Tipos de Particionamento

| Tipo | Uso | Exemplo |
|------|-----|---------|
| **RANGE** | Faixas contínuas | Data (mensal), ID (100k-200k) |
| **LIST** | Valores discretos | Região (Norte/Sul), Status (Ativo/Inativo) |
| **HASH** 🆕 | Distribuição uniforme | Hash do ID (apenas sintaxe moderna) |

---

## 2. Particionamento RANGE

### Exercício 2.1: Particionamento Mensal

```sql
-- Cria tabela vendas particionada
CREATE TABLE part_vendas (
    venda_id BIGSERIAL,
    data_venda DATE NOT NULL,
    cliente_id INTEGER,
    valor NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=column, compresstype=zstd, compresslevel=3)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda); 
```
```sql
-- Cria partições
create table part_vendas_202401 partition of part_vendas for values from ('2024-01-01') to ('2024-02-01');
create table part_vendas_202402 partition of part_vendas for values from ('2024-02-01') to ('2024-03-01');
```
```sql
-- O intervalo é semi-aberto, não inclui o último item da lista.
-- Teste:
 insert into part_vendas (data_venda, cliente_id, valor) values ('2024-02-01', 1, 100.00);
```
```sql
-- Em qual partição está o registro 2024-02-01?
select tableoid::regclass as partition, data_venda
from part_vendas   
where data_venda = '2024-02-01';
```

#### O particionamento Range é sempre 'semi-aberto', não apenas para datas
```sql
-- Integer
create table produtos_faixa_preco (
    produto_id bigserial,
    preco integer not null,
    nome text
) partition by range (preco);
```
```sql
-- Partição 1: [0, 100)
create table produtos_0_100 partition of produtos_faixa_preco
for values from (0) to (100);
-- ✅ Aceita: 0, 50, 99
-- ❌ Rejeita: 100

-- Partição 2: [100, 500)
create table produtos_100_500 partition of produtos_faixa_preco
for values from (100) to (500);
-- ✅ Aceita: 100, 250, 499
-- ❌ Rejeita: 500
```
```sql
-- Inserindo registro 100
insert into produtos_faixa_preco (preco, nome) values (100, 'teste');
```
```sql
-- Em qual partição está o registro 100?
select tableoid::regclass as partition, preco
from produtos_faixa_preco   
where preco = 100;
```

**Visualizar partições criadas:**

```sql
-- Visualizar partições criadas
SELECT * FROM pg_partition_tree('part_vendas');
```
```sql
-- Visualizar partições criadas
SELECT * FROM pg_partition_tree('produtos_faixa_preco');
```


**Ver detalhes das partições:**

```sql
-- Detalhes das partições
SELECT 
    parent.relname AS table_name,
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_boundary
FROM pg_class parent
JOIN pg_inherits i ON parent.oid = i.inhparent
JOIN pg_class child ON i.inhrelid = child.oid
WHERE parent.relname = 'part_vendas'
ORDER BY child.relname;
```
---

### Exercício 2.2: Particionamento Mensal

**Passo 1:** Criar tabela particionada (sem definir partições)
```sql
drop table vendas;
```
```sql
-- Tabela particionada (sem START/END/EVERY)
CREATE TABLE vendas (
    venda_id BIGSERIAL,
    data_venda DATE NOT NULL,
    cliente_id INTEGER,
    valor NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=column, compresstype=zstd, compresslevel=3)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda);  -- ✅ Sem parênteses!
```

**Passo 2:** Criar partições 
```sql
-- A criação de partições no Greenplum 7 passa a ser declarativa:
CREATE TABLE vendas_202401 PARTITION OF vendas FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE vendas_202402 PARTITION OF vendas FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE vendas_202403 PARTITION OF vendas FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```


**Visualizar partições:**

```sql
SELECT * FROM pg_partition_tree('vendas');
```

---

### Exercício 2.3: Inserir Dados e Testar Partition Pruning

**Inserir 1 vendas conforme as partições**

```sql
-- 202401
INSERT INTO vendas (data_venda, cliente_id, valor)
SELECT 
    '2024-01-01'::DATE + (random() * 15)::INTEGER,
    (random() * 100000)::INTEGER,
    (random() * 10000)::NUMERIC(10,2)
FROM generate_series(1, 10000);
```
```sql
-- 202402
INSERT INTO vendas (data_venda, cliente_id, valor)
SELECT 
    '2024-02-01'::DATE + (random() * 15)::INTEGER,
    (random() * 100000)::INTEGER,
    (random() * 10000)::NUMERIC(10,2)
FROM generate_series(1, 10000);
```
```sql
-- 202403
INSERT INTO vendas (data_venda, cliente_id, valor)
SELECT 
    '2024-03-01'::DATE + (random() * 15)::INTEGER,
    (random() * 100000)::INTEGER,
    (random() * 10000)::NUMERIC(10,2)
FROM generate_series(1, 10000);
```
**Query COM partition pruning:**
```sql
-- Busca apenas Feveriro/2024
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(valor)
FROM vendas
WHERE data_venda >= '2024-02-01' 
  AND data_venda < '2024-02-10';
```
**O que acontece se inserir Maio?**
```sql
-- 202405
INSERT INTO vendas (data_venda, cliente_id, valor)
SELECT 
    '2024-05-01'::DATE + (random() * 15)::INTEGER,
    (random() * 100000)::INTEGER,
    (random() * 10000)::NUMERIC(10,2)
FROM generate_series(1, 10000);
```

```sql
-- ERROR:  no partition of relation "vendas" found for row  (seg1 127.0.0.1:6001 pid=1626300)
-- DETAIL:  Partition key of the failing row contains (data_venda) = (2024-05-08).
```
**💡 Análise:**
- Manter um monitoramento das partições, criando sempre as partições necessárias
- Possuir uma partição default para acolher os registros órfãos


**Criar partição DEFAULT** (rede de segurança):
```sql
CREATE TABLE vendas_outros PARTITION OF vendas DEFAULT;
-- ⚠️ Degrada partition pruning! Precisa DETACH/SPLIT periodicamente.
-- O otimizador não sabe o que tem na Default, então sempre escaneia ela.
```

**Quantos registros por partição?**
```sql
-- Conta registros em cada partição
SELECT 
    tableoid::regclass AS partition_name,
    COUNT(*) AS row_count
FROM vendas
GROUP BY tableoid
ORDER BY partition_name;
```

**Detach/Split**
```sql
-- 1 - Detach
ALTER TABLE vendas DETACH PARTITION vendas_outros;
-- vendas_outros agora é tabela independente
-- ⚠️ INSERTs em vendas FALHAM se não tiverem partição!
```

```sql
-- 2 - Cria partição
CREATE TABLE vendas_202405 PARTITION OF vendas
FOR VALUES FROM ('2024-05-01') TO ('2024-06-01');
```

```sql
-- 2 - Insere dados
INSERT INTO vendas_202405
SELECT * FROM vendas_outros
WHERE data_venda >= '2024-05-01' AND data_venda < '2024-06-01';
```

```sql
-- Limpa partição default
DELETE FROM vendas_outros
WHERE data_venda >= '2024-05-01' AND data_venda < '2024-06-01';
```

```sql
-- Re-Attach
ALTER TABLE vendas ATTACH PARTITION vendas_outros DEFAULT;
```

---

## 3. Particionamento LIST

### Exercício 3.1: LIST com DEFAULT Partition

**DEFAULT captura valores não mapeados:**

```sql
CREATE TABLE pedidos_status (
    pedido_id BIGSERIAL,
    status VARCHAR(20) NOT NULL,
    valor NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (pedido_id)
PARTITION BY LIST (status);
```

**Criar partições:**

```sql
-- Partições específicas
CREATE TABLE prt_pedidos_ativos PARTITION OF pedidos_status
FOR VALUES IN ('PENDENTE', 'PROCESSANDO', 'EM_ENTREGA');
```

```sql
CREATE TABLE prt_pedidos_concluidos PARTITION OF pedidos_status
FOR VALUES IN ('CONCLUIDO', 'ENTREGUE');
```

```sql
CREATE TABLE prt_pedidos_problemas PARTITION OF pedidos_status
FOR VALUES IN ('CANCELADO', 'DEVOLVIDO');
```

```sql
-- DEFAULT para capturar novos status!
CREATE TABLE prt_pedidos_outros PARTITION OF pedidos_status
DEFAULT;
```

**💡 Partição Defaul:** Novos status (ex: 'EM_ANALISE') não causam erro!

**Inserir dados:**

```sql
INSERT INTO pedidos_status (status, valor)
SELECT 
    CASE 
        WHEN random() < 0.1 THEN 'PENDENTE'
        WHEN random() < 0.9 THEN 'CONCLUIDO'
        WHEN random() < 0.95 THEN 'CANCELADO'
        ELSE 'NOVO_STATUS'  -- Vai para DEFAULT!
    END,
    (random() * 2000)::NUMERIC(10,2)
FROM generate_series(1, 1000000);
```

```sql
ANALYZE pedidos_status;
```

**Verificar distribuição nas partições:**

```sql
SELECT 
    tableoid::regclass AS partition,
    COUNT(*)
FROM pedidos_status
GROUP BY tableoid;
```

**📊 Resultado(exemplo):**
```
      partition        | count
-----------------------+--------
 pedidos_ativos        | 100000
 pedidos_concluidos    | 800000
 pedidos_problemas     |  50000
 pedidos_outros        |  50000  ← DEFAULT capturou 'NOVO_STATUS'!
```

---

## 4. Gerenciamento de Partições

### 4.1: Adicionar Novas Partições

#### Tabela criada com PARTITION BY RANGE:

```sql
drop table vendas;
```
```sql
-- Tabela particionada (sem START/END/EVERY)
CREATE TABLE vendas (
    venda_id BIGSERIAL,
    data_venda DATE NOT NULL,
    cliente_id INTEGER,
    valor NUMERIC(10,2)
)
WITH (appendoptimized=true, orientation=column, compresstype=zstd, compresslevel=3)
DISTRIBUTED BY (venda_id)
PARTITION BY RANGE (data_venda); 
```

```sql
-- Adiciona janeiro/2025 (você escolhe o nome!)
CREATE TABLE prt_vendas_202501 PARTITION OF vendas
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

**Loop para múltiplas partições:**

```sql
-- Create
DO $$
DECLARE
    mes INTEGER;
    mes_inicio DATE;
    mes_fim DATE;
    partition_name TEXT;
BEGIN
    FOR mes IN 2..12 LOOP
        mes_inicio := ('2025-' || LPAD(mes::TEXT, 2, '0') || '-01')::DATE;
        mes_fim := mes_inicio + INTERVAL '1 month';
        partition_name := 'vendas_' || TO_CHAR(mes_inicio, 'YYYYMM');
        
        EXECUTE format(
            'CREATE TABLE prt_%I PARTITION OF vendas FOR VALUES FROM (%L) TO (%L)',
            partition_name, mes_inicio, mes_fim
        );
    END LOOP;
    RAISE NOTICE 'Criadas 12 partições de 2025';
END $$;
```


---

### 4.2: Remover Partições (Data Retention)

**🆕 GP7:** Partições são **tabelas reais**, use `DROP TABLE`!

```sql
-- Remove partição de janeiro/2022
DROP TABLE prt_vendas_202501;
```

**✅ Vantagens:**
- Instantâneo (não DELETE lento)
- Libera espaço imediatamente
- Sem bloat

**Também pode ser feito um loop para remover partições**

```sql
-- drop
DO $$
DECLARE
    mes INTEGER;
    mes_inicio DATE;
    mes_fim DATE;
    partition_name TEXT;
BEGIN
    FOR mes IN 2..12 LOOP
        mes_inicio := ('2025-' || LPAD(mes::TEXT, 2, '0') || '-01')::DATE;
        mes_fim := mes_inicio + INTERVAL '1 month';
        partition_name := 'vendas_' || TO_CHAR(mes_inicio, 'YYYYMM');
        
        EXECUTE format(
            'drop TABLE prt_%I', partition_name
        );
    END LOOP;
    RAISE NOTICE 'Drop de Partições OK';
END $$;
```

**⚠️ Alternativa segura: DETACH antes de DROP**

```sql
-- Desanexa partição (preserva dados, não acessível pela tabela pai)
ALTER TABLE vendas DETACH PARTITION prt_vendas_202501;
```
```sql
-- vendas_202201 agora é tabela independente
-- Faça backup antes de dropar!
-- Copiar a tabela ou pgdump
create table bkp_prt_vendas_202501 as select * from prt_vendas_202501;
```
```sql
-- Agora pode dropar com segurança
DROP TABLE prt_vendas_202501;
```

---

### 4.3: EXCHANGE PARTITION (Trocar Dados)

**Cenário:** Carregar dados em staging, depois trocar com partição.

**Passo 1:** Criar staging com constraint de CHECK (obrigatório!)

```sql
CREATE TABLE vendas_202501_staging (
    venda_id BIGINT not null,
    data_venda DATE NOT NULL,
    cliente_id INTEGER,
    valor NUMERIC(10,2),
    CONSTRAINT staging_check 
        CHECK (data_venda >= '2025-01-01' AND data_venda < '2025-02-01')
)
WITH (appendoptimized=true, orientation=column, compresstype=zstd, compresslevel=3)
DISTRIBUTED BY (venda_id);
-- drop table vendas_202501_staging
```

**💡 CHECK constraint é obrigatória para ATTACH no GP7!**

**Passo 2:** Carregar dados na staging

```sql
INSERT INTO vendas_202501_staging (venda_id, data_venda, cliente_id, valor)
SELECT 
    i,  -- venda_id sequencial
    '2025-01-01'::DATE + (random() * 30)::INTEGER,  -- Datas aleatórias em janeiro/2025
    (random() * 100000)::INTEGER,  -- cliente_id aleatório
    (random() * 10000)::NUMERIC(10,2)  -- valor entre 0-10000
FROM generate_series(1, 100000) i;  -- 100k registros
```

**Passo 3:** EXCHANGE (sintaxe clássica)
```sql
-- Verifica partições da tabela antes do exchange
SELECT 
    tableoid::regclass AS partition,
    COUNT(*)
FROM vendas
GROUP BY tableoid;
```

```sql
ALTER TABLE vendas
EXCHANGE PARTITION FOR ('2025-01-15')  -- Identifica partição
WITH TABLE vendas_202501_staging;
```

```sql
-- Verifica partições da tabela antes do exchange
SELECT 
    tableoid::regclass AS partition,
    COUNT(*)
FROM vendas
GROUP BY tableoid;
```

**Ou usando DETACH/ATTACH:**


```sql
-- 1. Detach partição antiga
ALTER TABLE vendas DETACH PARTITION prt_vendas_202501;

-- 2. Renomeia staging para nome da partição
ALTER TABLE vendas_202501_staging RENAME TO prt_vendas_202501;

-- 3. Attach como partição
ALTER TABLE vendas ATTACH PARTITION prt_vendas_202501
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

### 4.4: TRUNCATE de Partição

**Limpa dados mas mantém estrutura:**

```sql
TRUNCATE TABLE prt_vendas_202501;
```

**✅ Partição continua existindo, mas vazia.**

---

## 5. Manutenção e Boas Práticas

### 5.1: Monitoramento de Partições

**Tamanho de cada partição:**

```sql
SELECT 
    child.relname AS partition_name,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS total_size,
    pg_size_pretty(pg_relation_size(child.oid)) AS data_size,
    child.reltuples::BIGINT AS estimated_rows
FROM pg_class parent
JOIN pg_inherits i ON parent.oid = i.inhparent
JOIN pg_class child ON i.inhrelid = child.oid
WHERE parent.relname = 'vendas'
ORDER BY pg_total_relation_size(child.oid) DESC;
```

**Última manutenção (VACUUM/ANALYZE):**

```sql
SELECT 
    schemaname || '.' || relname AS partition_name,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows
FROM pg_stat_user_tables
WHERE relname LIKE 'vendas_%'
ORDER BY relname;
```

---

### 5.3: Boas Práticas GP7

#### ✅ FAÇA

1. **Use sintaxe moderna para novos projetos**
   ```sql
   CREATE TABLE t (...) PARTITION BY RANGE (col);
   CREATE TABLE p PARTITION OF t FOR VALUES FROM ... TO ...;
   ```

2. **Sempre ANALYZE após INSERT**
   ```sql
   INSERT INTO vendas ...;
   ANALYZE vendas;  -- ✅ Essencial para partition pruning!
   ```

3. **Filtre pela coluna de partição**
   ```sql
   -- ✅ BOM
   SELECT * FROM vendas WHERE data_venda >= '2024-06-01';
   
   -- ❌ RUIM
   SELECT * FROM vendas WHERE cliente_id = 12345;
   ```

4. **Use DEFAULT partition (LIST)**
   ```sql
   CREATE TABLE p_default PARTITION OF t DEFAULT;  -- ✅ Captura novos valores
   ```

5. **Partições com tamanho similar**
   - Evite < 1GB ou > 500GB
   - Mensal é ideal para maioria

#### ❌ NÃO FAÇA

1. **Não particione tabelas pequenas (< 100GB)**
   - Overhead > benefício

2. **Não crie centenas de partições**
   - GP7 performa bem até ~200 partições

3. **Não use sub-partições (multi-level)**
   - GPORCA não suporta no GP7

4. **Não remova sem backup**
   ```sql
   -- ❌ PERIGOSO
   DROP TABLE vendas_202201;
   
   -- ✅ SEGURO
   ALTER TABLE vendas DETACH PARTITION vendas_202201;
   pg_dump -t vendas_202201 > backup.sql
   DROP TABLE vendas_202201;
   ```

---

### 5.4: Estratégia por Caso de Uso

| Caso | Estratégia | Exemplo |
|------|-----------|---------|
| **Time-series** | RANGE diário/mensal | Logs, eventos |
| **Data Warehouse** | RANGE mensal/trimestral | Fato vendas |
| **Multi-tenant** | LIST por tenant | Cada cliente = 1 partição |
| **Geográfico** | LIST por região | Vendas por estado |
| **Hot/Cold** | RANGE + compressão variável | Recentes (zstd 3), antigos (zstd 9) |

---

### 5.5: Checklist Final

**Antes de particionar:**
- [ ] Tabela > 100GB?
- [ ] Queries filtram por coluna específica (data, região)?

**Após criar partições:**
- [ ] ANALYZE executado?
- [ ] EXPLAIN mostra partition pruning?
- [ ] Partições balanceadas?

**Manutenção contínua:**
- [ ] Novas partições criadas antecipadamente?
- [ ] VACUUM ANALYZE agendado?
- [ ] Partições antigas arquivadas?

---

## 📝 Resumo

### Comandos-Chave

```sql
-- Criar (moderna)
CREATE TABLE t (...) PARTITION BY RANGE (col);
CREATE TABLE p PARTITION OF t FOR VALUES FROM (v1) TO (v2);

-- Listar partições
SELECT * FROM pg_partition_tree('nome_tabela');

-- Remover
DROP TABLE partition_name;

-- Detach (preserva)
ALTER TABLE t DETACH PARTITION p;

-- Attach
ALTER TABLE t ATTACH PARTITION p FOR VALUES FROM (v1) TO (v2);

-- Manutenção
ANALYZE partition_name;
VACUUM partition_name;
```

### Funções Novas GP7

| Função | Uso |
|--------|-----|
| `pg_partition_tree()` | Lista partições |
| `pg_partition_ancestors()` | Lista ancestrais |
| `pg_partition_root()` | Retorna tabela raiz |

```sql
-- 1. pg_partition_tree() - Lista todas as partições e hierarquia
SELECT * FROM pg_partition_tree('vendas');
```
```sql
-- 2. pg_partition_ancestors() - Lista ancestrais de uma partição específica
SELECT * FROM pg_partition_ancestors('prt_vendas_202501');
```
```sql
-- 3. pg_partition_root() - Retorna a tabela raiz de uma partição
SELECT pg_partition_root('prt_vendas_202501');
```