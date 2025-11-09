# Módulo 1: Preparação do Ambiente

**Duração Total:** 45-60 minutos  
**Objetivo:** Familiarizar os participantes com o ambiente Greenplum, conexões básicas e comandos de navegação.

---

## Lab 1.1: Conexão ao Cluster Greenplum (15-20 min)

### Objetivos
- Compreender os componentes de conexão do Greenplum
- Estabelecer conexão via psql
- Verificar conectividade e credenciais
- Entender mensagens de conexão e prompt do psql

### Pré-requisitos
- Cliente psql instalado
- Credenciais de acesso fornecidas (host, porta, database, usuário, senha)
- Acesso de rede ao cluster Greenplum

### Conceitos Abordados
- **Master Node:** Ponto de entrada para conexões
- **Porta padrão:** 5432 (pode variar)
- **psql:** Cliente de linha de comando PostgreSQL/Greenplum
- **String de conexão:** Formato de conexão ao banco

---

### Exercício 1.1.1: Primeira Conexão via psql

**Objetivo:** Conectar ao cluster Greenplum pela primeira vez

**Passos:**

1. Abra o terminal/prompt de comando

2. Execute o comando de conexão:
```bash
psql -h <hostname> -p <porta> -d <database> -U <usuario>
```

**Exemplo:**
```bash
psql -h gpmaster.empresa.com -p 5432 -d training_db -U gpadmin
```

3. Digite a senha quando solicitado

4. Observe o prompt do Greenplum:
```
training_db=#
```

**Saída Esperada:**
```
psql (9.4.24)
Type "help" for help.

training_db=#
```

**Perguntas para Reflexão:**
- O que significa o símbolo `#` no prompt?
- Qual a diferença entre `#` e `>` no psql?

---

### Exercício 1.1.2: Conexão com String de Conexão Completa

**Objetivo:** Usar formato URI de conexão

**Passos:**

1. Teste a conexão usando string URI:
```bash
psql "postgresql://usuario:senha@hostname:porta/database"
```

**Exemplo:**
```bash
psql "postgresql://gpadmin:senha123@gpmaster.empresa.com:5432/training_db"
```

2. Compare com o método anterior

**Nota de Segurança:**
⚠️ Evite colocar senhas diretamente em comandos em ambientes produtivos. Use variáveis de ambiente ou arquivo `.pgpass`

---

### Exercício 1.1.3: Configuração de Variáveis de Ambiente

**Objetivo:** Simplificar conexões futuras

**Passos:**

1. Configure variáveis de ambiente (Linux/Mac):
```bash
export PGHOST=gpmaster.empresa.com
export PGPORT=5432
export PGDATABASE=training_db
export PGUSER=gpadmin
```

**Windows (PowerShell):**
```powershell
$env:PGHOST="gpmaster.empresa.com"
$env:PGPORT="5432"
$env:PGDATABASE="training_db"
$env:PGUSER="gpadmin"
```

2. Após configurar, conecte simplesmente com:
```bash
psql
```

**Resultado Esperado:**
- Conexão automática sem precisar especificar parâmetros

---

## Lab 1.2: Verificação da Configuração e Status do Cluster (15-20 min)

### Objetivos
- Verificar versão do Greenplum
- Identificar configuração do cluster
- Entender arquitetura master/segment
- Consultar informações do sistema

### Conceitos Abordados
- **Master vs Segments:** Arquitetura distribuída
- **gp_segment_configuration:** Tabela de configuração
- **System Catalogs:** Metadados do Greenplum

---

### Exercício 1.2.1: Verificar Versão do Greenplum

**Objetivo:** Identificar a versão instalada

**Comandos SQL:**

1. Verifique a versão completa:
```sql
SELECT version();
```

**Saída Esperada:**
```
PostgreSQL 9.4.24 (Greenplum Database 6.x.x build ...)
```

2. Versão simplificada:
```sql
SELECT gp_version();
```

3. Via comando psql:
```
\! psql --version
```

**Análise:**
- Qual versão do PostgreSQL está sendo usada?
- Qual versão do Greenplum?
- Por que isso é importante?

---

### Exercício 1.2.2: Consultar Configuração do Cluster

**Objetivo:** Entender a topologia do cluster

**Comandos SQL:**

1. Visualize todos os segments:
```sql
SELECT * FROM gp_segment_configuration 
ORDER BY content, role;
```

2. Conte quantos segments existem:
```sql
SELECT 
    role,
    COUNT(*) as total
FROM gp_segment_configuration
GROUP BY role
ORDER BY role;
```

3. Identifique segments ativos:
```sql
SELECT 
    content,
    role,
    hostname,
    port,
    status
FROM gp_segment_configuration
WHERE status = 'u'  -- 'u' = up
ORDER BY content;
```

**Interpretação dos Resultados:**
- **content = -1:** Master node
- **content >= 0:** Segment primário
- **role = 'p':** Primary segment
- **role = 'm':** Mirror segment
- **status = 'u':** Up (ativo)

**Perguntas para Reflexão:**
- Quantos segments primários existem?
- Existem mirrors configurados?
- Todos os segments estão ativos?

---

### Exercício 1.2.3: Informações do Sistema e Banco de Dados

