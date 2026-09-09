# ADR-18 — Correção de `modo_execucao` e categorização de anomalias (Teste vs. Operacional)

**Status:** Aceito
**Data:** 2026-09-04

## Contexto

Uma inconsistência real foi encontrada ao revisar o histórico de execuções: `03_silver`, `04_gold`, `05_reconciliacao` e `07_auditoria_execucoes` gravavam `modo_execucao = "reprocessamento_manual"` como texto fixo no código — mesmo quando disparados automaticamente pelo Job. Além disso, a Task `bronze` nunca teve o Parameter `modo_execucao` configurado no Job; ela só aparentava estar correta porque o valor padrão do seu Widget coincidia, por acaso, com `"agendado"`.

Paralelamente, ao investigar as anomalias detectadas pela auditoria automática (ADR-15), ficou claro que a maioria delas (12 de 19, na contagem mais recente) tinha origem no próprio processo de desenvolvimento e teste do projeto — não em comportamento real do pipeline em operação. Sem uma forma de distinguir as duas categorias, um avaliador revisando o histórico poderia interpretar erroneamente "muitas anomalias" como sinal de instabilidade de produção, quando na maior parte são ruído de construção do projeto.

## Decisão

**Widget `modo_execucao` adicionado aos 4 notebooks que não tinham** (`03_silver`, `04_gold`, `05_reconciliacao`, `07_auditoria_execucoes`), com padrão `"reprocessamento_manual"` — a leitura do valor passou de string fixa para `dbutils.widgets.get("modo_execucao")`.

**Parameter `modo_execucao = agendado` configurado explicitamente nas 7 Tasks do Job** (incluindo `bronze`, que nunca teve isso configurado corretamente) — elimina a dependência acidental de um valor padrão coincidente.

**Reconhecimento de limitação, não resolvida por completo**: mesmo com a correção, `modo_execucao` continua sendo um valor fixo definido na configuração da Task, não uma detecção real do gatilho da execução (agendamento vs. clique manual em "Run now"). Ou seja, a correção resolveu a *consistência* entre as 7 Tasks (todas dizem "agendado" quando disparadas pelo Job, incluindo execuções manuais via "Run now" do Job inteiro), mas não resolveu a *precisão* quanto ao gatilho real — avaliado e aceito como suficiente para o propósito do projeto.

**Novo notebook `databricks/auditoria/08_investigacao_anomalias`**, dedicado e centralizado, para o trabalho manual de preenchimento de causa raiz — antes espalhado em células soltas em múltiplos notebooks (`05`, `07`, notebook de testes), o que já havia causado confusão real sobre onde cada correção deveria morar.

**Taxonomia de prefixo para a causa raiz**, aplicada tanto em novas investigações quanto retroativamente nas 17 anomalias já documentadas:
- `[TESTE]` — anomalia originada do processo de desenvolvimento/teste do projeto (12 casos)
- `[OPERACIONAL]` — anomalia originada de comportamento real do pipeline em produção (5 casos, todos relacionados à sincronização tardia do CSV do KNIME em 28/08)
- `NAO DETERMINADA` (sem prefixo, padrão já existente desde o ADR-15) — causa não confirmada (2 casos)

**Dashboard atualizado** para refletir as 4 categorias resultantes (`Causa conhecida - Teste`, `Causa conhecida - Operacional`, `Requer atenção`, `Aguardando investigação`), tanto na tabela de anomalias (cores condicionais) quanto no gráfico de rosca de status de investigação — a lógica de classificação permanece dinâmica (baseada em `LIKE` sobre o conteúdo real da coluna `causa_raiz`), sem lista fixa de anomalias em nenhum lugar.

**Painel de Ranking de Valorização ajustado**: a consulta que buscava "o dia de maior dispersão de toda a janela" ficava permanentemente presa em 28/08 (único dia com dispersão real até então), já que nenhum dia seguinte superou esse valor — um problema de "congelamento" que só apareceria com o tempo, não nos testes iniciais. Corrigido para buscar a maior dispersão dentro de uma janela móvel dos últimos 5 dias corridos, equilibrando atualidade com robustez contra dias de dispersão zero.

## Alternativas consideradas

- **Investigar e implementar detecção real do gatilho de execução** (agendado vs. manual, via alguma API/contexto do Databricks): avaliada, mas não implementada — o ganho de precisão não justificava o esforço adicional nesta fase final do projeto, e a limitação já está documentada com clareza.
- **Automatizar o preenchimento de causa raiz por categoria** (ex.: toda anomalia de um certo tipo recebe automaticamente uma explicação): descartada — contradiria o princípio central do ADR-15 (detecção automática, explicação humana); automatizar a causa arriscaria aplicar explicação errada a um caso que parece igual mas não é.
- **Quarto valor de categoria para "hipótese reportada mas não confirmada"** (diferenciando de "nenhuma pista"): avaliada para o caso de 03/09 (que tinha uma hipótese do autor, mas não confirmada tecnicamente), descartada em favor de manter as 3 categorias originais — reclassificado como `NAO DETERMINADA`, mantendo a hipótese como texto dentro da causa raiz, sem criar uma quarta categoria de nuance.
- **Ranking de Valorização voltando a usar "último dia disponível"**: descartada — reintroduziria o risco original de barras zeradas na maioria das execuções (comportamento conhecido da API perto do fechamento, ver ADR-06).

## Consequências

- Os 7 notebooks e as 7 Tasks do Job agora são consistentes entre si quanto a `modo_execucao`, eliminando a fonte de confusão que motivou esta ADR.
- A distinção Teste vs. Operacional torna o histórico de anomalias interpretável de forma justa: das 19 anomalias registradas até 04/09, apenas 5 refletem comportamento real de operação (todas com a mesma causa raiz — dependência externa do KNIME), o restante é artefato do próprio processo de construção do projeto.
- O painel de Ranking deixa de correr o risco de ficar "congelado" indefinidamente num dia antigo, sem reintroduzir o problema de barras zeradas que motivou o desenho original.
- Este é o último ADR previsto do projeto — a partir daqui, o projeto entra em fase de **manutenção e validação**, sem novo desenvolvimento planejado, exceto em caso de algo parar de funcionar.