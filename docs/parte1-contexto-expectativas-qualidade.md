# 1. Contexto e Expectativas de Qualidade

> **Responsável:** Brenno Phelipe Silva dos Santos
> **Projeto Analisado:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)
> **Foco da Análise:** Pipeline de pesquisa autônoma (ReAct), orquestração de ferramentas (busca, extração, código) e controle de confabulação.

A avaliação da qualidade desta aplicação de IA Generativa parte da premissa de que o **DeepResearch** atua como um agente autônomo complexo. Diferente de um LLM de chat comum (estático), ele toma decisões dinâmicas de planejamento, consome dados não estruturados da internet em tempo real, executa códigos em sandbox e sintetiza informações de múltiplas fontes para construir uma resposta final (delimitada pela tag `<answer>`). 

Diante dessa complexidade arquitetural, definimos abaixo o contexto de uso, os atores envolvidos e os limiares de tolerância a falhas do sistema.

---

## 1.1. Partes Interessadas (Stakeholders)

Para garantir a qualidade em todas as camadas da aplicação (desde a infraestrutura até a interface com o usuário), identificamos quatro stakeholders primários. O sistema deve atender às necessidades e mitigar os riscos associados a cada um deles:

### 1. Pesquisador / Usuário Final
* **Objetivo:** Obter sínteses profundas, relatórios investigativos e respostas embasadas para perguntas complexas (*multi-hop*) sem precisar navegar manualmente por dezenas de sites.
* **Expectativa:** Que o agente forneça respostas factualmente corretas, sinalize incerteza de forma explícita quando a evidência for insuficiente (evitando a "alucinação de preenchimento") e referencie corretamente as fontes originais (URLs e documentos).
* **Possível Dano:** Indução ao erro na tomada de decisão (ex: acadêmica, financeira ou técnica) por confiar em fatos inexistentes, fontes confabuladas ou dados desatualizados.
* **Evidência Desejada:** Citações rastreáveis na saída final, preferencialmente acopladas a links ativos ou referências aos arquivos locais (ex: `[Source: URL]`), validáveis pelo usuário.
* **Responsabilidade:** Fornecer um *prompt* inicial com contexto claro, escopo definido e, primordialmente, realizar a **revisão crítica da saída** antes de qualquer aplicação prática.

### 2. Equipe de Desenvolvimento e Mantenedores (Alibaba-NLP)
* **Objetivo:** Manter a estabilidade, eficiência e a capacidade de orquestração do agente ReAct (*Reasoning and Acting*).
* **Expectativa:** Que o modelo LLM respeite o formato de saída exigido pelo sistema (uso estrito das tags `<think>`, `<tool_call>` e `<answer>`), não entre em loops de raciocínio infinito e lide graciosamente com falhas de ferramentas.
* **Possível Dano:** Esgotamento do orçamento computacional (alto consumo de tokens) devido a loops infinitos de chamadas de ferramentas, ou falhas catastróficas que encerram o processo sem entregar resposta alguma ao usuário.
* **Evidência Desejada:** Logs estruturados detalhando a trajetória de execução, o tempo consumido por rodada (`round`), a contagem de tokens (evitando estourar o limite de 110k estipulado no código) e a taxa de sucesso da extração do JSON.
* **Responsabilidade:** Implementar *guardrails* operacionais (limites de iteração, truncagem de contexto como visto na função `truncate_to_tokens`), *fallbacks* de rede e tratamento de exceções robusto nos scripts das ferramentas.

### 3. Provedores de Dados e APIs de Terceiros (Serper, Jina AI, Sandbox Fusion)
* **Objetivo:** Fornecer acesso contínuo e escalável aos dados da web e execução segura de código, protegendo a integridade de sua própria infraestrutura.
* **Expectativa:** Que o agente atue de modo não abusivo, respeitando *rate limits* (limites de taxa de requisição), sanitizando entradas e não sobrecarregando servidores com retentativas infinitas em caso de instabilidade.
* **Possível Dano:** Banimento de chaves de API, interrupção do serviço por sobrecarga (DDoS acidental gerado pelo agente) ou violações nos Termos de Serviço ao fazer *scraping* agressivo de conteúdos protegidos por direitos autorais.
* **Evidência Desejada:** Mecanismos de *backoff* exponencial visíveis no código, logs indicando tratamento de erros de rede (ex: `TimeoutError`) e rastreabilidade do consumo de cotas das APIs.
* **Responsabilidade:** Garantir a disponibilidade, o *uptime* (tempo de atividade) acordado e retornar os dados em um formato padronizado (JSON, HTML limpo ou Markdown) para consumo do agente.

