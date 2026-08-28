# ADR-07 — Reconciliação Gold vs. KNIME: formato, tolerância e execução manual

**Status:** Aceito
**Data:** 2026-08-28

## Contexto

O notebook `05_reconciliacao` compara os resultados do KNIME (sistema legado simulado) com a Gold do Databricks, por data — o critério central de "migração validada" definido desde o início do projeto (ver [ADR-01](adr-01-ingestao-independente-landing.md)). Durante a construção, dois problemas reais precisaram de decisão: um formato de dado inconsistente entre os dois sistemas, e como interpretar diferenças numéricas encontradas na primeira execução real.

**Formato inconsistente descoberto**: o KNIME calcula o retorno diário e o índice-proxy como **fração decimal** (ex.: `0.00227`), enquanto a Gold do Databricks já armazena esses valores em **percentual** (ex.: `0.23`), decisão tomada em [ADR-05](adr-05-indicadores-gold-automatizada.md). Sem conversão, a reconciliação compararia números em escalas diferentes do mesmo conceito, invalidando qualquer comparação.

**Primeira divergência real encontrada**: na reconciliação de 27/08, o índice-proxy do KNIME (`0.227%`, capturado às ~17:22-17:24 horário local) divergiu do Databricks (`0%`, capturado às 17:15 via Job agendado) em todos os 4 tickers, na mesma direção. Investigação apontou causa única: a defasagem de poucos minutos entre as duas execuções coincidiu com movimento real de mercado — não indica falha na lógica de negócio migrada.

## Decisão

**Conversão de fração para percentual no lado do KNIME**, antes de qualquer comparação, alinhando ambos os sistemas à mesma unidade (percentual, 3 casas decimais).

**Comparação com tolerância de `0,01%`** (não igualdade exata), classificando cada linha como `bate` ou `diverge`. Tolerância mantida deliberadamente apertada — mesmo sabendo, pela primeira execução real, que diferenças de horário entre as duas capturas frequentemente produzem divergência acima desse limiar.

**Toda divergência recebe uma `causa_raiz` textual explícita**, gravada junto com o resultado — em vez de afrouxar a tolerância até os números "baterem artificialmente", o projeto opta por manter o critério rigoroso e **explicar a causa** quando ela existe e é conhecida (defasagem de horário de captura). Essa é uma escolha deliberada de honestidade analítica: mascarar a divergência ajustando o critério esconderia informação real sobre o comportamento do sistema; documentar a causa raiz preserva a informação e ainda demonstra capacidade de diagnóstico.

**Execução mantida manual**, não incorporada como Task automática do Job `pipeline_diario_b3` (ver [ADR-06](adr-06-orquestracao-workflows-yaml.md)). Motivo: a reconciliação depende do CSV do KNIME já estar commitado no GitHub e sincronizado no Git folder do Databricks (via Pull) — dependência externa ao próprio Job, que roda de forma manual e no ritmo do autor, não sincronizada ao agendamento das 17h15. Automatizar a reconciliação como Task do mesmo dia arriscaria tentar ler um arquivo que ainda não existe.

**Duas tabelas gravadas**: `reconciliation.resultado_indice` (nível agregado, por data) e `reconciliation.resultado_detalhado` (nível ticker, por data), ambas via MERGE idempotente, seguindo o mesmo padrão das demais camadas.

**Schema `reconciliation` criado via código** (não pela interface), no notebook `databricks/setup/00_setup_catalog` — junto com a reafirmação idempotente dos 4 schemas já existentes (`landing`, `bronze`, `silver`, `gold`). Essa é a primeira peça de infraestrutura do projeto representada como código, no sentido restrito do termo (comandos SQL versionados e idempotentes) — distinto de Databricks Asset Bundles, que continua sendo o significado mais estrito de "IaC" no ecossistema Databricks (arquivo `databricks.yml` declarativo, CLI de deploy, múltiplos ambientes), ainda não adotado no projeto.

## Alternativas consideradas

- **Aumentar a tolerância de comparação** (ex.: para `0,1%` ou mais) para acomodar a defasagem esperada entre execuções: descartada — mascararia divergências reais sob um critério artificialmente frouxo; preferível manter o critério rigoroso e documentar a causa.
- **Reconciliação como 5ª Task automática do Job diário**: descartada nesta fase — dependência de sincronização externa (commit manual do KNIME) tornaria a automação frágil; considerada como evolução futura com defasagem proposital de 1 dia (reconciliar D-1, quando o dado já está garantidamente sincronizado).
- **Converter a Gold para fração em vez do KNIME para percentual**: equivalente em resultado, mas descartada — manter a Gold em percentual já é a decisão registrada em ADR-05, e reverter isso quebraria consistência com o restante do projeto (dashboards futuros, leitura humana) só para simplificar a reconciliação.

## Consequências

- A reconciliação de 27/08 registrou `status = diverge` em todos os níveis, com causa raiz documentada — resultado correto e esperado, não uma falha do pipeline ou da migração.
- Interpretação de qualquer reconciliação futura precisa considerar a causa raiz registrada antes de concluir se uma divergência é real (erro de lógica) ou esperada (timing de captura).
- A reconciliação de 28/08 e 31/08 continuará sendo executada manualmente, exigindo que o autor confirme a sincronização do CSV do KNIME (commit + Pull no Git folder) antes de rodar o notebook — registrado como passo operacional, não eliminável sem a automação com defasagem D-1 mencionada nas alternativas.
- A criação de infraestrutura via código (`00_setup_catalog`) estabelece um padrão reaproveitável caso o projeto precise de mais schemas ou catalogs no futuro, e é o primeiro passo concreto em direção a uma eventual adoção de Databricks Asset Bundles.