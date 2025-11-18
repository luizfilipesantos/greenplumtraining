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
COPY clientes_importacao (cliente_id, nome, cpf, email, data_cadastro)
FROM '/tmp/clientes_erros.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE
);
-- ERRO: tipo de dados inválido
```

3. Use SEGMENT REJECT LIMIT para tolerar erros:
```sql
COPY clientes_importacao (cliente_id, nome, cpf, email, data_cadastro)
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
split -l 10000 /tmp/clientes_1m.csv /tmp/clientes_batch_
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

## Lab 4.2: External Tables - Leitura e Escrita (30-35 min)

### Objetivos
- Criar External Tables para leitura
- Usar External Tables para escrita
- Integrar com sistemas de arquivos (GPFS, HDFS, S3)
- Implementar padrão de ETL com External Tables

### Conceitos Abordados
- **Readable External Tables:** Ler dados sem importar
- **Writable External Tables:** Exportar dados
- **Protocolos:** file://, gpfdist://, s3://, hdfs://
- **gpfdist:** Servidor de arquivos paralelo

### O Que São External Tables?
External Tables são tabelas virtuais que permitem acessar dados armazenados fora do banco de dados como se fossem tabelas normais, mas sem importar os dados fisicamente pra dentro do banco. Seria como 'anexar' arquivos externos, tratando esses arquivos como tabelas de origem e/ou destino.

### External Tables vs COPY

| Aspecto | External Table | COPY |
|---------|---------------|------|
| **Dados** | Não carregados (virtual) | Carregados fisicamente |
| **Performance Query** | Mais lento (I/O externo) | Rápido (dados locais) |
| **Espaço** | Zero no GP | Ocupa espaço |
| **Uso** | Staging, integração | Carga permanente |
| **Modificação** | Read-only ou write-only | Read-write normal |

---

### Exercício 4.2.1: External Table com gpfdist

**Objetivo:** Usar servidor de arquivos paralelo

**Cenário:** Carregar arquivos grandes com paralelismo

**Passos:**

1. Inicie gpfdist (no servidor de arquivos):
```bash
# Inicie gpfdist em um diretório
mkdir -p /tmp/gpfdist_data
cd /tmp/gpfdist_data

# Copie arquivo de dados
cp /tmp/clientes_1m.csv .

# Inicie gpfdist (porta padrão: 8080)
gpfdist -d /tmp/gpfdist_data -p 8080 &
```

2. Crie External Table usando gpfdist:
```sql
CREATE EXTERNAL TABLE ext_clientes_gpfdist (
    cliente_id INTEGER,
    nome VARCHAR(200),
    cpf VARCHAR(14),
    email VARCHAR(100),
    data_cadastro DATE
)
LOCATION ('gpfdist://hostname:8080/clientes_1m.csv')
FORMAT 'CSV' (
    DELIMITER ','
)
SEGMENT REJECT LIMIT 100 ROWS;
```

3. Teste leitura paralela:
```sql
-- Leitura será paralela através de gpfdist
EXPLAIN ANALYZE
SELECT COUNT(*) FROM ext_clientes_gpfdist;

-- Carga rápida para tabela interna
CREATE TABLE clientes_from_gpfdist AS
SELECT * FROM ext_clientes_gpfdist
DISTRIBUTED BY (cliente_id);
```

4. Múltiplos arquivos com gpfdist:
```bash
# Divida arquivo grande em partes (no servidor)
split -l 250000 /tmp/clientes_1m.csv /tmp/gpfdist_data/clientes_part_
```

```sql
-- External table que lê todos os arquivos
CREATE EXTERNAL TABLE ext_clientes_multifile (
    cliente_id INTEGER,
    nome VARCHAR(200),
    cpf VARCHAR(14),
    email VARCHAR(100),
    data_cadastro DATE
)
LOCATION ('gpfdist://hostname:8080/clientes_part_*')
FORMAT 'CSV';
```

**Vantagens do gpfdist:**
- ✅ **Paralelo:** Múltiplos segmentos leem simultaneamente
- ✅ **Rápido:** Network throughput máximo
- ✅ **Escalável:** Múltiplos servidores gpfdist
- ✅ **Wildcards:** Ler múltiplos arquivos

---

### Exercício 4.2.3: Writable External Table - Exportação

**Objetivo:** Exportar dados usando external tables

**Cenário:** Gerar arquivos para sistemas externos

**Passos:**

1. Crie Writable External Table:
```sql
CREATE WRITABLE EXTERNAL TABLE ext_vendas_export (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    valor_total NUMERIC(10,2)
)
LOCATION ('gpfdist://hostname:8080/vendas_export.csv')
FORMAT 'CSV' (
    HEADER
    DELIMITER ','
);
```

