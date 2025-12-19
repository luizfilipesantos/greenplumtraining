# Template de Documentação para Labs - Greenplum Training

Este documento define a estrutura padrão para documentação de laboratórios de treinamento para cases e cenários reais enviados pelos clientes, baseado no modelo desenvolvido para o Lab 4.4.1.

A documentação quando utilizar este template, precisa ser mais conceitual, ou seja, não criar exercícios e não fornecer exemplos de comando SQL ou de qualquer tipo.

Não é necessário informar isso no documento, ou seja, não é necessário colocar comentários como "gerando dados conceituais sem comandos sql" ou algo do tipo.

Todos os diagramas precisam ser mermaid.

Sempre procurar montar o diagrama de forma que se viabilize uma boa experiência para o usuário, evitando diagramas que cresçam demais para os lados tornando a visualização muito pequena. Se for preciso, quebrar os diagramas em partes menores e fazer um diagrama 'pai' juntando as partes filhas.



---

## Estrutura Geral do Documento

Um Lab bem documentado deve conter as seguintes seções, nesta ordem:

```
## Lab X.X.X: [Título Descritivo do Lab]
├── Contexto do Problema
├── O Dilema / Problema Central / A descrição do case enviado pelo cliente.
├── Análise para Tomada de Decisão (se aplicável)
├── Arquitetura Proposta
├── Justificativas para as Decisões
├── Implementação Técnica - Visão Geral
├── Resumo das Estratégias
├── Conceitos Abordados
└── Considerações Finais
```

---

## Descrição de Cada Seção

### 1. Título do Lab

**Formato:** `## Lab X.X.X: [Padrão/Técnica] - [Sistema/Contexto]`

**Exemplo:** `## Lab 4.4.1: Padrão CDC - Arrecadação Oracle`

O título deve indicar claramente:
- O padrão ou técnica principal abordada
- O sistema ou contexto de negócio utilizado como exemplo

---

### 2. Contexto do Problema

**Objetivo:** Apresentar o cenário real que motivou a necessidade da solução.

**Elementos obrigatórios:**
- **Cenário Real:** Uma frase descrevendo o sistema e a necessidade
- **Tabela de Aspectos:** Informações estruturadas sobre o cenário

**Tabela de Aspectos (modelo):**

| Aspecto | Descrição |
|---------|-----------|
| **Origem** | Sistema e tabelas de origem |
| **Estrutura** | Tipo de estrutura (Mestre-Detalhe, Tabelão, etc.) |
| **Volumetria** | Quantidade de registros (mensal e total) |
| **Campo de Controle** | Campo(s) usado(s) para identificar alterações |
| **Processo Atual** | Fluxo resumido (ex: Oracle → CSV → gpload → GP) |

**Desafios Identificados:** Lista numerada dos principais problemas a resolver.

---

### 3. O Dilema / Problema Central (quando aplicável)

**Objetivo:** Explicar o trade-off ou conflito central que a solução precisa resolver.

**Quando usar:** Quando houver duas ou mais abordagens conflitantes, onde cada uma tem vantagens e desvantagens.

**Formato recomendado:** Diagrama Mermaid comparativo mostrando:
- As opções conflitantes com prós e contras
- A solução proposta
- O resultado esperado

**Exemplo de estrutura Mermaid:**
```
flowchart LR
    subgraph Opcao1["Opção A"]
        prós e contras
    end
    subgraph Opcao2["Opção B"]
        prós e contras
    end
    subgraph Solucao["Solução"]
        abordagem escolhida
    end
    subgraph Resultado["Resultado"]
        benefícios alcançados
    end
```

---

### 4. Análise para Tomada de Decisão (quando aplicável)

**Objetivo:** Apresentar dados ou análises que fundamentam as decisões arquiteturais.

**Quando usar:** Quando a decisão depende de análise de dados históricos, métricas ou comportamento do sistema.

**Formato recomendado:**
- Explicação do que será analisado
- Query ou método de análise (se relevante)
- Tabela com resultados esperados/típicos
- Conclusão objetiva baseada nos dados

---

### 5. Arquitetura Proposta

**Objetivo:** Apresentar visualmente a solução técnica de forma conceitual.

**Formato obrigatório:** Diagrama Mermaid (flowchart) mostrando:
- Sistemas/componentes envolvidos
- Fluxo de dados entre eles
- Transformações ou processos principais

**Boas práticas:**
- Usar cores para diferenciar sistemas (Oracle = vermelho, Greenplum = verde)
- Agrupar componentes relacionados em subgraphs
- Indicar volumetrias quando relevante
- Usar ícones/emojis para facilitar identificação visual

---

