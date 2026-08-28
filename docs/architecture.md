# Arquitetura — Modernização de Dados B3 (KNIME → Databricks)

> Este documento descreve as decisões técnicas de arquitetura do projeto. Contexto de negócio (o que é o índice-proxy, por que esses tickers, indicadores) vive em `docs/business-context.md`. Decisões pontuais e seus trade-offs detalhados vivem em `docs/adr/`.

## 1. Entendimento do problema

Construir um projeto de portfólio que simule o escopo de uma vaga real de modernização de dados da B3: migração de pipelines legados (ferramenta visual + procedures, aqui simulada por KNIME no lugar de Alteryx) para uma arquitetura moderna em Databricks, com foco em governança, escalabilidade e cálculo de um índice financeiro — 100% dentro do Databricks Free Edition, dentro de um prazo de execução curto.

## 2. Premissas adotadas

- Plataforma alvo: **Databricks Free Edition** — serverless-only, sem private networking, outbound restrito a domínios confiáveis (testado e confirmado compatível com `brapi.dev` e GitHub — ver `databricks/tests/`), sem SLA/suporte.
- **KNIME Analytics Platform** no lugar de Alteryx, por ser gratuito e open source sem exigência de conta corporativa — não é um clone funcional do Alteryx, simula o *padrão* de migração ferramenta-visual → código.
- Fonte única de dados: [brapi.dev](https://brapi.dev), API REST gratuita, sandbox sem token, para 4 tickers (PETR4, VALE3, MGLU3, ITUB4). A API retorna a cotação **atual** no momento da chamada — não há endpoint de cotação histórica no plano gratuito.
- "Índice" calculado é um **proxy simplificado** (média simples do retorno diário dos 4 papéis), não a metodologia oficial de nenhum índice real da B3 — ver `docs/business-context.md`.
- Governança e time: projeto solo, portfólio público — sem requisitos de compliance regulatório real, sem SLA contratual.

## 3. Arquitetura proposta

```
[Fonte única]              [Legado simulado]          [Novo — Databricks]
brapi.dev (API)     ──┬──> KNIME (transforma,
  4 tickers            │     calcula índice-proxy,
                       │     grava CSV versionado por data)
                       │
                       └──> Volume UC (raw, landing) ──> Bronze ──> Silver ──> Gold
                                                                                  │
                                                          [Reconciliação: Gold vs. saída do KNIME]
                                                          (contagem de linhas, valor do índice)
```

KNIME e Databricks consomem a **mesma fonte de forma independente** (ver [ADR-01](adr/adr-01-ingestao-independente-landing.md)) — o Databricks não lê o CSV de saída do KNIME em nenhum ponto do fluxo Bronze/Silver/Gold.

> **Correção de escopo (27/08):** a comparação histórica entre KNIME e Databricks cobre **27/08, 28/08 e 31/08** — não 26/08. A infraestrutura Databricks (catalog, schemas, Volume) só foi criada em 27/08, e a API `brapi.dev` não permite obter retroativamente o preço de um dia anterior à criação. O CSV do KNIME de 26/08 permanece no repositório como primeiro registro histórico do sistema legado, mas fica fora da janela de reconciliação de 3 dias.

### Unity Catalog — convenção de nomes

Catalog único `poc_b3_modernizacao`, schema por camada, criados em 27/08/2026:

| Schema | Conteúdo | Tags aplicadas |
|---|---|---|
| `poc_b3_modernizacao.landing` | Volume `raw` — arquivos JSON brutos da API, organizados por `data=AAAA-MM-DD/` | `ambiente=poc`, `dominio=mercado-financeiro`, `projeto=b3-modernizacao-dados`, `camada=landing` |
| `poc_b3_modernizacao.bronze` | Tabelas Delta, cópia fiel por sistema, sem tratamento de tipo ou conteúdo | `camada=bronze` (+ tags comuns acima) |
| `poc_b3_modernizacao.silver` | Tabelas Delta tipadas, deduplicadas, com regra de qualidade e quarentena | `camada=silver` (+ tags comuns acima) |
| `poc_b3_modernizacao.gold` | Tabelas Delta com o índice-proxy, gerada de forma automatizada via Job | `camada=gold` (+ tags comuns acima) |
| `poc_b3_modernizacao.reconciliation` | Resultado da comparação entre Gold e KNIME, por data — nível agregado e detalhado | `camada=reconciliation` (+ tags comuns acima) |
| `poc_b3_modernizacao.observability` | Log de execução dos notebooks do pipeline — responde se quebrou, quando e por quê | `camada=observability` (+ tags comuns acima) |

Todos os catalog/schemas têm descrição registrada no momento da criação. A partir do schema `reconciliation`, a criação passou a ser feita via código (`databricks/setup/00_setup_catalog`), não mais manualmente pela interface — ver [ADR-07](adr/adr-07-reconciliacao-formato-tolerancia-manual.md).

### Camadas

| Camada | Conteúdo | Status |
|---|---|---|
| **Landing (Volume UC)** | Cópia bruta e imutável da resposta da API `brapi.dev`, com metadados de ingestão. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md) e [ADR-02](adr/adr-02-widgets-idempotencia-landing.md). | ✅ Implementado (`databricks/bronze/01_ingestao_landing`) |
| **Bronze** | Tabelas Delta, cópia fiel da fonte, imutável. | 🔧 Próximo passo |
| **Silver** | Tipagem, deduplicação, pelo menos 1 regra de qualidade com tabela de quarentena (ex.: preço nulo/negativo desviado, sem quebrar o pipeline). | ✅ Implementado (`databricks/silver/03_silver`) — quarentena unificada, ver [ADR-04](adr/adr-04-quarentena-dedup-schema-silver.md) |
| **Gold** | Cálculo do índice-proxy em PySpark/SQL, orquestrado como Job no Databricks Workflows. | ✅ Cálculo implementado (`databricks/gold/04_gold`) — 4 indicadores, ver [ADR-05](adr/adr-05-indicadores-gold-automatizada.md). Orquestração via Job ainda pendente. |
| **Reconciliação** | Notebook comparando Gold (Databricks) com a saída do KNIME — contagem de linhas e valor do índice, por data (27/08, 28/08, 31/08). | ✅ Implementado, execução manual (`databricks/reconciliation/05_reconciliacao`) — ver [ADR-07](adr/adr-07-reconciliacao-formato-tolerancia-manual.md) |

