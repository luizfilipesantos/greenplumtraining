# 🐘 Treinamento Greenplum - Laboratório Prático

![License](https://img.shields.io/badge/License-Proprietary-red.svg)
![Greenplum](https://img.shields.io/badge/Greenplum-6.x-green.svg)
![Status](https://img.shields.io/badge/Status-Active-success.svg)

## 📋 Sobre o Treinamento

Treinamento prático de **Greenplum Database** focado em usuários iniciantes, tanto em bancos de dados quanto em Greenplum especificamente. Este material oferece uma abordagem hands-on com laboratórios práticos que cobrem desde conceitos fundamentais até features avançadas específicas do Greenplum.

### 🎯 Público-Alvo

- Iniciantes em bancos de dados relacionais
- Profissionais migrando para Greenplum
- Desenvolvedores e analistas de dados
- DBAs em transição para ambientes MPP (Massively Parallel Processing)

### 🚀 Pré-requisitos

- Conhecimento básico de SQL (queries serão fornecidas nos exercícios)
- Acesso a um cluster Greenplum (produção, desenvolvimento ou sandbox)
- Cliente psql instalado
- Terminal/prompt de comando

---

## 📚 Estrutura do Curso

### **Módulo 1: Preparação do Ambiente** *(45-60 min)*
- Conexão ao cluster Greenplum
- Verificação da configuração e status
- Comandos básicos de navegação

### **Módulo 2: Fundamentos de DDL** *(75-90 min)*
- Criação de schemas e databases
- Criação de tabelas
- Tipos de distribuição (DISTRIBUTED BY vs RANDOMLY)
- Modificação de estruturas (ALTER TABLE)

### **Módulo 3: Manipulação de Dados - DML** *(120-150 min)*
- Inserção de dados (INSERT)
- Carregamento via COPY
- Carregamento usando gpload
- Atualização e exclusão (UPDATE/DELETE)
- Importação de arquivos CSV

### **Módulo 4: Características Específicas do Greenplum** *(150-180 min)*
- Análise de distribuição de dados
- Particionamento de tabelas
- External Tables
- Programação procedural (Functions e Procedures)
- Window Functions
- Verificação de data skew
- Comandos de manutenção (ANALYZE, VACUUM)

### **Módulo 5: Monitoramento Básico** *(60-75 min)*
- Consultas de sistema (pg_stat_activity)
- Análise de planos de execução (EXPLAIN)
- Identificação de gargalos

### **Módulo 6: Projeto Prático Final** *(90-120 min)*
- Cenário completo com dataset real
- Criação de estrutura, carga e análises
- Otimização e boas práticas

---

## ⏱️ Cronograma Sugerido

O treinamento foi projetado para ser ministrado em **6 sessões de até 2 horas cada**:

| Sessão | Duração | Conteúdo |
|--------|---------|----------|
| **Dia 1** | 2h | Módulos 1 e 2 |
| **Dia 2** | 2h | Módulo 3 (Parte 1) |
| **Dia 3** | 1h30min | Módulo 3 (Parte 2) + Módulo 4 (Início) |
| **Dia 4** | 2h | Módulo 4 (Continuação) |
| **Dia 5** | 2h | Módulo 4 (Final) + Módulo 5 |
| **Dia 6** | 2h | Módulo 6 (Projeto Final) |

**Duração Total:** 11h30min

---

## 🛠️ Tecnologias e Ferramentas

- **Greenplum Database** 6.x ou superior
- **PostgreSQL** (base do Greenplum)
- **psql** - Cliente de linha de comando
- **gpload** - Ferramenta de carregamento paralelo
- Opcional: DBeaver, pgAdmin ou outro cliente gráfico

---

## 📖 Como Usar Este Material

### Para Estudantes

1. Clone ou faça fork deste repositório
2. Siga os módulos em ordem sequencial
3. Execute todos os exercícios propostos
4. Complete o projeto final para consolidar o aprendizado

### Para Instrutores

1. Revise todo o material antes de ministrar
2. Prepare o ambiente Greenplum com antecedência
3. Adapte exemplos conforme necessário para seu contexto
4. Utilize as notas do instrutor em cada módulo

---

## 📁 Estrutura do Repositório

```
Greenplum_Training/
├── LICENSE.md                          # Licença do material
├── README.md                           # Este arquivo
├── Modulo_1_Preparacao_Ambiente.md    # Módulo 1 completo
├── Modulo_2_Fundamentos_DDL.md        # (Em desenvolvimento)
├── Modulo_3_Manipulacao_Dados.md      # (Em desenvolvimento)
├── Modulo_4_Features_Greenplum.md     # (Em desenvolvimento)
├── Modulo_5_Monitoramento.md          # (Em desenvolvimento)
├── Modulo_6_Projeto_Final.md          # (Em desenvolvimento)
└── datasets/                           # Dados de exemplo
    └── (arquivos CSV, SQL, etc.)
```

---

## 🎓 Habilidades Desenvolvidas

Ao completar este treinamento, você será capaz de:

✅ Conectar e navegar em um cluster Greenplum  
✅ Criar e gerenciar estruturas de dados (DDL)  
✅ Carregar dados usando diferentes métodos (COPY, gpload)  
✅ Trabalhar com particionamento e distribuição de dados  
✅ Criar External Tables para integração de dados  
✅ Desenvolver procedures e functions em PL/pgSQL  
✅ Monitorar e otimizar queries  
✅ Identificar e resolver problemas de performance  
✅ Aplicar boas práticas em ambientes Greenplum  

---

## 📄 Licença

**Este material possui licença proprietária com permissão de uso educacional.**

- ✅ **Permitido:** Uso pessoal e educacional para aprendizado
- ❌ **Proibido:** Uso comercial sem autorização

Para uso comercial, treinamentos corporativos ou consultorias, entre em contato para licenciamento.

Veja [LICENSE.md](LICENSE.md) para detalhes completos.

---

## 📧 Contato

Para dúvidas, sugestões ou solicitação de licenciamento comercial:

- **Email:** [seu-email@exemplo.com]
- **LinkedIn:** [seu-perfil]
- **GitHub:** [seu-usuario]

---

## 🙏 Agradecimentos

Este material foi desenvolvido com base em experiências práticas e documentação oficial do Greenplum Database.

---

## 🔄 Atualizações

- **v1.0** - Outubro 2025: Lançamento inicial com Módulo 1

---

**© 2025 [SEU NOME/EMPRESA]. Todos os direitos reservados.**
