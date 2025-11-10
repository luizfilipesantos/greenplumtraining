# Treinamento Greenplum Database

Treinamento completo e hands-on para Greenplum Database 7.5.4, cobrindo desde conceitos fundamentais até técnicas avançadas de otimização e manutenção.

---

## Sumário

- [Visão Geral](#-visão-geral)
- [Objetivos do Treinamento](#-objetivos-do-treinamento)
- [Estrutura dos Módulos](#-estrutura-dos-módulos)
  - [Módulo 1: Preparação do Ambiente](#módulo-1-preparação-do-ambiente)
  - [Módulo 2: DDL e Estratégias de Distribuição](#módulo-2-ddl-e-estratégias-de-distribuição)
  - [Módulo 3: Particionamento, Índices e Otimização](#módulo-3-particionamento-índices-e-otimização)
  - [Módulo 4: Carregamento de Dados e ETL](#módulo-4-carregamento-de-dados-e-etl)
  - [Módulo 5A: Manutenção e VACUUM](#módulo-5a-manutenção-e-vacuum)
  - [Módulo 5B: Detecção de Skew e EXPLAIN Avançado](#módulo-5b-detecção-de-skew-e-explain-avançado)
- [Ferramentas e Tecnologias](#️-ferramentas-e-tecnologias)
- [Metodologia](#-metodologia)
- [Público-Alvo](#-público-alvo)
- [Estrutura do Repositório](#-estrutura-do-repositório)
- [Como Usar Este Treinamento](#-como-usar-este-treinamento)
- [Dicas de Estudo](#-dicas-de-estudo)
- [Recursos Adicionais](#-recursos-adicionais)
- [Contribuições](#-contribuições)
- [Licença](#-licença)
- [Sobre o Treinamento](#-sobre-o-treinamento)

---

##  Visão Geral

Este treinamento foi desenvolvido para capacitar profissionais em **Greenplum Database**, um sistema de banco de dados analítico massivamente paralelo (MPP) baseado em PostgreSQL. O curso combina teoria com exercícios práticos extensivos, preparando os participantes para cenários reais de uso.

**Duração Total:** 8-10 horas  
**Nível:** Intermediário a Avançado  
**Pré-requisitos:** Conhecimento básico de SQL e PostgreSQL

---

## 🎯 Objetivos do Treinamento

Ao concluir este treinamento, você será capaz de:

- ✅ Configurar e navegar no ambiente Greenplum
- ✅ Projetar tabelas distribuídas otimizadas
- ✅ Implementar estratégias eficientes de particionamento
- ✅ Executar cargas de dados de alto volume
- ✅ Realizar manutenção preventiva e corretiva
- ✅ Detectar e corrigir problemas de desempenho
- ✅ Otimizar queries complexas usando EXPLAIN

---

## 📚 Estrutura dos Módulos

### **Módulo 1: Preparação do Ambiente**
**Duração:** 45-60 minutos

Fundamentos do Greenplum e preparação do ambiente de trabalho.

**Tópicos principais:**
- Arquitetura MPP (Master, Segments, Interconnect)
- Instalação e configuração
- Ferramentas essenciais (psql, gpfdist, gppkg)
- Navegação e catálogos do sistema
- Primeiras queries e verificações

---

### **Módulo 2: DDL e Estratégias de Distribuição**
**Duração:** 90-120 minutos

Design de tabelas e estratégias de distribuição de dados.

**Tópicos principais:**
- Políticas de distribuição (HASH, REPLICATED, RANDOM)
- Orientação de armazenamento (Heap vs AO/AOCO)
- Algoritmos de compressão (zstd, zlib, RLE)
- Escolha de distribution keys
- Tabelas temporárias e externas
- Comparação de performance entre estratégias

**Laboratórios:**
- Criação de tabelas com diferentes distribuições
- Análise comparativa de compressão
- Otimização de schemas para analytics

---

### **Módulo 3: Particionamento, Índices e Otimização**
**Duração:** 120-150 minutos

Técnicas avançadas de organização e acesso a dados.

**Tópicos principais:**
- Particionamento RANGE e LIST
- Multi-level partitioning
- Partition pruning e otimização
- Tipos de índices (B-tree, BRIN, Bitmap)
- Quando usar (e não usar) índices no Greenplum
- Análise básica de EXPLAIN

**Laboratórios:**
- Implementação de particionamento temporal
- Gerenciamento de partições (ADD, DROP, SPLIT)
- Criação e análise de índices
- Otimização de queries com partition pruning

---

### **Módulo 4: Carregamento de Dados e ETL**
**Duração:** 120-150 minutos

Estratégias de carga massiva e padrões de ETL.

**Tópicos principais:**
- COPY para cargas rápidas
- External Tables (gpfdist, file, http)
- gpload e arquivos de controle YAML
- Cargas incrementais e full
- Padrões de ETL (SCD Type 1 e 2, Watermark)
- Tratamento de erros e logging
- Pipelines automatizados

**Laboratórios:**
- Carga via COPY de arquivos CSV
- Configuração de external tables com gpfdist
- Implementação de SCD Type 2
- Pipeline completo de ETL automatizado

---

### **Módulo 5A: Manutenção e VACUUM**
**Duração:** 60-75 minutos

Manutenção preventiva e gerenciamento de bloat.

**Tópicos principais:**
- MVCC (Multi-Version Concurrency Control)
- Dead tuples e table bloat
- VACUUM vs VACUUM FULL vs VACUUM ANALYZE
- VACUUM FREEZE e transaction wraparound
- Manutenção de tabelas AO/AOCO
- Tuning de parâmetros (maintenance_work_mem, vacuum_cost_delay)
- Monitoramento e automação

**Laboratórios:**
- Visualização de dead tuples
- Execução de diferentes tipos de VACUUM
- Detecção e quantificação de bloat
- Implementação de sistema de monitoramento
- Criação de scripts de manutenção automatizada

---

### **Módulo 5B: Detecção de Skew e EXPLAIN Avançado**
**Duração:** 90-120 minutos

Diagnóstico e otimização avançada de performance.

**Tópicos principais:**

**Data Skew:**
- Detecção com gp_toolkit.gp_skew_coefficients
- Análise de distribution keys
- Correção de distribuição desbalanceada
- Monitoramento contínuo

**Processing Skew:**
- Diferença entre data skew e processing skew
- Identificação de bottlenecks por segmento
- Análise de JOINs problemáticos
- Otimização de redistribuição

**EXPLAIN Avançado:**
- Anatomia completa do plano de execução
- Motion Nodes (Gather, Redistribute, Broadcast)
- Join Methods (Hash, Nested Loop, Merge)
- Scan Methods (Sequential, Index, Bitmap)
- Slices e paralelização
- Otimização baseada em planos

**Laboratórios:**
- Simulação e detecção de data skew
- Correção de tabelas desbalanceadas
- Identificação de processing skew em queries
- Análise profunda de EXPLAIN ANALYZE
- Diagnóstico completo end-to-end

---

## 🛠️ Ferramentas e Tecnologias

- **Greenplum Database:** 7.5.4
- **PostgreSQL Base:** 12.12
- **Ferramentas:** psql, gpfdist, gpload, gppkg
- **Linguagens:** SQL, PL/pgSQL
- **Utilitários:** gp_toolkit, pg_stat_statements

---

## 📊 Metodologia

Cada módulo segue uma estrutura consistente:

1. **Conceitos Teóricos:** Explicação clara e objetiva
2. **Exercícios Práticos:** Hands-on com cenários reais
3. **Exemplos de Código:** Scripts prontos para uso
4. **Troubleshooting:** Guias de resolução de problemas
5. **Resumo:** Checklist de habilidades adquiridas

---

## 🎓 Público-Alvo

- Database Administrators (DBAs)
- Data Engineers
- Analytics Engineers
- Desenvolvedores que trabalham com Big Data
- Arquitetos de Dados
- Profissionais de BI e Analytics

---

## 📁 Estrutura do Repositório

```
greenplumtraining/
├── README.md
├── LICENSE.md
├── Setup/
│   ├── conectar.md          # Guia de conexão
│   └── sefaz_users.md       # Configuração de usuários
└── Training/
    ├── Modulo_1_Preparacao_Ambiente.md
    ├── Modulo_2_DDL_Tabelas_Distribuicao.md
    ├── Modulo_3_Particionamento_Indices_Otimizacao.md
    ├── Modulo_4_Carregamento_Dados_ETL.md
    ├── Modulo_5A_Manutencao_Vacuum.md
    └── Modulo_5B_Skew_Explain.md
```

---

## 🚀 Como Usar Este Treinamento

1. **Clone o repositório:**
   ```bash
   git clone https://github.com/luizfilipesantos/greenplumtraining.git
   cd greenplumtraining
   ```

2. **Siga a ordem dos módulos:**
   - Comece pelo Módulo 1 e avance sequencialmente
   - Cada módulo constrói sobre conceitos do anterior

3. **Execute os exercícios:**
   - Todos os scripts SQL estão prontos para execução
   - Adapte conforme seu ambiente

4. **Pratique:**
   - Refaça exercícios com variações
   - Experimente com seus próprios dados

---

## 💡 Dicas de Estudo

- **Hands-on é essencial:** Execute todos os exercícios
- **Não pule módulos:** A progressão é intencional
- **Anote suas observações:** Documente aprendizados
- **Experimente variações:** Mude parâmetros e observe resultados
- **Use em produção com cautela:** Sempre teste em ambiente de desenvolvimento

---

## 📖 Recursos Adicionais

- [Documentação Oficial Greenplum](https://docs.vmware.com/en/VMware-Greenplum/index.html)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Greenplum GitHub](https://github.com/greenplum-db/gpdb)
- [Greenplum Community](https://greenplum.org/)

---

## 🤝 Contribuições

Contribuições são bem-vindas! Sinta-se à vontade para:

- Reportar erros ou inconsistências
- Sugerir melhorias nos exercícios
- Adicionar novos casos de uso
- Compartilhar suas experiências

---

## 📄 Licença

Este projeto está sob a licença especificada no arquivo LICENSE.md.

---

## ✨ Sobre o Treinamento

Este treinamento foi desenvolvido com foco em **aplicabilidade prática** e **cenários reais**. Cada exercício foi testado e validado, representando situações comuns em ambientes de produção com Greenplum.

**Bom treinamento e sucesso na sua jornada com Greenplum Database!** 🚀
