# ADR-17 — Dashboard: correções de dado, causa raiz dinâmica e página de observabilidade técnica

**Status:** Aceito
**Data:** 2026-09-03

## Contexto

O Dashboard publicado no [ADR-11](adr-11-dashboard-genie-consumo-executivo.md) precisou de revisão após dado real revelar dois problemas de representação, e o projeto avançou com a extensão já planejada (página de observabilidade técnica, consumindo as tabelas de auditoria criadas no [ADR-15](adr-15-auditoria-automatica-execucoes.md)). Nenhuma das mudanças aqui contradiz o ADR-11 — são decisões novas, tomadas em cima do que já existia.

**Problema 1 — Ranking de Valorização com dado sem valor comunicativo**: usando "o último dia disponível" (como definido no ADR-11), o painel corria o risco real de mostrar as 4 barras zeradas — o comportamento já documentado no ADR-06, onde a API retorna `fechamento_anterior` idêntico ao `preco_atual` perto de certos horários. Um painel de "ranking" com todos os valores em 0% não comunica nada, mesmo sendo dado tecnicamente correto.

**Problema 2 — bug de formatação (percentual duplicado)**: a coluna `retorno_diario_pct` já vem em percentual do banco (ex.: `0,26` = 0,26%). O widget do gráfico, configurado com formatação "Percentage" nativa, multiplicou esse valor por 100 de novo — exibindo `26%` em vez de `0,26%` (e `-43%` em vez de `-0,43%`). Não gerou erro técnico algum; o gráfico renderizou normalmente, só que com um valor 100 vezes maior que o real.

**Problema 3 — causa raiz estática mascarando a correção do ADR-16**: o painel de Reconciliação usava um texto fixo (`CASE WHEN status = 'diverge' THEN 'Defasagem de horário...'`) para resumir a causa raiz, ignorando o conteúdo real da coluna `causa_raiz` na tabela — que já havia sido corrigida para refletir o bug de cache do KNIME (ADR-16) em 4 das 5 datas. Sem correção, o Dashboard voltaria a mostrar a explicação **incorreta**, mesmo depois de já termos corrigido a fonte de verdade.

## Decisão

**Ranking de Valorização passa a mostrar o dia de maior dispersão da janela**, não o mais recente: `ORDER BY dispersao_pct DESC, data_referencia DESC LIMIT 1`. Garante que o painel sempre exiba o dia mais informativo disponível, com barras visíveis — descrição do card ajustada para não presumir "último pregão" como texto fixo.

**Coluna de retorno convertida para fração antes de chegar ao widget** (`retorno_diario_pct / 100 AS retorno_diario_fracao`), eliminando o risco de dupla conversão — a formatação "Percentage" do widget agora opera sobre um valor realmente fracionário, produzindo o percentual correto.

**Cor condicional por categoria, não por regra de valor**: como o tipo de gráfico (barra) não ofereceu formatação condicional direta por valor (diferente da tabela), a solução foi calcular a categoria (`CASE WHEN retorno_diario_pct >= 0 THEN 'Alta' ELSE 'Baixa' END`) na própria consulta SQL e usar essa coluna como campo de cor do gráfico — vermelho para queda, verde para alta.

**Causa raiz do painel de Reconciliação passa a ler a coluna real da tabela**, com uma camada de tradução para o texto resumido (não o texto completo do ADR, que é longo demais para uma tabela) — mas a lógica de tradução (`CASE WHEN causa_raiz LIKE 'CORRECAO%' THEN ... WHEN status = 'diverge' THEN ...`) responde ao **conteúdo real** da coluna, não a uma suposição fixa. Isso significa que a página se autocorrige automaticamente sempre que a tabela de origem for corrigida no futuro, sem exigir edição manual do dashboard.

**Página "Observabilidade Técnica" criada como segunda página do mesmo Dashboard**, não um dashboard separado — mantém contexto (mesmo Global filters, mesma navegação), mas separa claramente o público executivo (página 1, enxuta) do público técnico (página 2, com profundidade de auditoria). Estrutura:
- Histórico de execuções (reaproveitando o dataset já existente do ADR-11)
- Anomalias detectadas, com causa raiz e cores condicionais reais (verde/vermelho/laranja por status de investigação)
- Gaps de execução (com tratamento explícito de "tabela vazia é o resultado correto", evitando parecer painel quebrado)
- Dois gráficos de indicadores de saúde (taxa de sucesso, status de investigação das anomalias), com **descrições sem números fixos no texto** — o valor certo aparece no próprio gráfico, atualizado automaticamente a cada execução, sem risco de a descrição ficar desatualizada (ex.: "100% de sucesso hoje" ficaria errado no dia em que houver falha).

## Alternativas consideradas

- **Manter "último dia disponível" no Ranking, aceitando o risco de barras zeradas**: descartada — o painel existe para comunicar, não para ser tecnicamente correto e visualmente vazio ao mesmo tempo.
- **Corrigir a formatação de percentual direto no widget, sem alterar a query**: não foi encontrada opção de configuração de fórmula/transformação direta no tipo de gráfico usado — resolvido na origem do dado (SQL) em vez de depender de um controle de interface que não existia.
- **Manter a causa raiz resumida como texto fixo, aceitando que ficaria desatualizada**: descartada — repetiria, na camada de apresentação, exatamente o mesmo tipo de erro que o ADR-16 já corrigiu na camada de dado; o valor da correção anterior seria anulado.
- **Dashboard separado para observabilidade técnica, em vez de segunda página do mesmo Dashboard**: descartada — perderia contexto compartilhado (filtros globais, navegação única) sem ganho real, já que o público técnico também se beneficia de conseguir alternar rapidamente para a visão executiva.
- **Escrever descrições dos gráficos de saúde com o número atual (ex.: "77 execuções, 100% de sucesso")**: descartada — ficaria desatualizada na primeira mudança de estado (ex.: primeira falha registrada), exigindo manutenção manual que ninguém lembraria de fazer.

## Consequências

- O Dashboard agora reflete automaticamente qualquer correção futura de causa raiz nas tabelas de reconciliação e auditoria — reduz risco de inconsistência entre "o que a documentação diz" e "o que o dashboard mostra", sem exigir sincronização manual.
- A correção do bug de percentual duplicado (26% → 0,26%) é um lembrete concreto de que gráficos configurados corretamente na aparência ainda podem estar matematicamente errados — validação com dado real conhecido (não só "o gráfico renderizou sem erro") é necessária antes de confiar em qualquer painel.
- A página de Observabilidade Técnica torna visível, para qualquer avaliador técnico, o estado real de saúde do pipeline (77 execuções, 100% sucesso, 16 anomalias com 15 já investigadas, 0 gaps) sem depender de consultar notebooks ou tabelas diretamente.
- Este é o último ADR planejado antes da apresentação da próxima semana — a partir daqui, o foco muda para consolidação de `architecture.md` (incluindo o diagrama visual pendente) e testes finais, sem novas funcionalidades.