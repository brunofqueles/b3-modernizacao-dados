# ADR-09 — Índice acumulado (base 100) via capitalização composta

**Status:** Aceito
**Data:** 2026-08-28

## Contexto

O 5º indicador da Gold, já registrado como pendente desde [ADR-05](adr-05-indicadores-gold-automatizada.md), dependia de pelo menos 2 dias de dados válidos na Silver — condição satisfeita a partir de 28/08 (27/08 e 28/08 disponíveis). O objetivo é expressar como o índice-proxy evoluiu ao longo dos dias da janela de reconciliação, não apenas o retorno isolado de cada dia.

## Decisão

**Capitalização composta, não soma simples**: cada dia parte do nível acumulado do dia anterior (`nivel(dia) = nivel(dia-1) * (1 + indice_proxy_pct(dia)/100)`), refletindo como índices de mercado reais (Ibovespa, S&P 500) de fato acumulam variação — soma simples ignoraria que cada dia de variação incide sobre o valor já acumulado, não sobre uma base fixa.

**Base 100 no primeiro dia da janela** (27/08/2026), mesma convenção usada por índices de mercado reais para expressar nível sem depender de unidade monetária.

**Implementação via log/soma/exponencial**: o Spark não possui agregação nativa de produto acumulado (apenas soma). A capitalização composta foi implementada convertendo cada fator diário para logaritmo, somando os logaritmos numa janela (`Window`) ordenada por data, e revertendo com exponencial — matematicamente equivalente a multiplicar os fatores diários em sequência, mas utilizando apenas operações nativas do Spark.

**Reprocessamento completo do histórico a cada execução**: diferente dos demais indicadores da Gold (que processam apenas o dia da execução), o índice acumulado lê toda a tabela `gold.indice_proxy` a cada vez que roda, recalculando a série completa. Necessário porque o valor de cada dia depende de todos os dias anteriores — não é possível calcular incrementalmente sem manter o último nível acumulado como estado externo, o que adicionaria complexidade desproporcional ao benefício nesta fase.

**Nova tabela dedicada**: `gold.indice_acumulado`, gravada via `merge_ou_cria` com chave `data_referencia` — mesmo padrão idempotente das demais tabelas.

## Alternativas consideradas

- **Soma simples dos retornos diários**: descartada — matematicamente incorreta para retornos financeiros, não reflete como o valor realmente evolui ao longo do tempo.
- **Calcular incrementalmente, mantendo apenas o último nível como estado**: descartada nesta fase — exigiria uma tabela ou variável de estado externa e lógica de atualização incremental; reprocessar o histórico completo (ainda pequeno, poucos dias) é mais simples e igualmente correto, ao custo de recalcular dados já processados a cada execução.
- **Adicionar a coluna diretamente em `gold.indice_proxy`**, em vez de uma tabela nova: descartada — misturaria um indicador de comportamento diário (índice-proxy, retorno do dia) com um indicador de série histórica (nível acumulado), com granularidade e propósito de leitura diferentes; tabelas separadas mantêm cada uma com responsabilidade única.

## Consequências

- O projeto agora tem os 5 indicadores planejados desde o ADR-05: retorno diário, índice-proxy, ranking, dispersão, e índice acumulado.
- A tabela `gold.indice_acumulado` cresce em volume conforme a janela de dias aumenta, mas continua pequena o suficiente (poucas dezenas de linhas ao longo de todo o projeto) para que o reprocessamento completo a cada execução não gere custo de performance perceptível.
- O aviso do Spark sobre "Window sem partição" (`No Partition Defined for Window operation`) é esperado e inofensivo na escala atual do projeto — seria uma preocupação real apenas em volumes de dados muito maiores, fora do escopo atual.
- Caso o projeto seja retomado com volume de dados significativamente maior (ex.: histórico de meses ou mais tickers), a estratégia de reprocessamento completo do histórico deveria ser revisada em favor de cálculo incremental.