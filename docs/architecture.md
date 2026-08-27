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

Todos os catalog/schemas têm descrição registrada no momento da criação (não adicionada retroativamente).

### Camadas

| Camada | Conteúdo | Status |
|---|---|---|
| **Landing (Volume UC)** | Cópia bruta e imutável da resposta da API `brapi.dev`, com metadados de ingestão. Ver [ADR-01](adr/adr-01-ingestao-independente-landing.md) e [ADR-02](adr/adr-02-widgets-idempotencia-landing.md). | ✅ Implementado (`databricks/bronze/01_ingestao_landing`) |
| **Bronze** | Tabelas Delta, cópia fiel da fonte, imutável. | 🔧 Próximo passo |
| **Silver** | Tipagem, deduplicação, pelo menos 1 regra de qualidade com tabela de quarentena (ex.: preço nulo/negativo desviado, sem quebrar o pipeline). | ✅ Implementado (`databricks/silver/03_silver`) — quarentena unificada, ver [ADR-04](adr/adr-04-quarentena-dedup-schema-silver.md) |
| **Gold** | Cálculo do índice-proxy em PySpark/SQL, orquestrado como Job no Databricks Workflows. | ✅ Cálculo implementado (`databricks/gold/04_gold`) — 4 indicadores, ver [ADR-05](adr/adr-05-indicadores-gold-automatizada.md). Orquestração via Job ainda pendente. |
| **Reconciliação** | Notebook comparando Gold (Databricks) com a saída do KNIME — contagem de linhas e valor do índice, por data (27/08, 28/08, 31/08). | Pendente |

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

## 5. Limitações conhecidas da Free Edition e como o desenho já as absorve

| Limite | Como o desenho contorna |
|---|---|
| Outbound restrito a domínios confiáveis | Testado e confirmado no Dia 1 (`databricks/tests/teste_b3.ipynb`): `brapi.dev` e `api.github.com`/`github.com` acessíveis sem bloqueio. |
| Sem SLA/suporte, uso não comercial | Aceitável — natureza de portfólio, não produto em produção. |
| 1 workspace / 1 metastore por conta | Sem impacto — projeto não requer múltiplos ambientes (dev/staging/prod) nesta fase. |

## 6. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Divergência de preço entre a chamada do KNIME e a do Databricks no mesmo dia (dado em tempo real, muda durante o pregão) | Execução padronizada de ambos após o fechamento do pregão (~17h), eliminando a janela de variação intraday. |
| "Índice" proxy confundido com metodologia oficial de um índice real (ex.: Ibovespa) numa entrevista | Ressalva explícita no README e em `docs/business-context.md`. |
| Prazo curto competindo com escopo de automação (agente de commit) | Agente tratado como extra condicional, só entra se o cronograma técnico central fechar no prazo — não faz parte do caminho crítico. |
| Execução de teste sobrescrevendo silenciosamente a execução oficial do dia, sem registro | Reconhecido como lacuna de observabilidade — ver [ADR-02](adr/adr-02-widgets-idempotencia-landing.md) e item "Observabilidade" na seção 7. |

## 7. Evolução — possíveis próximos passos

Itens abaixo são **roadmap reconhecido**, não decisões já tomadas — não implementados nesta versão do projeto por restrição de prazo, mas documentados deliberadamente para demonstrar maturidade de arquitetura mesmo sem tempo de execução.

- **Databricks Asset Bundles (IaC)**: padrão mais robusto para deploy/CI-CD entre ambientes.
- **Agente de automação de commit/PR** via API REST do GitHub — mecânica já validada em projeto anterior (branch → commit → PR automáticos, merge sempre manual). Candidato a extra, condicionado a sobra de tempo.
- **Observabilidade do pipeline**: hoje o pipeline não registra suas próprias execuções — uma falha ou uma sobrescrita indevida (ver riscos, seção 6) não deixa rastro. Evolução: tabela de log de execução (`pipeline_runs` ou similar) respondendo a três perguntas — o pipeline quebrou? quando? por quê? — usando o campo `modo_execucao` já presente no notebook de ingestão como um dos insumos.
- **Reconciliação automatizada com alerta de divergência**: a reconciliação básica (Gold vs. KNIME, contagem de linhas e valor do índice) é escopo core do projeto, mas hoje seria uma comparação manual em notebook. Evolução: alertar automaticamente *quando* e *quais* valores divergem entre as duas implementações, não apenas apresentar o número final para leitura manual.
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

