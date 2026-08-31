# ADR-11 — Camada de consumo executivo: AI/BI Dashboard e Genie Space

**Status:** Aceito
**Data:** 2026-08-31

## Contexto

Com o pipeline técnico completo (Landing → Bronze → Silver → Gold → Reconciliação, orquestrado e observável), faltava uma camada de consumo voltada a um público não técnico — a apresentação do projeto é dirigida a uma audiência executiva, que não vai ler notebooks, ADRs, nem consultar tabelas diretamente. O Databricks oferece dois recursos nativos para isso: **AI/BI Dashboard** (visualização) e **Genie Space** (consulta em linguagem natural).

## Decisão

**Dashboard consolidado em 4 painéis, não 5**: a proposta original de indicadores foi condensada para caber no volume real de dados disponível (2-3 dias, janela de reconciliação ainda em andamento) — um dashboard com muitos painéis esparsos comunicaria pior do que poucos painéis robustos. Estrutura final, em ordem de apresentação (do mais executivo ao mais técnico):
1. Índice acumulado (linha) — visão geral da evolução
2. Ranking de valorização (barra) — resposta direta a "quais ações mais se valorizaram"
3. Reconciliação (tabela, com formatação condicional) — prova de migração
4. Observabilidade (tabela) — maturidade operacional, posicionado por último por ser o mais técnico

**SQL como exceção justificada ao padrão PySpark-only**: os datasets do Dashboard são definidos nativamente em SQL (não há alternativa em PySpark para essa ferramenta específica) — exceção pontual e documentada ao padrão de desenvolvimento já registrado no projeto, não uma mudança de rumo.

**Formatação condicional restrita aos valores reais da coluna `status`**: apenas duas regras — `diverge` (laranja) e `bate` (verde) — refletindo exatamente os dois únicos valores que a lógica de reconciliação (ver [ADR-07](adr-07-reconciliacao-formato-tolerancia-manual.md)) pode gerar. Deliberadamente **não foi criada** uma terceira regra para um estado "crítico"/vermelho, por não existir essa categoria na lógica real do pipeline — criar uma regra visual para um estado inexistente enganaria o leitor do dashboard sobre o que o sistema realmente distingue.

**Genie Space com acesso curado, não irrestrito**: apenas tabelas Gold e Reconciliação foram conectadas (`gold.indice_proxy`, `gold.indicadores_diarios`, `gold.indice_acumulado`, `reconciliation.resultado_indice`, `reconciliation.resultado_detalhado`) — Bronze, Silver e Landing foram deliberadamente excluídas. Motivos: (1) precisão — mais tabelas aumentam o risco do Genie escolher a fonte errada para perguntas ambíguas; (2) coerência com o modelo de governança já documentado em `architecture.md` (consumidores de negócio veem dado curado, não dado bruto ou em processo de limpeza).

**Instruções do Genie embutindo as ressalvas de domínio já documentadas no projeto**: o campo de instruções replica, em linguagem dirigida à IA, as mesmas ressalvas já presentes em `business-context.md` e nos ADRs — que o índice-proxy não é a metodologia oficial de nenhum índice real da B3; que o índice acumulado usa base 100 com capitalização composta; que uma divergência na reconciliação tem causa raiz conhecida e não indica falha de migração; que os valores já vêm em percentual. Sem essas instruções, o Genie poderia gerar respostas tecnicamente incorretas ou alarmantes ao ser questionado por alguém sem contexto do projeto.

**3 exemplos de pergunta+SQL** (tipo "Example query"), cobrindo os 3 principais painéis do dashboard (ranking, índice acumulado, reconciliação) — número deliberadamente pequeno, proporcional ao volume real de dados e casos de uso do projeto nesta fase, evitando prometer uma cobertura de perguntas que os dados ainda não sustentam.

## Alternativas consideradas

- **Manter os 5 painéis originalmente planejados**: descartada — dado o volume de dados disponível (2-3 dias), ao menos 1-2 painéis ficariam visualmente esparsos (ex.: um gráfico de série temporal com 2 pontos), enfraquecendo a comunicação em vez de fortalecê-la.
- **Conectar todo o catalog ao Genie** (todos os schemas): descartada — ver justificativa de precisão e governança na decisão acima.
- **Criar uma terceira categoria de status "crítico" com cor vermelha**, sugerida durante a construção: descartada — não existe essa categoria na lógica real do pipeline; adicioná-la visualmente sem lastro no código criaria uma representação enganosa do sistema.

## Consequências

- O dashboard e o Genie Space dependem de dados que ainda estão numa janela curta (2-3 dias); a experiência de consumo tende a melhorar significativamente com mais dias de execução acumulados, sem necessidade de nenhuma mudança estrutural — os painéis e datasets já foram desenhados para escalar automaticamente (uso de `MAX(data_referencia)`, sem datas fixas no código).
- Durante os testes desta etapa, observou-se uma anomalia não investigada a fundo: o Job `pipeline_diario_b3` parece ter dois conjuntos de execuções muito próximos entre si no dia 28/08 (~17:15 e ~17:28-17:31, horário local), visível na tabela `observability.pipeline_runs`. A escrita idempotente (MERGE) evitou qualquer duplicação ou inconsistência de dado, mas a causa da segunda execução (disparo manual sobreposto ao agendamento, ou outro motivo) não foi determinada — registrado aqui como observação honesta, não investigada a fundo por restrição de tempo antes da apresentação.
- A camada de consumo executivo (Dashboard + Genie) está pronta para demonstração, mas depende do restante do pipeline continuar funcionando corretamente nas execuções restantes (31/08) para que os dados exibidos reflitam a janela completa de reconciliação na hora da apresentação.