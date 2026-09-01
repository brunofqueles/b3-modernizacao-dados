# ADR-12 — Alertas nativos de falha e duração no Job

**Status:** Aceito
**Data:** 2026-09-01

## Contexto

O Job `pipeline_diario_b3` já contava com notificação de falha por e-mail desde sua criação (ver [ADR-06](adr-06-orquestracao-workflows-yaml.md)), mas sem endereço confirmado nem alerta de duração. Avaliou-se ampliar os alertas nativos do Databricks Jobs (sem código customizado) antes de investir em alertas customizados (ex.: divergência de reconciliação).

## Decisão

**Dois endereços de e-mail, com propósitos distintos**: `bruno.queles.dataeng@gmail.com` (endereço real, dedicado a projetos de portfólio, já validado em projeto anterior) recebe falha e aviso de duração; `bruno.quelestech@outlook.com` (endereço pessoal) permanece recebendo apenas falha, simulando um segundo interessado (ex.: "grupo de trabalho") sendo notificado de um evento crítico — mesmo padrão de dois destinatários já usado como referência em outro projeto do autor.

**Limiar de duração calibrado pelo tempo real observado, não por um valor arbitrário de livro-texto**: o tempo de execução medido (Tasks individuais entre 20-40s; Job completo entre 2-3,5 minutos) levou à escolha de **5 minutos** como limiar de aviso — folga suficiente (7-15x o tempo real por Task) para não gerar alarme falso, mas sensível o bastante para capturar uma lentidão real (ex.: API respondendo devagar, MERGE anormalmente lento) sem esperar um valor desproporcional como 15 ou 30 minutos.

**Dois níveis de limiar configurados, não apenas um**: threshold por Task individual (`Metric thresholds`, 5 min em cada uma das 5 Tasks) e threshold do Job como um todo (`Job health configuration`, 5 min na soma total). Justificativa: o primeiro identifica *qual* etapa específica ficou lenta; o segundo captura um cenário onde nenhuma Task isoladamente ultrapassa o limiar, mas a soma total ainda assim foge do padrão — os dois níveis respondem perguntas diferentes.

**Opções de "mute" (execuções puladas, canceladas, ou até o último retry) mantidas desmarcadas**, deliberadamente — cada uma dessas situações pode carregar informação relevante (ex.: um retry automático mascarando uma falha real na primeira tentativa), e silenciá-las por padrão reduziria a visibilidade sem ganho real de redução de ruído.

## Alternativas consideradas

- **Um limiar de duração alto (ex.: 15 minutos), inspirado em outro projeto do autor**: descartada para este projeto — aquele outro pipeline tem volume e tempo de execução reais muito maiores (Tasks de geração/ingestão de múltiplos sistemas); replicar o mesmo número sem calibrar pelo tempo real deste projeto seria copiar uma configuração, não fazer engenharia baseada em dado observado.
- **Apenas um endereço de e-mail para todos os alertas**: descartada — perderia a oportunidade de simular, de forma simples e sem custo, o cenário real de múltiplos interessados recebendo alertas críticos.
- **Configurar apenas o limiar por Task, ou apenas o do Job inteiro**: descartada — cada um cobre um cenário de detecção que o outro não cobre (ver Decisão).

## Consequências

- Uma execução real de validação (01/09) confirmou o funcionamento sem falso positivo: Job completo em 3,42 minutos, abaixo do limiar de 5 minutos configurado nos dois níveis, nenhum alerta de duração disparado.
- O limiar de 5 minutos é um baseline inicial, não uma calibração estatística rigorosa — o projeto ainda tem poucos dias de histórico real de execução. Recalibração é esperada caso o projeto seja retomado com mais tempo de operação acumulado.
- O alerta de falha ainda não carrega informação agregada (quantas vezes um notebook específico já falhou, por qual motivo) — essa limitação depende da extensão do tratamento explícito de erro aos demais notebooks, já registrada como evolução pendente (ver ADR-08, ADR-10).