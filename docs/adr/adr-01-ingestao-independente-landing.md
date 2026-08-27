# adr-01 — Ingestão independente via Volume UC (Landing)

**Status:** Aceito
**Data:** 2026-08-27

## Contexto

O projeto simula uma migração de pipeline legado (KNIME, representando Alteryx) para uma arquitetura moderna em Databricks. Uma decisão central do desenho geral do projeto é que o KNIME e o Databricks devem consumir a mesma fonte (`brapi.dev`) de forma **independente** — o Databricks não deve ler o CSV de saída do KNIME como sua fonte de dados.

O motivo é direto: se o Databricks ingerisse o output do KNIME, estaríamos migrando apenas a *saída* do sistema legado, não a *lógica de negócio* em si. Isso não representaria uma migração real de pipeline, e a reconciliação final (Gold vs. KNIME) perderia o sentido — estaríamos comparando um sistema com uma cópia dele mesmo.

## Decisão

Adotar um Volume do Unity Catalog como camada de pouso (landing) para a cópia bruta da fonte, **antes** da tabela Bronze — não gravar direto a resposta da API numa tabela Delta.

Fluxo:

```
brapi.dev (API) → Volume UC (raw, arquivo JSON/CSV bruto) → tabela Bronze (Delta)
```

- **Volume UC como landing**: preserva a resposta bruta da API exatamente como recebida, antes de qualquer parsing ou tipagem — é a evidência primária caso seja necessário auditar ou reprocessar.
- **Separação landing → Bronze**: desacopla "buscar o dado" de "estruturar o dado em tabela", permitindo testar e reprocessar cada etapa isoladamente.
- **Ingestão independente do KNIME**: o Databricks chama a mesma API (`brapi.dev`), com os mesmos 4 tickers, sem depender do CSV gerado pelo KNIME em nenhum momento do fluxo Bronze/Silver/Gold.

## Alternativas consideradas

- **Bronze lendo direto o CSV do KNIME**: descartada — sustentaria apenas a migração da *saída* do legado, não da lógica de negócio, e invalidaria a reconciliação como prova de migração real (comparar um sistema consigo mesmo).
- **Escrita direta da resposta da API em tabela Delta, sem landing intermediária**: descartada — perde a cópia fiel/imutável da fonte bruta antes de qualquer transformação, dificultando auditoria e indo contra o princípio de Bronze como "cópia fiel da fonte" definido no desenho geral do projeto.

## Consequências

- KNIME e Databricks têm cada um sua própria chamada à API, com risco de pequenas divergências de preço caso não sejam executados próximos no tempo (ver decisão operacional de rodar ambos após o fechamento do pregão, ~17h).
- A reconciliação (Gold vs. KNIME) se torna uma comparação genuína entre duas implementações independentes da mesma lógica de negócio, não uma comparação trivial.
- Exige versionamento por data tanto no Volume UC quanto nos CSVs do KNIME, para permitir comparação histórica entre execuções de dias diferentes (qui/sex/seg).