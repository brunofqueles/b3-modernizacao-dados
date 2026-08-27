# Modernização de Dados B3 — KNIME → Databricks

Projeto de portfólio que simula a modernização de um pipeline de dados de mercado de capitais: migração de um sistema legado baseado em ferramenta visual (KNIME, no lugar de Alteryx) para uma arquitetura moderna em Databricks, com foco em governança, escalabilidade e cálculo de um índice financeiro.

## Status do projeto

🚧 Em construção — última atualização: 27/08/2026

- [x] Repositório estruturado, com Git folders conectando o Databricks ao GitHub
- [x] Workflow KNIME funcional (sistema legado simulado): busca cotações, calcula retorno diário e índice-proxy
- [x] Teste de conectividade do Databricks Free Edition (API externa + GitHub)
- [x] ADR-01: decisão de ingestão independente via Volume UC
- [x] ADR-02: parametrização via Widgets e escrita idempotente na Landing Zone
- [x] Catalog `poc_b3_modernizacao` e schemas por camada (landing, bronze, silver, gold), com tags e descrição
- [x] Volume UC (`landing.raw`) + notebook de ingestão parametrizado (`01_ingestao_landing`)
- [ ] Tabela Bronze
- [ ] Silver (tipagem, deduplicação, regra de qualidade)
- [ ] Gold (cálculo do índice-proxy em PySpark/SQL)
- [ ] Orquestração via Databricks Workflows
- [ ] Reconciliação Gold vs. KNIME
- [ ] Diagrama de arquitetura final
- [ ] Lições aprendidas

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

Diagrama detalhado por camada será adicionado conforme Bronze/Silver/Gold forem implementados.

## Estrutura do repositório

```
b3-modernizacao-dados/
├── docs/
│   └── adr/                     # decisões de arquitetura, uma por arquivo
├── knime/                       # workflow .knwf + CSVs de saída (versionados por data)
└── databricks/
    ├── bronze/
    ├── silver/
    ├── gold/
    ├── jobs/                    # definição do Databricks Workflow
    ├── reconciliation/
    └── tests/                   # notebooks de validação técnica (ex.: teste de conectividade)
```

## Como rodar

*Seção em construção — será preenchida conforme cada camada for implementada.*

1. **KNIME**: abrir `knime/b3_pipeline_legado.knwf` no KNIME Analytics Platform, executar "Execute all". Gera dois CSVs versionados por data em `knime/`.
2. **Databricks**: *(pendente)*

## Documentação completa

- [Arquitetura técnica](docs/architecture.md)
- [Contexto de negócio](docs/business-context.md)

## Decisões de arquitetura (ADRs)

- [ADR-01 — Ingestão independente via Volume UC (Landing)](docs/adr/adr-01-ingestao-independente-landing.md)

## Limitações conhecidas

- O índice calculado é um **proxy simplificado**, não a metodologia oficial de um índice real da B3.
- Databricks Free Edition: compute serverless apenas, sem SLA, uso não comercial, outbound restrito a domínios confiáveis (testado e confirmado compatível com `brapi.dev` e GitHub).
- Comparação histórica (reconciliação) limitada a 3 dias de pregão (qui/sex/seg), com execução padronizada após o fechamento (~17h) para evitar divergência de preço intraday entre KNIME e Databricks.

## Possíveis evoluções futuras

- Adoção de **Databricks Asset Bundles (DAB)** para deploy e CI/CD estruturado entre ambientes — não implementado neste projeto por restrição de prazo, mas reconhecido como padrão mais robusto para produção.
- Agente de automação de commit/PR via API do GitHub (mecânica já validada em projeto anterior) — candidato a extra, condicionado a sobra de tempo no cronograma.