**Infraestrutura já criada:**
- Repositório GitHub estruturado (`docs/adr`, `knime`, `databricks/{landing,bronze,silver,gold,jobs,reconciliation,tests}`)
- Git folder conectando o Databricks ao repositório, fluxo branch → commit → PR → merge validado
- Workflow KNIME completo e funcional (`knime/b3_pipeline_legado.knwf`)
- Catalog `poc_b3_modernizacao` e os 4 schemas (`landing`, `bronze`, `silver`, `gold`), com tags e descrição
- Volume `poc_b3_modernizacao.landing.raw`
- Tabelas: `bronze.cotacoes`, `silver.cotacoes`, `silver.cotacoes_quarentena`, `gold.indicadores_diarios`, `gold.indice_proxy`

**Testes realizados:**
- Conectividade do Databricks Free Edition com `brapi.dev` e GitHub (`databricks/tests/teste_b3.ipynb`) — ambos confirmados acessíveis.

**Padrão de desenvolvimento adotado:** PySpark como linguagem primária em todos os notebooks — SQL evitado, usado apenas quando genuinamente inevitável (nenhum caso até o momento).

**Dívida técnica reconhecida:** função `merge_ou_cria` (lógica de MERGE idempotente) duplicada manualmente em 3 notebooks (`03_silver`, `04_gold`, e implicitamente repetida em conceito na Bronze) — Databricks não compartilha funções entre notebooks sem um passo explícito de importação (`%run` ou módulo Python). Candidata a extração futura para um utilitário compartilhado, quando o número de notebooks reaproveitando a mesma lógica justificar o investimento.

**Código já implementado:**
- `databricks/landing/01_ingestao_landing`: busca os 4 tickers na API, grava JSON bruto no Volume `landing.raw`, parametrizado via Widgets (ticker, data de referência, modo de execução), com célula de validação pós-gravação.
- `databricks/bronze/02_bronze`: lê todos os arquivos do Volume `landing.raw`, valida integridade (data da partição vs. data real do conteúdo), grava via MERGE idempotente na tabela `bronze.cotacoes`, com colunas de auditoria (`data_carga`, `arquivo_origem`, `data_valida`). Ver [ADR-03](adr/adr-03-merge-bronze-validacao-integridade.md).
- `databricks/silver/03_silver`: tipa (`date`, `timestamp`, `double`), deduplica (mantém registro mais recente por `data_carga`), separa registros problemáticos em quarentena unificada, com schema final explícito. Ver [ADR-04](adr/adr-04-quarentena-dedup-schema-silver.md).
- `databricks/gold/04_gold`: calcula 4 indicadores (retorno diário, índice-proxy, ranking de valorização, dispersão), gerados exclusivamente via código, sem edição manual. Ver [ADR-05](adr/adr-05-indicadores-gold-automatizada.md).

**Estado dos dados:**
- CSVs do KNIME gerados e versionados por data (`knime/cotacoes_detalhado_AAAA-MM-DD.csv`, `knime/indice_proxy_AAAA-MM-DD.csv`), com histórico iniciado em 26/08/2026.
- Pipeline Databricks completo para 27/08/2026: 4 registros válidos em cada camada (Bronze → Silver → Gold), 4 registros em quarentena (rastreáveis, não descartados).
- Índice-proxy de 27/08: `0.01%` (dispersão: `0.02%`). Ranking do dia: VALE3 (1º, +0,04%), demais papéis sem variação registrada no momento da execução.

**Próximo passo real:** orquestração via Databricks Workflows (Job agendado, encadeando Landing → Bronze → Silver → Gold sem intervenção manual) e, em seguida, o notebook de reconciliação (Gold vs. KNIME).