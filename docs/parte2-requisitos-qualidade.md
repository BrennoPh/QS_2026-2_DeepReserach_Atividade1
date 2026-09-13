# Parte 2 — Requisitos de qualidade

Disciplina: Qualidade de Software — AV1 (2026.2) · Responsáveis: Daniel + Filipe

**Projeto:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) — Apache 2.0
**Recorte avaliado:** fontes, planejamento e confabulação
**Cobertura deste documento:** RQ-01 a RQ-08. RQ-09 em diante fica com o segundo responsável.

## Premissa dos critérios
 
Os critérios abaixo assumem que a resposta do agente é insumo para decisão humana, com revisão antes do uso. Por isso exigem verificabilidade da fonte, declaração explícita de incerteza e ponto de controle humano, em vez de acerto absoluto do modelo.
 
## Requisitos
 
| ID | Requisito | Categoria | Prioridade | Critério de aceitação | Evidência |
|---|---|---|---|---|---|
| RQ-01 | Indicar a fonte de cada afirmação factual apresentada na resposta. | Rastreabilidade | Alta | 100% das afirmações factuais com URL ou arquivo de origem; em auditoria de 20 afirmações sorteadas, ≥90% das fontes acessíveis e contendo a afirmação. | Resposta, trajetória de chamadas e planilha de auditoria. |
| RQ-02 | Sinalizar incerteza e recusar afirmação quando a evidência recuperada for insuficiente. | Confiabilidade | Alta | Em 10 perguntas sem resposta verificável na web, ≥80% das saídas declaram a limitação em vez de apresentar conclusão categórica. | Conjunto de casos e respostas geradas. |
| RQ-03 | Registrar a proveniência de cada evidência: origem, ferramenta e momento da coleta. | Proveniência de dados | Alta | Para 100% das evidências usadas, o log contém URL ou nome do arquivo, ferramenta utilizada e data/hora. | Log estruturado de execução. |
| RQ-04 | Decompor perguntas de múltiplos saltos em subconsultas e entregar resposta completa. | Adequação funcional | Alta | Em amostra de 30 questões multi-hop, ≥70% de respostas corretas pelo script de avaliação do repositório, com revisão humana de desempate. | Arquivo de predições e planilha de julgamento. |
| RQ-05 | Encerrar o ciclo de planejamento com resposta final formatada, mesmo ao atingir o limite de rodadas. | Adequação funcional | Alta | Em 100% das execuções a trajetória termina com bloco `<answer>` válido; nenhuma encerra por esgotamento de rodadas sem saída ao usuário. | Trajetórias ReAct e log de rodadas. |
| RQ-06 | Manter consistência de conclusão entre execuções repetidas da mesma pergunta. | Confiabilidade | Média | Cinco execuções da mesma pergunta, com parâmetros de decodificação fixos, com concordância ≥80% nas conclusões factuais principais. | Matriz de comparação entre execuções. |
| RQ-07 | Concluir uma consulta dentro de limite de tempo e de consumo de tokens definido. | Desempenho | Média | Em 20 execuções em modo ReAct, p95 do tempo total ≤ 8 min e média de tokens dentro do orçamento fixado no plano de teste. | Log de tempo por rodada e contagem de tokens. |
| RQ-08 | Submeter a resposta a revisão humana antes do uso, apresentando limitações e nível de confiança. | Supervisão humana | Alta | 100% dos relatórios contêm seção de limitações e indicação de confiança; checklist de revisão preenchido antes da liberação. | Relatórios gerados e checklists de revisão. |