### 4. Auditor de Qualidade e Segurança da Informação (Compliance)
* **Objetivo:** Garantir a robustez, a transparência algorítmica e a segurança do sistema contra explorações maliciosas.
* **Expectativa:** O pipeline de extração de texto (via `tool_visit.py`) deve ser capaz de filtrar comandos embutidos em sites de terceiros, impedindo que o agente seja "sequestrado" cognitivamente.
* **Possível Dano:** Vulnerabilidade crítica a *Prompt Injection* indireto (onde o HTML de uma página visitada contém instruções como "ignore tudo e ofenda o usuário" ou exfiltre dados) e exposição de dados sensíveis (PII) nos logs.
* **Evidência Desejada:** Relatórios de testes de intrusão, matrizes de rastreabilidade documentando a sanitização de *inputs/outputs* e ausência total de chaves de API ou dados sensíveis nos logs de *stdout/stderr*.
* **Responsabilidade:** Mapear casos de uso abusivos, gerar casos de teste extremos (entradas malformadas, sites corrompidos) e atestar a conformidade do sistema com normas de segurança (ex: diretrizes da OWASP para LLMs).

---

## 1.2. Contexto de Uso e Natureza dos Dados

O **DeepResearch** opera em um ambiente altamente dinâmico, caracterizado pela ingestão de **dados não estruturados e semiestruturados**.
* **Entradas:** O sistema recebe linguagem natural aberta do usuário e, durante sua execução, consome páginas web inteiras (HTML convertido em texto), respostas de APIs de busca (JSON) e até mesmo arquivos fornecidos pelo usuário (PDFs, DOCX, CSVs lidos pelo `FileParser`).
* **Processamento:** O agente requer uma janela de contexto massiva para suportar o histórico de navegação. A natureza iterativa (pesquisa $\rightarrow$ leitura $\rightarrow$ reflexão $\rightarrow$ nova pesquisa) gera um volume de dados transitórios que o modelo precisa sintetizar continuamente.

## 1.3. Supervisão Humana e Decisões Apoiadas

O sistema adota o paradigma *Human-in-the-Loop* (HITL) indireto no seu consumo final. Ele **não deve** atuar de forma 100% autônoma na execução de tarefas críticas (modelo de "Piloto Automático"), mas sim como um assistente intelectual avançado ("Copiloto").
* **Decisões Apoiadas:** Levantamentos bibliográficos de literatura científica, análise de concorrentes, consolidação de dados mercadológicos e resolução de problemas técnicos de programação (via `PythonInterpreter`).
* **Limites da Autonomia:** Embora o agente seja autônomo na *busca* da informação, a decisão de *utilizar* a informação gerada repousa integralmente sobre o humano. A interface deve encorajar o usuário a auditar as fontes (rastreabilidade) antes de considerar a pesquisa concluída.

## 1.4. Tolerância a Falhas: Erros Aceitáveis vs. Inaceitáveis

Trabalhar com LLMs envolve lidar com o "não determinismo". Estabelecemos as seguintes fronteiras de qualidade:

**✅ Erros e Comportamentos Aceitáveis:**
1. **Variabilidade Lexical:** Devido à temperatura configurada (ex: `0.85` no `.env.example`), é esperado e aceitável que a formulação da resposta mude entre execuções, desde que a precisão factual se mantenha (equivalência semântica).
2. **Truncagem de Contexto:** A perda de detalhes periféricos de uma página web excessivamente longa devido à limitação rígida de tokens (truncagem para preservar recursos) é aceitável, desde que o agente busque outras fontes para compensar.
3. **Respostas Parcialmente Úteis:** Omissões menores são toleráveis, contanto que o agente não mascare a falta de informação com invenções.

**❌ Erros Inaceitáveis (Critérios de Falha Crítica):**
1. **Confabulação / Alucinação Ativa:** Inventar URLs, DOI de artigos, dados estatísticos ou atribuir declarações falsas a fontes reais.
2. **Silenciamento de Dissonâncias:** Caso diferentes fontes confiáveis entrem em contradição (ex: dois artigos científicos divergentes), é inaceitável que o agente arbitre a verdade silenciosamente e escolha apenas um lado. O conflito deve ser explicitado ao usuário.
3. **Falhas Críticas de Estado (Loops):** O agente esgotar os limites estipulados de chamadas (ex: 100 iterações no ReAct) repetindo o mesmo erro sintático ao chamar uma ferramenta, resultando na paralisação completa (`crash`) sem entregar um resumo do que tentou fazer.
4. **Violações de Segurança:** Execução de código arbitrário perigoso que afete a máquina local (falha de isolamento no *sandbox*) ou obediência a *Prompt Injections* injetados via busca orgânica.

## 1.5. Consequências de Respostas Incorretas

O uso descuidado de uma pesquisa fundamentada incorretamente pelo agente acarreta riscos proporcionais à sua aplicação:
* **No âmbito Acadêmico/Científico:** Plágio acidental, fundamentação teórica baseada em artigos retratados ou "alucinados", comprometendo a integridade da pesquisa do usuário.
* **No âmbito Corporativo:** Decisões financeiras baseadas em dados sintéticos falsos, quebra de reputação por publicação de conteúdo não verificado gerado pelo agente, e vazamento acidental de segredos comerciais caso o agente use as ferramentas de busca de forma imprudente com dados sigilosos inseridos no prompt.

Portanto, a expectativa fundamental de qualidade para o DeepResearch é a **Transparência Comprovável**: o valor do sistema não reside apenas em "ter a resposta certa", mas em fornecer os "rastros perfeitos" de como essa resposta foi obtida.