2. Exporte dados:
```sql
-- Insira dados na external table (serão escritos no arquivo)
INSERT INTO ext_vendas_export
SELECT 
    venda_id,
    data_venda,
    cliente_id,
    produto_id,
    valor_total
FROM vendas_particionada
WHERE data_venda >= '2024-01-01'
  AND data_venda < '2024-02-01';
```

3. Verifique arquivo gerado:
```bash
# No servidor gpfdist
ls -lh /tmp/gpfdist_data/vendas_export.csv*
head /tmp/gpfdist_data/vendas_export.csv
```

4. Export distribuído (múltiplos arquivos por segmento):
```sql
CREATE WRITABLE EXTERNAL TABLE ext_vendas_distributed (
    venda_id BIGINT,
    data_venda DATE,
    valor_total NUMERIC(10,2)
)
LOCATION (
    'gpfdist://host1:8080/vendas_seg#.csv',
    'gpfdist://host2:8080/vendas_seg#.csv',
    'gpfdist://host3:8080/vendas_seg#.csv'
)
FORMAT 'CSV'
DISTRIBUTED BY (venda_id);

-- Cada segmento escreve seu próprio arquivo
INSERT INTO ext_vendas_distributed
SELECT venda_id, data_venda, valor_total
FROM vendas_particionada
WHERE data_venda >= CURRENT_DATE - 365;
```

---

### Exercício 4.2.5: Pattern - External Table para Staging

**Objetivo:** Implementar padrão de staging com external tables

**Cenário:** Pipeline de carga incremental

**Passos:**

1. Crie estrutura de staging:
```sql
-- External table (landing zone)
CREATE EXTERNAL TABLE ext_staging_vendas (
    venda_id BIGINT,
    data_venda TEXT,
    cliente_id TEXT,
    produto_id TEXT,
    quantidade TEXT,
    valor TEXT,
    arquivo TEXT,
    linha INTEGER
)
LOCATION ('gpfdist://hostname:8080/staging/vendas/*.csv')
FORMAT 'CSV' (
    DELIMITER ','
    FILL MISSING FIELDS
)
SEGMENT REJECT LIMIT 5 PERCENT;

-- Tabela staging interna (validação)
CREATE TABLE staging_vendas (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor NUMERIC(10,2),
    data_carga TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    origem_arquivo TEXT
)
DISTRIBUTED BY (venda_id);

-- Tabela de controle de carga
CREATE TABLE load_control (
    load_id SERIAL PRIMARY KEY,
    arquivo TEXT,
    data_inicio TIMESTAMP,
    data_fim TIMESTAMP,
    linhas_lidas INTEGER,
    linhas_validas INTEGER,
    linhas_rejeitadas INTEGER,
    status VARCHAR(20)
)
DISTRIBUTED RANDOMLY;
```

2. Processo de carga (ETL):
```sql
-- Passo 1: Carregar da external table para staging
INSERT INTO staging_vendas (
    venda_id, data_venda, cliente_id, produto_id, quantidade, valor, origem_arquivo
)
SELECT 
    venda_id::BIGINT,
    data_venda::DATE,
    cliente_id::INTEGER,
    produto_id::INTEGER,
    quantidade::INTEGER,
    valor::NUMERIC(10,2),
    arquivo
FROM ext_staging_vendas
WHERE data_venda IS NOT NULL
  AND cliente_id IS NOT NULL;

-- Passo 2: Validar dados em staging
SELECT 
    COUNT(*) as total,
    COUNT(CASE WHEN venda_id IS NULL THEN 1 END) as sem_id,
    COUNT(CASE WHEN cliente_id NOT IN (SELECT cliente_id FROM dim_clientes) THEN 1 END) as cliente_invalido
FROM staging_vendas;

-- Passo 3: Carregar na tabela final (apenas válidos)
INSERT INTO vendas_particionada (
    venda_id, data_venda, cliente_id, produto_id, quantidade, valor_total
)
SELECT 
    s.venda_id,
    s.data_venda,
    s.cliente_id,
    s.produto_id,
    s.quantidade,
    s.valor * s.quantidade
FROM staging_vendas s
WHERE EXISTS (SELECT 1 FROM dim_clientes c WHERE c.cliente_id = s.cliente_id)
  AND EXISTS (SELECT 1 FROM dim_produtos p WHERE p.produto_id = s.produto_id);

-- Passo 4: Registrar carga
INSERT INTO load_control (arquivo, data_fim, linhas_validas, status)
SELECT 
    origem_arquivo,
    CURRENT_TIMESTAMP,
    COUNT(*),
    'COMPLETED'
FROM staging_vendas
GROUP BY origem_arquivo;

-- Passo 5: Limpar staging
TRUNCATE staging_vendas;
```

---

## Lab 4.3: gpload e Ferramentas de Carga (25-30 min)

### Objetivos
- Usar gpload para ETL declarativo
- Configurar arquivos de controle YAML
- Executar cargas incrementais
- Monitorar e logging

