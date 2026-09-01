# Modernização de Dados B3 — KNIME → Databricks

Projeto de portfólio que simula a modernização de um pipeline de dados de mercado de capitais: migração de um sistema legado baseado em ferramenta visual (KNIME, no lugar de Alteryx) para uma arquitetura moderna em Databricks, com foco em governança, escalabilidade e cálculo de um índice financeiro.

## Status do projeto

✅ MVP concluído — projeto apresentado com sucesso, aprovado para etapa final — última atualização: 01/09/2026

- [x] Repositório estruturado, com Git folders conectando o Databricks ao GitHub
- [x] Workflow KNIME funcional (sistema legado simulado): busca cotações, calcula retorno diário e índice-proxy
- [x] Teste de conectividade do Databricks Free Edition (API externa + GitHub)
- [x] ADR-01: decisão de ingestão independente via Volume UC
- [x] ADR-02: parametrização via Widgets e escrita idempotente na Landing Zone
- [x] ADR-03: escrita idempotente na Bronze via MERGE e coluna de validação de integridade
- [x] ADR-04: quarentena unificada, deduplicação e schema explícito na Silver
- [x] ADR-05: indicadores da camada Gold e geração exclusivamente automatizada
- [x] Catalog `poc_b3_modernizacao` e schemas por camada (landing, bronze, silver, gold, reconciliation, observability), com tags e descrição
- [x] Volume UC (`landing.raw`) + notebook de ingestão parametrizado (`01_ingestao_landing`)
- [x] Tabela Bronze (`bronze.cotacoes`), com MERGE idempotente e coluna de validação de integridade
- [x] Silver (tipagem, deduplicação, quarentena unificada)
- [x] Gold (5 indicadores: retorno diário, índice-proxy, ranking de valorização, dispersão, índice acumulado)
- [x] ADR-06: orquestração via Databricks Workflows e YAML como documentação leve de IaC
- [x] Orquestração via Databricks Workflows (Job `pipeline_diario_b3`, 5 Tasks, agendado dias úteis às 17h15)
- [x] ADR-07: reconciliação Gold vs. KNIME (formato, tolerância, execução manual)
- [x] ADR-08: observabilidade via log de execução e módulo utilitário compartilhado
- [x] Observabilidade (tabela `observability.pipeline_runs`) e eliminação da duplicação de código (`merge_ou_cria`)
- [x] ADR-09: índice acumulado (5º indicador, base 100, capitalização composta)
- [x] ADR-10: reconciliação automatizada D-1, tratamento explícito de erro
- [x] Reconciliação automatizada (5ª Task no Job, detecção de D-1, sem intervenção manual)
- [x] ADR-11: camada de consumo executivo (AI/BI Dashboard + Genie Space)
- [x] AI/BI Dashboard publicado (índice acumulado, ranking, reconciliação, observabilidade)
- [x] Genie Space configurado e testado (acesso curado, instruções de domínio, 3 exemplos)
- [x] Janela de reconciliação completa (27/08, 28/08, 31/08)
- [x] [Lições aprendidas](docs/licoes-aprendidas.md)
- [x] Documento de escalabilidade (4 → 500 tickers) — material de apoio, fora da documentação formal
- [ ] Alerta de divergência (implementação aditiva) — release futuro
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
```

KNIME e Databricks consomem a **mesma fonte de forma independente** — o Databricks não lê o CSV de saída do KNIME. Essa decisão está detalhada na [ADR-01](docs/adr/adr-01-ingestao-independente-landing.md).

## Estrutura do repositório

```
b3-modernizacao-dados/
├── docs/
│   └── adr/                     # decisões de arquitetura, uma por arquivo
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
2. **Databricks**: conectar o workspace ao repositório via Git folder, dar Pull. O pipeline completo roda via Job `pipeline_diario_b3` (agendado dias úteis às 17h15) ou manualmente, notebook por notebook, na ordem `setup/00_setup_catalog` → `setup/01_utilitarios_pipeline` → `landing/01_ingestao_landing` → `bronze/02_bronze` → `silver/03_silver` → `gold/04_gold` → `reconciliation/05_reconciliacao`.
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

## Limitações conhecidas

- O índice calculado é um **proxy simplificado**, não a metodologia oficial de um índice real da B3.
- Databricks Free Edition: compute serverless apenas, sem SLA, uso não comercial, outbound restrito a domínios confiáveis (testado e confirmado compatível com `brapi.dev` e GitHub).
- Comparação histórica (reconciliação) limitada a 3 dias de pregão (27/08, 28/08, 31/08), com execução padronizada após o fechamento (**17h15**) para evitar divergência de preço intraday entre KNIME e Databricks.
- Falha é detectada por ausência de registro em `observability.pipeline_runs`, não por evento explícito, exceto na reconciliação (que já tem tratamento explícito de erro).

## Possíveis evoluções futuras

- Adoção de **Databricks Asset Bundles (DAB)** para deploy e CI/CD estruturado entre ambientes — não implementado neste projeto por restrição de prazo, mas reconhecido como padrão mais robusto para produção.
- Agente de automação de commit/PR via API do GitHub (mecânica já validada em projeto anterior) — candidato a extra, condicionado a sobra de tempo no cronograma.
- Alerta automatizado de divergência na reconciliação (implementação aditiva, sem alterar os notebooks já validados).
- Estender o tratamento explícito de erro (hoje só na reconciliação) aos demais 4 notebooks.
- Governança de acesso (RBAC real no Unity Catalog) — hoje documentada como modelo pretendido.
- Escalabilidade de 4 para uma lista maior de tickers — ver documento de referência técnica dedicado (não incluído neste repositório).