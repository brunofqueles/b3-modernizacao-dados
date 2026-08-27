# ADR-06 — Orquestração via Databricks Workflows e YAML como documentação leve de IaC

**Status:** Aceito
**Data:** 2026-08-27

## Contexto

Com Landing, Bronze, Silver e Gold implementados como notebooks independentes, o projeto precisava de uma forma de encadeá-los e executá-los sem intervenção manual — completando a automação real (distinta de apenas ter lógica em código, ver [ADR-05](adr-05-indicadores-gold-automatizada.md)) e viabilizando a execução padronizada após o fechamento do pregão nos 3 dias da janela de reconciliação.

Adicionalmente, o projeto já havia decidido não adotar Databricks Asset Bundles (DAB) nesta fase, por restrição de prazo (ver `architecture.md`, seção de Evolução). Durante a criação do Job, avaliou-se uma alternativa leve que aproxima o projeto de práticas de IaC sem pagar o custo de aprender a sintaxe do `databricks.yml` ou do fluxo `bundle deploy`.

## Decisão

**Orquestração via Databricks Workflows**, Job único (`pipeline_diario_b3`) com 4 Tasks em cadeia linear, cada uma dependente da anterior com a condição "All succeeded" (uma falha interrompe a cadeia, evitando erro em cascata sobre dado inexistente ou incompleto):

```
ingestao_landing → bronze → silver → gold
```

**Agendamento restrito a dias úteis**, não diário: `0 15 17 ? * MON-FRI` (Quartz cron), timezone `America/Sao_Paulo`. Horário ajustado de 17h para **17h15** — margem adicional após o fechamento regular do pregão da B3, e compatibilidade com a agenda do autor (execução de 28/08 e apresentação do projeto às 17h45 de 31/08).

**Notificação de falha por e-mail**, configurada nativamente no Job (`email_notifications.on_failure`) — primeira peça real de observabilidade do projeto, ainda que a tabela de log de execução completa (`pipeline_runs` ou similar) permaneça como item de evolução futura.

**YAML exportado como documentação, não como fonte de verdade deployada**: após configurar o Job pela interface (UI), o YAML resultante foi exportado via "View as code" e commitado em `databricks/jobs/pipeline_diario_b3.yml`, com cabeçalho explícito informando que o arquivo é uma cópia de referência — editá-lo não altera o Job real, e ele deve ser reexportado sempre que a configuração mudar pela UI. Essa prática aproxima o projeto de Databricks Asset Bundles sem exigir a curva de aprendizado completa de `bundle deploy` agora, servindo como ponto de partida caso o projeto seja retomado e a migração para DAB completo se torne viável.

## Alternativas consideradas

- **Executar os 4 notebooks manualmente, célula por célula, nos 3 dias da janela de reconciliação**: descartada — contradiz diretamente o objetivo de demonstrar migração para uma arquitetura moderna com orquestração real, e reintroduziria risco de erro humano (esquecer uma etapa, rodar fora de ordem).
- **Configurar o Job diretamente via `databricks.yml` e `bundle deploy` (DAB completo)**: descartada nesta fase pelo mesmo motivo já registrado na decisão original de não adotar DAB — custo de aprendizado da sintaxe e do fluxo de deploy não compensa dentro do prazo do projeto.
- **Agendamento diário, incluindo fins de semana**: descartada — a B3 não opera aos sábados e domingos; rodar o pipeline nesses dias geraria execuções sem dado novo relevante, sem benefício real.
- **Não configurar notificação de falha**: descartada — mesmo sendo uma peça pequena, é a primeira evidência concreta de observabilidade funcionando no projeto, reforçando a seção de evolução já documentada em `architecture.md`.

## Consequências

- O pipeline agora roda de ponta a ponta sem intervenção manual, validado em execução real (27/08, ~2 minutos de duração total, sem erros) — primeira prova de que a automação completa (código automatizado + execução agendada) funciona, não apenas testada célula por célula.
- A janela de reconciliação (27/08, 28/08, 31/08) passa a depender do agendamento automático nos dois dias restantes, reduzindo risco de esquecimento em relação à execução manual.
- O arquivo YAML em `databricks/jobs/` precisa de manutenção manual (reexportação) sempre que o Job for editado pela UI — não há sincronização automática entre os dois; essa é uma limitação conhecida da abordagem "documentação leve" escolhida, aceitável dado que o Job não deve mudar com frequência no restante do prazo do projeto.
- Identificada, na primeira execução agendada, uma observação relevante para a interpretação da reconciliação: nesse horário (próximo ao fechamento), a API retornou `regularMarketPreviousClose` idêntico a `regularMarketPrice` para os 4 tickers, resultando em índice-proxy e dispersão zerados — registrado como limitação de comportamento da fonte externa em `architecture.md`, seção de riscos, não como falha do pipeline.