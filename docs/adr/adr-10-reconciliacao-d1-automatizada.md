# ADR-10 — Reconciliação automatizada D-1, tratamento explícito de erro e correção de dado residual

**Status:** Aceito
**Data:** 2026-08-28

## Contexto

A reconciliação (`05_reconciliacao`) era executada manualmente, com uma data de início fixa no código (`DATA_INICIO_RECONCILIACAO`), processando um intervalo aberto (tudo a partir daquela data) — decisão original registrada em [ADR-07](adr-07-reconciliacao-formato-tolerancia-manual.md), que também documentou a dependência de sincronização do CSV do KNIME como motivo para manter a execução manual. Avaliou-se automatizar essa etapa como 5ª Task do Job `pipeline_diario_b3`, com a condição de que a solução não dependesse de calendário fixo (a B3 não opera aos fins de semana, e "ontem" no calendário nem sempre corresponde ao último dia com dado disponível).

Durante a transição do desenho por intervalo para o desenho por dia único, um problema real de qualidade de dado foi identificado nas próprias tabelas de reconciliação: uma linha para 28/08 registrada com `status = diverge` e uma `causa_raiz` de "execuções em horários diferentes" — mas sem nenhum valor real do KNIME para comparar (`indice_proxy_pct_knime = null`, `diferenca_absoluta = null`). A causa raiz atribuída era factualmente incorreta: não havia divergência de horário nenhuma, porque não havia comparação real — o KNIME simplesmente não tinha dado para aquele dia ainda, e o `join` tipo `outer` (usado na versão anterior do notebook) produziu uma linha "fantasma" comparando um dado real (Gold) com uma ausência (KNIME).

## Decisão

**Detecção de D-1 por consulta aos dados, não por cálculo de calendário**: a data a reconciliar é a última `data_referencia` disponível em `gold.indice_proxy` anterior à data de execução (`< current_date()`), descoberta via consulta Spark. Essa abordagem se adapta automaticamente a fins de semana, feriados ou qualquer lacuna, sem exigir um calendário de pregão da B3 embutido no código.

**Widget de override manual mantido** (`data_reconciliacao`, texto, padrão vazio): quando vazio, aciona a detecção automática de D-1; quando preenchido, reconcilia a data especificada — preserva a capacidade de reprocessamento pontual já usada em outros notebooks do projeto.

**Filtro por dia exato, não por intervalo aberto**: tanto a leitura dos CSVs do KNIME quanto a leitura da Gold passaram a filtrar exatamente `data_referencia == DATA_RECONCILIACAO`, eliminando a possibilidade de comparar dias sem correspondência real entre os dois sistemas.

**Falha explícita quando o CSV do KNIME não existe**: `raise FileNotFoundError` com mensagem indicando a data ausente, interrompendo a execução de forma clara — primeira ocorrência de tratamento de erro explícito no projeto (distinto da detecção por ausência de registro descrita no [ADR-08](adr-08-observabilidade-modulo-compartilhado.md)), adequado especificamente a este caso porque a causa (KNIME não sincronizado) é conhecida e comunicável de forma precisa.

**Correção do dado residual incorreto**: as linhas de 28/08 geradas pela versão anterior do notebook (antes da migração para filtro exato) foram removidas manualmente de `reconciliation.resultado_indice` e `resultado_detalhado` via `DELETE`, após confirmação de que representavam uma comparação inválida, não uma divergência real.

**5ª Task adicionada ao Job `pipeline_diario_b3`** (`reconciliacao`, dependente de `gold`, "All succeeded"), sem necessidade de Parameters — o valor padrão vazio do Widget já aciona a detecção automática de D-1.

## Alternativas consideradas

- **Calcular D-1 como `data_execucao - 1 dia`, no calendário**: descartada — quebraria em segundas-feiras (D-1 seria sábado, sem dado nenhum) e em qualquer feriado da B3, exigindo manutenção de um calendário de pregão que o projeto não possui.
- **Manter o join tipo `outer` e apenas ignorar linhas com valores nulos na apresentação**: descartada — resolveria o sintoma (não mostrar a linha incorreta), mas manteria o dado inválido persistido na tabela, disponível para qualquer consulta futura que não aplicasse esse filtro adicional. Preferível eliminar a causa (comparar apenas dias com dado real dos dois lados) a mascarar o efeito.
- **Não corrigir a linha residual, apenas documentar como limitação conhecida**: descartada — o dado incorreto já estava persistido e acessível; corrigir na origem foi possível com baixo custo (`DELETE` direcionado) e preferível a conviver com um registro sabidamente errado numa tabela cujo propósito é ser fonte de verdade sobre a validade da migração.

## Consequências

- A reconciliação agora roda automaticamente como parte do Job diário, eliminando a dependência manual de "lembrar de rodar depois de confirmar a sincronização do KNIME" registrada como limitação no ADR-07 — a Task falha explicitamente (`FileNotFoundError`) se o KNIME não estiver sincronizado a tempo, tornando o problema visível via notificação de falha por e-mail, em vez de silenciosamente ausente.
- O incidente da linha residual é um exemplo concreto do próprio mecanismo de reconciliação (e do processo de desenvolvimento iterativo do projeto) expondo e corrigindo um problema real de qualidade de dado — material relevante para a seção de lições aprendidas do projeto.
- A execução de 31/08 (segunda-feira, dia da apresentação) poderá depender inteiramente da automação: desde que o KNIME de 31/08 seja executado e commitado antes das 17h15, a reconciliação D-1 (comparando 28/08, o dia anterior disponível a partir de segunda) e a cadeia completa do Job devem rodar sem intervenção manual.
- Caso o KNIME de algum dia da janela não seja commitado a tempo, a Task de reconciliação falhará de forma visível (e-mail de notificação), preservando a integridade das tabelas de reconciliação em vez de gravar dados inválidos ou incompletos.