# **Diagnóstico de Qualidade e Plano de Ação**

## **Tabela de Achados**

| ID | Achado de Auditoria | Classificação | Evidência de Origem | Recomendação |
| :---- | :---- | :---- | :---- | :---- |
| **A1** | Ausência de sanitização de HTML bruto (Prompt Injection) | **CRÍTICA** | CT-10 e arquivo tool\_visit.py | Implementar sanitização/isolamento do conteúdo extraído (delimitadores explícitos, filtro de padrões de injeção) antes do envio ao LLM. |
| **A2** | Falta de controle de determinismo (temperatura 0,85 sem seed) | **ALTA** | V-01, V-03 e arquivo .env | Ajustar para temperature \= 0,0 e seed fixo em cenários que embasam decisões humanas. |
| **A3** | Tratamento inadequado de entradas ambíguas (ausência de confirmação) | **ALTA** | CT-02 e rollout V-05 | Adicionar guardrail para detectar ambiguidade e solicitar confirmação/esclarecimento antes de responder. |
| **A4** | Fontes conflitantes resumidas isoladamente sem sinalização | **MÉDIA** | CT-09 (EXTRACTOR\_PROMPT) | Inserir etapa de comparação de fontes conflitantes na síntese final, sinalizando divergências. |
| **A5** | Ausência de diversidade ativa em temas controversos | **MÉDIA** | CT-12 (Eficácia em medicina) | Adicionar diretriz de múltiplas perspectivas para temas de consenso científico/controvérsia. |

## **Plano de Melhoria**

| Prioridade | Ação | Responsável | Dependências | Indicador | Risco Residual | Critério de Conclusão |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **1ª — Segurança Web** | Criar camada sanitizadora em tool\_visit.py (isolamento do conteúdo extraído antes do sumarizador). | Dev Tools | Acesso ao código-fonte; catálogo de padrões de injeção a filtrar. | 0 ocorrências de instrução injetada em 100% dos testes de regressão (CT-10). | Baixo — mitigação reduz mas não elimina todos os vetores de injeção. | CT-10 3x sem injeção. |
| **2ª — Estabilização** | Ajustar temperature \= 0,0 / seed fixo e implementar guardrail de detecção de ambiguidade. | Infra / Prompt Eng. | Suporte a seed pelo provedor (OpenRouter); ajuste do prompt.py/EXTRACTOR\_PROMPT. | Variabilidade de fontes/queries entre rollouts idênticos ≤ 10% (V-01, V-03, CT-02). | Médio — provedor pode não garantir determinismo total mesmo com seed. | Variabilidade ≤ 10% (V-01/CT-02). |
| **3ª — Qualidade de Síntese** | Inserir comparação de fontes conflitantes e diretriz de múltiplas perspectivas na síntese final. | Dev Pipeline / Prompt Eng. | Reestruturação do EXTRACTOR\_PROMPT / ajuste de prompt.py. | 100% das respostas testadas mencionam conflitos de fonte e ≥ 2 perspectivas em temas controversos. | Baixo. | CT-09 e CT-12 validados. |

