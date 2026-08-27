# ADR-02 — Parametrização via Widgets e escrita idempotente na Landing Zone

**Status:** Aceito
**Data:** 2026-08-27

## Contexto

O notebook `01_ingestao_landing` (primeira peça do pipeline Databricks) precisa ser reutilizável: rodar para todos os tickers ou apenas um específico, apontar para uma data de referência específica (útil para reprocessamento), e não gerar dados duplicados caso seja executado mais de uma vez no mesmo dia (ex.: um teste manual de manhã, seguido da execução oficial às 17h).

Além disso, o projeto já sinalizou escalabilidade (crescer de 4 tickers para uma lista maior) como uma evolução futura — o desenho do notebook precisa comportar isso sem reescrita, na medida do possível.

## Decisão

**Parametrização via `dbutils.widgets`:**
- `data_referencia` (texto, padrão vazio): quando vazio, assume a data de hoje automaticamente (`datetime.now()`); só é preenchido manualmente em casos pontuais de reprocessamento de uma data específica.
- `ticker` (dropdown com opção `todos` + os 4 tickers individuais): permite rodar o pipeline completo ou isolar um único papel para teste/depuração, sem editar código.
- `modo_execucao` (dropdown: `agendado` / `reprocessamento_manual`): não altera o comportamento do notebook nesta fase, mas registra a intenção da execução — insumo para a futura observabilidade (ver seção "Evolução" em `architecture.md`).

**Escrita idempotente por sobrescrita:** o nome do arquivo gravado no Volume (`cotacoes_<data_referencia>.json`) é determinístico a partir da data de referência. Executar o notebook duas vezes no mesmo dia sobrescreve o arquivo anterior, em vez de duplicar ou acumular. Isso mantém a Landing Zone com **um snapshot por dia**, alinhado à mesma decisão já tomada no workflow KNIME (CSVs versionados por data, não por execução).

## Alternativas consideradas

- **Widget `tickers` como campo de texto livre (múltiplos valores separados por vírgula)**: descartada — sujeita a erro de digitação (ex.: "PETR" sem o "4") e exige o usuário lembrar a sintaxe exata.
- **Widget `tickers` como multiselect**: avaliada e implementada inicialmente, mas substituída por dropdown único com opção `todos` — mais simples de operar para o caso de uso real do projeto (rodar tudo, ou isolar um único papel por vez), sem a complexidade extra de seleção múltipla que não tinha demanda concreta ainda.
- **Nome de arquivo incluindo timestamp de execução (ex.: `cotacoes_2026-08-27_17-03-00.json`)**: descartada — acumularia múltiplos arquivos por dia, indo contra a decisão de "um snapshot por dia" e complicando a leitura posterior pela Bronze (que precisaria escolher qual arquivo do dia é o "oficial").

## Consequências

- O notebook é reutilizável para reprocessamento pontual de datas passadas, mas **não busca preço histórico real** — a API `brapi.dev` retorna sempre a cotação atual no momento da chamada; o widget `data_referencia` apenas rotula/particiona onde o dado é salvo, não obtém dados retroativos.
- Execuções de teste (ex.: manhã) são silenciosamente substituídas por execuções posteriores do mesmo dia (ex.: 17h), sem nenhum registro de que a sobrescrita ocorreu — lacuna de observabilidade reconhecida e documentada como item de evolução futura, não resolvida nesta fase.
- Adicionar novos tickers no futuro (escalabilidade) exige editar a lista fixa de opções do dropdown `ticker` — não é uma solução dinâmica baseada em catálogo de tickers; suficiente para o escopo atual (4 papéis), mas seria um ponto de revisão caso o projeto escale para uma lista maior (ex.: 500 papéis).