# ADR-04 — Quarentena unificada, deduplicação e schema explícito na Silver

**Status:** Aceito
**Data:** 2026-08-27

## Contexto

O notebook `03_silver` precisa transformar a Bronze (cópia fiel, tudo string, incluindo registros com integridade divergente — ver [ADR-03](adr-03-merge-bronze-validacao-integridade.md)) em dados limpos e confiáveis. Três problemas precisavam de decisão: como tratar registros com data divergente versus registros com preço inválido; como garantir uma única linha por ticker/dia quando a mesma combinação aparece mais de uma vez; e qual schema exato a tabela Silver deveria expor.

Durante a construção, um problema real de design foi identificado e corrigido: a primeira versão do notebook selecionava as colunas numéricas tipadas (`preco_atual_num`, `fechamento_anterior_num`) **em cima** do DataFrame original da Bronze, sem remover as colunas antigas (string) nem as colunas de auditoria que não fazem sentido na Silver (`data_valida`, `motivo_quarentena` na tabela limpa). O resultado foi uma tabela com colunas duplicadas e fora de ordem — sintoma de não ter definido o schema final explicitamente antes de gravar.

## Decisão

**Quarentena unificada**: registros com `data_valida = False` (ver ADR-03) e registros com preço nulo ou não-positivo (`preco_atual` ou `fechamento_anterior`) são desviados para a **mesma** tabela de quarentena (`silver.cotacoes_quarentena`), com uma coluna `motivo_quarentena` identificando a causa específica (`data_particao_divergente`, `preco_atual_invalido`, `fechamento_anterior_invalido`). Um único mecanismo de desvio cobre os dois tipos de problema de qualidade previstos no escopo original do projeto.

**Deduplicação por registro mais recente**: quando mais de um registro existe para a mesma combinação `ticker + data_referencia`, mantém-se o de `data_carga` mais recente (via `row_number()` particionado por ticker+data, ordenado por `data_carga` decrescente). Reflete a intenção de que uma reexecução do pipeline para o mesmo dia deve refletir a versão mais atual do dado, não a primeira.

**Schema final explícito**: após tipagem e antes da gravação, cada tabela (limpa e quarentena) passa por uma seleção de colunas explícita e definitiva — `silver.cotacoes` expõe apenas `ticker`, `data_referencia` (tipo `date`), `preco_atual`/`fechamento_anterior` (tipo `double`), `data_hora_mercado` (tipo `timestamp`), `data_carga`, `arquivo_origem`; `silver.cotacoes_quarentena` mantém adicionalmente `data_valida` e `motivo_quarentena`, por ser tabela de auditoria. Nenhuma coluna intermediária de cálculo (ex.: sufixo `_num`) ou coluna herdada da Bronze sem função na Silver chega à tabela final.

## Alternativas consideradas

- **Tabelas de quarentena separadas por tipo de problema** (uma para data divergente, outra para preço inválido): descartada — fragmentaria a auditoria sem ganho real, já que a coluna `motivo_quarentena` cumpre a mesma função de diferenciação dentro de uma única tabela.
- **Manter o registro mais antigo em vez do mais recente na deduplicação**: descartada — não haveria motivo de negócio para preferir um dado desatualizado quando uma versão mais nova do mesmo dia está disponível.
- **Selecionar colunas apenas na gravação final, sem uma camada de schema explícito intermediária**: foi o que a primeira versão do notebook fez, e gerou o problema descrito no Contexto — corrigido introduzindo `df_silver_final`/`df_quarentena_final` como um passo deliberado e nomeado, não implícito.

## Consequências

- A tabela `silver.cotacoes` está pronta para consumo por qualquer camada seguinte (Gold) sem necessidade de tratamento adicional de tipo ou limpeza de colunas.
- A correção de schema exigiu recriar as tabelas do zero (`DROP TABLE` seguido de nova gravação) uma vez, já que o MERGE não reconcilia automaticamente uma mudança de schema entre execuções — ciclo registrado como parte do processo de desenvolvimento, não escondido.
- A função `merge_ou_cria`, criada neste notebook para evitar repetição da lógica de MERGE entre as duas tabelas, precisou ser duplicada manualmente no notebook `04_gold` — Databricks não compartilha funções entre notebooks automaticamente. Reconhecido como dívida técnica pequena (ver seção de evolução em `architecture.md`): candidata a extração futura para um módulo Python compartilhado, importado via `%run` ou pacote, quando o número de notebooks reaproveitando a mesma lógica justificar o investimento.