### Conceitos Abordados
- **gpload:** Utilitário de carga high-level
- **YAML Control File:** Configuração declarativa
- **Update/Insert/Merge:** Diferentes modos
- **Fast/Slow Mode:** Trade-offs de performance

---

### Exercício 4.3.1: gpload - Primeira Carga

**Objetivo:** Usar gpload para carregar dados

**Cenário:** Carga de arquivo CSV com gpload

**Passos:**

1. Crie arquivo de controle YAML:
```bash
cat > /tmp/gpload_clientes.yaml << 'EOF'
VERSION: 1.0.0.1
DATABASE: seu_database
USER: seu_usuario
HOST: localhost
PORT: 5432

GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - localhost
         PORT: 8081
         FILE:
           - /tmp/clientes_*.csv
    - FORMAT: csv
    - DELIMITER: ','
    - QUOTE: '"'
    - HEADER: true
    - ERROR_LIMIT: 100
   
   OUTPUT:
    - TABLE: clientes_bulk
    - MODE: insert
   
   SQL:
    - BEFORE: "TRUNCATE clientes_bulk"
EOF
```

2. Execute gpload:
```bash
gpload -f /tmp/gpload_clientes.yaml -v
```

3. Verifique log e resultados:
```sql
SELECT 
    COUNT(*) as total,
    COUNT(DISTINCT cidade) as cidades,
    COUNT(DISTINCT estado) as estados
FROM clientes_bulk;
```

**Opções do gpload:**
- **MODE:** insert, update, merge
- **MATCH_COLUMNS:** Chaves para update/merge
- **UPDATE_COLUMNS:** Colunas a atualizar
- **ERROR_LIMIT:** Tolerância a erros

---

### Exercício 4.3.2: gpload - Update e Merge

**Objetivo:** Atualizar registros existentes

**Cenário:** Carga incremental com merge

**Passos:**

1. Crie arquivo de controle para merge:
```yaml
VERSION: 1.0.0.1
DATABASE: seu_database
USER: seu_usuario

GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - localhost
         PORT: 8081
         FILE:
           - /tmp/clientes_atualizacao.csv
    - FORMAT: csv
    - HEADER: true
   
   OUTPUT:
    - TABLE: clientes_bulk
    - MODE: merge
    - MATCH_COLUMNS:
        - cliente_id
    - UPDATE_COLUMNS:
        - nome
        - email
        - telefone
        - cidade
   
   SQL:
    - BEFORE: "CREATE TEMP TABLE clientes_tmp (LIKE clientes_bulk)"
    - AFTER: "ANALYZE clientes_bulk"
```

2. Crie arquivo de atualização:
```bash
cat > /tmp/clientes_atualizacao.csv << 'EOF'
cliente_id,nome,email,telefone,cidade
1,João Silva - Atualizado,joao.novo@email.com,11999999999,São Paulo
2,Maria Santos - VIP,maria.vip@email.com,21999999999,Rio de Janeiro
EOF
```

3. Execute merge:
```bash
gpload -f /tmp/gpload_merge.yaml -v
```

4. Verifique atualizações:
```sql
SELECT * FROM clientes_bulk WHERE cliente_id IN (1, 2);
```

---

### Exercício 4.3.3: Agendamento e Automação

**Objetivo:** Automatizar cargas com cron

**Cenário:** Carga diária automatizada

**Passos:**

1. Crie script de carga:
```bash
cat > /home/gpadmin/scripts/carga_diaria.sh << 'EOF'
#!/bin/bash

# Configurações
LOGDIR="/home/gpadmin/logs"
DATE=$(date +%Y%m%d_%H%M%S)
LOGFILE="$LOGDIR/carga_$DATE.log"

echo "Iniciando carga: $(date)" >> $LOGFILE

# Verifica se há arquivos para processar
if [ -z "$(ls -A /data/staging/*.csv 2>/dev/null)" ]; then
    echo "Nenhum arquivo para processar" >> $LOGFILE
    exit 0
fi

# Executa gpload
gpload -f /home/gpadmin/config/gpload_vendas.yaml -v >> $LOGFILE 2>&1
RESULT=$?

if [ $RESULT -eq 0 ]; then
    echo "Carga concluída com sucesso" >> $LOGFILE
    # Move arquivos processados
    mv /data/staging/*.csv /data/archive/
else
    echo "Erro na carga: $RESULT" >> $LOGFILE
    # Alerta ou notificação
    echo "Erro no gpload" | mail -s "ALERTA: Falha na carga" admin@empresa.com
fi

echo "Finalizando carga: $(date)" >> $LOGFILE
EOF

chmod +x /home/gpadmin/scripts/carga_diaria.sh
```

2. Configure cron:
```bash
# Edite crontab
crontab -e

# Adicione linha (executa diariamente às 2h)
0 2 * * * /home/gpadmin/scripts/carga_diaria.sh
```