### 6. Justificativas para as Decisões

**Objetivo:** Explicar o "porquê" de cada decisão arquitetural.

**Formato obrigatório:** Tabela com duas colunas:

| Decisão | Justificativa |
|---------|---------------|
| Nome da decisão | Explicação do motivo e benefício |

**Importante:** Cada decisão técnica relevante deve ter uma justificativa clara. O leitor deve entender não apenas "o que" foi decidido, mas "por que".

---

### 7. Implementação Técnica - Visão Geral

**Objetivo:** Mostrar como a arquitetura conceitual se traduz em implementação real.

**Formato recomendado:** Diagrama Mermaid detalhado mostrando:
- Servidores/ambientes físicos envolvidos
- Caminhos de arquivos e diretórios
- Ferramentas utilizadas (gpfdist, gpload, scripts)
- Sequência numerada das etapas
- Fluxo de transferência de arquivos

**Nível de detalhe:** Suficiente para que um desenvolvedor entenda onde cada componente se encaixa, sem entrar em código específico.

---

### 8. Resumo das Estratégias

**Objetivo:** Consolidar as principais estratégias em formato de referência rápida.

**Formato obrigatório:** Tabela com três colunas:

| Estratégia | Benefício | Impacto |
|------------|-----------|---------|
| Nome da estratégia | O que ela resolve | Ganho quantificável (quando possível) |

**Boas práticas:**
- Usar métricas quando disponíveis (ex: "-91% registros no merge")
- Manter descrições concisas
- Ordenar por importância ou sequência lógica

---

### 9. Conceitos Abordados

**Objetivo:** Listar os conceitos técnicos que o leitor aprenderá com este lab.

**Formato:** Lista com marcadores, onde cada item tem:
- **Nome do conceito em negrito:** Breve descrição

**Exemplo:**
- **Janela Adaptativa**: Análise histórica para definir período ideal de extração
- **Hash de Registro**: Identificação eficiente de mudanças sem comparação campo a campo

---

### 10. Considerações Finais

**Objetivo:** Abordar aspectos operacionais, de manutenção e cuidados que vão além da implementação inicial.

**Formato:** Subseções com títulos em H4 (####), contendo texto descritivo SEM código.

**Tópicos recomendados (quando aplicáveis):**
- **Manutenção:** VACUUM, estatísticas, monitoramento de bloat
- **Impacto no Sistema de Origem:** Horários, índices, paralelismo
- **Tratamento de Falhas:** Idempotência, transações, rollback
- **Política de Retenção:** Histórico, arquivamento, compressão
- **Validação e Qualidade:** Verificações pós-carga, integridade
- **Escalabilidade:** Como o processo se comporta com crescimento

**Importante:** Esta seção deve ser puramente conceitual e descritiva, sem blocos de código.

---

## Diagramas Mermaid - Padrões Visuais

### Cores por Tipo de Sistema

| Sistema | Fill Color | Stroke Color |
|---------|------------|--------------|
| Oracle/Origem | #ffcccc | #cc0000 |
| File Server/Transfer | #fff3cd | #ffc107 |
| Greenplum Master | #cce5ff | #004085 |
| Greenplum Cluster | #c3e6cb ou #ccffcc | #28a745 ou #00cc00 |
| Processamento/Delta | #ffffcc | #cccc00 |
| Scripts/Controle | #e2d5f1 | #6f42c1 |

### Emojis Recomendados

| Elemento | Emoji |
|----------|-------|
| Oracle/Database | 🔴 ou 🗄️ |
| Greenplum | 🟢 ou 🐘 |
| Arquivo/CSV | 📄 |
| Diretório | 📂 |
| Query/Script | 📜 |
| Processo | ⚡ ou ▶️ |
| Staging | 📥 |
| Raw/Final | 💾 ou 📊 |
| Controle | 📈 |
| Servidor | 🖥️ |
| Comparação | 🔍 |
| Merge | 🔄 |

---

## Checklist de Revisão

Antes de finalizar um Lab, verifique:

- [ ] O título é claro e indica padrão + contexto?
- [ ] O contexto do problema tem tabela de aspectos completa?
- [ ] Os desafios estão listados de forma numerada?
- [ ] Há diagrama conceitual da arquitetura proposta?
- [ ] Todas as decisões técnicas têm justificativa?
- [ ] O diagrama de implementação mostra o fluxo físico?
- [ ] O resumo de estratégias inclui benefícios e impactos?
- [ ] Os conceitos abordados estão listados?
- [ ] As considerações finais abordam aspectos operacionais?
- [ ] Os diagramas Mermaid seguem o padrão de cores?
- [ ] O documento está sem código nas considerações finais?

---