**Objetivo:** Coletar informações básicas do ambiente

**Comandos SQL:**

1. Database atual:
```sql
SELECT current_database();
```

2. Usuário conectado:
```sql
SELECT current_user;
```

3. Data/hora do servidor:
```sql
SELECT now();
```

4. Configurações importantes:
```sql
SELECT name, setting, unit 
FROM pg_settings 
WHERE name IN (
    'max_connections',
    'shared_buffers',
    'gp_vmem_protect_limit'
)
ORDER BY name;
```

5. Tamanho do database:
```sql
SELECT 
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;
```

---

## Lab 1.3: Comandos Básicos de Navegação (psql/cliente) (15-20 min)

### Objetivos
- Dominar comandos meta do psql
- Navegar entre schemas e objetos
- Explorar estruturas do banco
- Conhecer comandos de ajuda

### Conceitos Abordados
- **Meta-comandos:** Comandos iniciados com `\` no psql
- **Schemas:** Organização lógica de objetos
- **System Catalogs:** pg_catalog, information_schema
- **Tab completion:** Autocompletar no psql

---

### Exercício 1.3.1: Comandos Meta Básicos do psql

**Objetivo:** Conhecer comandos essenciais de navegação

**Comandos:**

1. Listar todos os databases:
```
\l
```

2. Conectar a outro database:
```
\c <nome_database>
```

3. Listar tabelas:
```
\dt
```

4. Descrever estrutura de uma tabela:
```
\d <nome_tabela>
```

5. Ajuda sobre comandos:
```
\?
```

6. Sair do psql:
```
\q
```

---

### Exercício 1.3.2: Explorando Informações do Greenplum

**Objetivo:** Consultar metadados específicos do Greenplum

**Passos:**

1. Consulte informações sobre tabelas do usuário:
```sql
SELECT 
    schemaname,
    tablename,
    tableowner
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'gp_toolkit')
ORDER BY schemaname, tablename;
```

2. Verifique a configuração de distribuição de uma tabela (conceito importante no Greenplum):
```sql
SELECT 
    tablename,
    schemaname
FROM pg_tables
WHERE schemaname = 'public'
LIMIT 5;
```

---

## Exercício Integrador: Checklist de Conexão

**Objetivo:** Consolidar todos os conhecimentos do módulo

**Tarefa:** Crie um script de verificação que execute as seguintes validações:

```sql
-- Verificação Completa do Ambiente Greenplum
-- ============================================

-- 1. Versão
SELECT 'Versão do Greenplum:' as info, version() as detalhes;

-- 2. Database atual
SELECT 'Database Conectado:' as info, current_database() as detalhes;

-- 3. Usuário
SELECT 'Usuário:' as info, current_user as detalhes;

-- 4. Total de Segments
SELECT 
    'Total de Segments:' as info,
    COUNT(*) as detalhes
FROM gp_segment_configuration
WHERE role = 'p' AND content >= 0;

-- 5. Status do Cluster
SELECT 
    'Status do Cluster:' as info,
    CASE 
        WHEN COUNT(*) = 0 THEN 'Todos segments UP'
        ELSE 'Existem segments DOWN: ' || COUNT(*)::text
    END as detalhes
FROM gp_segment_configuration
WHERE status != 'u';

-- 6. Schemas disponíveis
SELECT 'Schemas Disponíveis:' as info, string_agg(nspname, ', ') as detalhes
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%' AND nspname != 'information_schema';
```

**Salve este script como:** `verificacao_ambiente.sql`

**Execute:**
```
\i verificacao_ambiente.sql
```

---

## Resumo do Módulo 1

### Habilidades Adquiridas
✅ Conectar ao cluster Greenplum via psql  
✅ Verificar versão e configuração do cluster  
✅ Identificar master e segments  
✅ Navegar entre databases e schemas  
✅ Usar comandos básicos do psql  
✅ Consultar informações do sistema  

### Comandos Principais
| Comando | Descrição |
|---------|-----------|
| `psql -h host -p porta -d db -U user` | Conectar ao Greenplum |
| `SELECT version();` | Ver versão |
| `\l` | Listar databases |
| `\c database` | Trocar database |
| `\dt` | Listar tabelas |
| `\d tabela` | Descrever tabela |
| `\q` | Sair |
| `SELECT * FROM gp_segment_configuration;` | Ver configuração do cluster |

### Próximos Passos
No **Módulo 2**, você aprenderá a criar e modificar estruturas de banco de dados (DDL), incluindo schemas, databases e tabelas com distribuição específica do Greenplum.

---

## Notas do Instrutor

### Pontos de Atenção
- [ ] Garantir que todos conseguiram conectar
- [ ] Validar credenciais antes da aula
- [ ] Preparar ambiente com databases de exemplo
- [ ] Testar conectividade de rede
- [ ] Ter backup de acesso via ferramenta gráfica (DBeaver, pgAdmin)

### Dicas de Apresentação
- Demonstre ao vivo cada conexão
- Mostre erros comuns e como resolver
- Incentive uso de tab completion
- Explique visualmente a arquitetura master/segments

### Materiais de Apoio
- Diagrama da arquitetura do cluster
- Cheat sheet de comandos psql
- Troubleshooting de conexão

---

**Fim do Módulo 1**
