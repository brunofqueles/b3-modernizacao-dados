# Entrega do Projeto — Modernização de Dados B3 (KNIME → Databricks)

> Resumo executivo da jornada completa do projeto, do início ao estado final. Para detalhe técnico de qualquer decisão específica, ver `docs/architecture.md` e `docs/adr/`. Para a experiência de quem construiu, ver `docs/licoes-aprendidas.md`.

**Data de entrega:** 04/09/2026
**Status:** MVP concluído, apresentado com sucesso, aprovado para etapa final do processo seletivo. A partir desta data, o projeto entra em manutenção e validação — sem novo desenvolvimento planejado, exceto em caso de falha real.

## O que foi construído

Um pipeline de dados completo simulando a modernização de um sistema legado de mercado de capitais: migração de uma ferramenta visual (KNIME, no lugar de Alteryx, indisponível sem conta corporativa) para uma arquitetura moderna em Databricks — Landing → Bronze → Silver → Gold, com reconciliação automatizada provando que a lógica de negócio foi preservada na migração, não apenas o resultado final.

Além do pipeline em si, o projeto entrega uma camada completa de observabilidade e governança: alertas automáticos (falha, duração, divergência anormal), auditoria que detecta e categoriza anomalias de execução, simulação de custo (FinOps) com descoberta de um gargalo real de escalabilidade, e uma camada de consumo (Dashboard executivo + agente de IA conversacional) testada com dado real antes de qualquer apresentação.

Todo o processo de decisão está documentado em **18 ADRs** (Architecture Decision Records) — cada escolha técnica registrada com contexto, alternativas consideradas e consequências, no momento em que foi tomada, sem retrospecto embelezado.

## Linha do tempo

| Data | Marco |
|---|---|
| 26/08 | Primeiro contato com o projeto: definição de escopo, primeira execução de teste do KNIME |
| 27/08 | Infraestrutura Databricks criada (catalog, schemas, Volume); início real da janela de dados |
| 28/08 | Pipeline completo (Landing → Gold) funcionando; reconciliação Gold vs. KNIME pela primeira vez |
| 31/08 | Orquestração via Job; observabilidade (`pipeline_runs`); segundo dia de reconciliação |
| 01/09 | **Apresentação da etapa inicial — aprovado para a etapa final.** Try/except completo, alertas nativos, alerta de divergência anormal |
| 02/09 | **Bug de cache do KNIME descoberto e corrigido** — 6 dias de dado congelado no sistema legado, nunca detectado até então. Reconciliação genuinamente válida a partir deste dia |
| 03/09 | Diagrama de arquitetura visual; simulação de FinOps (armazenamento + DBU); auditoria automática de anomalias (notebook dedicado, 2 tabelas novas) |
| 04/09 | Correção de consistência (`modo_execucao` em todas as Tasks do Job); notebook de investigação centralizado; categorização de anomalias (Teste vs. Operacional); ranking de valorização corrigido (2 bugs reais de dashboard); **entrega final** |

## Os 3 momentos que mais definem este projeto

**1. A decisão de ingestão independente (ADR-01), provada na prática, não só na teoria.** Desde o primeiro dia, KNIME e Databricks foram desenhados para consumir a mesma fonte de forma independente — nenhum lê o output do outro. Essa decisão parecia só "boa prática" até o dia em que um bug real no KNIME (cache mascarando 6 dias de dado congelado) nunca contaminou o pipeline Databricks, que continuou produzindo dado real o tempo todo. A arquitetura se defendeu sozinha, exatamente como projetada.

**2. O bug de cache do KNIME (ADR-16) — o incidente mais significativo do projeto.** Um comportamento de cache do KNIME (ele preserva o estado de execução junto com o workflow salvo) manteve o índice-proxy congelado por 6 dias seguidos, sem erro técnico visível — cada execução "funcionava" normalmente. Descoberto por comparação ativa de valores entre dias, não por acidente. A decisão de não recalcular ou inventar o dado histórico perdido, e em vez disso corrigir a explicação (causa raiz) mantendo os números reais, é o exemplo mais forte de honestidade técnica do projeto: a janela de reconciliação genuinamente válida como prova de migração é mais curta do que o planejado originalmente, e isso está registrado sem meias-palavras.

**3. A auditoria automática que se provou útil de verdade, não só teórica.** O sistema de detecção de anomalias (execuções próximas, gaps, duração fora do padrão) não ficou só no papel — capturou, sem intervenção manual, tanto o processo de debug do próprio bug do KNIME quanto uma execução real lenta em produção. A separação final entre anomalias de teste (12) e operacionais (5) mostra que a maior parte do "ruído" detectado veio do próprio processo de construção, não de instabilidade real — uma distinção que só fez sentido depois de ter dado real para analisar.

## Estado final

- **Pipeline**: 7 Tasks em cadeia (Landing → Bronze → Silver → Gold → Reconciliação → Alerta de Divergência → Auditoria), agendado dias úteis às 17h15, 100% de sucesso nas execuções registradas
- **KNIME**: 8 execuções reais, processo operacional corrigido (Reset all obrigatório antes de Execute all)
- **Reconciliação**: válida como prova de migração a partir de 02/09 (dias anteriores documentados como afetados pelo bug de cache, não descartados)
- **Observabilidade**: 19 anomalias históricas, todas com causa raiz classificada (12 teste, 5 operacional, 2 sem causa confirmada — nenhuma escondida)
- **Consumo**: Dashboard de 2 páginas (executiva + técnica) e Genie Space, ambos testados com pergunta/uso real
- **Documentação**: 18 ADRs, `architecture.md`, `business-context.md`, `licoes-aprendidas.md`, diagrama visual, simulação de FinOps, documento de escalabilidade

## Como navegar esta documentação

- **Quer entender o "porquê" de negócio?** → `docs/business-context.md`
- **Quer entender a arquitetura técnica completa?** → `docs/architecture.md`
- **Quer entender uma decisão específica, com alternativas consideradas?** → `docs/adr/` (18 arquivos, um por decisão)
- **Quer saber o que foi aprendido no processo, incluindo erros?** → `docs/licoes-aprendidas.md`
- **Quer material de apoio (custo, escalabilidade)?** → `docs/anexos/`
- **Quer rodar o projeto você mesmo?** → seção "Como rodar" no `README.md`

## Nota final

Este projeto foi construído com uma prioridade deliberada: **honestidade técnica acima de aparência de perfeição**. Bugs reais (cache do KNIME, formatação de percentual duplicada, configuração de Task ausente) estão documentados como aconteceram, não escondidos ou reescritos como se nunca tivessem existido. Limitações conhecidas (índice-proxy simplificado, Free Edition, um caso de anomalia sem causa confirmada) permanecem registradas, não silenciadas. Essa é a entrega — não um sistema que finge não ter tido problemas, mas um que mostra como problemas reais foram encontrados, investigados e resolvidos.