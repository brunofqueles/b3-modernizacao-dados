# ADR-14 — Simulação de FinOps: armazenamento e consumo de DBU

**Status:** Aceito
**Data:** 2026-09-01

## Contexto

O projeto roda inteiramente no Databricks Free Edition, sem custo real — mas uma vaga de modernização de dados espera familiaridade com **FinOps** (gestão financeira de nuvem) como competência real, não só técnica de pipeline. Foi solicitada uma simulação de custo caso o projeto rodasse num ambiente pago, cobrindo dois ângulos: armazenamento (via calculadora Databricks/Unity Catalog Managed Storage e via Azure Storage direto) e consumo de compute (DBU), este último detalhado nesta ADR.

## Decisão — Armazenamento

Duas planilhas (`docs/anexos/custo_armazenamento_databricks.xlsx`, `docs/anexos/custo_armazenamento_azure.xlsx`) simulam o volume de dados do projeto de baixo para cima (por camada, a partir de tamanhos reais já observados — ex.: 4,32 KB para a resposta de 4 tickers em um dia), em dois cenários: atual (4 tickers, 1 ano) e escala (500 tickers, 1 ano).

**Resultado**: mesmo em 500 tickers com 1 ano de histórico, o volume total fica abaixo de 0,5 GB — custo mensal de armazenamento, em qualquer combinação de camada (Hot/Cool) e redundância (LRS/GRS), não ultrapassa poucos centavos de dólar. Isso é esperado: o projeto lida com dados estruturados pequenos (cotações diárias), não volume massivo.

## Decisão — Consumo de DBU (compute)

**Como funciona o modelo de cobrança**: o Databricks não cobra por hora de máquina diretamente — cobra por **DBU** (Databricks Unit), uma medida normalizada de capacidade de processamento, consumida por segundo de execução e multiplicada pela taxa do tipo de compute utilizado. O tipo de compute que o projeto usa — **Jobs Compute em modo Serverless** — tem uma característica importante no Azure: o custo da máquina virtual subjacente fica **embutido na própria taxa de DBU**, ao contrário do compute clássico (cluster provisionado manualmente), onde Databricks e Azure cobram em linhas separadas. Isso simplifica a conta, mas também significa que não há uma "economia de infraestrutura" a buscar separadamente — o único alavanca de custo é reduzir o consumo de DBU em si.

**Taxa de referência**: Jobs Compute, tier Premium, Azure — taxa pública consolidada entre **US$ 0,15 e US$ 0,30 por DBU-hora**, dependendo da fonte consultada (não há um número único e definitivo publicado; a calculadora oficial do Databricks é a fonte final para orçamento real). Usado o ponto médio (US$ 0,225/DBU-hora) como referência nesta simulação.

**Estimativa de consumo, cenário atual (4 tickers)**: o Job completo (6 Tasks) tem levado, em execuções reais observadas, entre 3,4 e 5,5 minutos. Assumindo um consumo conservador de 1 a 2 DBU/hora para essa carga leve (baixo volume de dado, transformações simples, sem cluster grande — ponto médio 1,5 DBU/hora), o consumo diário fica em torno de **0,13 DBU** (5 min × 1,5 DBU/hora). Ao longo de 250 dias úteis/ano, isso equivale a aproximadamente **32,5 DBU/ano** — custo estimado de **US$ 7 a US$ 8/ano** (ponto médio da taxa).

**Estimativa de consumo, cenário de escala (500 tickers)**: aqui está o ponto central desta ADR, e o motivo pelo qual vale a pena documentar compute separadamente de armazenamento. Diferente do volume de dado (que escala ~125x, mas continua irrisório em termos absolutos), o **tempo de execução** do Job não escala da mesma forma — e o principal motivo não é o processamento Spark (que lida bem com volumes maiores), mas sim o notebook de ingestão (`01_ingestao_landing`), que hoje faz **uma chamada HTTP sequencial por ticker**, dentro de um laço `for`. Com 4 tickers, isso é imperceptível (poucos segundos). Com 500 tickers, 500 chamadas HTTP sequenciais — mesmo a ~0,3-1 segundo cada — podem levar sozinhas entre **4 e 8 minutos**, antes mesmo de qualquer processamento Spark começar. Estimativa de tempo total do Job em 500 tickers: **10 a 15 minutos** (contra 3,4-5,5 minutos hoje) — um aumento de tempo desproporcional ao aumento de volume de dado, causado por uma característica de implementação (chamadas sequenciais), não por limitação de processamento distribuído.

Consumo estimado em 500 tickers: 12,5 min × 1,5 DBU/hora ≈ **0,31 DBU/dia**, ou **78 DBU/ano** — custo estimado de **US$ 17 a US$ 18/ano**. Mais que o dobro do cenário atual, apesar do volume de dado ter crescido 125x — a relação entre "volume de dado" e "custo real" não é direta neste projeto; o gargalo é a forma de ingestão, não o tamanho do dado.

## Alternativas consideradas

- **Assumir consumo de DBU proporcional ao volume de dado (mesmo fator 125x usado no armazenamento)**: descartada — tecnicamente incorreta. Processamento Spark distribuído não degrada linearmente com o aumento de linhas numa base já pequena; o verdadeiro gargalo identificado (chamadas HTTP sequenciais) não tem relação com volume de dado processado, e sim com quantidade de chamadas de rede.
- **Não estimar DBU por falta de dado real de faturamento** (Free Edition não expõe consumo de DBU, já que é gratuito): descartada — mesmo sem medição real, uma estimativa fundamentada (taxa pública + tempo de execução observado) tem valor de FinOps maior do que nenhuma estimativa, desde que as premissas fiquem claramente marcadas como tal.
- **Citar um único valor de taxa de DBU sem faixa**: descartada — as fontes públicas convergem para uma faixa (US$ 0,15-0,30/DBU-hora para Jobs Compute Premium no Azure), não um número único; apresentar falsa precisão seria menos honesto do que expor a faixa real encontrada na pesquisa.

## Consequências

- **A verdadeira alavanca de FinOps deste projeto, numa eventual operação paga, seria compute, não armazenamento** — confirma e aprofunda a conclusão já registrada nas planilhas de armazenamento.
- **A descoberta do gargalo de chamadas HTTP sequenciais reforça, com número concreto, o Passo 2 do documento de escalabilidade** (`docs/anexos/escalabilidade_500_tickers.pdf`), que já recomendava avaliar paginação/lote antes de escalar — antes um risco qualitativo, agora com estimativa quantitativa de impacto em tempo e custo.
- Nenhum destes números é medição real — são estimativas fundamentadas em taxa pública e tempo de execução observado. Qualquer decisão de orçamento real exigiria a calculadora oficial do Databricks e, idealmente, um piloto medido com dado real de faturamento (não disponível na Free Edition).
- Caso o projeto seja retomado com foco em escala, a prioridade técnica correta — validada agora também sob a ótica financeira — é resolver a ingestão sequencial (paralelizar ou usar endpoint de cotação em lote, se existir) antes de qualquer otimização de armazenamento ou processamento Spark.