## 4. Componentes e justificativa

| Componente | Por que esse e não outro |
|---|---|
| **brapi.dev** | API REST gratuita, sandbox funciona sem token para os tickers escolhidos — preferida ao `.zip` oficial da B3 (COTAHIST), incompatível com o prazo do projeto. |
| **Volume UC como landing, antes da Bronze** | Desacopla "buscar o dado" de "estruturar o dado em tabela", preserva a resposta bruta como evidência auditável. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md). |
| **KNIME e Databricks consumindo a fonte de forma independente** | Sustenta a reconciliação Gold vs. KNIME como prova de migração de lógica de negócio, não apenas de output. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md). |
| **CSVs do KNIME e JSONs do Volume versionados por data** | Permite comparação histórica real entre execuções de dias diferentes, em vez de um único snapshot sobrescrito. |
| **Widgets (`dbutils.widgets`) no notebook de ingestão** | Parametrização de ticker, data de referência e modo de execução — permite reprocessamento pontual e isolamento de um único papel sem editar código. Ver [ADR-02](adr/adr-02-widgets-idempotencia-landing.md). |
| **Databricks Workflows para orquestração** | Nativo, sem custo extra, disponível na Free Edition — orquestra Bronze → Silver → Gold como Job. |
| **Databricks Git folders** | Conecta o workspace diretamente ao repositório GitHub, permitindo commits via branch + PR direto da interface do Databricks. |
| **Geração da Gold exclusivamente via código** | Nenhum valor é editado manualmente em tabela Gold — implementa a primeira metade do modelo de governança pretendido (a segunda metade, RBAC por papel, permanece pendente). Ver [ADR-05](adr/adr-05-indicadores-gold-automatizada.md). |
| **Databricks Workflows (Job `pipeline_diario_b3`)** | Orquestra as 4 Tasks em cadeia (Landing → Bronze → Silver → Gold), agendado para dias úteis às 17h15, com notificação de falha por e-mail. Completa a automação real do pipeline, sem intervenção manual. Ver [ADR-06](adr/adr-06-orquestracao-workflows-yaml.md). |
| **YAML do Job exportado como documentação** | Aproxima o projeto de IaC sem o custo completo de Databricks Asset Bundles — cópia de referência versionada, não fonte de verdade deployada. Ver [ADR-06](adr/adr-06-orquestracao-workflows-yaml.md). |
| **Notebook utilitário compartilhado (`%run`)** | Centraliza `merge_ou_cria` e `registrar_execucao`, eliminando duplicação de código entre 4 notebooks e viabilizando a observabilidade de forma consistente. Ver [ADR-08](adr/adr-08-observabilidade-modulo-compartilhado.md). |

