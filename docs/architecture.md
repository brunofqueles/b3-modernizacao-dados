# Arquitetura — Modernização de Dados B3 (KNIME → Databricks)

> Este documento descreve as decisões técnicas de arquitetura do projeto. Contexto de negócio (o que é o índice-proxy, por que esses tickers, indicadores) vive em `docs/business-context.md`. Decisões pontuais e seus trade-offs detalhados vivem em `docs/adr/`.

## 1. Entendimento do problema

Construir um projeto de portfólio que simule o escopo de uma vaga real de modernização de dados da B3: migração de pipelines legados (ferramenta visual + procedures, aqui simulada por KNIME no lugar de Alteryx) para uma arquitetura moderna em Databricks, com foco em governança, escalabilidade e cálculo de um índice financeiro — 100% dentro do Databricks Free Edition, dentro de um prazo de execução curto (3 dias dedicados, com buffer).

## 2. Premissas adotadas

- Plataforma alvo: **Databricks Free Edition** — serverless-only, sem private networking, outbound restrito a domínios confiáveis (testado e confirmado compatível com `brapi.dev` e GitHub — ver `databricks/tests/`), sem SLA/suporte.
- **KNIME Analytics Platform** no lugar de Alteryx, por ser gratuito e open source sem exigência de conta corporativa — não é um clone funcional do Alteryx, simula o *padrão* de migração ferramenta-visual → código.
- Fonte única de dados: [brapi.dev](https://brapi.dev), API REST gratuita, sandbox sem token, para 4 tickers (PETR4, VALE3, MGLU3, ITUB4).
- "Índice" calculado é um **proxy simplificado** (média simples do retorno diário dos 4 papéis), não a metodologia oficial de nenhum índice real da B3 — ver `docs/business-context.md`.
- Governança e time: projeto solo, portfólio público — sem requisitos de compliance regulatório real, sem SLA contratual.

## 3. Arquitetura proposta

```
[Fonte única]              [Legado simulado]          [Novo — Databricks]
brapi.dev (API)     ──┬──> KNIME (transforma,
  4 tickers            │     calcula índice-proxy,
                       │     grava CSV versionado por data)
                       │
                       └──> Volume UC (raw, landing) ──> Bronze ──> Silver ──> Gold
                                                                                  │
                                                          [Reconciliação: Gold vs. saída do KNIME]
                                                          (contagem de linhas, valor do índice)
```

KNIME e Databricks consomem a **mesma fonte de forma independente** (ver [ADR-01](adr/adr-01-ingestao-independente-landing.md)) — o Databricks não lê o CSV de saída do KNIME em nenhum ponto do fluxo Bronze/Silver/Gold.

### Unity Catalog — convenção de nomes

*Pendente de definição formal — será registrada aqui e em ADR próprio ao criar o catalog/schema, no início do trabalho de Bronze.*

Convenção provisória adotada: `catalog.schema.table`, com um schema por camada (`bronze`, `silver`, `gold`), definida **antes** da criação das tabelas, não depois (conforme já planejado no cronograma do projeto).

### Camadas

| Camada | Conteúdo |
|---|---|
| **Landing (Volume UC)** | Cópia bruta e imutável da resposta da API `brapi.dev`, com metadados de ingestão. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md). |
| **Bronze** | Tabelas Delta, cópia fiel da fonte, imutável. |
| **Silver** | Tipagem, deduplicação, pelo menos 1 regra de qualidade com tabela de quarentena (ex.: preço nulo/negativo desviado, sem quebrar o pipeline). |
| **Gold** | Cálculo do índice-proxy em PySpark/SQL, orquestrado como Job no Databricks Workflows. |
| **Reconciliação** | Notebook comparando Gold (Databricks) com a saída do KNIME — contagem de linhas e valor do índice, por data. |

## 4. Componentes e justificativa

