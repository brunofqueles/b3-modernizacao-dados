# ADR-15 — Auditoria automática de execuções, preservando investigação humana

**Status:** Aceito
**Data:** 2026-09-02

## Contexto

O projeto já registrava execuções em `observability.pipeline_runs` (ver [ADR-08](adr-08-observabilidade-modulo-compartilhado.md)), mas nada **analisava** esse histórico em busca de padrões que merecessem explicação. Durante a investigação manual de uma anomalia real (execuções duplicadas do Job em 28/08 e 01/09), ficou claro que esse tipo de análise — execuções próximas do mesmo notebook, gaps de execução esperada, duração fora do padrão — poderia (e deveria) ser automatizado, mas com um limite importante: **a causa raiz de uma anomalia é investigação humana, não algo que o sistema pode inferir sozinho** para casos futuros ainda não vistos.

Um segundo problema técnico surgiu ao decidir como gravar essas anomalias: a função `merge_ou_cria` já usada em todo o projeto ([ADR-08](adr-08-observabilidade-modulo-compartilhado.md)) faz `whenMatchedUpdateAll` — se aplicada aqui, uma reexecução diária apagaria qualquer `causa_raiz` anotada manualmente após investigação, silenciosamente.

## Decisão

**Nova função utilitária `inserir_se_novo`**, em `01_utilitarios_pipeline`, com semântica deliberadamente diferente de `merge_ou_cria`: insere apenas registros cuja chave ainda não existe (`whenNotMatchedInsertAll`, sem `whenMatchedUpdateAll`). Uma anomalia detectada automaticamente entra na tabela uma única vez; qualquer anotação humana feita depois (via `UPDATE` direto, como já fizemos para os casos reais) permanece intacta em reexecuções futuras do notebook.

**Novo notebook `databricks/auditoria/07_auditoria_execucoes`**, com três detecções automáticas sobre `observability.pipeline_runs`:
1. **Execuções próximas**: mesmo notebook, mesmo dia, menos de 30 minutos entre uma execução e a anterior.
2. **Gaps de execução esperada**: dias úteis sem registro de `01_ingestao_landing`.
3. **Duração anormal**: execuções com duração além de 2 desvios-padrão da média histórica do próprio notebook (métrica relativa, não um limiar fixo — cada notebook tem seu próprio padrão de "normal").

Grava em duas tabelas novas (`observability.auditoria_anomalias`, `observability.auditoria_gaps`), cada uma com uma coluna `causa_raiz` que nasce vazia e é preenchida manualmente após investigação — o mesmo processo humano que já fizemos para os 4 casos reais encontrados nesta sessão, agora persistido em tabela em vez de documentado apenas em texto solto.

**Execução automática via Task 7 no Job** (`auditoria_execucoes`, dependente de `alerta_divergencia`), garantindo que a tabela nunca fique desatualizada — mas o notebook continua **também executável manualmente** a qualquer momento, para checagem pontual sem esperar o próximo ciclo agendado.

**Achados reais desta auditoria, com causa raiz registrada**: das 12 anomalias detectadas no histórico completo do projeto, 11 têm causa conhecida e documentada (reprocessamentos de desenvolvimento do índice acumulado; reprocessamento por sincronização tardia do CSV do KNIME em 28/08, incluindo a observação de que o Widget `modo_execucao` não reflete de forma confiável o gatilho real quando não é alterado manualmente antes de um reprocessamento; testes controlados do try/except em 01/09). **1 permanece sem causa confirmada**: uma execução de `01_ingestao_landing` em 01/09 com duração de 157,87 segundos (2,57 desvios-padrão acima da média) — o autor confirma ter notado a lentidão na hora, mas sem registro do motivo exato. Registrada como "não determinada", não descartada nem inventada.

## Alternativas consideradas

- **Usar `merge_ou_cria` (mesma função das demais tabelas) para gravar as anomalias**: descartada — apagaria qualquer `causa_raiz` anotada manualmente a cada reexecução diária, destruindo o próprio propósito da tabela.
- **Tentar automatizar a causa raiz (ex.: correlacionar automaticamente com commits do Git, ou com o Widget `modo_execucao`)**: descartada nesta fase — o caso real do Widget `modo_execucao` mostrou que esse campo não é confiável sozinho como fonte de causa; uma correlação automática arriscaria inferir causas erradas com aparência de certeza. Investigação humana, ainda que mais lenta, é mais confiável nesta escala do projeto.
- **Não registrar a anomalia com causa "não determinada", omitindo o caso sem explicação**: descartada — omitir seria menos honesto do que registrar explicitamente "investigado, sem causa confirmada", que é a situação real.
- **Rodar a auditoria apenas sob demanda, sem Task automática no Job**: descartada como decisão final (foi a proposta inicial) — substituída para garantir que o Dashboard (próximo passo, página de observabilidade) sempre reflita dado atualizado, sem depender de alguém lembrar de rodar manualmente antes de cada apresentação.

## Consequências

- O projeto agora tem uma segunda camada de observabilidade: `pipeline_runs` registra o que aconteceu; `auditoria_anomalias`/`auditoria_gaps` registram o que **chamou atenção** dentro do que aconteceu, com investigação humana anexada.
- O padrão `inserir_se_novo` fica disponível para qualquer tabela futura do projeto que precise da mesma característica (detecção automática + anotação humana preservada) — não limitado a esta auditoria.
- A auditoria, rodando dentro do Job (Task 7), foi validada em execução real: as 7 Tasks concluíram em 4,50 minutos (dentro do limiar de 5 min), sem gerar anomalias novas — confirmando que Tasks diferentes rodando em sequência dentro do mesmo Job não disparam falsamente a detecção de "execução próxima" (que compara apenas execuções do *mesmo* notebook entre si).
- O caso de duração anormal sem causa confirmada permanece como item aberto — não bloqueia a apresentação, mas está registrado, rastreável, e não escondido.
- Este notebook ainda não alimenta o Dashboard — essa é a extensão natural planejada em seguida (nova página de observabilidade técnica), tratada em documento próprio quando implementada.