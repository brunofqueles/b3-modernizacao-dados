# Contexto de negócio — Modernização de Dados B3

> Este documento explica **por que** este projeto existe: o contexto de mercado que ele simula, a motivação de negócio por trás da modernização, e como as competências demonstradas aqui se conectam a uma necessidade real do setor. Decisões técnicas de arquitetura vivem em `architecture.md`; decisões pontuais em `docs/adr/`.

## 1. O problema de negócio que este projeto simula

Empresas do setor financeiro — B3 incluída — frequentemente operam com pipelines de dados construídos sobre ferramentas visuais de ETL (como Alteryx), somados a procedures de banco de dados acumuladas ao longo de anos. Esse modelo tem um custo real de manutenção:

- **Baixa governança e rastreabilidade**: fluxos construídos visualmente são difíceis de versionar, revisar por pares e auditar da mesma forma que código.
- **Dificuldade de escala**: ferramentas de ETL visual não escalam tão bem quanto processamento distribuído (Spark) para volumes crescentes de dados de mercado.
- **Conhecimento perdido na saída de pessoas**: fluxos visuais sem documentação formal dependem de quem os construiu para serem entendidos e mantidos.

Esse é exatamente o cenário de uma vaga real de modernização de dados na B3: ambiente com forte uso de Alteryx, migração para Databricks já em andamento (~1,5 ano), com a necessidade de evoluir a arquitetura, migrar pipelines críticos (incluindo procedures) e desenvolver novos índices — usando IA como alavanca de produtividade no processo.

Este projeto simula esse cenário de ponta a ponta, com escopo reduzido para caber num portfólio: um pipeline "legado" (KNIME, no lugar de Alteryx) sendo migrado para uma arquitetura moderna em Databricks, com prova de que a lógica de negócio foi preservada na migração (reconciliação Gold vs. KNIME).

## 2. Por que o índice-proxy, e o que ele representa

O projeto calcula um **índice-proxy** e três indicadores complementares, todos derivados dos mesmos 4 papéis (PETR4, VALE3, MGLU3, ITUB4), gerados de forma automatizada na camada Gold. O índice-proxy em si não é a metodologia oficial de nenhum índice real da B3 (como o Ibovespa, que usa critérios de liquidez, free float e peso de capitalização) — é uma simplificação deliberada, documentada com essa ressalva desde o início ([ADR-01](adr/adr-01-ingestao-independente-landing.md), `architecture.md`).

**Os 4 indicadores calculados, e como interpretar cada um:**

- **Retorno diário por ticker**: `(preço atual - fechamento anterior) / fechamento anterior`, expresso em percentual. Um valor de `0,65%`, por exemplo, significa que o papel valorizou 0,65% no dia.
- **Índice-proxy**: a média simples dos 4 retornos diários. Representa, de forma simplificada, "como esses 4 papéis se comportaram, em média, no dia" — não uma medida de mercado amplo.
- **Ranking de valorização**: ordena os 4 papéis do maior para o menor retorno do dia, respondendo diretamente "quais ações mais se valorizaram?".
- **Dispersão do dia**: o desvio padrão dos 4 retornos diários. Um valor baixo indica que os papéis se moveram de forma parecida entre si naquele dia; um valor alto indica que alguns subiram e outros caíram, ou que a intensidade do movimento variou bastante entre eles.

**Indicador pendente, documentado deliberadamente:** variação acumulada do índice entre dias consecutivos (ex.: "como o índice evoluiu de 27/08 para 28/08"). Ainda não implementado porque depende de pelo menos 2 dias de dados válidos — assim que a janela de reconciliação (27/08, 28/08, 31/08) avançar, esse é o próximo indicador natural a ser adicionado. Ver [ADR-05](adr/adr-05-indicadores-gold-automatizada.md).

A escolha por proxy simplificado (em vez de tentar replicar a metodologia oficial de um índice real) foi consciente: dentro do prazo do projeto, não havia uma fonte gratuita confirmada para a carteira teórica oficial de um índice real, e apresentar um proxy como se fosse metodologia oficial seria uma imprecisão grave numa conversa técnica sobre mercado de capitais.

## 3. Por que os 4 tickers escolhidos

PETR4, VALE3, MGLU3 e ITUB4 foram escolhidos por serem confirmados como acessíveis sem token no sandbox gratuito da API `brapi.dev` — critério prático de viabilidade dentro do prazo do projeto, não uma seleção por relevância setorial específica. Representam, ainda assim, uma amostra de setores diferentes (petróleo, mineração, varejo, bancário), o que dá alguma diversidade ao proxy mesmo sendo pequeno.

## 4. O que este projeto demonstra, além do pipeline em si

Este projeto foi construído deliberadamente para evidenciar, além do resultado técnico funcional, um conjunto de práticas de engenharia que fazem diferença numa avaliação real:

- **Databricks e PySpark como ferramenta central de transformação de dados** — não apenas SQL, mas o uso do ecossistema Databricks (Unity Catalog, Volumes, Delta Lake, Workflows) de ponta a ponta, incluindo os limites reais da Free Edition e como o desenho foi adaptado a eles.
- **Arquitetura em camadas (Medallion — Bronze/Silver/Gold)**, com separação clara de responsabilidade entre ingestão, limpeza/qualidade e regra de negócio — não um pipeline monolítico.
- **Decisões de arquitetura registradas e justificadas** (ADRs), incluindo alternativas descartadas e por quê — não apenas o resultado final, mas o raciocínio por trás dele, replicável em entrevista.
- **Documentação organizada e evolutiva**: README, arquitetura e contexto de negócio como documentos vivos, atualizados ao longo da construção do projeto (não escritos de uma vez no fim), com histórico real de commits mostrando a evolução.
- **Prova de migração real, não só de output**: a decisão de KNIME e Databricks consumirem a fonte de forma independente ([ADR-01](adr/adr-01-ingestao-independente-landing.md)) existe especificamente para que a reconciliação final prove que a *lógica de negócio* foi migrada — não apenas que um sistema copia a saída do outro.
- **Honestidade técnica sobre limitações**: o índice como proxy simplificado, os limites da Free Edition, o que ficou como evolução futura (ex.: Databricks Asset Bundles) — tudo documentado explicitamente, em vez de omitido ou apresentado como mais robusto do que é.

## 5. Conexão direta com os requisitos da vaga simulada

| Requisito da vaga | Como este projeto demonstra |
|---|---|
| Databricks e/ou Spark | Pipeline Bronze/Silver/Gold construído em PySpark/SQL sobre Databricks Free Edition |
| SQL | Transformações e cálculo do índice-proxy na camada Gold |
| Engenharia de dados / ETL | Pipeline completo de ingestão, limpeza, transformação e agregação |
| Ambiente cloud (Azure/Databricks) | Databricks Free Edition, Unity Catalog, Volumes |
| Experiência com Alteryx (migração) | KNIME como simulação do padrão de migração ferramenta-visual → código |
| Lakehouse/Medallion | Arquitetura Bronze/Silver/Gold explícita |
| Dados financeiros/mercado de capitais | Cotações reais da B3 via `brapi.dev`, cálculo de retorno diário e índice-proxy |
| IA aplicada a desenvolvimento | Uso de IA como copiloto ao longo de toda a construção do projeto (este próprio processo de desenvolvimento assistido) |
| Data Governance e Data Quality | Unity Catalog, regra de qualidade com quarentena na Silver, ADRs documentando decisões de governança de dados |