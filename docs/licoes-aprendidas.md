# Lições Aprendidas — Modernização de Dados B3

> Registro honesto do que o processo de construção deste projeto ensinou — incluindo erros cometidos e corrigidos, não só acertos. Complementa as decisões já formalizadas em `docs/adr/`.

## 1. Arquitetura e dados

**Separar geração de ingestão evita acoplamento — e paga dividendo na reconciliação.** A decisão de KNIME e Databricks consumirem a API de forma independente ([ADR-01](adr/adr-01-ingestao-independente-landing.md)) parecia, no início, apenas uma questão de "fazer certo". Na prática, foi o que deu sentido real à reconciliação: sem essa independência, comparar Gold com KNIME seria comparar um sistema com uma cópia dele mesmo.

**Um dado de teste esquecido virou prova de controle de qualidade, não um problema escondido.** Um arquivo de teste do Widget de data (rotulado como 26/08, mas contendo cotação real de 27/08) poderia ter sido silenciosamente ignorado ou apagado. Em vez disso, a coluna `data_valida` ([ADR-03](adr-03-merge-bronze-validacao-integridade.md)) capturou a divergência automaticamente, e o dado foi mantido — rastreável, não escondido. Isso reforçou um princípio: erros de desenvolvimento não precisam ser apagados da história, podem virar evidência de que o sistema de qualidade funciona.

**Um incidente real na reconciliação D-1 mostrou o valor de testar a própria correção.** Ao migrar de filtro por intervalo para filtro por dia exato, uma linha residual incorreta (28/08, comparando dado real com ausência de dado do KNIME) ficou gravada nas tabelas de reconciliação, com uma causa raiz factualmente errada. Foi identificada, investigada e corrigida via `DELETE` direcionado ([ADR-10](adr/adr-10-reconciliacao-d1-automatizada.md)) — lição prática: mudar a lógica de um pipeline não corrige automaticamente dados já gravados pela lógica antiga.

**Formatos inconsistentes entre sistemas só aparecem quando você tenta compará-los de verdade.** O KNIME calculava retorno em fração decimal; a Gold do Databricks, em percentual. Isso não era um "bug" em nenhum dos dois lados isoladamente — só virou visível na hora de reconciliar ([ADR-07](adr/adr-07-reconciliacao-formato-tolerancia-manual.md)). Lição: escrever testes de integração (ou, neste caso, a própria reconciliação) cedo revela inconsistências que nenhum dos lados percebe sozinho.

**Capitalização composta importa, e não é só rigor acadêmico.** Calcular o índice acumulado por soma simples teria sido mais fácil e "quase certo" — mas matematicamente errado para retornos financeiros ([ADR-09](adr/adr-09-indice-acumulado-capitalizacao-composta.md)). Entender a diferença entre os dois métodos, e por que índices reais usam capitalização composta, foi um investimento pequeno de tempo com retorno real de credibilidade técnica.

**Nem toda prática "mais robusta" vale o custo no momento certo.** OOP e Databricks Asset Bundles foram avaliados repetidamente ao longo do projeto (não só uma vez) e, cada vez, a resposta permaneceu "ainda não" — não por desconhecimento, mas porque a escala real do projeto (uma fonte, um pipeline linear) não justificava a abstração. A lição não é "nunca use OOP/IaC", é "julgue pela necessidade real, não pelo que parece mais avançado".

## 2. Processo de desenvolvimento e Git

**Arquivos binários (`.knwf`) em Git exigem atenção redobrada.** Diferente de arquivos de texto, o Git não consegue mesclar automaticamente um `.knwf` — qualquer edição simultânea em dois lugares (local e Databricks) vira conflito manual. Isso aconteceu mais de uma vez ao longo do projeto, e a solução (`git stash`, `checkout --ours`/`--theirs`) só ficou clara na prática, não por antecipação.

**Trabalhar em dois ambientes (local + Databricks) exige disciplina constante de sincronização.** Mais de um erro (`Couldn't find remote ref`, conflitos de merge, branch desatualizada) veio de esquecer de dar `Pull` ou `git pull` antes de começar a trabalhar em um dos dois lados. A lição prática: sincronizar **antes** de cada sessão de trabalho, não só depois.

**Branches "zumbi" (já mergeadas, mas ainda referenciadas localmente) causam erro silencioso.** Continuar numa branch antiga depois do merge gerou, mais de uma vez, comandos que falhavam de forma confusa (`git pull` reclamando de uma ref que não existe mais). A correção de hábito (sempre partir de uma `main` atualizada para nova branch) só veio depois de sentir esse atrito repetidamente.

## 3. Planejamento e priorização

**Cronograma rígido por dia não sobrevive a um projeto que evolui com decisões reais.** O plano inicial (Dia 1/Dia 2/Dia 3) foi abandonado cedo em favor de um formato de 3 blocos (já feito / falta fazer / melhorias) — mais honesto sobre como o trabalho realmente avançava, e mais fácil de manter atualizado sem reescrever tudo a cada rodada.

**Nem toda melhoria vale o mesmo, e o contexto (tempo, risco, visibilidade) muda a prioridade.** Observabilidade e eliminação de duplicação de código foram corretamente priorizadas por serem baixo risco e alto retorno; estender tratamento de erro explícito a todos os notebooks foi corretamente **adiado** na manhã da apresentação, por ser uma mudança em código já validado, com risco desproporcional ao benefício visível para a plateia. A lição: "está na lista de melhorias" não significa "deve ser feito agora" — o momento importa tanto quanto o valor.

**Corrigir o próprio escopo, quando necessário, é mais honesto do que manter uma inconsistência documentada.** A janela de reconciliação teve que ser corrigida de "26/08 em diante" para "27/08 em diante" quando ficou claro que o Databricks não tinha como obter retroativamente o preço de um dia anterior à criação da infraestrutura. Atualizar o README/architecture com essa correção explícita, em vez de deixar a inconsistência silenciosa, evitou uma armadilha em qualquer entrevista futura sobre o projeto.

**Para audiência executiva, menos painéis robustos comunicam melhor que muitos painéis esparsos.** O dashboard foi reduzido de 5 para 4 painéis exatamente porque o volume real de dados (2-3 dias) não sustentava certos gráficos com solidez visual. Ajustar a ambição do artefato ao volume real de dado disponível, em vez de forçar todos os indicadores planejados, resultou em um material mais forte para apresentação.

**Embutir ressalvas de domínio na camada de consumo de IA evita erro na frente do público certo.** As instruções do Genie Space, escritas antes de qualquer pergunta real, garantiram respostas que já incluíam automaticamente as ressalvas corretas (proxy simplificado, causa raiz de divergência) sem precisar de correção humana durante os testes — prova de que o cuidado na configuração antecipa problemas que só apareceriam ao vivo.

## 4. Resultado

O projeto foi apresentado com sucesso e avançou para a etapa final do processo seletivo — validação prática de que a combinação de honestidade técnica (documentar limitações e não apenas conquistas), profundidade de arquitetura (Medallion completo, reconciliação real, observabilidade) e cuidado na camada de consumo (Dashboard e Genie testados antes da apresentação, não na hora) formou um material técnico e narrativo coerente.