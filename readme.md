# Modernização de Dados B3 — KNIME → Databricks

Projeto de portfólio que simula a modernização de um pipeline de dados de mercado de capitais: migração de um sistema legado baseado em ferramenta visual (KNIME, no lugar de Alteryx) para uma arquitetura moderna em Databricks, com foco em governança, escalabilidade e cálculo de um índice financeiro.

## Status do projeto

✅ MVP concluído — projeto apresentado com sucesso, aprovado para etapa final — última atualização: 01/09/2026

- [x] Repositório estruturado, com Git folders conectando o Databricks ao GitHub
- [x] Workflow KNIME funcional (sistema legado simulado): busca cotações, calcula retorno diário e índice-proxy — 5 execuções reais (26/08, 27/08, 28/08, 31/08, 01/09)
- [x] Teste de conectividade do Databricks Free Edition (API externa + GitHub)
- [x] Catalog `poc_b3_modernizacao` e 6 schemas (landing, bronze, silver, gold, reconciliation, observability), com tags e descrição, criados via código
- [x] Pipeline completo: Landing → Bronze → Silver → Gold (5 indicadores) → Reconciliação (D-1 automático) → Alerta de divergência
- [x] Orquestração via Databricks Workflows (Job `pipeline_diario_b3`, **6 Tasks**, agendado dias úteis às 17h15)
- [x] Observabilidade (tabela `observability.pipeline_runs`) com try/except completo nos 5 notebooks do pipeline
- [x] Alertas nativos (falha + duração, 2 e-mails com propósitos distintos) e alerta customizado de divergência anormal (limiar 1%)
- [x] Databricks Secret Scope para credencial de e-mail (nunca em texto no código)
- [x] AI/BI Dashboard publicado (índice acumulado, ranking, reconciliação, observabilidade)
- [x] Genie Space configurado e testado (acesso curado, instruções de domínio, 3 exemplos)
- [x] Janela de reconciliação completa (27/08, 28/08, 31/08)
- [x] 13 ADRs documentando cada decisão técnica
- [x] Simulação de FinOps (armazenamento + consumo de DBU), com identificação de gargalo real de escalabilidade
- [x] [Lições aprendidas](docs/licoes-aprendidas.md)
- [x] Documento de escalabilidade (4 → 500 tickers) e planilhas de custo — em `docs/anexos/`
- [ ] Painel de observabilidade com falhas por notebook (dado real disponível, widget ainda pendente)
- [ ] Recalibração do limiar de duração do Job (aguardando observação de execuções agendadas com 6 Tasks)
- [ ] Investigação da execução dupla do Job (observada em 28/08 e 01/09, sem impacto de dado)
- [ ] Diagrama de arquitetura final (visual)

> **Nota sobre a janela de reconciliação:** a comparação histórica entre KNIME e Databricks cobre **27/08, 28/08 e 31/08** (não 26/08) — a infraestrutura Databricks só foi criada em 27/08, e a API não permite obter retroativamente o preço de um dia anterior. Detalhe completo em `docs/architecture.md`.

## Objetivo

Simular o escopo de uma vaga real de modernização de dados: migração de pipelines legados (ferramenta visual + procedures) para Databricks, cobrindo governança, escalabilidade e desenvolvimento de índices financeiros — incluindo uso de IA aplicada a produtividade no fluxo de desenvolvimento.

## Fonte de dados

