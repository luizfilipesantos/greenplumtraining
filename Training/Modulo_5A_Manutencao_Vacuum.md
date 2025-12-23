# Módulo 5A: Manutenção e VACUUM

**Objetivo:** Dominar técnicas de manutenção de banco de dados, compreender o funcionamento do VACUUM e otimizar operações de limpeza no Greenplum.

---

## Índice
1. [Lab 5A.1: Fundamentos do VACUUM](#lab-5a1-fundamentos-do-vacuum)
2. [Lab 5A.2: VACUUM Avançado e Tuning](#lab-5a2-vacuum-avançado-e-tuning)
3. [Lab 5A.3: Monitoramento e Automação](#lab-5a3-monitoramento-e-automação)

---

## Lab 5A.1: Fundamentos do VACUUM

### Objetivos
- Compreender quando e por que usar VACUUM
- Diferenciar VACUUM, VACUUM FULL e ANALYZE
- Identificar problemas de bloat

### Conceitos Abordados
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
    relname as tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
WHERE relname = 'teste_vacuum';
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
    relname as tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) * 100 as percentual_mortas,
    pg_size_pretty(pg_relation_size(relid)) as tamanho_tabela
FROM pg_stat_user_tables
WHERE relname = 'teste_vacuum';
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

2. Atualize as estatísticas:
```sql
-- VACUUM remove dead tuples mas NÃO atualiza pg_stat_user_tables automaticamente
-- É necessário executar ANALYZE para atualizar as estatísticas
ANALYZE teste_vacuum;
```

3. Verifique resultados:
```sql
SELECT 
    schemaname,
    relname as tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) * 100 as percentual_mortas,
    pg_size_pretty(pg_relation_size(relid)) as tamanho_tabela
FROM pg_stat_user_tables
WHERE relname = 'teste_vacuum';
```

**Observações:**
- `n_dead_tup` deve estar próximo de 0
- `n_live_tup` agora mostra o valor correto (após ANALYZE)
- `last_vacuum` atualizado
- ⚠️ **Tamanho da tabela NÃO diminuiu!**

4. Por que o tamanho não diminuiu?
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
    relname as tablename,
    n_live_tup as linhas_vivas,
    n_dead_tup as linhas_mortas,
    n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) * 100 as percentual_mortas,
    pg_size_pretty(pg_relation_size(relid)) as tamanho_tabela
FROM pg_stat_user_tables
WHERE relname = 'produtos_teste';

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

## Resumo do Módulo 5A

### Checklist importante
✅ Executar VACUUM, VACUUM FULL e VACUUM ANALYZE  
✅ Detectar e quantificar linhas mortas dos arquivos.
✅ Otimizar configurações de VACUUM  
✅ Monitorar execuções de VACUUM  
✅ Automatizar manutenção  
✅ Criar alertas 


### Boas Práticas

**Rotina Regular:**
1. VACUUM ANALYZE semanal em tabelas grandes
2. VACUUM ANALYZE após cargas massivas
3. Monitore diariamente
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