## 5. Limitações conhecidas da Free Edition e como o desenho já as absorve

| Limite | Como o desenho contorna |
|---|---|
| Outbound restrito a domínios confiáveis | Testado e confirmado no Dia 1 (`databricks/tests/teste_b3.ipynb`): `brapi.dev` e `api.github.com`/`github.com` acessíveis sem bloqueio. |
| Sem SLA/suporte, uso não comercial | Aceitável — natureza de portfólio, não produto em produção. |
| 1 workspace / 1 metastore por conta | Sem impacto — projeto não requer múltiplos ambientes (dev/staging/prod) nesta fase. |

## 6. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Divergência de preço entre a chamada do KNIME e a do Databricks no mesmo dia (dado em tempo real, muda durante o pregão) | Execução padronizada de ambos após o fechamento do pregão (**17h15**), eliminando a janela de variação intraday. |
| "Índice" proxy confundido com metodologia oficial de um índice real (ex.: Ibovespa) numa entrevista | Ressalva explícita no README e em `docs/business-context.md`. |
| Prazo curto competindo com escopo de automação (agente de commit) | Agente tratado como extra condicional, só entra se o cronograma técnico central fechar no prazo — não faz parte do caminho crítico. |
| Execução de teste sobrescrevendo silenciosamente a execução oficial do dia, sem registro | Reconhecido como lacuna de observabilidade — ver [ADR-02](adr/adr-02-widgets-idempotencia-landing.md) e item "Observabilidade" na seção 7. |
| API `brapi.dev` retornando `regularMarketPreviousClose` idêntico a `regularMarketPrice` quando consultada muito próxima ao fechamento, zerando o índice-proxy do dia | Observado na execução real de 27/08 (17h15) — comportamento da fonte externa, não falha do pipeline. Relevante para a interpretação da reconciliação: um índice zerado num dos 3 dias não deve ser lido como "sem movimento no mercado", mas como possível efeito do horário de captura. Ver [ADR-06](adr/adr-06-orquestracao-workflows-yaml.md). |

## 7. Evolução — possíveis próximos passos

Itens abaixo são **roadmap reconhecido**, não decisões já tomadas — não implementados nesta versão do projeto por restrição de prazo, mas documentados deliberadamente para demonstrar maturidade de arquitetura mesmo sem tempo de execução.

