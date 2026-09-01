# ADR-13 — Try/except completo no pipeline e alerta de divergência anormal aditivo

**Status:** Aceito
**Data:** 2026-09-01

## Contexto

Desde o [ADR-08](adr-08-observabilidade-modulo-compartilhado.md), o projeto reconhecia que a detecção de falha funcionava apenas por **ausência** de registro de sucesso — só o `05_reconciliacao` tinha tratamento explícito de erro (`raise FileNotFoundError`, ver [ADR-10](adr-10-reconciliacao-d1-automatizada.md)). Os outros 4 notebooks (`01_ingestao_landing`, `02_bronze`, `03_silver`, `04_gold`) não registravam `status = "falha"` de forma explícita, deixando uma lacuna real de diagnóstico. Paralelamente, o [ADR-12](adr-12-alertas-nativos-falha-duracao.md) havia estabelecido alertas nativos de falha e duração, mas sem cobrir o cenário "a execução teve sucesso técnico, mas os valores da reconciliação divergem de forma anormal".

## Decisão

**Try/except consolidado numa única célula de execução principal, em cada um dos 4 notebooks restantes.** Databricks executa células de notebook de forma isolada — um erro numa célula não pode ser capturado por um `except` em célula diferente. Isso exigiu reestruturar cada notebook, absorvendo a lógica antes espalhada em múltiplas células (leitura, transformação, gravação, validação) num único bloco `try`, com `registrar_execucao(status="falha", mensagem_erro=str(e))` no `except`, seguido de `raise` (preserva a falha visível na interface do Databricks e no Job).

**Correção de lacuna real descoberta em teste**: `response.raise_for_status()` adicionado à chamada da API em `01_ingestao_landing`. Sem essa linha, um status HTTP de erro (ex.: 404) com corpo JSON válido não gerava exceção Python — o notebook registraria `status = "sucesso"` mesmo diante de uma falha real da fonte de dados.

**Novo notebook aditivo `06_alerta_divergencia`**, não vinculado à lógica de `05_reconciliacao` (não a modifica, apenas lê seu resultado já gravado em `reconciliation.resultado_indice`). Dispara e-mail HTML apenas quando a divergência entre KNIME e Databricks ultrapassa **1% de diferença absoluta** — limiar escolhido por ser o dobro da maior divergência real já observada no projeto (0,43%), calibrado para permanecer silencioso em condições normais e disparar apenas diante de algo genuinamente fora do padrão.

**Credencial de e-mail via Databricks Secret Scope**, criado pelo Databricks CLI através do Web Terminal (não pelo terminal local, que não tinha o CLI instalado) — `b3-secrets/gmail-app-password`, nunca em texto no código. Validado com teste de redação automática (`print()` do secret retorna `[REDACTED]`).

**6ª Task adicionada ao Job `pipeline_diario_b3`** (`alerta_divergencia`, dependente de `reconciliacao`), mesmo padrão de threshold de duração (5 min) das demais Tasks, YAML reexportado.

**Decisão explícita de não customizar as mensagens dos alertas nativos** (falha e duração do [ADR-12](adr-12-alertas-nativos-falha-duracao.md)): o Databricks Jobs não oferece campo de edição de corpo para essas notificações — são template fixo da plataforma. Customizar exigiria replicar o mesmo mecanismo `smtplib` usado no alerta de divergência dentro do `except` de cada um dos 4 notebooks. Avaliado e **descartado por desproporção entre custo e benefício**: os e-mails nativos já entregam o valor essencial (avisar que algo falhou ou demorou); o retrabalho de tocar novamente nos 4 notebooks recém-estabilizados não se justificou.

**Limpeza dos registros de teste em `observability.pipeline_runs`**: 10 registros removidos via `DELETE` direcionado (7 falhas propositais geradas durante os testes de try/except — URLs, caminhos e tabelas inválidos usados de propósito — e 3 registros de sucesso com timestamp confuso, artefato de testes fora de ordem). Mesma disciplina de correção de dado já aplicada no [ADR-10](adr-10-reconciliacao-d1-automatizada.md).

## Alternativas consideradas

- **Try/except célula por célula**: descartada — tecnicamente impossível, uma célula não pode capturar exceção lançada por outra célula no Databricks.
- **Alertar em toda divergência da reconciliação**: descartada — reabriria o problema de ruído já resolvido no [ADR-07](adr-07-reconciliacao-formato-tolerancia-manual.md); o limiar de 1% mantém o alerta raro e significativo.
- **Customizar os e-mails nativos de falha/duração**: avaliada e descartada nesta fase — custo desproporcional (reestruturar 4 notebooks de novo) frente ao ganho (mensagem mais bonita, sem novo dado funcional além do que já existe no Databricks e em `pipeline_runs`).
- **Manter os registros de teste na tabela de observabilidade, sem limpeza**: descartada — poluiria qualquer análise futura (incluindo o próprio painel de observabilidade do dashboard) com dados fabricados.

## Consequências

- Os 5 notebooks do pipeline agora têm tratamento de erro explícito e consistente — a lacuna registrada desde o ADR-08 está fechada.
- O alerta de divergência anormal foi validado apenas por simulação controlada (nunca disparou por um caso real, já que todas as divergências observadas até agora ficaram entre 0,07% e 0,43%, abaixo do limiar) — seu comportamento em um cenário genuinamente anormal permanece não testado com dado real.
- A 6ª Task aumentou o tempo total do Job — uma execução manual registrou 5,49 minutos, próximo ou acima do limiar de 5 minutos definido no nível do Job inteiro (ADR-12). Recalibração desse limiar foi deliberadamente adiada até observar o comportamento de execuções agendadas reais, evitando ajustar um número a partir de uma única amostra manual possivelmente não representativa.
- As mensagens dos alertas nativos (falha, duração) permanecem no template padrão do Databricks — decisão consciente de escopo, não lacuna não percebida.
- Painel de observabilidade do dashboard ainda não reflete "falhas por notebook" — item que só se tornou viável a partir desta etapa (agora existe dado real de falha) e permanece como próximo passo.