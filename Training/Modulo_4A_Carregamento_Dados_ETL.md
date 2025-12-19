# Módulo 4: Carregamento de Dados e ETL

**Objetivo:** Dominar técnicas de carregamento massivo de dados, uso de External Tables, padrões de ETL e integração com ferramentas externas no Greenplum.

---

## Índice
1. [Lab 4.1: COPY - Carregamento Básico ](#lab-41-copy---carregamento-básico)
2. [Lab 4.2: External Tables - Leitura e Escrita ](#lab-42-external-tables---leitura-e-escrita)
3. [Lab 4.3: gpload e Ferramentas de Carga ](#lab-43-gpload-e-ferramentas-de-carga)
4. [Lab 4.4: Padrões de ETL e Boas Práticas ](#lab-44-padrões-de-etl-e-boas-práticas)

---

## Lab 4.1: COPY - Carregamento Básico

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

## Lab 4.2: External Tables - Leitura e Escrita

### Objetivos
- Entender o conceito e arquitetura de External Tables
- Criar External Tables para leitura (READABLE)
- Criar External Tables para escrita (WRITABLE)
- Configurar e usar gpfdist para carga paralela
- Tratar erros em External Tables

### Conceitos Abordados

**External Tables** são tabelas que referenciam dados armazenados fora do banco de dados Greenplum. Diferentemente de tabelas regulares, os dados não são armazenados dentro do cluster — eles permanecem em arquivos externos e são lidos/escritos sob demanda.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Arquitetura External Tables (gpfdist)        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐          ┌──────────────────────────────┐    │
│   │  Arquivos    │          │       Greenplum Cluster      │    │
│   │  Externos    │          │                              │    │
│   │              │          │  ┌────────┐                  │    │
│   │ ┌──────────┐ │ gpfdist: │  │Coordin.│ (coordena)       │    │
│   │ │*.csv     │◄├──────────┤  └────────┘                  │    │
│   │ └──────────┘ │ (paralelo│       │                      │    │
│   │     ▲        │  via     │       ▼                      │    │
│   │     │        │ segments)│  ┌────────┐ ┌────────┐       │    │
│   │ ┌───┴────┐   │          │  │ Seg 0  │ │ Seg 1  │ ...   │    │
│   │ │gpfdist │   │◄─────────┼──└────────┘ └────────┘       │    │
│   │ │:8080   │   │          │   (leitura paralela direta)  │    │
│   │ └────────┘   │          │                              │    │
│   │              │          │  Cada segment lê um chunk    │    │
│   │  (serve      │          │  diferente do arquivo        │    │
│   │   chunks)    │          │  = PARALELISMO MÁXIMO        │    │
│   └──────────────┘          └──────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

- **READABLE External Table:** Permite SELECT para ler dados de arquivos externos
- **WRITABLE External Table:** Permite INSERT para gravar dados em arquivos externos
- **Protocolos Suportados:**
  - `gpfdist://` - Servidor de arquivos paralelo do Greenplum (recomendado)
  - `gpfdists://` - gpfdist com SSL
  - `file://` - Arquivos locais no coordinator (requer superuser, não paralelo)

### COPY vs External Table

| Aspecto | COPY | External Table |
|---------|------|----------------|
| **Persistência** | Comando único | Tabela reutilizável (DDL) |
| **Paralelismo** | Via coordinator | Direto nos segments (gpfdist) |
| **Performance** | Boa | Excelente para arquivos grandes |
| **Uso típico** | Cargas ad-hoc | Pipelines ETL, cargas recorrentes |
| **gpfdist** | Não suporta | Suporta nativamente |
| **Metadados** | Não mantém | Mantém schema no catálogo |
| **Joins** | Não diretamente | Sim, como qualquer tabela |

**Quando usar External Tables:**
- ✅ Arquivos grandes (>100MB)
- ✅ Cargas recorrentes/agendadas
- ✅ Pipelines ETL automatizados
- ✅ Joins com dados externos
- ✅ Necessidade de paralelismo máximo

**Quando usar COPY:**
- ✅ Cargas únicas/ad-hoc
- ✅ Arquivos pequenos
- ✅ Simplicidade é prioridade

---

### Exercício 4.2.1: gpfdist - Configuração e Uso

**Objetivo:** Configurar gpfdist e criar External Table com carga paralela

**Cenário:** Carregar arquivo grande de transações usando paralelismo total

O **gpfdist** é o servidor de arquivos paralelo do Greenplum. Cada segment se conecta diretamente ao gpfdist para ler sua porção dos dados, maximizando throughput.

```
┌─────────────────────────────────────────────────────────────┐
│                    Arquitetura gpfdist                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐         ┌─────────────────────────┐      │
│   │   gpfdist    │         │    Greenplum Cluster    │      │
│   │   :8080      │         │                         │      │
│   │              │         │  ┌─────────────────┐    │      │
│   │ ┌──────────┐ │◄────────┼──│    Segment 0    │    │      │
│   │ │ dados    │ │         │  └─────────────────┘    │      │
│   │ │ .csv     │ │◄────────┼──│    Segment 1    │    │      │
│   │ └──────────┘ │         │  └─────────────────┘    │      │
│   │              │◄────────┼──│    Segment 2    │    │      │
│   │  (serve      │         │  └─────────────────┘    │      │
│   │   chunks)    │◄────────┼──│    Segment N    │    │      │
│   │              │         │  └─────────────────┘    │      │
│   └──────────────┘         └─────────────────────────┘      │
│                                                             │
│   Cada segment lê um chunk diferente = PARALELISMO MÁXIMO   │
└─────────────────────────────────────────────────────────────┘
```

**Passos:**

> **Importante:** 
> - Coloque os arquivos no diretório configurado no gpfdist (verifique com `ps aux | grep gpfdist`)
> - Use seu nome de usuário como prefixo nos arquivos, ex: `usuario_transacoes.csv`

1. Crie arquivo de transações para teste (no diretório do gpfdist):

```bash
cat > /tmp/gpfdist_data/usuario_transacoes_gpfdist.csv << 'EOF'
trans_id,cliente_id,produto_id,data_trans,valor,status
1001,101,1,2024-01-15,3500.00,APROVADA
1002,102,2,2024-01-15,150.00,APROVADA
1003,103,3,2024-01-16,450.00,PENDENTE
1004,101,4,2024-01-16,1800.00,APROVADA
1005,104,5,2024-01-17,280.00,CANCELADA
1006,105,6,2024-01-17,400.00,APROVADA
1007,102,7,2024-01-18,350.00,APROVADA
1008,103,1,2024-01-18,3500.00,PENDENTE
1009,106,2,2024-01-19,300.00,APROVADA
1010,101,8,2024-01-19,1200.00,APROVADA
EOF
```

2. Verifique se o gpfdist está rodando (ou inicie se necessário):

```bash
ps aux | grep gpfdist
```

Se não estiver rodando, inicie:

```bash
gpfdist -d /tmp/gpfdist_data -p 8080 -l /tmp/gpfdist.log &
```

> **Parâmetros:**
> - `-d /tmp/gpfdist_data` - Diretório base dos arquivos
> - `-p 8080` - Porta do servidor
> - `-l /tmp/gpfdist.log` - Arquivo de log
> - `&` - Executa em background

3. Teste se o gpfdist está servindo o arquivo:

```bash
curl http://localhost:8080/usuario_transacoes_gpfdist.csv | head -5
```

4. Crie a External Table com protocolo gpfdist://:

```sql
CREATE EXTERNAL TABLE ext_transacoes_gpfdist (
    trans_id INTEGER,
    cliente_id INTEGER,
    produto_id INTEGER,
    data_trans DATE,
    valor NUMERIC(10,2),
    status VARCHAR(20)
)
LOCATION ('gpfdist://hostname:8080/usuario_transacoes_gpfdist.csv')
FORMAT 'CSV' (HEADER DELIMITER ',')
ENCODING 'UTF8';
```

> **Nota:** Substitua `hostname` pelo nome do servidor acessível pelos segments (ex: `worksefaz`, `mdw`, ou IP). Verifique com `hostname` no terminal.

5. Consulte os dados:

```sql
SELECT * FROM ext_transacoes_gpfdist ORDER BY trans_id;
```

6. Verifique distribuição de leitura pelos segments:

```sql
SELECT 
    gp_segment_id,
    COUNT(*) as linhas_lidas
FROM ext_transacoes_gpfdist
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

7. Carregue dados em tabela permanente:

```sql
CREATE TABLE transacoes_carregadas (
    trans_id INTEGER,
    cliente_id INTEGER,
    produto_id INTEGER,
    data_trans DATE,
    valor NUMERIC(10,2),
    status VARCHAR(20)
)
DISTRIBUTED BY (trans_id);
```

```sql
INSERT INTO transacoes_carregadas
SELECT * FROM ext_transacoes_gpfdist;
```

```sql
SELECT COUNT(*) FROM transacoes_carregadas;
```

8. Encerre o gpfdist quando terminar (opcional):

```bash
pkill gpfdist
```

**Referência:** [Documentação oficial gpfdist - VMware Greenplum 7](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-external-g-gpfdist-protocol.html)

---

### Exercício 4.2.2: External Tables com Múltiplos Arquivos

**Objetivo:** Carregar múltiplos arquivos usando wildcards e múltiplos gpfdist

**Cenário:** Dados de vendas particionados por região em arquivos separados

**Passos:**

1. Crie múltiplos arquivos de vendas por região (no diretório do gpfdist):

```bash
cat > /tmp/gpfdist_data/usuario_vendas_norte.csv << 'EOF'
venda_id,regiao,produto,quantidade,valor,data_venda
1,NORTE,Produto_A,10,1500.00,2024-01-10
2,NORTE,Produto_B,5,800.00,2024-01-11
3,NORTE,Produto_C,8,1200.00,2024-01-12
EOF
```

```bash
cat > /tmp/gpfdist_data/usuario_vendas_sul.csv << 'EOF'
venda_id,regiao,produto,quantidade,valor,data_venda
4,SUL,Produto_A,15,2250.00,2024-01-10
5,SUL,Produto_B,12,1920.00,2024-01-11
6,SUL,Produto_D,20,3000.00,2024-01-12
EOF
```

```bash
cat > /tmp/gpfdist_data/usuario_vendas_leste.csv << 'EOF'
venda_id,regiao,produto,quantidade,valor,data_venda
7,LESTE,Produto_B,8,1280.00,2024-01-10
8,LESTE,Produto_C,6,900.00,2024-01-11
9,LESTE,Produto_A,11,1650.00,2024-01-12
EOF
```

2. Verifique se gpfdist está rodando:

```bash
ps aux | grep gpfdist
```

3. Crie External Table usando wildcard para ler todos os arquivos:

```sql
CREATE EXTERNAL TABLE ext_vendas_todas_regioes (
    venda_id INTEGER,
    regiao VARCHAR(20),
    produto VARCHAR(50),
    quantidade INTEGER,
    valor NUMERIC(10,2),
    data_venda DATE
)
LOCATION ('gpfdist://hostname:8080/usuario_vendas_*.csv')
FORMAT 'CSV' (HEADER DELIMITER ',')
ENCODING 'UTF8';
```

> **Nota:** O wildcard `usuario_vendas_*.csv` irá ler todos os arquivos que começam com `usuario_vendas_`.

4. Consulte dados consolidados de todas as regiões:

```sql
SELECT * FROM ext_vendas_todas_regioes ORDER BY venda_id;
```

5. Agregue por região:

```sql
SELECT 
    regiao,
    COUNT(*) as total_vendas,
    SUM(quantidade) as qtd_total,
    SUM(valor) as valor_total
FROM ext_vendas_todas_regioes
GROUP BY regiao
ORDER BY valor_total DESC;
```

6. **Alternativa - Múltiplas URLs gpfdist** (para alta disponibilidade):

```sql
-- Exemplo com múltiplos servidores gpfdist
-- (requer gpfdist rodando em múltiplos hosts)
CREATE EXTERNAL TABLE ext_vendas_multi_gpfdist (
    venda_id INTEGER,
    regiao VARCHAR(20),
    produto VARCHAR(50),
    quantidade INTEGER,
    valor NUMERIC(10,2),
    data_venda DATE
)
LOCATION (
    'gpfdist://host1:8080/vendas_*.csv',
    'gpfdist://host2:8080/vendas_*.csv'
)
FORMAT 'CSV' (HEADER DELIMITER ',')
ENCODING 'UTF8';
```

> **Nota:** Com múltiplas URLs, o Greenplum distribui a carga entre os servidores gpfdist, aumentando throughput e tolerância a falhas.

**Padrões de wildcard suportados:**
- `*.csv` - Todos os arquivos .csv
- `vendas_*.csv` - Arquivos começando com "vendas_"
- `data_202[0-9]*.csv` - Arquivos de 2020-2029

---

### Exercício 4.2.3: WRITABLE External Table - Exportação

**Objetivo:** Criar External Table para exportar dados em paralelo

**Cenário:** Exportar relatório de vendas para arquivo externo via pipeline ETL

**Importante:** WRITABLE External Tables só permitem INSERT (não SELECT). A cláusula `DISTRIBUTED BY` é obrigatória.

**Passos:**

1. Verifique se gpfdist está rodando:

```bash
ps aux | grep gpfdist
```

> **Nota:** O gpfdist suporta escrita (WRITABLE) por padrão. Não é necessário parâmetro especial.

2. Crie a WRITABLE External Table:

```sql
CREATE WRITABLE EXTERNAL TABLE ext_relatorio_export (
    regiao VARCHAR(20),
    produto VARCHAR(50),
    total_vendas BIGINT,
    valor_total NUMERIC(12,2),
    data_geracao DATE
)
LOCATION ('gpfdist://hostname:8080/usuario_relatorio_vendas_export.csv')
FORMAT 'CSV' (DELIMITER ',' HEADER)
DISTRIBUTED BY (regiao);
```

3. Exporte dados agregados para o arquivo:

```sql
INSERT INTO ext_relatorio_export
SELECT 
    regiao,
    produto,
    COUNT(*) as total_vendas,
    SUM(valor) as valor_total,
    CURRENT_DATE as data_geracao
FROM ext_vendas_todas_regioes
GROUP BY regiao, produto
ORDER BY regiao, valor_total DESC;
```

4. Verifique o arquivo gerado no servidor:

```bash
cat /tmp/gpfdist_data/usuario_relatorio_vendas_export.csv
```

5. **Exemplo prático - Pipeline ETL completo:**

```sql
-- Tabela de staging para transformações
CREATE TABLE stg_vendas_processadas (
    regiao VARCHAR(20),
    mes DATE,
    total_vendas BIGINT,
    receita_total NUMERIC(12,2),
    ticket_medio NUMERIC(10,2)
)
DISTRIBUTED BY (regiao);
```

```sql
-- Processa e insere na staging
INSERT INTO stg_vendas_processadas
SELECT 
    regiao,
    DATE_TRUNC('month', data_venda)::DATE as mes,
    COUNT(*) as total_vendas,
    SUM(valor) as receita_total,
    AVG(valor)::NUMERIC(10,2) as ticket_medio
FROM ext_vendas_todas_regioes
GROUP BY regiao, DATE_TRUNC('month', data_venda);
```

```sql
-- WRITABLE External Table para export final
CREATE WRITABLE EXTERNAL TABLE ext_kpi_mensal_export (
    regiao VARCHAR(20),
    mes DATE,
    total_vendas BIGINT,
    receita_total NUMERIC(12,2),
    ticket_medio NUMERIC(10,2)
)
LOCATION ('gpfdist://hostname:8080/usuario_kpi_mensal.csv')
FORMAT 'CSV' (DELIMITER ',' HEADER)
DISTRIBUTED BY (regiao);
```

```sql
-- Exporta KPIs processados
INSERT INTO ext_kpi_mensal_export
SELECT * FROM stg_vendas_processadas;
```

```bash
cat /tmp/gpfdist_data/usuario_kpi_mensal.csv
```

**WRITABLE External Table - Considerações:**
- ✅ Exportação paralela (cada segment escreve simultaneamente)
- ✅ Ideal para pipelines ETL automatizados
- ✅ Integração com sistemas externos
- ❌ Não permite SELECT (só INSERT)
- ❌ `DISTRIBUTED BY` é obrigatório
- ⚠️ Arquivo é sobrescrito a cada INSERT (não append)

---

### Exercício 4.2.4: Error Handling em External Tables

**Objetivo:** Configurar tratamento de erros para cargas robustas

**Cenário:** Carregar arquivo com dados inconsistentes sem falhar completamente

**Passos:**

1. Crie arquivo com erros propositais (no diretório do gpfdist):

```bash
cat > /tmp/gpfdist_data/usuario_clientes_com_erros.csv << 'EOF'
cliente_id,nome,idade,salario,data_cadastro
1,Ana Silva,28,5500.00,2024-01-15
2,Bruno Costa,IDADE_INVALIDA,4200.00,2024-01-16
3,Carlos Lima,35,3800.00,DATA_ERRADA
4,Diana Souza,42,6100.00,2024-01-18
5,,25,3000.00,2024-01-19
6,Eduardo Reis,31,SALARIO_ERRADO,2024-01-20
7,Fernanda Luz,29,4800.00,2024-01-21
8,Gustavo Alves,abc,5200.00,2024-01-22
EOF
```

2. Crie External Table COM tratamento de erros:

```sql
CREATE EXTERNAL TABLE ext_clientes_com_erros (
    cliente_id INTEGER,
    nome VARCHAR(100),
    idade INTEGER,
    salario NUMERIC(10,2),
    data_cadastro DATE
)
LOCATION ('gpfdist://hostname:8080/usuario_clientes_com_erros.csv')
FORMAT 'CSV' (HEADER DELIMITER ',')
ENCODING 'UTF8'
LOG ERRORS
SEGMENT REJECT LIMIT 5 ROWS;
```

> **Parâmetros de Error Handling:**
> - `LOG ERRORS` - Registra erros na tabela de erros interna
> - `SEGMENT REJECT LIMIT 5 ROWS` - Aceita até 5 linhas com erro por segment

3. Carregue os dados (linhas válidas serão processadas):

```sql
SELECT * FROM ext_clientes_com_erros;
```

4. Consulte os erros registrados:

```sql
SELECT 
    cmdtime,
    relname,
    filename,
    linenum,
    errmsg,
    rawdata
FROM gp_read_error_log('ext_clientes_com_erros')
ORDER BY linenum;
```

5. Analise estatísticas de erro:

```sql
SELECT 
    COUNT(*) as total_erros,
    COUNT(DISTINCT linenum) as linhas_com_erro
FROM gp_read_error_log('ext_clientes_com_erros');
```

6. Carregue apenas dados válidos em tabela permanente:

```sql
CREATE TABLE clientes_validos (
    cliente_id INTEGER,
    nome VARCHAR(100),
    idade INTEGER,
    salario NUMERIC(10,2),
    data_cadastro DATE
)
DISTRIBUTED BY (cliente_id);
```

```sql
INSERT INTO clientes_validos
SELECT * FROM ext_clientes_com_erros;
```

```sql
SELECT * FROM clientes_validos ORDER BY cliente_id;
```

7. Limpe o log de erros quando não precisar mais:

```sql
SELECT gp_truncate_error_log('ext_clientes_com_erros');
```

8. **Alternativa - SEGMENT REJECT LIMIT por porcentagem:**

```sql
CREATE EXTERNAL TABLE ext_clientes_percent (
    cliente_id INTEGER,
    nome VARCHAR(100),
    idade INTEGER,
    salario NUMERIC(10,2),
    data_cadastro DATE
)
LOCATION ('gpfdist://hostname:8080/usuario_clientes_com_erros.csv')
FORMAT 'CSV' (HEADER DELIMITER ',')
LOG ERRORS
SEGMENT REJECT LIMIT 10 PERCENT;
```

**Opções de SEGMENT REJECT LIMIT:**
- `SEGMENT REJECT LIMIT n ROWS` - Máximo de n linhas com erro por segment
- `SEGMENT REJECT LIMIT n PERCENT` - Máximo de n% de erros do total

**Funções de Error Log:**
- `gp_read_error_log('nome_tabela')` - Lê erros registrados
- `gp_truncate_error_log('nome_tabela')` - Limpa log de erros
- `gp_truncate_error_log('*')` - Limpa todos os logs de erro

---

### Boas Práticas - External Tables

**Quando usar cada protocolo:**

| Protocolo | Uso Recomendado | Performance |
|-----------|-----------------|-------------|
| `gpfdist://` | Produção, arquivos grandes | ⭐⭐⭐⭐⭐ (paralelo) |
| `gpfdists://` | Produção com SSL | ⭐⭐⭐⭐ |
| `file://` | Apenas superuser, testes | ⭐ (sem paralelismo) |

**Checklist de Boas Práticas:**

✅ **FAÇA:**
- Use `gpfdist://` para arquivos maiores que 100MB
- Configure `SEGMENT REJECT LIMIT` para tolerância a erros
- Use wildcards para carregar múltiplos arquivos
- Distribua arquivos entre múltiplos servidores gpfdist
- Execute `ANALYZE` após carregar dados em tabelas permanentes
- Use `LOG ERRORS` para diagnóstico de problemas
- Defina `ENCODING` explicitamente

❌ **EVITE:**
- Usar `file://` em produção (não escala, requer superuser)
- WRITABLE External Table sem `DISTRIBUTED BY` explícito
- Arquivos muito pequenos (overhead > benefício)
- External Tables sem tratamento de erros em produção

**Performance Tips:**
- Arquivos entre 100MB-1GB têm melhor performance
- Comprima arquivos grandes com gzip (gpfdist descomprime automaticamente)
- Use múltiplos gpfdist em hosts diferentes para máximo throughput
- Nomeie arquivos com padrões consistentes para usar wildcards

**Referência Oficial:**
- [External Tables - VMware Greenplum 7](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/admin_guide-external-g-external-tables.html)
- [gpfdist - VMware Greenplum 7](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/utility_guide-ref-gpfdist.html)

---

### Limpeza do Lab 4.2

```sql
-- Remove External Tables criadas (substitua 'usuario' pelo seu prefixo)
DROP EXTERNAL TABLE IF EXISTS ext_transacoes_gpfdist;
DROP EXTERNAL TABLE IF EXISTS ext_vendas_todas_regioes;
DROP EXTERNAL TABLE IF EXISTS ext_vendas_multi_gpfdist;
DROP EXTERNAL TABLE IF EXISTS ext_relatorio_export;
DROP EXTERNAL TABLE IF EXISTS ext_kpi_mensal_export;
DROP EXTERNAL TABLE IF EXISTS ext_clientes_com_erros;
DROP EXTERNAL TABLE IF EXISTS ext_clientes_percent;
```

```sql
-- Remove tabelas auxiliares (substitua 'usuario' pelo seu prefixo)
DROP TABLE IF EXISTS usuario_transacoes_carregadas;
DROP TABLE IF EXISTS stg_vendas_processadas;
DROP TABLE IF EXISTS clientes_validos;
```

```bash
# Remove arquivos de teste (substitua 'usuario' pelo seu prefixo)
rm -f /tmp/gpfdist_data/usuario_transacoes_gpfdist.csv
rm -f /tmp/gpfdist_data/usuario_vendas_*.csv
rm -f /tmp/gpfdist_data/usuario_clientes_com_erros.csv
rm -f /tmp/gpfdist_data/usuario_relatorio_*.csv
rm -f /tmp/gpfdist_data/usuario_kpi_*.csv
```

```bash
# Para o gpfdist se ainda estiver rodando
pkill gpfdist
```

---

### Próximos Passos
No **Lab 4.3** veremos sobre gpload e Ferramentas de Carga automatizadas.

---

## Lab 4.3: gpload e Ferramentas de Carga

### Objetivos
- Entender o gpload e suas vantagens sobre External Tables manuais
- Criar arquivos de controle YAML para cargas automatizadas
- Usar diferentes modos de carga: INSERT, UPDATE, MERGE
- Configurar transformações e mapeamento de colunas
- Implementar tratamento de erros e logs

### Conceitos Abordados

O **gpload** é uma ferramenta de linha de comando que automatiza o processo de carga de dados no Greenplum. Ele gerencia internamente o gpfdist e as External Tables, simplificando pipelines ETL.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Arquitetura gpload                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐    ┌─────────────┐    ┌──────────────────┐   │
│   │  Arquivo     │    │   gpload    │    │    Greenplum     │   │
│   │  YAML        │────│             │────│    Database      │   │
│   │  (controle)  │    │  ┌───────┐  │    │                  │   │
│   └──────────────┘    │  │gpfdist│  │    │  ┌────────────┐  │   │
│                       │  │(auto) │  │    │  │Target Table│  │   │
│   ┌──────────────┐    │  └───────┘  │    │  └────────────┘  │   │
│   │  Arquivos    │────│             │    │                  │   │
│   │  de Dados    │    │  ┌───────┐  │    │  ┌────────────┐  │   │
│   │  (CSV, TXT)  │    │  │ExtTab │  │    │  │ Staging    │  │   │
│   └──────────────┘    │  │(temp) │  │    │  │ (opcional) │  │   │
│                       │  └───────┘  │    │  └────────────┘  │   │
│                       └─────────────┘    └──────────────────┘   │
│                                                                 │
│   gpload automatiza: gpfdist + External Table + INSERT/UPDATE   │
└─────────────────────────────────────────────────────────────────┘
```

- **gpload:** Utilitário de carga automatizada baseado em YAML
- **Arquivo YAML:** Define fonte, destino, mapeamento e comportamento
- **Modos de Carga:**
  - `INSERT` - Insere novos registros
  - `UPDATE` - Atualiza registros existentes
  - `MERGE` - Insere novos e atualiza existentes (UPSERT)

### gpload vs External Tables Manuais

| Aspecto | External Table Manual | gpload |
|---------|----------------------|--------|
| **Configuração** | DDL + comandos SQL | Arquivo YAML único |
| **gpfdist** | Gerenciar manualmente | Automático |
| **Modos de carga** | Apenas INSERT | INSERT, UPDATE, MERGE |
| **Transformações** | Via SQL | Via YAML (COLUMNS) |
| **Reuso** | Recriar External Table | Reusar arquivo YAML |
| **Logs** | Manual | Automático |
| **Ideal para** | Cargas ad-hoc | Pipelines ETL |

**Quando usar gpload:**
- ✅ Cargas recorrentes/agendadas
- ✅ Necessidade de UPDATE ou MERGE
- ✅ Pipelines ETL automatizados
- ✅ Controle de versão dos jobs de carga

**Quando usar External Tables:**
- ✅ Cargas ad-hoc únicas
- ✅ Joins com dados externos
- ✅ Maior controle sobre o processo

---

### Exercício 4.3.1: gpload - Estrutura do Arquivo YAML

**Objetivo:** Entender a estrutura do arquivo de controle YAML do gpload

**Cenário:** Criar arquivo YAML para carga de dados de produtos

**Estrutura básica do arquivo YAML:**

```yaml
---
VERSION: 1.0.0.1
DATABASE: nome_banco
USER: usuario
HOST: localhost
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - hostname
         PORT: 8080
         FILE:
           - /caminho/arquivo.csv
    - FORMAT: csv
    - DELIMITER: ','
    - HEADER: true
    - ENCODING: UTF8
    - ERROR_LIMIT: 25
    - LOG_ERRORS: true
   OUTPUT:
    - TABLE: schema.tabela_destino
    - MODE: INSERT
   PRELOAD:
    - TRUNCATE: false
    - REUSE_TABLES: true
```

**Passos:**

1. Crie a tabela de destino para produtos:

```sql
CREATE TABLE produtos_gpload (
    produto_id INTEGER,
    nome VARCHAR(100),
    categoria VARCHAR(50),
    preco NUMERIC(10,2),
    estoque INTEGER,
    ativo BOOLEAN,
    data_cadastro DATE
)
DISTRIBUTED BY (produto_id);
```

2. Crie o arquivo de dados (no diretório do gpfdist):

```bash
cat > /tmp/gpfdist_data/usuario_produtos_gpload.csv << 'EOF'
produto_id,nome,categoria,preco,estoque,ativo,data_cadastro
1,Notebook Dell,Eletrônicos,3500.00,50,true,2024-01-15
2,Mouse Logitech,Periféricos,150.00,200,true,2024-01-16
3,Teclado Mecânico,Periféricos,450.00,80,true,2024-01-17
4,Monitor 27pol,Eletrônicos,1800.00,30,true,2024-01-18
5,Webcam HD,Periféricos,280.00,120,true,2024-01-19
6,SSD 1TB,Armazenamento,400.00,150,true,2024-01-20
7,Headset Gamer,Periféricos,350.00,90,false,2024-01-21
8,Impressora Laser,Eletrônicos,1200.00,25,true,2024-01-22
EOF
```

3. Crie o arquivo YAML de controle:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_gpload.yml << 'EOF'
---
VERSION: 1.0.0.1
DATABASE: db_usuario
USER: usuario
HOST: localhost
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - worksefaz
         PORT: 8081
         FILE:
           - /tmp/gpfdist_data/usuario_produtos_gpload.csv
    - FORMAT: csv
    - DELIMITER: ','
    - HEADER: true
    - QUOTE: '"'
    - ENCODING: UTF8
    - ERROR_LIMIT: 10
    - LOG_ERRORS: true
   OUTPUT:
    - TABLE: public.produtos_gpload
    - MODE: INSERT
   PRELOAD:
    - TRUNCATE: true
    - REUSE_TABLES: false
EOF
```

> **Nota:** Substitua `db_usuario`, `usuario` e `worksefaz` pelos valores do seu ambiente.

**Seções principais do YAML:**

| Seção | Descrição |
|-------|-----------|
| `VERSION` | Versão do formato (sempre 1.0.0.1) |
| `DATABASE` | Banco de dados destino |
| `USER` | Usuário de conexão |
| `INPUT` | Configuração da fonte de dados |
| `OUTPUT` | Configuração do destino |
| `PRELOAD` | Ações antes da carga |
| `SQL` | Comandos SQL antes/depois (opcional) |

---

### Exercício 4.3.2: gpload - Carga Básica com INSERT

**Objetivo:** Executar carga de dados usando gpload

**Cenário:** Carregar produtos usando o arquivo YAML criado

**Passos:**

1. Verifique se o gpfdist **NÃO** está rodando na porta 8081:

```bash
ps aux | grep gpfdist
```

> **Importante:** O gpload inicia seu próprio gpfdist automaticamente. Se já houver um gpfdist na mesma porta, haverá conflito.

2. Execute o gpload:

```bash
gpload -f /tmp/gpfdist_data/usuario_produtos_gpload.yml
```

3. Verifique os dados carregados:

```sql
SELECT * FROM produtos_gpload ORDER BY produto_id;
```

```sql
SELECT 
    categoria,
    COUNT(*) as qtd_produtos,
    SUM(estoque) as estoque_total,
    AVG(preco)::NUMERIC(10,2) as preco_medio
FROM produtos_gpload
GROUP BY categoria
ORDER BY estoque_total DESC;
```

4. Execute novamente e observe o TRUNCATE:

```bash
gpload -f /tmp/gpfdist_data/usuario_produtos_gpload.yml
```

```sql
SELECT COUNT(*) FROM produtos_gpload;
```

> **Nota:** Como `TRUNCATE: true`, a tabela é limpa antes de cada carga.

**Opções úteis do gpload:**

| Opção | Descrição |
|-------|-----------|
| `-f arquivo.yml` | Especifica arquivo de controle |
| `-l logfile` | Define arquivo de log |
| `-v` | Modo verbose |
| `-V` | Modo muito verbose (debug) |
| `--gpfdist_timeout` | Timeout para gpfdist |

---

### Exercício 4.3.3: gpload - Modos UPDATE e MERGE

**Objetivo:** Usar gpload para atualizar e fazer UPSERT de dados

**Cenário:** Atualizar preços e estoque de produtos existentes

**Passos:**

1. Crie arquivo com dados atualizados:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_update.csv << 'EOF'
produto_id,nome,categoria,preco,estoque,ativo,data_cadastro
1,Notebook Dell,Eletrônicos,3200.00,45,true,2024-01-15
2,Mouse Logitech,Periféricos,140.00,180,true,2024-01-16
3,Teclado Mecânico,Periféricos,420.00,75,true,2024-01-17
9,Placa de Vídeo,Eletrônicos,2500.00,20,true,2024-02-01
10,Memória RAM 16GB,Armazenamento,350.00,100,true,2024-02-02
EOF
```

2. Crie arquivo YAML para modo UPDATE:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_update.yml << 'EOF'
---
VERSION: 1.0.0.1
DATABASE: db_usuario
USER: usuario
HOST: localhost
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - worksefaz
         PORT: 8081
         FILE:
           - /tmp/gpfdist_data/usuario_produtos_update.csv
    - FORMAT: csv
    - DELIMITER: ','
    - HEADER: true
    - ENCODING: UTF8
    - ERROR_LIMIT: 10
   OUTPUT:
    - TABLE: public.produtos_gpload
    - MODE: UPDATE
    - MATCH_COLUMNS:
       - produto_id
    - UPDATE_COLUMNS:
       - preco
       - estoque
   PRELOAD:
    - REUSE_TABLES: true
EOF
```

> **Parâmetros UPDATE:**
> - `MATCH_COLUMNS`: Colunas usadas para encontrar registros (WHERE)
> - `UPDATE_COLUMNS`: Colunas que serão atualizadas

3. Execute o UPDATE:

```bash
gpload -f /tmp/gpfdist_data/usuario_produtos_update.yml
```

4. Verifique as atualizações (produtos 1, 2, 3 devem ter novos preços):

```sql
SELECT produto_id, nome, preco, estoque 
FROM produtos_gpload 
WHERE produto_id IN (1, 2, 3, 9, 10)
ORDER BY produto_id;
```

> **Nota:** Produtos 9 e 10 **NÃO** foram inseridos porque MODE: UPDATE só atualiza existentes.

5. Crie arquivo YAML para modo MERGE (UPSERT):

```bash
cat > /tmp/gpfdist_data/usuario_produtos_merge.yml << 'EOF'
---
VERSION: 1.0.0.1
DATABASE: db_usuario
USER: usuario
HOST: localhost
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - worksefaz
         PORT: 8081
         FILE:
           - /tmp/gpfdist_data/usuario_produtos_update.csv
    - FORMAT: csv
    - DELIMITER: ','
    - HEADER: true
    - ENCODING: UTF8
    - ERROR_LIMIT: 10
   OUTPUT:
    - TABLE: public.produtos_gpload
    - MODE: MERGE
    - MATCH_COLUMNS:
       - produto_id
    - UPDATE_COLUMNS:
       - preco
       - estoque
   PRELOAD:
    - REUSE_TABLES: true
EOF
```

> **MERGE:** Atualiza se existe (MATCH_COLUMNS), insere se não existe. As colunas não listadas em UPDATE_COLUMNS são usadas no INSERT de novos registros.

6. Execute o MERGE:

```bash
gpload -f /tmp/gpfdist_data/usuario_produtos_merge.yml
```

7. Verifique - agora produtos 9 e 10 devem existir:

```sql
SELECT produto_id, nome, preco, estoque 
FROM produtos_gpload 
ORDER BY produto_id;
```

**Comparativo dos modos:**

| Modo | Comportamento | Uso típico |
|------|---------------|------------|
| `INSERT` | Apenas insere novos | Carga inicial, append |
| `UPDATE` | Apenas atualiza existentes | Atualização de dados |
| `MERGE` | Insere novos + atualiza existentes | Sincronização (UPSERT) |

---

### Exercício 4.3.4: gpload - Transformações com SQL BEFORE/AFTER

**Objetivo:** Aplicar transformações durante a carga com gpload usando tabela staging e SQL

**Cenário:** Carregar dados em tabela staging e transformar para tabela final

> **Nota GP7:** O parâmetro `MAPPING` foi removido no Greenplum 7. A abordagem recomendada é usar tabela staging + seção `SQL` com comandos `BEFORE` e `AFTER`.

**Passos:**

1. Crie arquivo CSV de origem:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_transform.csv << 'EOF'
cod,descricao,valor
101,câmera dslr,4500.00
102,tripé profissional,800.00
103,flash externo,650.00
EOF
```

2. Crie a tabela de staging (estrutura igual ao arquivo):

```sql
CREATE TABLE produtos_staging (
    cod INTEGER,
    descricao VARCHAR(100),
    valor NUMERIC(10,2)
)
DISTRIBUTED BY (cod);
```

3. Crie a tabela de destino final com colunas derivadas:

```sql
CREATE TABLE produtos_transformados (
    produto_id INTEGER,
    nome_upper VARCHAR(100),
    preco_com_imposto NUMERIC(12,2),
    data_carga TIMESTAMP,
    origem VARCHAR(50)
)
DISTRIBUTED BY (produto_id);
```

4. Crie arquivo YAML com SQL AFTER para transformações:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_transform.yml << 'EOF'
---
VERSION: 1.0.0.1
DATABASE: db_usuario
USER: usuario
HOST: localhost
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - worksefaz
         PORT: 8081
         FILE:
           - /tmp/gpfdist_data/usuario_produtos_transform.csv
    - COLUMNS:
       - cod: integer
       - descricao: text
       - valor: numeric
    - FORMAT: csv
    - DELIMITER: ','
    - HEADER: true
    - ENCODING: UTF8
   OUTPUT:
    - TABLE: public.produtos_staging
    - MODE: INSERT
   PRELOAD:
    - TRUNCATE: true
   SQL:
    - BEFORE: "TRUNCATE TABLE produtos_transformados"
    - AFTER: "INSERT INTO produtos_transformados SELECT cod, UPPER(descricao), valor * 1.18, CURRENT_TIMESTAMP, 'gpload_import' FROM produtos_staging"
EOF
```

> **SQL BEFORE:** Executado antes da carga (preparar ambiente).
> **SQL AFTER:** Executado após a carga (transformar e mover dados).
> **Transformações aplicadas:** `UPPER()` no nome, cálculo de imposto (18%), timestamp e valor fixo.

5. Execute a carga com transformação:

```bash
gpload -f /tmp/gpfdist_data/usuario_produtos_transform.yml
```

6. Verifique os dados na tabela staging:

```sql
SELECT * FROM produtos_staging ORDER BY cod;
```

7. Verifique os dados transformados na tabela final:

```sql
SELECT * FROM produtos_transformados ORDER BY produto_id;
```

**Opções da seção SQL:**

| Recurso | Descrição | Exemplo |
|---------|-----------|---------|
| `BEFORE` | SQL executado antes da carga | `"TRUNCATE TABLE destino"` |
| `AFTER` | SQL executado após a carga | `"INSERT INTO final SELECT..."` |
| Múltiplos | Lista de comandos | `- BEFORE: "cmd1"` e `- BEFORE: "cmd2"` |

**Padrão Staging + Transform:**

```
┌─────────────┐    gpload    ┌─────────────┐    SQL AFTER    ┌─────────────┐
│ Arquivo CSV │ ──────────► │   Staging   │ ─────────────► │    Final    │
│  (origem)   │             │  (raw data) │   (transform)   │ (processed) │
└─────────────┘             └─────────────┘                 └─────────────┘
```

---

### Exercício 4.3.5: gpload - Error Handling e Logs

**Objetivo:** Configurar tratamento de erros e logging no gpload

**Cenário:** Carregar arquivo com erros e analisar logs

**Passos:**

1. Crie arquivo com dados problemáticos:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_erros.csv << 'EOF'
produto_id,nome,categoria,preco,estoque,ativo,data_cadastro
201,Produto OK,Teste,100.00,10,true,2024-04-01
202,Produto Erro Preco,Teste,INVALIDO,20,true,2024-04-02
203,Outro OK,Teste,150.00,15,true,2024-04-03
204,Erro Data,Teste,200.00,5,true,DATA_ERRADA
205,Mais Um OK,Teste,250.00,8,true,2024-04-05
206,Erro Estoque,Teste,300.00,abc,true,2024-04-06
EOF
```

2. Crie arquivo YAML com error handling:

```bash
cat > /tmp/gpfdist_data/usuario_produtos_erros.yml << 'EOF'
---
VERSION: 1.0.0.1
DATABASE: db_usuario
USER: usuario
HOST: localhost
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - worksefaz
         PORT: 8081
         FILE:
           - /tmp/gpfdist_data/usuario_produtos_erros.csv
    - FORMAT: csv
    - DELIMITER: ','
    - HEADER: true
    - ENCODING: UTF8
    - ERROR_LIMIT: 5
    - LOG_ERRORS: true
   OUTPUT:
    - TABLE: public.produtos_gpload
    - MODE: INSERT
   PRELOAD:
    - REUSE_TABLES: true
EOF
```

> **Parâmetros de Error Handling:**
> - `ERROR_LIMIT: 5` - Aceita até 5 erros antes de falhar
> - `LOG_ERRORS: true` - Registra erros na tabela de erros

3. Execute o gpload com logging:

```bash
gpload -f /tmp/gpfdist_data/usuario_produtos_erros.yml -l /tmp/gpload_erros.log -v
```

4. Verifique quantas linhas foram carregadas:

```sql
SELECT COUNT(*) as total, 
       COUNT(CASE WHEN produto_id >= 201 THEN 1 END) as novos_produtos
FROM produtos_gpload;
```

5. Verifique o log de execução:

```bash
cat /tmp/gpload_erros.log
```

6. Consulte erros na tabela de erros (se LOG_ERRORS: true):

```sql
SELECT 
    relname,
    linenum,
    errmsg,
    rawdata
FROM gp_read_error_log('ext_gpload_*')
ORDER BY linenum;
```

> **Nota:** O gpload cria External Tables temporárias com prefixo `ext_gpload_`.

**Opções de Error Handling:**

| Parâmetro | Descrição |
|-----------|-----------|
| `ERROR_LIMIT: n` | Número máximo de erros tolerados |
| `LOG_ERRORS: true` | Registra erros em tabela interna |
| `ERROR_TABLE: tabela` | Tabela customizada para erros (obsoleto no GP7) |

---

### Boas Práticas - gpload

**Checklist de Boas Práticas:**

✅ **FAÇA:**
- Sempre defina `ERROR_LIMIT` para tolerância a falhas
- Use `LOG_ERRORS: true` para diagnóstico
- Defina `REUSE_TABLES: true` para cargas frequentes (performance)
- Use modo `MERGE` para sincronização de dados
- Versione seus arquivos YAML no controle de versão
- Teste com `-v` (verbose) antes de produção
- Defina `ENCODING` explicitamente

❌ **EVITE:**
- Deixar gpfdist rodando manualmente na mesma porta
- Usar `TRUNCATE: true` sem confirmação em produção
- `ERROR_LIMIT` muito alto (pode mascarar problemas)
- Caminhos relativos nos arquivos YAML

**Dicas de Performance:**
- Use `REUSE_TABLES: true` para evitar recriar External Tables
- Agrupe múltiplos arquivos usando wildcards: `FILE: - /path/*.csv`
- Para tabelas grandes, considere particionar a carga
- Execute `ANALYZE` após cargas grandes

**Referência Oficial:**
- [gpload - VMware Greenplum 7](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/utility_guide-ref-gpload.html)

---

### Limpeza do Lab 4.3

```sql
-- Remove tabelas criadas
DROP TABLE IF EXISTS produtos_gpload;
DROP TABLE IF EXISTS produtos_staging;
DROP TABLE IF EXISTS produtos_transformados;
```

```bash
# Remove arquivos de teste (substitua 'usuario' pelo seu prefixo)
rm -f /tmp/gpfdist_data/usuario_produtos_gpload.csv
rm -f /tmp/gpfdist_data/usuario_produtos_gpload.yml
rm -f /tmp/gpfdist_data/usuario_produtos_update.csv
rm -f /tmp/gpfdist_data/usuario_produtos_update.yml
rm -f /tmp/gpfdist_data/usuario_produtos_merge.yml
rm -f /tmp/gpfdist_data/usuario_produtos_transform.csv
rm -f /tmp/gpfdist_data/usuario_produtos_transform.yml
rm -f /tmp/gpfdist_data/usuario_produtos_erros.csv
rm -f /tmp/gpfdist_data/usuario_produtos_erros.yml
rm -f /tmp/gpload_erros.log
```

---

### Próximos Passos
No **Lab 4.4** veremos sobre Padrões de ETL e Boas Práticas.

---

**Fim do Módulo 4**