3. Tabela de controle de execuções:
```sql
CREATE TABLE etl_executions (
    execution_id SERIAL PRIMARY KEY,
    job_name VARCHAR(100),
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    status VARCHAR(20),
    rows_processed BIGINT,
    error_message TEXT
)
DISTRIBUTED RANDOMLY;

-- Procedure para registrar
CREATE OR REPLACE FUNCTION log_etl_execution(
    p_job_name VARCHAR,
    p_status VARCHAR,
    p_rows BIGINT,
    p_error TEXT DEFAULT NULL
) RETURNS INTEGER AS $$
DECLARE
    v_exec_id INTEGER;
BEGIN
    INSERT INTO etl_executions (job_name, start_time, end_time, status, rows_processed, error_message)
    VALUES (p_job_name, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, p_status, p_rows, p_error)
    RETURNING execution_id INTO v_exec_id;
    
    RETURN v_exec_id;
END;
$$ LANGUAGE plpgsql;
```

---

## Lab 4.4: Padrões de ETL e Boas Práticas (35-40 min)

### Objetivos
- Implementar padrões de ETL robustos
- Gerenciar cargas incrementais
- Implementar CDC (Change Data Capture)
- Otimizar performance de ETL

### Conceitos Abordados
- **Slowly Changing Dimensions (SCD):** Tipos 1, 2, 3
- **Incremental Load:** Delta processing
- **Idempotência:** Execuções repetíveis
- **Data Quality:** Validação e auditoria

---

### Exercício 4.4.1: Slowly Changing Dimension - Tipo 2

**Objetivo:** Implementar SCD Tipo 2 (histórico completo)

**Cenário:** Dimensão de clientes com histórico

**Passos:**

1. Crie tabela dimensão com SCD Tipo 2:
```sql
CREATE TABLE dim_clientes_scd (
    cliente_sk BIGSERIAL PRIMARY KEY,  -- Surrogate key
    cliente_id INTEGER,                 -- Business key
    nome VARCHAR(200),
    email VARCHAR(100),
    cidade VARCHAR(100),
    estado CHAR(2),
    categoria VARCHAR(50),
    -- Campos de controle SCD
    data_inicio DATE NOT NULL,
    data_fim DATE,
    versao INTEGER,
    ativo BOOLEAN DEFAULT TRUE
)
DISTRIBUTED BY (cliente_sk);

-- Índice na business key
CREATE INDEX idx_dim_clientes_scd_id ON dim_clientes_scd(cliente_id);
```

2. Carregue versão inicial:
```sql
INSERT INTO dim_clientes_scd (
    cliente_id, nome, email, cidade, estado, categoria,
    data_inicio, data_fim, versao, ativo
)
SELECT 
    cliente_id,
    nome,
    email,
    cidade,
    estado,
    'Regular' as categoria,
    CURRENT_DATE as data_inicio,
    '9999-12-31'::DATE as data_fim,
    1 as versao,
    TRUE as ativo
FROM clientes_importacao;
```

3. Crie tabela staging para updates:
```sql
CREATE TEMP TABLE clientes_staging (
    cliente_id INTEGER,
    nome VARCHAR(200),
    email VARCHAR(100),
    cidade VARCHAR(100),
    estado CHAR(2),
    categoria VARCHAR(50)
)
DISTRIBUTED BY (cliente_id);

-- Simule mudanças
INSERT INTO clientes_staging VALUES
    (1, 'João Silva', 'joao@email.com', 'Rio de Janeiro', 'RJ', 'VIP'),  -- Mudou cidade e categoria
    (2, 'Maria Santos', 'maria.novo@email.com', 'Rio de Janeiro', 'RJ', 'Regular'),  -- Mudou email
    (6, 'Novo Cliente', 'novo@email.com', 'Brasília', 'DF', 'Regular');  -- Cliente novo
```

