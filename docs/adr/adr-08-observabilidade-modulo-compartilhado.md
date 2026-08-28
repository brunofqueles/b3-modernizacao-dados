# ADR-08 — Observabilidade via log de execução e módulo utilitário compartilhado

**Status:** Aceito
**Data:** 2026-08-28

## Contexto

Desde o [ADR-02](adr-02-widgets-idempotencia-landing.md), o projeto reconhecia uma lacuna: nenhum notebook registrava sua própria execução, tornando impossível responder de forma automatizada "o pipeline quebrou? quando? por quê?" sem inspecionar manualmente cada tabela. Paralelamente, a função `merge_ou_cria` (lógica de MERGE idempotente) havia sido duplicada manualmente em `03_silver`, `04_gold` e `05_reconciliacao` — dívida técnica registrada desde o [ADR-04](adr-04-quarentena-dedup-schema-silver.md) — e a Bronze (`02_bronze`) sequer usava essa função, escrevendo a lógica de MERGE diretamente, inline, específica para sua tabela.

Ambos os problemas compartilham a mesma causa raiz técnica: notebooks do Databricks não compartilham funções ou estado entre si automaticamente, forçando repetição de código a menos que um mecanismo explícito de importação seja adotado.

## Decisão

**Notebook utilitário compartilhado** (`databricks/setup/01_utilitarios_pipeline`), importado pelos demais via `%run ../setup/01_utilitarios_pipeline` — mecanismo nativo do Databricks para reaproveitar código entre notebooks sem duplicação, mais simples que empacotar como módulo Python instalável, adequado à escala atual do projeto (5 notebooks, não uma biblioteca de produção). Contém duas funções:

- `merge_ou_cria(df, nome_tabela, colunas_chave)`: centraliza a lógica de MERGE idempotente, agora reaproveitada nos 4 notebooks que gravam tabelas Delta (incluindo a Bronze, que passou a usá-la pela primeira vez, abandonando sua lógica de MERGE inline).
- `registrar_execucao(notebook, data_referencia, modo_execucao, status, inicio, fim, mensagem_erro)`: grava um evento de execução na nova tabela `observability.pipeline_runs`.

**Novo schema `observability`**, criado via código no `00_setup_catalog` (mesmo padrão idempotente dos demais schemas), com a tabela `pipeline_runs`: `execucao_id` (UUID), `notebook`, `data_referencia`, `modo_execucao`, `status`, `mensagem_erro`, `inicio`, `fim`, `duracao_segundos`.

**Escrita em modo `append`, não MERGE**: diferente de todas as demais tabelas do projeto (que usam `merge_ou_cria` para refletir o estado mais atual por chave), `pipeline_runs` acumula um registro por execução — é um log de eventos, não um snapshot de estado. Cada execução gera uma linha nova, mesmo que rode várias vezes no mesmo dia.

**Detecção de falha por ausência de registro**, não por registro explícito de `status = "falha"`: cada notebook registra sucesso apenas ao final de sua execução completa. Se uma célula anterior falhar, o notebook interrompe e a célula de registro nunca roda — a ausência de um registro de sucesso para uma execução esperada é, em si, o sinal de falha. Um registro explícito de falha (com `mensagem_erro` detalhado) exigiria envolver cada notebook inteiro em blocos `try/except`, reestruturação mais invasiva, avaliada e adiada nesta fase para não introduzir risco sobre notebooks já validados e funcionando, próximo à data de entrega do projeto.

**`modo_execucao` inconsistente entre notebooks**: `01_ingestao_landing` e `02_bronze` possuem Widget próprio para esse valor; `03_silver`, `04_gold` e `05_reconciliacao` não têm Widget e registram valor fixo (`"reprocessamento_manual"`), não propagado automaticamente a partir da Task do Job `pipeline_diario_b3` que os executa. Reconhecido como limitação, não corrigida nesta fase.

## Alternativas consideradas

- **Empacotar as funções como módulo Python instalável (`.whl` ou similar)**: descartada — complexidade de empacotamento e deploy desproporcional à escala do projeto (5 notebooks); `%run` resolve o mesmo problema de forma nativa e imediata.
- **Registrar falha explicitamente via `try/except` em cada notebook, nesta fase**: descartada — risco de introduzir erro de estrutura em notebooks já validados e testados em execução real (incluindo via Job agendado), próximo da data de entrega. Adiada como evolução futura, priorizando estabilidade do que já funciona.
- **AI/BI Dashboard para visualizar `pipeline_runs`**: descartada nesta fase — consulta simples (`display()`) já cumpre o propósito de provar que a observabilidade funciona; dashboard formal registrado como evolução futura, não crítico para o objetivo do projeto.
- **Propagar `modo_execucao` do Job para todas as Tasks via Parameters**: avaliada, mas não implementada nesta fase para os 3 notebooks sem Widget — ajuste pequeno, mas não crítico, adiado para não atrasar a entrega dos itens de maior prioridade (observabilidade e eliminação de duplicação de código).

## Consequências

- A dívida técnica de duplicação de `merge_ou_cria`, reconhecida desde o ADR-04, está resolvida: a função existe em um único lugar, reaproveitada em 4 notebooks (Bronze, Silver, Gold, Reconciliação).
- O projeto agora tem visibilidade real sobre sua própria execução: 5 registros em `pipeline_runs` (um por notebook), validados numa execução completa em 28/08, todos com `status = sucesso`.
- Um erro técnico real foi encontrado e corrigido durante a implementação: `spark.createDataFrame` sem schema explícito falhou ao inferir tipo com `mensagem_erro=None` (`CANNOT_DETERMINE_TYPE`), corrigido definindo `StructType` explícito; e um segundo erro (`ArrowTypeError`) ao passar `data_referencia` como string em vez de `date`, corrigido com conversão defensiva dentro da própria função `registrar_execucao`.
- A detecção de falha por ausência de registro é funcionalmente mais fraca que um registro explícito de falha — distinguir "notebook não rodou" de "notebook falhou no meio" exige consultar outras fontes (ex.: histórico de execução do Job) até que a evolução com `try/except` seja implementada.
- A inconsistência de `modo_execucao` entre notebooks é uma limitação conhecida e aceita nesta fase, sem impacto no funcionamento do pipeline, apenas na granularidade da informação de rastreabilidade.