| Componente | Por que esse e não outro |
|---|---|
| **brapi.dev** | API REST gratuita, sandbox funciona sem token para os tickers escolhidos, endpoint de cotação pronto para consumo — preferida ao `.zip` oficial da B3 (COTAHIST), que tem layout de largura fixa e histórico de captcha no download, incompatível com o prazo do projeto. |
| **Volume UC como landing, antes da Bronze** | Desacopla "buscar o dado" de "estruturar o dado em tabela", preserva a resposta bruta como evidência auditável. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md). |
| **KNIME e Databricks consumindo a fonte de forma independente** | Sustenta a reconciliação Gold vs. KNIME como prova de migração de lógica de negócio, não apenas de output. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md). |
| **CSVs do KNIME versionados por data** | Permite comparação histórica real entre execuções de dias diferentes (qui/sex/seg), em vez de um único snapshot sobrescrito. |
| **Databricks Workflows para orquestração** | Nativo, sem custo extra, disponível na Free Edition — orquestra Bronze → Silver → Gold como Job. |
| **Databricks Git folders** | Conecta o workspace diretamente ao repositório GitHub, permitindo commits via branch + PR direto da interface do Databricks, sem depender exclusivamente do terminal local. |

## 5. Limitações conhecidas da Free Edition e como o desenho já as absorve

| Limite | Como o desenho contorna |
|---|---|
| Outbound restrito a domínios confiáveis | Testado e confirmado no Dia 1 (`databricks/tests/teste_b3.ipynb`): `brapi.dev` e `api.github.com`/`github.com` acessíveis sem bloqueio. |
| Sem SLA/suporte, uso não comercial | Aceitável — natureza de portfólio, não produto em produção. |
| 1 workspace / 1 metastore por conta | Sem impacto — projeto não requer múltiplos ambientes (dev/staging/prod) nesta fase. |

## 6. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Divergência de preço entre a chamada do KNIME e a do Databricks no mesmo dia (dado em tempo real, muda durante o pregão) | Execução padronizada de ambos após o fechamento do pregão (~17h), eliminando a janela de variação intraday. |
| "Índice" proxy confundido com metodologia oficial de um índice real (ex.: Ibovespa) numa entrevista | Ressalva explícita no README e em `docs/business-context.md`. |
| Prazo curto (3 dias dedicados) competindo com escopo de automação (agente de commit) | Agente tratado como extra condicional, só entra se o cronograma técnico central fechar no prazo — não faz parte do caminho crítico. |

## 7. Evolução — possíveis próximos passos

- **Databricks Asset Bundles (IaC)**: não implementado neste projeto por restrição de prazo — reconhecido como padrão mais robusto para deploy/CI-CD entre ambientes, documentado aqui como evolução consciente, não como lacuna não percebida.
- **Agente de automação de commit/PR** via API REST do GitHub — mecânica já validada em projeto anterior (branch → commit → PR automáticos, merge sempre manual). Candidato a extra, condicionado a sobra de tempo no cronograma (ver cronograma no histórico do projeto).
- Ampliação do índice-proxy para mais tickers ou ponderação por `marketCap`, caso o projeto seja retomado após a entrega.

## 8. Status atual e próximos passos

**ADRs escritos:**
- [ADR-01 — Ingestão independente via Volume UC (Landing)](adr/adr-01-ingestao-independente-landing.md)

**Infraestrutura já criada:**
- Repositório GitHub estruturado (`docs/adr`, `knime`, `databricks/{bronze,silver,gold,jobs,reconciliation,tests}`)
- Git folder conectando o Databricks ao repositório
- Workflow KNIME completo e funcional (`knime/b3_pipeline_legado.knwf`)

**Testes realizados:**
- Conectividade do Databricks Free Edition com `brapi.dev` e GitHub (`databricks/tests/teste_b3.ipynb`) — ambos confirmados acessíveis.

**Estado dos dados:**
- CSVs do KNIME gerados e versionados por data (`knime/cotacoes_detalhado_AAAA-MM-DD.csv`, `knime/indice_proxy_AAAA-MM-DD.csv`), com histórico iniciado em 26/08/2026.

**Próximo passo real:** criação do Volume UC e da tabela Bronze, consumindo `brapi.dev` diretamente (independente do KNIME), conforme [ADR-01](adr/adr-01-ingestao-independente-landing.md).