4. Procedure de merge SCD Tipo 2:
```sql
CREATE OR REPLACE FUNCTION merge_scd_tipo2() RETURNS VOID AS $$
BEGIN
    -- Passo 1: Expire registros alterados
    UPDATE dim_clientes_scd d
    SET 
        data_fim = CURRENT_DATE - 1,
        ativo = FALSE
    FROM clientes_staging s
    WHERE d.cliente_id = s.cliente_id
      AND d.ativo = TRUE
      AND (
          d.nome != s.nome OR
          d.email != s.email OR
          d.cidade != s.cidade OR
          d.estado != s.estado OR
          d.categoria != s.categoria
      );
    
    -- Passo 2: Insira novas versões dos registros alterados
    INSERT INTO dim_clientes_scd (
        cliente_id, nome, email, cidade, estado, categoria,
        data_inicio, data_fim, versao, ativo
    )
    SELECT 
        s.cliente_id,
        s.nome,
        s.email,
        s.cidade,
        s.estado,
        s.categoria,
        CURRENT_DATE,
        '9999-12-31'::DATE,
        COALESCE((SELECT MAX(versao) + 1 FROM dim_clientes_scd WHERE cliente_id = s.cliente_id), 1),
        TRUE
    FROM clientes_staging s
    WHERE EXISTS (
        SELECT 1 FROM dim_clientes_scd d
        WHERE d.cliente_id = s.cliente_id
          AND d.data_fim < '9999-12-31'::DATE  -- Foi expirado
    );
    
    -- Passo 3: Insira novos clientes
    INSERT INTO dim_clientes_scd (
        cliente_id, nome, email, cidade, estado, categoria,
        data_inicio, data_fim, versao, ativo
    )
    SELECT 
        s.cliente_id,
        s.nome,
        s.email,
        s.cidade,
        s.estado,
        s.categoria,
        CURRENT_DATE,
        '9999-12-31'::DATE,
        1,
        TRUE
    FROM clientes_staging s
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_clientes_scd d
        WHERE d.cliente_id = s.cliente_id
    );
END;
$$ LANGUAGE plpgsql;
```

5. Execute merge:
```sql
SELECT merge_scd_tipo2();

-- Verifique histórico
SELECT 
    cliente_sk,
    cliente_id,
    nome,
    cidade,
    categoria,
    data_inicio,
    data_fim,
    versao,
    ativo
FROM dim_clientes_scd
ORDER BY cliente_id, versao;
```

---

### Exercício 4.4.2: Carga Incremental com Watermark

**Objetivo:** Carregar apenas dados novos/modificados

**Cenário:** Carga incremental de transações

**Passos:**

1. Crie tabela de controle de watermark:
```sql
CREATE TABLE etl_watermark (
    tabela VARCHAR(100) PRIMARY KEY,
    ultima_atualizacao TIMESTAMP,
    ultima_chave BIGINT,
    proxima_execucao TIMESTAMP
)
DISTRIBUTED RANDOMLY;

-- Inicialize watermark
INSERT INTO etl_watermark VALUES
    ('vendas', '2024-01-01 00:00:00', 0, CURRENT_TIMESTAMP);
```

2. Procedure de carga incremental:
```sql
CREATE OR REPLACE FUNCTION load_vendas_incremental() RETURNS TABLE(
    novas_linhas BIGINT,
    atualizacoes BIGINT
) AS $$
DECLARE
    v_ultimo_ts TIMESTAMP;
    v_ultima_chave BIGINT;
    v_novas BIGINT;
    v_updates BIGINT;
BEGIN
    -- Obter watermark
    SELECT ultima_atualizacao, ultima_chave
    INTO v_ultimo_ts, v_ultima_chave
    FROM etl_watermark
    WHERE tabela = 'vendas';
    
    -- Carregar dados novos da staging
    INSERT INTO vendas_particionada (
        venda_id, data_venda, cliente_id, produto_id, quantidade, valor_total
    )
    SELECT 
        s.venda_id,
        s.data_venda,
        s.cliente_id,
        s.produto_id,
        s.quantidade,
        s.valor
    FROM ext_staging_vendas s
    WHERE s.data_atualizacao > v_ultimo_ts
       OR (s.data_atualizacao = v_ultimo_ts AND s.venda_id > v_ultima_chave)
    ON CONFLICT DO NOTHING;
    
    GET DIAGNOSTICS v_novas = ROW_COUNT;
    
    -- Atualizar watermark
    UPDATE etl_watermark
    SET ultima_atualizacao = (SELECT MAX(data_atualizacao) FROM ext_staging_vendas),
        ultima_chave = (SELECT MAX(venda_id) FROM ext_staging_vendas 
                        WHERE data_atualizacao = (SELECT MAX(data_atualizacao) FROM ext_staging_vendas)),
        proxima_execucao = CURRENT_TIMESTAMP + INTERVAL '1 hour'
    WHERE tabela = 'vendas';
    
    -- Retornar estatísticas
    RETURN QUERY SELECT v_novas, v_updates::BIGINT;
END;
$$ LANGUAGE plpgsql;
```

3. Execute carga incremental:
```sql
SELECT * FROM load_vendas_incremental();
```

---

### Exercício 4.4.3: Data Quality e Validação

**Objetivo:** Implementar validações de qualidade

**Cenário:** Validar dados antes de carregar

**Passos:**

