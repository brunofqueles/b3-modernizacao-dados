# ADR-16 — Bug de cache no KNIME: dados congelados invalidam a reconciliação de 27/08 a 01/09

**Status:** Aceito
**Data:** 2026-09-03

## Contexto

Durante a construção da página 2 do Dashboard, uma inconsistência chamou atenção: o índice-proxy do KNIME (`indice_proxy_pct_knime`) aparecia com **exatamente o mesmo valor** (`0,227%`) em todas as datas da tabela `reconciliation.resultado_indice` — 27/08, 28/08, 31/08 e 01/09 — apesar de o lado Databricks mostrar valores diferentes e coerentes a cada dia (incluindo um dia com dispersão real, 28/08).

Investigação, passo a passo:
1. Consulta direta ao valor bruto de cada arquivo `indice_proxy_AAAA-MM-DD.csv` do KNIME confirmou: **26/08, 27/08, 28/08, 31/08 e 01/09 continham exatamente o mesmo número** (`0.002265146227187275`), inclusive o preço de cada ticker no CSV detalhado (ex.: PETR4 a R$41,45 repetido em múltiplos dias).
2. Consulta à `gold.indice_proxy` (Databricks) confirmou que o lado Databricks **nunca teve o mesmo problema** — valores distintos a cada dia, incluindo zeros legítimos já documentados (ver ADR-06).
3. No KNIME, o nó `Math Formula` (e os nós seguintes) apareciam com indicador **verde antes mesmo de clicar em "Execute all"** — sinal de que o workflow preservava o estado de execução anterior junto com o arquivo salvo.
4. `Reset all` seguido de `Execute all` gerou um valor novo (`0.0023014922397928753`) — confirmando que o cache estava mascarando a reexecução.

**Causa raiz confirmada**: o KNIME Analytics Platform salva o estado de execução (incluindo resultados) junto com o arquivo do workflow. Diferente do Databricks (onde cada execução de notebook é sempre do zero), "Execute all" no KNIME **não força reexecução** de nós que já aparentam estar prontos — ele reaproveita o resultado anterior sempre que julga que as entradas não mudaram. Isso afetou não apenas o cálculo final (`Math Formula`, `GroupBy`), mas também o **`GET Request`**: os preços "capturados" entre 27/08 e 01/09 nunca foram chamadas reais à API para essas datas — eram a mesma resposta de 27/08, salva em cache, reexportada com nomes de arquivo diferentes a cada dia.

## Decisão

**Não recalcular ou inventar os dados históricos afetados.** Não existe forma de saber, retroativamente, qual era o preço real de cada ticker às ~17h22-17h24 de 28/08, 31/08 ou 01/09 — o mercado já fechou, e a janela de captura passou. Qualquer valor "reconstruído" seria uma invenção com aparência de dado real, o oposto do padrão de honestidade já seguido no projeto inteiro.

**Corrigir a `causa_raiz` das 4 linhas afetadas** (`reconciliation.resultado_indice` e `resultado_detalhado`, datas 27/08 a 01/09), substituindo a explicação original ("defasagem de horário entre execuções") — que era **factualmente incorreta** para esses casos — pela causa real (bug de cache do KNIME), via `UPDATE` direcionado, mantido como código visível no notebook `05_reconciliacao` como evidência da correção, não apenas descrito em texto solto.

**Marcar essas 4 reconciliações como não-probatórias**: elas não confirmam nem invalidam a migração de lógica de negócio, porque um dos dois lados da comparação nunca teve dado real. **02/09/2026 passa a ser, oficialmente, o primeiro dia com reconciliação genuinamente válida** — o KNIME rodou com o cache limpo (`Reset all` + `Execute all`), gerando um valor real e diferente dos anteriores.

**A janela de reconciliação "27/08, 28/08, 31/08" documentada repetidamente no projeto (README, `architecture.md`, `business-context.md`) está, retroativamente, incorreta como prova de migração** — precisa ser corrigida em toda a documentação para refletir que a validade real começa em 02/09, não 27/08.

**Correção de hábito operacional**: a partir de agora, antes de qualquer execução do KNIME considerada "oficial" (a que gera o dado do dia), rodar **`Reset all` explicitamente antes de `Execute all`**, eliminando a possibilidade de cache mascarar uma não-execução real. Essa checagem entra no processo operacional documentado do projeto.

## Alternativas consideradas

- **Recalcular os valores de 27/08 a 01/09 usando alguma estimativa (ex.: interpolação, repetir o preço de 02/09 retroativamente)**: descartada — geraria dado fabricado com aparência de real, exatamente o tipo de imprecisão que o projeto evitou deliberadamente desde a decisão sobre o índice-proxy simplificado (ver `business-context.md`).
- **Apagar as 4 linhas afetadas da tabela de reconciliação**: descartada — apagar esconderia o incidente em vez de documentá-lo; mantê-las, com causa raiz corrigida, preserva o histórico completo e honesto do projeto, incluindo o processo de descoberta e correção do bug.
- **Manter a explicação original ("defasagem de horário") por ser "aproximadamente verdade" (afinal, também há defasagem real entre as execuções)**: descartada — a causa dominante nesses 4 dias não foi a defasagem de minutos (que é real e documentada desde o ADR-07), foi a ausência completa de captura nova. Manter a explicação antiga seria impreciso mesmo que parcialmente relacionado.
- **Reexecutar o KNIME retroativamente "fingindo" ser cada data antiga**: tecnicamente impossível — a API sempre retorna o preço atual, não haveria como simular o preço real de uma data passada (mesma limitação já documentada no ADR-01 sobre por que a janela de reconciliação não pôde incluir 26/08 retroativamente).

## Consequências

- **A decisão de ingestão independente (ADR-01) provou seu valor na prática, não só na teoria**: o bug de cache ficou inteiramente confinado ao KNIME — o pipeline Databricks (Landing → Bronze → Silver → Gold) nunca teve contato com o dado congelado do KNIME e continuou produzindo dado real todos os dias. Se a arquitetura tivesse acoplamento entre os dois sistemas (a alternativa descartada no ADR-01), o bug poderia ter se propagado.
- **A janela de reconciliação válida do projeto é, na prática, mais curta do que documentado até aqui**: apenas 02/09 (e dias seguintes) sustentam a alegação "Gold e KNIME comparados com dado real dos dois lados". Isso não invalida o restante do projeto (Bronze/Silver/Gold sempre foram reais), mas exige correção de linguagem em toda a documentação que citava "27/08, 28/08, 31/08" como a janela de reconciliação.
- **Novo item de processo operacional**: `Reset all` antes de `Execute all` no KNIME, para toda execução considerada oficial — mitigação direta contra a recorrência deste bug específico.
- **Achado com valor de portfólio real**: é um exemplo concreto e honesto de descoberta, investigação e correção de um bug de produção mascarado por comportamento de cache de ferramenta — exatamente o tipo de situação que aparece em ambientes reais de dados, tratado aqui com o mesmo rigor que o restante do projeto já demonstrava (não esconder, não inventar, documentar a causa real).