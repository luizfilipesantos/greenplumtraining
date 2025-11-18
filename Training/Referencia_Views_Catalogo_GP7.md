# Referência: Views de Catálogo no Greenplum 7

## ⚠️ Mudanças do GP6 para GP7

### Views que MUDARAM

| GP6 (ANTIGO) ❌ | GP7 (CORRETO) ✅ | Uso |
|-----------------|------------------|-----|
| `FROM pg_partitions` | `FROM pg_catalog.pg_partitions` | Listar partições |
| `FROM pg_partition_rule` | `FROM pg_catalog.pg_partition_rule` | Regras de partição |
| `FROM pg_partition_columns` | `FROM pg_catalog.pg_partition_columns` | Colunas de partição |

**💡 Regra geral:** Sempre use `pg_catalog.` antes de views de partição no GP7!

---

## ✅ Views que CONTINUAM FUNCIONANDO (gp_toolkit)

Estas views **são válidas no GP7** e **não precisam** de pg_catalog:

### Skew (Distribuição de Dados)

```sql
-- Coeficiente de skew por tabela
SELECT * FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'minha_tabela';
```

```sql
-- Detalhes do skew por segmento
SELECT * FROM gp_toolkit.gp_skew_details
WHERE skcrelname = 'minha_tabela';
```

### Tamanho de Tabelas

```sql
-- Tamanho total da tabela em todos os segmentos
SELECT * FROM gp_toolkit.gp_size_of_table_disk
WHERE sotdtablename = 'minha_tabela';
```

```sql
-- Tamanho de todas as tabelas do schema
SELECT * FROM gp_toolkit.gp_size_of_schema_disk
WHERE sodddatname = current_database();
```

### AO/AOCO (Append-Optimized)

```sql
-- Informações de visibility map (AO)
SELECT * FROM gp_toolkit.__gp_aovisimap_name('tabela_ao');
```

```sql
-- Bloat em tabelas AO/AOCO
SELECT * FROM gp_toolkit.gp_bloat_diag;
```

---

## 📋 Queries Comuns CORRIGIDAS para GP7

### 1. Listar todas as partições de uma tabela

```sql
-- ✅ CORRETO (GP7)
SELECT 
    schemaname,
    tablename,
    partitiontablename,
    partitionname,
    partitionrank,
    partitionboundary
FROM pg_catalog.pg_partitions
WHERE tablename = 'vendas_mensais'
ORDER BY partitionrank;
```

### 2. Tamanho de cada partição

```sql
-- ✅ CORRETO (GP7)
SELECT 
    pt.partitiontablename,
    pt.partitionboundary,
    pg_size_pretty(pg_total_relation_size(pt.partitiontablename::regclass)) AS tamanho
FROM pg_catalog.pg_partitions pt
WHERE pt.tablename = 'vendas_mensais'
ORDER BY pt.partitionrank;
```

### 3. Contagem de linhas por partição

```sql
-- ✅ CORRETO (GP7)
SELECT 
    pt.partitionname,
    pt.partitionboundary,
    (SELECT COUNT(*) 
     FROM vendas_mensais v
     WHERE v.tableoid = pt.partitiontablename::regclass::oid) AS row_count
FROM pg_catalog.pg_partitions pt
WHERE pt.tablename = 'vendas_mensais'
ORDER BY pt.partitionrank;
```

### 4. Verificar skew de distribuição

```sql
-- ✅ CORRETO (GP7) - gp_toolkit funciona normalmente
SELECT 
    skcrelname AS tabela,
    skccoeff AS coeficiente_skew
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname IN ('vendas', 'clientes', 'produtos')
ORDER BY skccoeff DESC;
```

### 5. Tabelas maiores do banco

```sql
-- ✅ CORRETO (GP7) - gp_toolkit funciona normalmente
SELECT 
    sotdtablename AS tabela,
    pg_size_pretty(sotdtoastsz + sotdtidsz) AS tamanho_total
FROM gp_toolkit.gp_size_of_table_disk
ORDER BY (sotdtoastsz + sotdtidsz) DESC
LIMIT 10;
```

---

## 🔍 Alternativas Nativas do PostgreSQL

Se preferir usar catálogo nativo (mais portável):

### Listar partições (pg_inherits)

```sql
-- Alternativa usando catálogo nativo do PostgreSQL
SELECT 
    c.relname AS partition_name,
    pg_get_expr(c.relpartbound, c.oid) AS partition_boundary,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS size
FROM pg_class p
JOIN pg_inherits i ON p.oid = i.inhparent
JOIN pg_class c ON i.inhrelid = c.oid
WHERE p.relname = 'vendas_mensais'
ORDER BY c.relname;
```

### Colunas de particionamento

```sql
-- Alternativa usando catálogo nativo
SELECT 
    a.attname AS partition_column,
    format_type(a.atttypid, a.atttypmod) AS data_type
FROM pg_class c
JOIN pg_partitioned_table pt ON c.oid = pt.partrelid
JOIN pg_attribute a ON a.attrelid = c.oid 
    AND a.attnum = ANY(pt.partattrs)
WHERE c.relname = 'vendas_mensais';
```

---

## 📝 Checklist de Migração GP6 → GP7

Ao revisar scripts/queries antigas do GP6:

- [ ] `FROM pg_partitions` → `FROM pg_catalog.pg_partitions`
- [ ] `FROM pg_partition_rule` → `FROM pg_catalog.pg_partition_rule`
- [ ] `FROM pg_partition_columns` → `FROM pg_catalog.pg_partition_columns`
- [ ] `FROM gp_toolkit.*` → **Manter como está** ✅
- [ ] `FROM pg_stat_*` → **Manter como está** ✅
- [ ] `FROM information_schema.*` → **Manter como está** ✅

---

## 🚨 Erros Comuns e Soluções

### Erro: "relation 'pg_partitions' does not exist"

**Causa:** Falta especificar schema `pg_catalog.`

**Solução:**
```sql
-- ❌ ERRADO
SELECT * FROM pg_partitions;

-- ✅ CORRETO
SELECT * FROM pg_catalog.pg_partitions;
```

### Erro: "column 'partitiontablename' does not exist"

**Causa:** Usando view nativa do PostgreSQL (`pg_class`) que tem nomes diferentes.

**Solução:** Use `pg_catalog.pg_partitions` (específico GP) ou adapte para nomes do PostgreSQL:
- `pg_partitions.partitiontablename` → `pg_class.relname`
- `pg_partitions.partitionboundary` → `pg_get_expr(pg_class.relpartbound, oid)`

---

## 📚 Documentação Oficial

- **GP7 System Catalogs:** https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/ref_guide-system_catalogs-catalog_ref.html
- **gp_toolkit Views:** https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/ref_guide-gp_toolkit.html
- **PostgreSQL Catalogs:** https://www.postgresql.org/docs/12/catalogs.html

---

**📅 Última atualização:** Novembro 2025 (GP 7.x)