1. Crie tabela de regras de qualidade:
```sql
CREATE TABLE dq_rules (
    rule_id SERIAL PRIMARY KEY,
    tabela VARCHAR(100),
    coluna VARCHAR(100),
    tipo_validacao VARCHAR(50),  -- NOT_NULL, RANGE, PATTERN, FOREIGN_KEY
    parametros JSONB,
    ativo BOOLEAN DEFAULT TRUE
)
DISTRIBUTED RANDOMLY;

-- Defina regras
INSERT INTO dq_rules (tabela, coluna, tipo_validacao, parametros) VALUES
    ('vendas', 'venda_id', 'NOT_NULL', '{}'),
    ('vendas', 'data_venda', 'RANGE', '{"min": "2020-01-01", "max": "2030-12-31"}'),
    ('vendas', 'quantidade', 'RANGE', '{"min": 1, "max": 1000}'),
    ('vendas', 'valor_total', 'RANGE', '{"min": 0, "max": 1000000}'),
    ('vendas', 'cliente_id', 'FOREIGN_KEY', '{"tabela_ref": "dim_clientes", "coluna_ref": "cliente_id"}');
```

2. Function de validação:
```sql
CREATE OR REPLACE FUNCTION validate_data(
    p_tabela VARCHAR,
    p_staging_table VARCHAR
) RETURNS TABLE(
    regra VARCHAR,
    linhas_invalidas BIGINT
) AS $$
DECLARE
    v_rule RECORD;
    v_count BIGINT;
    v_sql TEXT;
BEGIN
    FOR v_rule IN 
        SELECT * FROM dq_rules 
        WHERE tabela = p_tabela AND ativo = TRUE
    LOOP
        CASE v_rule.tipo_validacao
            WHEN 'NOT_NULL' THEN
                v_sql := format('SELECT COUNT(*) FROM %I WHERE %I IS NULL',
                    p_staging_table, v_rule.coluna);
            
            WHEN 'RANGE' THEN
                v_sql := format('SELECT COUNT(*) FROM %I WHERE %I NOT BETWEEN %L AND %L',
                    p_staging_table, v_rule.coluna,
                    v_rule.parametros->>'min', v_rule.parametros->>'max');
            
            WHEN 'FOREIGN_KEY' THEN
                v_sql := format('SELECT COUNT(*) FROM %I s WHERE NOT EXISTS (SELECT 1 FROM %I t WHERE t.%I = s.%I)',
                    p_staging_table,
                    v_rule.parametros->>'tabela_ref',
                    v_rule.parametros->>'coluna_ref',
                    v_rule.coluna);
        END CASE;
        
        EXECUTE v_sql INTO v_count;
        
        IF v_count > 0 THEN
            RETURN QUERY SELECT 
                v_rule.tipo_validacao || ' em ' || v_rule.coluna,
                v_count;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

3. Execute validação:
```sql
-- Valide dados em staging
SELECT * FROM validate_data('vendas', 'staging_vendas');
```

4. Tabela de auditoria:
```sql
CREATE TABLE dq_audit (
    audit_id BIGSERIAL PRIMARY KEY,
    data_validacao TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    tabela VARCHAR(100),
    regra VARCHAR(200),
    linhas_invalidas BIGINT,
    percentual_erro NUMERIC(5,2),
    acao_tomada VARCHAR(50)
)
DISTRIBUTED BY (audit_id);
```

---

### Exercício 4.4.4: Idempotência e Retry Logic

**Objetivo:** Garantir execuções repetíveis

**Cenário:** ETL que pode ser reexecutado sem duplicar dados

**Passos:**

1. Pattern com chave de execução:
```sql
CREATE TABLE etl_batch (
    batch_id VARCHAR(50) PRIMARY KEY,
    inicio TIMESTAMP,
    fim TIMESTAMP,
    status VARCHAR(20),
    registros_processados BIGINT
)
DISTRIBUTED RANDOMLY;

-- Procedure idempotente
CREATE OR REPLACE FUNCTION load_vendas_idempotent(
    p_batch_id VARCHAR
) RETURNS VOID AS $$
BEGIN
    -- Verifica se batch já foi processado
    IF EXISTS (SELECT 1 FROM etl_batch WHERE batch_id = p_batch_id AND status = 'COMPLETED') THEN
        RAISE NOTICE 'Batch % já processado, pulando...', p_batch_id;
        RETURN;
    END IF;
    
    -- Registra início
    INSERT INTO etl_batch (batch_id, inicio, status)
    VALUES (p_batch_id, CURRENT_TIMESTAMP, 'RUNNING')
    ON CONFLICT (batch_id) DO UPDATE
        SET inicio = CURRENT_TIMESTAMP, status = 'RUNNING';
    
    -- Processamento (com DELETE antes de INSERT para idempotência)
    DELETE FROM vendas_particionada
    WHERE data_venda::TEXT IN (
        SELECT DISTINCT data_venda::TEXT 
        FROM ext_staging_vendas 
        WHERE batch_id = p_batch_id
    );
    
    INSERT INTO vendas_particionada
    SELECT * FROM ext_staging_vendas
    WHERE batch_id = p_batch_id;
    
    -- Marca como completo
    UPDATE etl_batch
    SET fim = CURRENT_TIMESTAMP,
        status = 'COMPLETED',
        registros_processados = (SELECT COUNT(*) FROM vendas_particionada)
    WHERE batch_id = p_batch_id;
    
