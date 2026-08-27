# ADR-03 — Escrita idempotente na Bronze via MERGE e coluna de validação de integridade

**Status:** Aceito
**Data:** 2026-08-27

## Contexto

O notebook `02_bronze` lê todos os arquivos JSON disponíveis no Volume `landing.raw` (acumulados dia após dia) e precisa gravá-los como tabela Delta na camada Bronze. Diferente da Landing (que mantém um snapshot por dia, sobrescrito a cada execução — ver [ADR-02](adr-02-widgets-idempotencia-landing.md)), a Bronze precisa **acumular histórico**: o objetivo é que ela cresça conforme novos dias forem ingeridos, não que reflita apenas o dia mais recente.

Além disso, durante os testes deste notebook, identificou-se um caso real: um arquivo gravado na pasta `data=2026-08-26/` continha, na verdade, a cotação de `2026-08-27` (a API `brapi.dev` retorna sempre o preço atual, não um preço histórico — o widget de data do notebook de ingestão apenas rotula a pasta de destino, não obtém dado retroativo). Esse tipo de divergência entre "onde o arquivo foi salvo" e "o que o arquivo realmente contém" precisa ser detectável na Bronze, não silenciosamente ignorado ou escondido.

## Decisão

**Escrita via `MERGE INTO`**, com chave composta `ticker + data_referencia` (identifica unicamente "a cotação desse papel, nesse dia"):
- Registro novo (combinação ticker+data ainda não existente): inserido.
- Registro já existente: atualizado (`whenMatchedUpdateAll`).
- Na ausência da tabela (primeira execução), ela é criada diretamente, sem MERGE.

**Coluna `data_valida` como controle de qualidade e auditoria**: cada registro é validado comparando a data extraída do conteúdo real (`regularMarketTime`, dentro do JSON) com a data da partição onde o arquivo foi salvo (`data=AAAA-MM-DD`, extraída do caminho no Volume). Quando as duas divergem, o registro é marcado como `False`, mas **mantido na tabela**, não descartado — o objetivo é rastreabilidade, não limpeza (que é responsabilidade da Silver, não da Bronze).

**Colunas de auditoria adicionais**: `data_carga` (timestamp de ingestão na Bronze) e `arquivo_origem` (caminho do arquivo fonte), sustentando rastreabilidade completa de cada registro.

## Alternativas consideradas

- **`overwrite` da tabela inteira a cada execução**: descartada — embora funcionasse na primeira execução (que reprocessa todos os arquivos do Volume de uma vez), criaria risco real: se um arquivo de um dia anterior deixasse de existir no Volume por qualquer motivo (ex.: rotina futura de retenção/limpeza), aquele dia desapareceria silenciosamente da Bronze também. MERGE por chave evita essa dependência.
- **Descartar (não ingerir) registros com data divergente**: descartada — esconderia um problema real de origem em vez de torná-lo visível. A Bronze, por princípio de arquitetura já definido no projeto, é cópia fiel da fonte; filtrar dados nesse estágio contradiria esse princípio, além de eliminar a evidência de auditoria.
- **Sinalizar a divergência apenas em log/print, sem persistir como coluna**: descartada — não sobrevive além da execução do notebook; uma coluna na própria tabela permite consulta e auditoria a qualquer momento, por qualquer pessoa com acesso à Bronze.

## Consequências

- A tabela Bronze cresce de forma idempotente: reexecutar o notebook no mesmo dia não duplica registros, apenas atualiza o que já existe para aquela combinação ticker+data.
- A coluna `data_valida` transforma um problema real encontrado durante a construção (arquivo de teste com data de partição incorreta) em prova concreta de controle de qualidade funcionando, em vez de um erro escondido ou corrigido silenciosamente.
- Um registro com `data_valida = False` permanece na Bronze indefinidamente, a menos que uma rotina de limpeza futura seja criada — não há, hoje, expiração ou arquivamento desses registros (potencial item de evolução futura, na mesma linha da observabilidade já documentada).
- Consultas e transformações a partir da Silver precisarão decidir explicitamente como tratar registros com `data_valida = False` (ex.: excluir, sinalizar, ou enviar para quarentena) — essa decisão ainda não foi tomada e será registrada em ADR próprio ao construir a Silver.