- **Databricks Asset Bundles (IaC)**: padrão mais robusto para deploy/CI-CD entre ambientes.
- **Agente de automação de commit/PR** via API REST do GitHub — mecânica já validada em projeto anterior (branch → commit → PR automáticos, merge sempre manual). Candidato a extra, condicionado a sobra de tempo.
- **Observabilidade — refinamentos além do já implementado**: a tabela de log de execução (`observability.pipeline_runs`) já está implementada e validada (ver [ADR-08](adr/adr-08-observabilidade-modulo-compartilhado.md)), respondendo "quebrou? quando?" de forma automatizada. Evoluções ainda pendentes: (1) registro explícito de falha com `try/except` em cada notebook — hoje a falha é detectada apenas pela ausência de um registro de sucesso, não por um evento de falha com `mensagem_erro` detalhado; (2) propagação consistente de `modo_execucao` via Widget em todos os notebooks (hoje só `01` e `02` têm esse Widget); (3) AI/BI Dashboard visualizando os dados, hoje consultados apenas via `display()` simples.
- **Reconciliação automatizada com defasagem D-1**: a reconciliação hoje é executada manualmente (ver [ADR-07](adr/adr-07-reconciliacao-formato-tolerancia-manual.md)), pois depende do CSV do KNIME já estar commitado e sincronizado — dependência externa ao Job agendado. Evolução: incorporar como Task automática comparando sempre o dia anterior (D-1), quando o dado já está garantidamente disponível, incluindo alerta automático de divergência (não apenas registro passivo do resultado).
- **Governança de acesso (RBAC)**: modelo pretendido — arquitetos com acesso completo a todas as camadas; engenharia limitada a Bronze e Silver; negócio limitado a Silver. A camada Gold seria gerada exclusivamente de forma automatizada via Job (nunca editada manualmente), especificamente para eliminar divergência de números entre diferentes consumidores do dado. Documentado aqui como modelo pretendido — não implementado com RBAC real no Unity Catalog dentro do prazo do projeto.
- **Escalabilidade**: caminho de 4 tickers para uma lista maior (ex.: 500 papéis, ou o pregão completo). Pontos que precisariam de revisão: paginação/rate limit da API `brapi.dev` em volume maior; o dropdown fixo de tickers no Widget (ver [ADR-02](adr/adr-02-widgets-idempotencia-landing.md)) precisaria virar uma fonte dinâmica (tabela de referência) em vez de lista codificada; volume de dados por camada e se a arquitetura atual aguenta o salto sem redesenho de particionamento. Nesse cenário de múltiplas fontes/sistemas, também passaria a fazer sentido avaliar **Programação Orientada a Objetos** (classe base de ingestão com subclasses por fonte) — decisão deliberadamente não adotada nesta fase, por não haver ainda repetição de estrutura real que justifique a abstração (projeto tem uma única fonte, uma única forma de ingestão).
- Ampliação do índice-proxy para mais tickers ou ponderação por `marketCap`, caso o projeto seja retomado após a entrega.

## 8. Perguntas de negócio e operacionais — o que o projeto já responde

Registradas aqui deliberadamente, mesmo as que ainda não têm resposta plena pelo pipeline — mostrar honestamente o que está e o que não está coberto é parte da maturidade que o projeto pretende demonstrar.

**Quais ações mais se valorizaram?**
✅ Respondível pelo pipeline desde a conclusão da Gold: `gold.indicadores_diarios`, coluna `ranking_valorizacao`, ordenada por `retorno_diario_pct`. Ver [ADR-05](adr/adr-05-indicadores-gold-automatizada.md).

**Quando podemos encerrar o fluxo do KNIME?**
Já respondível conceitualmente: o KNIME existe exclusivamente para sustentar a reconciliação (prova de que a lógica de negócio foi preservada na migração). Pode ser formalmente encerrado após a conclusão e validação da janela de reconciliação de 3 dias (27/08, 28/08, 31/08) — a partir daí, não tem mais função no projeto.

**O que usamos para o fluxo de CI/CD?**
Hoje, nada equivalente a CI/CD real — o que existe é controle de versão (Git: branch → commit → PR → merge), que é disciplina de versionamento, não integração/entrega contínua. Um CI/CD real (testes automatizados por PR, deploy automatizado de notebooks/Jobs, validação de schema antes de promover) depende de **Databricks Asset Bundles**, já registrado como evolução futura (seção 7) e não implementado por restrição de prazo — decisão consciente, não lacuna não percebida.

## 9. Status atual e próximos passos

**ADRs escritos:**
- [ADR-01 — Ingestão independente via Volume UC (Landing)](adr/adr-01-ingestao-independente-landing.md)
- [ADR-02 — Parametrização via Widgets e escrita idempotente na Landing Zone](adr/adr-02-widgets-idempotencia-landing.md)
- [ADR-03 — Escrita idempotente na Bronze via MERGE e coluna de validação de integridade](adr/adr-03-merge-bronze-validacao-integridade.md)
- [ADR-04 — Quarentena unificada, deduplicação e schema explícito na Silver](adr/adr-04-quarentena-dedup-schema-silver.md)
- [ADR-05 — Indicadores da camada Gold e geração exclusivamente automatizada](adr/adr-05-indicadores-gold-automatizada.md)
- [ADR-06 — Orquestração via Databricks Workflows e YAML como documentação leve de IaC](adr/adr-06-orquestracao-workflows-yaml.md)
- [ADR-07 — Reconciliação Gold vs. KNIME: formato, tolerância e execução manual](adr/adr-07-reconciliacao-formato-tolerancia-manual.md)
- [ADR-08 — Observabilidade via log de execução e módulo utilitário compartilhado](adr/adr-08-observabilidade-modulo-compartilhado.md)