EXCEPTION WHEN OTHERS THEN
    UPDATE etl_batch
    SET status = 'FAILED',
        fim = CURRENT_TIMESTAMP
    WHERE batch_id = p_batch_id;
    
    RAISE;
END;
$$ LANGUAGE plpgsql;
```

2. Execute com batch_id:
```sql
SELECT load_vendas_idempotent('BATCH_20241109_001');

-- Reexecute (será pulado)
SELECT load_vendas_idempotent('BATCH_20241109_001');
```

---

### Exercício 4.4.5: Otimizações de Performance

**Objetivo:** Técnicas para acelerar ETL

**Passos:**

1. **Parallel Insert:**
```sql
-- Insira em partições paralelas
INSERT INTO vendas_particionada
SELECT /*+ PARALLEL(4) */ *
FROM ext_staging_vendas
WHERE data_venda >= '2024-01-01';
```

2. **Batch Processing:**
```sql
-- Processe em lotes
DO $$
DECLARE
    v_offset BIGINT := 0;
    v_batch_size BIGINT := 100000;
    v_total BIGINT;
BEGIN
    SELECT COUNT(*) INTO v_total FROM ext_staging_vendas;
    
    WHILE v_offset < v_total LOOP
        INSERT INTO vendas_particionada
        SELECT * FROM ext_staging_vendas
        LIMIT v_batch_size OFFSET v_offset;
        
        v_offset := v_offset + v_batch_size;
        
        RAISE NOTICE 'Processados % de % registros', v_offset, v_total;
        COMMIT;  -- Commit incremental
    END LOOP;
END $$;
```

3. **Disable Indexes durante carga:**
```sql
-- Antes da carga
DROP INDEX IF EXISTS idx_vendas_data;
DROP INDEX IF EXISTS idx_vendas_cliente;

-- Carga massiva
COPY vendas_particionada FROM '/tmp/huge_file.csv' CSV;

-- Recrie índices
CREATE INDEX idx_vendas_data ON vendas_particionada(data_venda);
CREATE INDEX idx_vendas_cliente ON vendas_particionada(cliente_id);

-- Atualize estatísticas
ANALYZE vendas_particionada;
```

4. **Configurações de sessão:**
```sql
-- Para ETL de longa duração
SET maintenance_work_mem = '2GB';
SET work_mem = '512MB';
SET gp_enable_fast_sri = ON;  -- Fast Single Row Insert
```

---

## Exercício Integrador: Pipeline ETL Completo

**Objetivo:** Implementar pipeline end-to-end

**Cenário:** Pipeline de vendas com todas as técnicas

```sql
-- 1. External Table (Landing)
CREATE EXTERNAL TABLE ext_landing_vendas (
    venda_id BIGINT,
    data_venda TEXT,
    cliente_cpf TEXT,
    produto_codigo TEXT,
    quantidade TEXT,
    valor_unitario TEXT,
    arquivo TEXT
)
LOCATION ('gpfdist://servidor:8080/landing/*.csv')
FORMAT 'CSV' (HEADER DELIMITER ',')
SEGMENT REJECT LIMIT 1000 ROWS;