[brapi.dev](https://brapi.dev) — API REST gratuita de cotações da B3, usada em modo sandbox (sem necessidade de token) para os seguintes tickers: **PETR4, VALE3, MGLU3, ITUB4**.

> **Nota importante sobre metodologia:** o "índice" calculado neste projeto é um **proxy simplificado** (média simples do retorno diário dos 4 papéis acima), não uma reprodução da metodologia oficial de nenhum índice real da B3 (como o Ibovespa). Essa distinção é deliberada — ver [ADR-01](docs/adr/adr-01-ingestao-independente-landing.md) e limitações abaixo.

## Arquitetura (visão geral)

```
[Fonte única]              [Legado simulado]         [Novo — Databricks]
brapi.dev (API) ──┬──> KNIME (transforma,
  4 tickers        │     calcula índice-proxy,
                   │     grava CSV versionado por data)
                   │
                   └──> Volume UC (raw) ──> Bronze ──> Silver ──> Gold
                                                                     │
                                             [Reconciliação: Gold vs. saída do KNIME]
                                                                     │
                                                     [Alerta de divergência anormal]
```

KNIME e Databricks consomem a **mesma fonte de forma independente** — o Databricks não lê o CSV de saída do KNIME. Essa decisão está detalhada na [ADR-01](docs/adr/adr-01-ingestao-independente-landing.md).

## Estrutura do repositório

```
b3-modernizacao-dados/
├── docs/
│   ├── adr/                     # decisões de arquitetura, uma por arquivo
│   └── anexos/                  # planilhas de custo (FinOps) e documento de escalabilidade
├── knime/                       # workflow .knwf + CSVs de saída (versionados por data)
└── databricks/
    ├── setup/                   # criação de catalog/schemas e funções utilitárias compartilhadas
    ├── landing/                 # ingestão da API para o Volume UC
    ├── bronze/
    ├── silver/
    ├── gold/
    ├── jobs/                    # definição do Databricks Workflow (YAML exportado)
    ├── reconciliation/
    └── tests/                   # notebooks de validação técnica (ex.: teste de conectividade)
```

## Como rodar

1. **KNIME**: abrir `knime/b3_pipeline_legado.knwf` no KNIME Analytics Platform, ajustar a data nos CSV Writers, executar "Execute all". Gera dois CSVs versionados por data em `knime/`, e exportar novamente o `.knwf`.
2. **Databricks**: conectar o workspace ao repositório via Git folder, dar Pull. O pipeline completo roda via Job `pipeline_diario_b3` (agendado dias úteis às 17h15) ou manualmente, notebook por notebook, na ordem `setup/00_setup_catalog` → `setup/01_utilitarios_pipeline` → `landing/01_ingestao_landing` → `bronze/02_bronze` → `silver/03_silver` → `gold/04_gold` → `reconciliation/05_reconciliacao` → `reconciliation/06_alerta_divergencia`.
3. **Camada de consumo**: AI/BI Dashboard "B3 - Modernização de Dados" e Genie Space "Genie B3 - Modernização de Dados", ambos no workspace do Databricks, atualizando automaticamente conforme novos dados entram na Gold e na Reconciliação.

## Documentação completa

- [Arquitetura técnica](docs/architecture.md)
- [Contexto de negócio](docs/business-context.md)
- [Lições aprendidas](docs/licoes-aprendidas.md)

## Decisões de arquitetura (ADRs)

- [ADR-01 — Ingestão independente via Volume UC (Landing)](docs/adr/adr-01-ingestao-independente-landing.md)
- [ADR-02 — Parametrização via Widgets e escrita idempotente na Landing Zone](docs/adr/adr-02-widgets-idempotencia-landing.md)
- [ADR-03 — Escrita idempotente na Bronze via MERGE e coluna de validação de integridade](docs/adr/adr-03-merge-bronze-validacao-integridade.md)
- [ADR-04 — Quarentena unificada, deduplicação e schema explícito na Silver](docs/adr/adr-04-quarentena-dedup-schema-silver.md)
- [ADR-05 — Indicadores da camada Gold e geração exclusivamente automatizada](docs/adr/adr-05-indicadores-gold-automatizada.md)
- [ADR-06 — Orquestração via Databricks Workflows e YAML como documentação leve de IaC](docs/adr/adr-06-orquestracao-workflows-yaml.md)
- [ADR-07 — Reconciliação Gold vs. KNIME: formato, tolerância e execução manual](docs/adr/adr-07-reconciliacao-formato-tolerancia-manual.md)
- [ADR-08 — Observabilidade via log de execução e módulo utilitário compartilhado](docs/adr/adr-08-observabilidade-modulo-compartilhado.md)
- [ADR-09 — Índice acumulado (base 100) via capitalização composta](docs/adr/adr-09-indice-acumulado-capitalizacao-composta.md)
- [ADR-10 — Reconciliação automatizada D-1, tratamento explícito de erro e correção de dado residual](docs/adr/adr-10-reconciliacao-d1-automatizada.md)
- [ADR-11 — Camada de consumo executivo: AI/BI Dashboard e Genie Space](docs/adr/adr-11-dashboard-genie-consumo-executivo.md)
- [ADR-12 — Alertas nativos de falha e duração no Job](docs/adr/adr-12-alertas-nativos-falha-duracao.md)
- [ADR-13 — Try/except completo no pipeline e alerta de divergência anormal aditivo](docs/adr/adr-13-tryexcept-completo-alerta-divergencia.md)
- [ADR-14 — Simulação de FinOps: armazenamento e consumo de DBU](docs/adr/adr-14-finops-armazenamento-dbu.md)

## Limitações conhecidas

- O índice calculado é um **proxy simplificado**, não a metodologia oficial de um índice real da B3.
- Databricks Free Edition: compute serverless apenas, sem SLA, uso não comercial, outbound restrito a domínios confiáveis (testado e confirmado compatível com `brapi.dev` e GitHub).
- Comparação histórica (reconciliação) limitada a 3 dias de pregão (27/08, 28/08, 31/08), com execução padronizada após o fechamento (**17h15**) para evitar divergência de preço intraday entre KNIME e Databricks.
- Mensagens dos alertas nativos (falha, duração) permanecem no template padrão do Databricks — decisão de escopo, não lacuna (ver ADR-13).
- Job apresentou execução dupla, próxima uma da outra, em pelo menos 2 dias — sem impacto de dado (MERGE idempotente), causa não investigada.

## Possíveis evoluções futuras

- Adoção de **Databricks Asset Bundles (DAB)** para deploy e CI/CD estruturado entre ambientes — não implementado neste projeto por restrição de prazo, mas reconhecido como padrão mais robusto para produção.
- Agente de automação de commit/PR via API do GitHub (mecânica já validada em projeto anterior) — candidato a extra, condicionado a sobra de tempo no cronograma.
- Painel de observabilidade do dashboard com falhas por notebook.
- Propagação consistente de `modo_execucao` via Widget em todos os notebooks (hoje só `01` e `02` têm esse Widget).
- Governança de acesso (RBAC real no Unity Catalog) — hoje documentada como modelo pretendido.
- Escalabilidade de 4 para uma lista maior de tickers — ver documento de referência técnica dedicado (`docs/anexos/`). Gargalo identificado: chamadas de API sequenciais na ingestão seriam o principal driver de tempo/custo em escala, não o volume de dado (ver [ADR-14](docs/adr/adr-14-finops-armazenamento-dbu.md)).