**Infraestrutura já criada:**
- Repositório GitHub estruturado (`docs/adr`, `knime`, `databricks/{setup,landing,bronze,silver,gold,jobs,reconciliation,tests}`)
- Git folder conectando o Databricks ao repositório, fluxo branch → commit → PR → merge validado (via Databricks e via terminal local)
- Workflow KNIME completo e funcional, 3 execuções reais (26/08, 27/08, 28/08)
- Catalog `poc_b3_modernizacao` e os 6 schemas (`landing`, `bronze`, `silver`, `gold`, `reconciliation`, `observability`), com tags e descrição — criados via código a partir de `reconciliation`
- Volume `poc_b3_modernizacao.landing.raw`
- Tabelas: `bronze.cotacoes`, `silver.cotacoes`, `silver.cotacoes_quarentena`, `gold.indicadores_diarios`, `gold.indice_proxy`, `reconciliation.resultado_indice`, `reconciliation.resultado_detalhado`, `observability.pipeline_runs`
- Job `pipeline_diario_b3`: 4 Tasks em cadeia, agendado dias úteis às 17h15, notificação de falha por e-mail. Validado em execução agendada real (27/08), sem intervenção manual.

**Testes realizados:**
- Conectividade do Databricks Free Edition com `brapi.dev` e GitHub — confirmados acessíveis.
- Execução real do Job agendado (27/08, 17h15): 4 Tasks concluídas com sucesso, ~2 minutos, sem intervenção manual.
- Execução completa instrumentada (28/08): 5 notebooks, 5 registros de observabilidade, todos `status = sucesso`.

**Padrão de desenvolvimento adotado:** PySpark como linguagem primária — SQL evitado, usado apenas quando genuinamente inevitável. Funções compartilhadas via `%run` a partir de `databricks/setup/01_utilitarios_pipeline`.

**Dívida técnica:** ~~função `merge_ou_cria` duplicada em 4 notebooks~~ — **resolvida** (ver [ADR-08](adr/adr-08-observabilidade-modulo-compartilhado.md)). Pendente: `modo_execucao` inconsistente entre notebooks (só `01` e `02` têm Widget); detecção de falha apenas por ausência de registro, não por evento explícito.

**Código já implementado:**
- `databricks/setup/00_setup_catalog`: cria catalog e os 6 schemas de forma idempotente e via código.
- `databricks/setup/01_utilitarios_pipeline`: funções compartilhadas `merge_ou_cria` e `registrar_execucao`. Ver [ADR-08](adr/adr-08-observabilidade-modulo-compartilhado.md).
- `databricks/landing/01_ingestao_landing`: busca os 4 tickers na API, grava JSON bruto no Volume, parametrizado via Widgets, instrumentado com observabilidade.
- `databricks/bronze/02_bronze`: lê o Volume, valida integridade, grava via MERGE (agora via função compartilhada), instrumentado.
- `databricks/silver/03_silver`: tipa, deduplica, quarentena unificada, instrumentado.
- `databricks/gold/04_gold`: calcula 4 indicadores, instrumentado.
- `databricks/jobs/pipeline_diario_b3.yml`: exportação de referência do Job.
- `databricks/reconciliation/05_reconciliacao`: compara Gold vs. KNIME, instrumentado.

**Estado dos dados:**
- CSVs do KNIME: 26/08 (fora da janela), 27/08 e 28/08 (válidos).
- Pipeline Databricks: dados de 26/08, 27/08 e 28/08 em todas as camadas.
- Observabilidade: 5 registros de execução (28/08), todos `status = sucesso`.

**Próximo passo real:** 5º indicador (variação acumulada entre dias — já viável com 28/08 disponível); execução de 31/08; lições aprendidas.