-- 2. Staging (Validação)
CREATE TABLE staging_vendas_full (
    venda_id BIGINT,
    data_venda DATE,
    cliente_id INTEGER,
    produto_id INTEGER,
    quantidade INTEGER,
    valor_total NUMERIC(12,2),
    batch_id VARCHAR(50),
    data_carga TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
DISTRIBUTED BY (venda_id);

-- 3. Procedure ETL Completo
CREATE OR REPLACE FUNCTION etl_vendas_completo(
    p_batch_id VARCHAR
) RETURNS TABLE(
    fase VARCHAR,
    registros BIGINT,
    status VARCHAR
) AS $$
DECLARE
    v_count BIGINT;
BEGIN
    -- Fase 1: Extract & Load to Staging
    INSERT INTO staging_vendas_full (
        venda_id, data_venda, cliente_id, produto_id, quantidade, valor_total, batch_id
    )
    SELECT 
        e.venda_id,
        e.data_venda::DATE,
        c.cliente_id,
        p.produto_id,
        e.quantidade::INTEGER,
        e.quantidade::INTEGER * e.valor_unitario::NUMERIC,
        p_batch_id
    FROM ext_landing_vendas e
    LEFT JOIN dim_clientes c ON c.cpf = e.cliente_cpf
    LEFT JOIN dim_produtos p ON p.codigo = e.produto_codigo
    WHERE e.arquivo LIKE '%' || p_batch_id || '%';
    
    GET DIAGNOSTICS v_count = ROW_COUNT;
    RETURN QUERY SELECT 'EXTRACT'::VARCHAR, v_count, 'OK'::VARCHAR;
    
    -- Fase 2: Validação
    v_count := (SELECT COUNT(*) FROM staging_vendas_full 
                WHERE batch_id = p_batch_id AND (cliente_id IS NULL OR produto_id IS NULL));
    
    IF v_count > 0 THEN
        RETURN QUERY SELECT 'VALIDATION'::VARCHAR, v_count, 'ERROR'::VARCHAR;
        RETURN;
    END IF;
    
    RETURN QUERY SELECT 'VALIDATION'::VARCHAR, 0::BIGINT, 'OK'::VARCHAR;
    
    -- Fase 3: Load to Target
    INSERT INTO vendas_particionada
    SELECT 
        venda_id, data_venda, cliente_id, produto_id, quantidade, valor_total
    FROM staging_vendas_full
    WHERE batch_id = p_batch_id;
    
    GET DIAGNOSTICS v_count = ROW_COUNT;
    RETURN QUERY SELECT 'LOAD'::VARCHAR, v_count, 'OK'::VARCHAR;
    
    -- Fase 4: Cleanup
    DELETE FROM staging_vendas_full WHERE batch_id = p_batch_id;
    
    RETURN QUERY SELECT 'CLEANUP'::VARCHAR, 0::BIGINT, 'OK'::VARCHAR;
END;
$$ LANGUAGE plpgsql;

-- Executar pipeline
SELECT * FROM etl_vendas_completo('BATCH_20241109');
```

---

## Resumo do Módulo 4

### Habilidades Adquiridas
✅ Carregar dados com COPY (import/export)  
✅ Criar e usar External Tables  
✅ Integrar com gpfdist, S3, HDFS  
✅ Usar gpload para ETL declarativo  
✅ Implementar SCD Tipo 2  
✅ Criar cargas incrementais com watermark  
✅ Validar qualidade de dados  
✅ Garantir idempotência em ETL  
✅ Otimizar performance de cargas  

### Comparação de Métodos

| Método | Velocidade | Complexidade | Uso Ideal |
|--------|-----------|--------------|-----------|
| **INSERT** | Lento | Baixa | Linhas individuais |
| **COPY** | Rápido | Baixa | Cargas batch simples |
| **External Table** | Médio | Média | Staging, integração |
| **gpload** | Rápido | Média | ETL automatizado |
| **gpfdist** | Muito Rápido | Alta | Cargas massivas paralelas |

### Comandos Principais

```sql
-- COPY
COPY tabela FROM '/path/file.csv' CSV HEADER DELIMITER ',';
COPY (SELECT ...) TO '/path/output.csv' CSV HEADER;

-- External Tables
CREATE EXTERNAL TABLE ext_dados (...) 
LOCATION ('gpfdist://host:8080/file.csv') FORMAT 'CSV';

-- gpload (linha de comando)
gpload -f control.yaml -v

-- Validação
SEGMENT REJECT LIMIT 100 ROWS;
LOG ERRORS INTO error_table;

-- Otimização
SET work_mem = '512MB';
ANALYZE tabela;
```

### Padrões de ETL

**Pattern 1: Landing → Staging → Target**
```
External Table (Landing)
    ↓
Staging Table (Validação)
    ↓
Target Table (Production)
```

**Pattern 2: Incremental com Watermark**
```
1. Ler watermark
2. Carregar apenas novos dados
3. Atualizar watermark
```

**Pattern 3: SCD Tipo 2**
```
1. Expire registros alterados
2. Insira novas versões
3. Insira novos registros
```

### Checklist de ETL

**Antes da Carga:**
- [ ] Validar esquema dos arquivos
- [ ] Verificar espaço em disco
- [ ] Testar conectividade (gpfdist, S3)
- [ ] Configurar error handling

**Durante a Carga:**
- [ ] Monitorar progresso
- [ ] Capturar erros
- [ ] Registrar métricas

**Após a Carga:**
- [ ] Executar ANALYZE
- [ ] Validar contagens
- [ ] Verificar qualidade
- [ ] Atualizar controle/audit

### Troubleshooting

| Problema | Causa | Solução |
|----------|-------|---------|
| COPY lento | Arquivo grande, sem compressão | Use gpfdist, comprima arquivos |
| External table não encontra arquivo | Path incorreto | Verifique LOCATION, permissões |
| gpload falha | YAML inválido | Valide sintaxe, logs |
| Duplicação de dados | ETL não idempotente | Use batch_id, DELETE antes de INSERT |
| Validação falha | Dados incompatíveis | Ajuste staging, use SEGMENT REJECT |

### Próximos Passos
No **Módulo 5** (se houver), você aprenderá sobre administração avançada, backup/restore, alta disponibilidade e troubleshooting de performance.

---

**Fim do Módulo 4**
