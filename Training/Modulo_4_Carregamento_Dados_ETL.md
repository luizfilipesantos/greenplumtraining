# Módulo 4: Carregamento de Dados e ETL

**Duração Total:** 120-150 minutos  
**Objetivo:** Dominar técnicas de carregamento massivo de dados, uso de External Tables, padrões de ETL e integração com ferramentas externas no Greenplum.

---

## Índice
1. [Lab 4.1: COPY - Carregamento Básico](#lab-41-copy---carregamento-básico-25-30-min)
2. [Lab 4.2: External Tables - Leitura e Escrita](#lab-42-external-tables---leitura-e-escrita-30-35-min)
3. [Lab 4.3: gpload e Ferramentas de Carga](#lab-43-gpload-e-ferramentas-de-carga-25-30-min)
4. [Lab 4.4: Padrões de ETL e Boas Práticas](#lab-44-padrões-de-etl-e-boas-práticas-35-40-min)

---

## Lab 4.1: COPY - Carregamento Básico (25-30 min)

### Objetivos
- Usar COPY para carregamento rápido
- Entender diferenças entre COPY e INSERT
- Trabalhar com diferentes formatos (CSV, TSV, TEXT)
- Tratar erros durante carregamento

### Conceitos Abordados
- **COPY FROM:** Importar dados de arquivo
- **COPY TO:** Exportar dados para arquivo
- **Formatos:** CSV, TEXT, BINARY
- **Delimitadores e Encodings:** Customização
- **Error Handling:** SEGMENT REJECT LIMIT

### Preparação
- Acessar a pasta /tmp do servidor
- Sempre colocar seu nome de usuário na frente de cada arquivo a ser lido gravado, ex: usuario_arquivo.csv.

### COPY vs INSERT

| Aspecto | COPY | INSERT |
|---------|------|--------|
| **Performance** | Muito rápido | Lento |
| **Uso** | Cargas em massa | Linhas individuais |
| **Validação** | Básica | Completa |
| **Transacional** | Sim | Sim |
| **Parallel** | Sim (automático) | Não |

---

### Exercício 4.1.1: COPY Básico - CSV

**Objetivo:** Carregar dados de arquivo CSV

**Cenário:** Importar lista de clientes de arquivo

**Passos:**

1. Crie tabela de destino:
```sql
CREATE TABLE clientes_importacao (
    cliente_id INTEGER,
    nome VARCHAR(200),
    cpf VARCHAR(14),
    email VARCHAR(100),
    telefone VARCHAR(20),
    data_cadastro DATE,
    cidade VARCHAR(100),
    estado CHAR(2),
    ativo BOOLEAN
)
DISTRIBUTED BY (cliente_id);
```

2. Crie arquivo CSV de exemplo (no servidor):
```bash
# No terminal do servidor Greenplum
cat > /tmp/clientes.csv << 'EOF'
cliente_id,nome,cpf,email,telefone,data_cadastro,cidade,estado,ativo
1,João Silva,12345678901,joao@email.com,11987654321,2024-01-15,São Paulo,SP,true
2,Maria Santos,98765432109,maria@email.com,21987654321,2024-02-20,Rio de Janeiro,RJ,true
3,Pedro Oliveira,11122233344,pedro@email.com,31987654321,2024-03-10,Belo Horizonte,MG,true
4,Ana Costa,55566677788,ana@email.com,41987654321,2024-04-05,Curitiba,PR,false
5,Carlos Souza,99988877766,carlos@email.com,51987654321,2024-05-12,Porto Alegre,RS,true
EOF
```

3. Carregue dados com COPY:
```sql
\COPY clientes_importacao
FROM '/tmp/clientes.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    DELIMITER ',',
    NULL ''
);
```

4. Verifique os dados carregados:
```sql
SELECT * FROM clientes_importacao ORDER BY cliente_id;
```
```sql
SELECT 
    COUNT(*) as total_clientes,
    COUNT(CASE WHEN ativo THEN 1 END) as ativos,
    COUNT(CASE WHEN NOT ativo THEN 1 END) as inativos
FROM clientes_importacao;
```

**Opções importantes do COPY:**
- **FORMAT:** CSV, TEXT, BINARY
- **HEADER:** Se arquivo tem linha de cabeçalho
- **DELIMITER:** Separador de campos (vírgula, tab, pipe)
- **NULL:** Representação de valores nulos
- **QUOTE:** Caractere de aspas (padrão: ")
- **ESCAPE:** Caractere de escape

---

### Exercício 4.1.2: COPY com Transformações

**Objetivo:** Aplicar transformações durante o carregamento

**Cenário:** Carregar dados com formatação diferente

**Passos:**

1. Crie arquivo com formato diferente:
```bash
cat > /tmp/vendas.txt << 'EOF'
1|2024-01-15|1001|P001|5|99.90
2|2024-01-15|1002|P002|3|149.50
3|2024-01-16|1003|P001|2|99.90
4|2024-01-16|1001|P003|1|299.00
5|2024-01-17|1004|P002|4|149.50
EOF
```

2. Crie tabela temporária para staging:
```sql
CREATE TEMP TABLE vendas_staging (
    venda_id INTEGER,
    data_venda_str TEXT,
    cliente_id INTEGER,
    produto_cod TEXT,
    quantidade INTEGER,
    valor_unitario_str TEXT
)
DISTRIBUTED BY (venda_id);
```

3. Carregue na tabela staging:
```sql
\COPY vendas_staging
FROM '/tmp/vendas.txt'
WITH (
    FORMAT TEXT,
    DELIMITER '|',
    NULL 'NULL'
);
```

4. Transforme e insira na tabela final:
```sql
CREATE TABLE vendas_final (
    venda_id INTEGER,
    data_venda date,
    cliente_id INTEGER,
    produto_id integer,
    quantidade INTEGER,
    valor_total numeric
)
DISTRIBUTED BY (venda_id);
```
```sql
INSERT INTO vendas_final (
    venda_id,
    data_venda,
    cliente_id,
    produto_id,
    quantidade,
    valor_total
)
SELECT 
    venda_id,
    data_venda_str::DATE,
    cliente_id,
    SUBSTRING(produto_cod, 2)::INTEGER as produto_id,  -- Remove 'P' prefix
    quantidade,
    quantidade * valor_unitario_str::NUMERIC(10,2) as valor_total
FROM vendas_staging;
```
```sql
-- Verifique
SELECT * FROM vendas_final 
WHERE venda_id IN (1,2,3,4,5)
ORDER BY venda_id;
```

**Pattern Staging + Transform:**
- ✅ Carrega raw data primeiro
- ✅ Valida e transforma em SQL
- ✅ Permite rollback fácil
- ✅ Mantém dados originais para auditoria

---

### Exercício 4.1.3: COPY TO - Exportação

**Objetivo:** Exportar dados para arquivo

**Cenário:** Gerar extratos para análise externa

**Passos:**

1. Exporte dados para CSV:
```sql
\COPY (
    SELECT 
        cliente_id,
        nome,
        email,
        cidade,
        estado
    FROM clientes_importacao
    WHERE ativo = TRUE
)
TO '/tmp/clientes_ativos.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    DELIMITER ',',
    QUOTE '"'
);
```

2. Exporte relatório agregado:
```sql
\COPY (
    SELECT 
        DATE_TRUNC('month', data_venda)::DATE as mes,
        COUNT(*) as total_vendas,
        SUM(valor_total) as receita,
        AVG(valor_total) as ticket_medio
    FROM vendas_final
    WHERE data_venda >= '2024-01-01'
    GROUP BY 1
    ORDER BY 1
)
TO '/tmp/relatorio_mensal.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE
);
```

3. Verifique arquivos criados (no servidor):
```bash
head /tmp/clientes_ativos.csv
head /tmp/relatorio_mensal.csv
```

---

### Exercício 4.1.4: Error Handling com SEGMENT REJECT LIMIT

**Objetivo:** Lidar com registros inválidos durante carga

**Cenário:** Arquivo com dados problemáticos

**Passos:**

1. Crie arquivo com erros:
```bash
cat > /tmp/clientes_erros.csv << 'EOF'
cliente_id,nome,cpf,email,data_cadastro
101,Cliente A,12345678901,clientea@email.com,2024-01-15
102,Cliente B,INVALIDO,clienteb@email.com,2024-02-20
103,Cliente C,98765432109,clientec@email.com,DATA_INVALIDA
104,Cliente D,11122233344,cliented@email.com,2024-04-05
105,,55566677788,clientee@email.com,2024-05-12
EOF
```

2. Tente carga sem error handling (falhará):
```sql
TRUNCATE clientes_importacao;

-- Este comando falhará
\COPY clientes_importacao (cliente_id, nome, cpf, email, data_cadastro)
FROM '/tmp/clientes_erros.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE
);
-- ERRO: tipo de dados inválido
```

3. Use SEGMENT REJECT LIMIT para tolerar erros:
```sql
\COPY clientes_importacao (cliente_id, nome, cpf, email, data_cadastro)
FROM '/tmp/clientes_erros.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE
)
SEGMENT REJECT LIMIT 10 ROWS;
```

4. Verifique quantas linhas foram carregadas:
```sql
SELECT COUNT(*) as carregadas FROM clientes_importacao;
-- Deve mostrar apenas as linhas válidas
```

5. Configure log de erros:
```sql
-- Crie tabela para log de erros
CREATE TABLE load_errors (
    cmdtime TIMESTAMP,
    relname TEXT,
    filename TEXT,
    linenum INTEGER,
    bytenum INTEGER,
    errmsg TEXT,
    rawdata TEXT,
    rawbytes BYTEA
);

-- Recarregue com log (requer configuração do servidor)
-- LOG ERRORS INTO load_errors
```

**Estratégias de Error Handling:**
- **SEGMENT REJECT LIMIT:** Tolera % ou número de erros
- **LOG ERRORS:** Registra linhas problemáticas
- **ROWS:** Limite por número de linhas
- **PERCENT:** Limite por porcentagem

---

### Exercício 4.1.5: Performance Tuning - COPY em Lote

**Objetivo:** Otimizar cargas grandes

**Cenário:** Carregar muitos de registros

**Passos:**

1. Gere arquivo grande:
```bash
# Gera arquivo com 100 mil linhas
for i in {1..100000}; do
    echo "$i,Cliente_$i,1000,cliente@email.com,2024-01-01"
done > /tmp/clientes_1m.csv
```

2. Crie tabela otimizada para carga:
```sql
CREATE TABLE clientes_bulk (
    cliente_id INTEGER,
    nome VARCHAR(200),
    cpf VARCHAR(14),
    email VARCHAR(100),
    data_cadastro DATE
)
WITH (
    appendoptimized=true,
    orientation=column,
    compresstype=zstd,
    compresslevel=5
)
DISTRIBUTED BY (cliente_id);
```

3. Meça performance da carga:
```sql
\timing on

\COPY clientes_bulk
FROM '/tmp/clientes_1m.csv'
WITH (
    FORMAT CSV,
    DELIMITER ','
);

\timing off
```

4. Otimizando a carga:
```sql
-- Split de arquivos
split -l 50000 /tmp/clientes_1m.csv /tmp/clientes_batch_
```
```sql
-- Load parallel
-- Sessão 1 (execute em uma janela psql)
\copy clientes_bulk FROM '/tmp/clientes_batch_aa' CSV DELIMITER ','
-- Sessão 2 (execute em OUTRA janela psql simultaneamente)
\copy clientes_bulk FROM '/tmp/clientes_batch_ab' CSV DELIMITER ','
```


4. Analise a distribuição:
```sql
-- Verifique distribuição entre segmentos
SELECT 
    gp_segment_id,
    COUNT(*) as rows_per_segment
FROM clientes_bulk
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

**Dicas de Performance:**
- ✅ Use tabelas AO/AOCO para cargas
- ✅ Desabilite índices antes de carregar (se existirem)
- ✅ Execute em transação única
- ✅ Rode ANALYZE após carga
- ✅ Use compressão para economizar espaço

---


### Próximos Passos
No **Módulo 5** veremos sobre Manutenção, Vacuum e Skew.

---

**Fim do Módulo 4**
