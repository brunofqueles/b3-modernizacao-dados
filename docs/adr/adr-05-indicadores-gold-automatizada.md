# ADR-05 — Indicadores da camada Gold e geração exclusivamente automatizada

**Status:** Aceito
**Data:** 2026-08-27

## Contexto

A camada Gold, por definição já registrada no projeto (ver seção de governança em `architecture.md`), deveria ser gerada **exclusivamente de forma automatizada**, sem edição manual em nenhuma hipótese — especificamente para eliminar divergência de números entre diferentes consumidores do dado. Além disso, o escopo original do projeto previa apenas 1 indicador (o índice-proxy); durante a construção, avaliou-se ampliar esse escopo para dar mais substância analítica à camada, sem inventar dado novo — apenas explorando o que já estava disponível na Silver.

## Decisão

**Geração exclusivamente via código**: a Gold é construída inteiramente pelo notebook `04_gold`, lendo a Silver e calculando tudo via PySpark — nenhum valor é digitado ou editado manualmente em nenhuma tabela Gold. Isso implementa, na prática, a primeira metade do modelo de governança já pretendido (a segunda metade — controle de acesso por papel via RBAC no Unity Catalog — permanece como modelo pretendido, não implementado, dado o prazo do projeto). A automação completa (execução agendada, sem intervenção humana nem para *disparar* o processo) depende da orquestração via Databricks Workflows, ainda não construída — tratada como uma segunda camada de automação, distinta da geração de código em si.

**Quatro indicadores calculados**, todos derivados de campos já existentes na Silver, sem necessidade de nova fonte de dado:
1. **Retorno diário por ticker** (`retorno_diario_pct`): `(preço atual - fechamento anterior) / fechamento anterior`, expresso já em percentual, arredondado a 2 casas decimais.
2. **Índice-proxy** (`indice_proxy_pct`): média simples do retorno diário dos 4 papéis, agregada por dia.
3. **Ranking de valorização** (`ranking_valorizacao`): posição de cada papel no dia, ordenado do maior para o menor retorno — resolve diretamente a pergunta de negócio "quais ações mais se valorizaram?", antes não respondível pelo pipeline (ver `architecture.md`, seção de perguntas).
4. **Dispersão do dia** (`dispersao_pct`): desvio padrão dos 4 retornos diários, agregado por dia — indica o quanto os papéis se moveram de forma semelhante ou distinta no mesmo pregão.

Duas tabelas Gold resultantes: `gold.indicadores_diarios` (indicadores 1 e 3, uma linha por ticker/dia) e `gold.indice_proxy` (indicadores 2 e 4, uma linha por dia).

**Indicador adiado, documentado deliberadamente**: variação acumulada do índice entre dias consecutivos — não implementável hoje porque a Silver contém apenas 1 dia de dados válidos (27/08); depende de pelo menos 2 dias para ter sentido. Registrado como próximo indicador natural assim que a janela de reconciliação avançar.

## Alternativas consideradas

- **Manter apenas a fração decimal do retorno (ex.: `0.0004`), sem versão em percentual**: descartada — dificulta leitura/apresentação sem ganho de precisão real (double já é preciso o suficiente em qualquer das duas escalas).
- **Manter simultaneamente uma coluna em fração e outra em percentual**: considerada inicialmente, mas descartada por redundância — nenhum cálculo posterior do projeto precisa da forma fracionária depois que o percentual já foi calculado.
- **Calcular variação acumulada entre dias já nesta fase, mesmo com 1 dia de dado**: descartada — resultado seria matematicamente vazio ou enganoso (variação "entre" um único ponto), preferível declarar explicitamente como pendente a apresentar um número sem significado real.

## Consequências

- A resposta à pergunta de negócio "quais ações mais se valorizaram?" passa de "não respondível pelo pipeline" para "respondível diretamente via `gold.indicadores_diarios`" — atualização registrada em `architecture.md`.
- A limitação de que `fechamento_anterior` vem pronto da API (não calculado a partir de execuções anteriores do próprio pipeline) permanece relevante: o indicador de variação acumulada entre dias, quando implementado, precisará comparar registros da própria Silver entre datas diferentes, não depender apenas do que a API entrega pronto a cada chamada.
- A função `merge_ou_cria` foi duplicada uma segunda vez (já duplicada da Silver para a Gold) — reforça o item de dívida técnica já registrado em [ADR-04](adr-04-quarentena-dedup-schema-silver.md), agora presente em 2 notebooks além do original.