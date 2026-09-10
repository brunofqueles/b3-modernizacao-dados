# ADR-19 — Resiliência a CSV ausente do KNIME: reconciliação não deve derrubar o pipeline

**Status:** Aceito
**Data:** 2026-09-10

## Contexto

Em produção real (fase de manutenção, pós-entrega), o pipeline falhou em dois dias consecutivos de execução agendada: **07/09** (feriado — Job disparou normalmente via cron, mas o autor não rodou o KNIME nesse dia, comportamento esperado) e **09/09** (dia útil, mesma causa: KNIME não rodado no dia anterior). O comportamento original do `05_reconciliacao` — `raise FileNotFoundError` quando o CSV do KNIME não existe para a data D-1 — propagava o erro e, por dependência ("All succeeded"), derrubava em cascata `alerta_divergencia` e `auditoria_execucoes`, mesmo com Landing/Bronze/Silver/Gold tendo funcionado perfeitamente.

Esse comportamento era correto **durante a construção do projeto** (queríamos saber imediatamente se um CSV estava faltando) — mas numa fase de manutenção real, um dia sem execução manual do KNIME (feriado, viagem, esquecimento) é esperado e não deveria comprometer o restante do pipeline, que continua tendo dado real do lado Databricks.

Um segundo problema, encontrado ao investigar o primeiro: **08/09 (dia útil, sem feriado) não teve nenhum registro em `pipeline_runs`** — nem sucesso, nem falha. Diferente dos outros dois casos, não há explicação técnica disponível: o agendamento do Job estava confirmado ativo, rodou normalmente antes e depois desse dia. **Causa não confirmada.**

## Decisão

**`05_reconciliacao` não propaga mais erro quando o CSV do KNIME está ausente.** Em vez de `raise FileNotFoundError`, registra `status = "pendente_knime"` (um terceiro valor de status, distinto de `sucesso` e `falha` — reflete a realidade: não é falha de pipeline, também não é uma reconciliação concluída) e encerra via `dbutils.notebook.exit(...)`, que finaliza o notebook sem lançar exceção — a Task é considerada bem-sucedida pelo Job, e as Tasks seguintes (`alerta_divergencia`, `auditoria_execucoes`) continuam rodando normalmente no mesmo dia.

**`06_alerta_divergencia` passa a checar o status da execução mais recente de `05_reconciliacao` antes de avaliar qualquer divergência.** Sem essa checagem, ele releria a última linha gravada em `reconciliation.resultado_indice` — que, num dia de reconciliação pendente, seria uma linha **antiga**, de um dia já avaliado antes. Na melhor hipótese isso é redundante; na pior, poderia reenviar um alerta de divergência alta sobre dado que já gerou e-mail anteriormente. Quando a reconciliação mais recente está `pendente_knime`, a avaliação é pulada por completo, sem tentativa de envio.

**Lacuna de observabilidade do `06_alerta_divergencia` fechada nesta mesma correção** (identificada desde o ADR-15/17, nunca resolvida até agora): adicionado Widget `modo_execucao`, marcação de início, `try/except` e `registrar_execucao` — igualando-o aos demais 6 notebooks do pipeline.

**Novo notebook `databricks/auditoria/09_verificacao_pos_deploy`**, para checagem ad-hoc de estado (ex.: "até que data cada camada tem dado disponível") — deliberadamente **não automatizado via Job**: hoje contém apenas consultas pontuais escritas durante esta investigação, sem uma lógica fixa e repetível que justificasse virar uma 8ª Task. Avaliado e descartado automatizá-lo nesta fase — decisão consciente, não esquecimento.

**Gap de 08/09 registrado sem causa confirmada**, mesmo padrão de honestidade já aplicado a outros casos do projeto (ex.: duração anormal sem explicação técnica) — não investigado a fundo por não haver evidência disponível para diagnosticar (ambiente Free Edition, sem log de infraestrutura acessível), não por falta de tentativa.

## Alternativas consideradas

- **Manter o `raise` e apenas ajustar as dependências das Tasks seguintes** (ex.: "Run if: at least one succeeded" em vez de "All succeeded"): descartada — mudaria a semântica de todas as dependências do Job de forma mais ampla e menos previsível do que tratar o caso específico na origem (a própria reconciliação).
- **Reconciliar automaticamente contra o CSV mais recente disponível, mesmo que não seja D-1**: descartada — misturaria datas de referência diferentes na reconciliação, comprometendo a integridade da comparação (mesmo tipo de raciocínio já aplicado no ADR-01, sobre não recuperar dado retroativo).
- **Investigar a fundo a causa de 08/09 antes de declarar como "não confirmada"**: avaliada, mas o ambiente (Databricks Free Edition) não expõe logs de infraestrutura de agendamento suficientes para diagnosticar com certeza; declarar honestamente a ausência de causa foi preferido a especular.
- **Automatizar `09_verificacao_pos_deploy` como 8ª Task do Job**: considerada durante a implementação desta correção, descartada por não haver ainda uma lógica de checagem fixa e repetível — permanece como ferramenta manual, reavaliável no futuro se um padrão de uso recorrente emergir.

## Consequências

- O pipeline agora é resiliente à ausência esperada de dado do sistema legado (feriados, dias sem execução manual) — um dia sem KNIME não compromete mais alertas nem auditoria.
- Um novo valor de status (`pendente_knime`) precisa ser considerado por qualquer análise futura sobre `observability.pipeline_runs` — não é `sucesso` nem `falha` no sentido usual.
- `06_alerta_divergencia` agora aparece em `pipeline_runs` como os demais notebooks, fechando a última lacuna de observabilidade conhecida do pipeline core.
- O gap de 08/09 permanece como item aberto, sem causa confirmada — monitorado, não resolvido; se o padrão se repetir, será o sinal de que algo sistemático (não pontual) está em jogo.
- Esta correção foi motivada por uso real em produção, não por teste — validação prática do valor da observabilidade construída ao longo do projeto: o problema foi detectado pelo próprio autor operando o sistema, com evidência suficiente (e-mails de alerta, tabela de execução) para diagnosticar e corrigir em uma sessão.