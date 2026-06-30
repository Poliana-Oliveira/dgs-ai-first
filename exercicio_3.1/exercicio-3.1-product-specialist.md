# Exercício 3.1 — Revisão Crítica das Respostas do Assistente
**Papel:** Product Specialist  
**Cenário:** 3 — Fase de Governança e Validação  
**Tópico:** Revisão Crítica de Outputs de IA  
**Data:** 29/06/2026

---

## 1. Minha Avaliação (antes do Claude)

| # | Pergunta | Classificação | Tipo de Erro | Ajuste Proposto |
|---|----------|--------------|-------------|-----------------|
| 1 | Prazo de devolução para produtos standard? | Parcialmente correta | Informação incompleta | Ajuste de prompt |
| 2 | SLA do cliente Silver? | Parcialmente correta | Informação incompleta | Ajuste de prompt |
| 3 | Posso devolver carga perigosa classe 3? | Correta | Informação incompleta | Ajuste de prompt |
| 4 | Política para carga danificada durante transporte? | **Incorreta — Alucinação** | Alucinação: o modelo inventou política específica com confiança alta e sem nenhuma fonte. Não existe documento formal sobre carga danificada na base | **Ajuste de pipeline** |
| 5 | SLA do cliente Enterprise? | **Correta** | — (assistente reconheceu o limite: tier Enterprise não existe na documentação; comportamento esperado e desejado) | Nenhum |
| 6 | Posso enviar carga perigosa com frete expresso? | **Problemática — Fonte não confiável** | Fonte não confiável: FAQ-Atendimento contém aviso explícito *"NÃO validado por Compliance ou Operações"*. Tema de carga perigosa exige documento formal (PROC/POL) | **Ajuste de pipeline** |

---

## 2. Avaliação do Claude (segundo avaliador)

| # | Pergunta | Classificação | Tipo de Erro | Ajuste Proposto |
|---|----------|--------------|-------------|-----------------|
| 1 | Prazo de devolução para produtos standard? | Parcialmente correta | Informação incompleta + **fonte incorreta** (citou seção 3.2; prazo está na 3.1) | Ajuste de prompt |
| 2 | SLA do cliente Silver? | Parcialmente correta | Informação incompleta (omitiu SLA de incidente crítico: 8h) | Ajuste de prompt |
| 3 | Posso devolver carga perigosa classe 3? | **Correta — sem problema** | Nenhum | Nenhum ajuste necessário |
| 4 | Política para carga danificada durante transporte? | **Incorreta — Alucinação** | Modelo inventou política específica com confiança alta e sem fonte. Não existe documento formal sobre carga danificada na base | **Ajuste de pipeline** (rejeitar resposta sem `source_document`) |
| 5 | SLA do cliente Enterprise? | **Correta** | Nenhum — assistente reconheceu o limite, nomeou tiers corretos e recomendou escalação | Nenhum ajuste necessário |
| 6 | Posso enviar carga perigosa com frete expresso? | **Problemática — Fonte não confiável** | FAQ-Atendimento é informal e explicitamente não validado por Compliance. Tema de carga perigosa exige documento formal (PROC/POL) | Ajuste de pipeline |

---

## 3. Comparação: Minha Avaliação × Claude

| # | Minha Classificação | Claude | Situação | Observação |
|---|--------------------|---------|---------|-----------:|
| 1 | Parcialmente correta | Parcialmente correta | ✅ Concordância | Claude acrescentou: fonte incorreta — seção 3.2 lista exceções, o prazo está na 3.1 |
| 2 | Parcialmente correta | Parcialmente correta | ✅ Concordância | Alinhadas após revisão. Claude detalha o gap: falta SLA de incidente crítico (8h) |
| 3 | Correta — informação incompleta | Correta — sem problema | ⚠️ Divergência leve | Avaliei como incompleta; Claude considera adequada. A menção a "supervisor" é equivalente ao ramal 4500 (Gestão de Riscos) conforme POL-001 |
| 4 | Incorreta — **Alucinação** | Incorreta — **Alucinação** | ✅ Concordância | Ambas identificaram alucinação: não existe documento formal sobre carga danificada na base. O modelo inventou política específica ("reembolso integral", "negligência comprovada", "laudo técnico") com confiança alta e sem fonte. Ajuste correto: pipeline (rejeitar resposta sem `source_document`) |
| 5 | **Correta** | **Correta** | ✅ Concordância | Ambas reconheceram como adequada: o assistente demonstrou o comportamento ideal ao reconhecer o limite do conhecimento, listar os tiers corretos e recomendar escalação. Confiança baixa foi apropriada |
| 6 | **Problemática — fonte não confiável** | **Problemática — fonte não confiável** | ✅ Concordância | Ambas identificaram a fonte como não confiável. O FAQ-Atendimento avisa explicitamente: *"NÃO validado por Compliance ou Operações"*. Para carga perigosa — tema de alto risco — a informação deve vir de documento formal (PROC/POL). Não existe PROC que formalize frete expresso para carga perigosa. Ajuste: pipeline |

---

## 4. Propostas de Ajuste para Respostas com Problema

### Resposta #1 — Ajuste de Prompt (informação incompleta + fonte incorreta)

**Problema identificado:** A resposta omite o CT-e e o detalhamento das fotos exigidos no procedimento (POL-001, seção 3.3). A fonte citada (seção 3.2) é a seção de exceções, não a de prazo geral (seção 3.1).

**Proposta:**
> Adicionar ao prompt instrução para que o assistente cite sempre a subseção correta do documento e liste todos os requisitos documentais do procedimento (não apenas os mais evidentes). Exemplo de instrução: *"Ao responder sobre procedimentos, liste todos os passos e documentos exigidos conforme a seção correspondente."*

---

### Resposta #2 — Ajuste de Prompt (informação incompleta)

**Problema identificado:** A resposta apresenta apenas o SLA de chamados gerais (48h), omitindo o SLA de incidentes críticos (8h). Um atendente pode aplicar o prazo errado em uma situação urgente.

**Proposta:**
> Adicionar ao prompt instrução para que, ao responder sobre SLA, o assistente sempre apresente as duas modalidades (geral e incidente crítico) com os respectivos prazos e definição de quando cada um se aplica.

---

### Resposta #3 — Ajuste de Prompt menor (opcional)

**Problema identificado (leve):** A resposta diz "escalar para o supervisor" sem especificar o canal correto (Gestão de Riscos, ramal 4500). O comportamento é adequado, mas pode ser mais preciso.

**Proposta (baixa prioridade):**
> Enriquecer os chunks da POL-001 com o ramal e o setor de destino para escalação, de modo que o assistente inclua essa informação automaticamente ao recuperar esse trecho.

---

### Resposta #4 — Ajuste de Pipeline (alucinação) ⚠️ Alta Prioridade

**Problema identificado:** O assistente inventou uma política inexistente na base com confiança alta e sem nenhuma fonte citada. Tipo de erro: **alucinação**.

**Por que ajuste de prompt não resolve:** O modelo pode ignorar instruções de prompt quando gera texto plausível sem âncora em documento. A mitigação deve ser determinística.

**Proposta:**
> **Pipeline:** Implementar rejeição automática de qualquer resposta cujo campo `source_document` esteja vazio ou nulo. Substituir por mensagem padrão: *"Não encontrei documentação formal sobre este tema. Recomendo escalar ao setor responsável."*  
> **Structured output:** Tornar `source_document` campo obrigatório no schema Zod da resposta. Respostas que não preencherem o campo são barradas antes de chegar ao atendente.

---

### Resposta #6 — Ajuste de Pipeline (fonte não confiável) ⚠️ Alta Prioridade

**Problema identificado:** Informação sobre carga perigosa foi fornecida com confiança alta a partir de fonte informal (FAQ-Atendimento), que é explicitamente não validada. Não existe documento formal que defina frete expresso para carga perigosa.

**Por que ajuste de prompt não resolve:** A resposta tecnicamente veio de um documento indexado. O problema está em qual documento pode ser usado para qual tema.

**Proposta:**
> **Pipeline:** Implementar regra de fonte por tema — respostas sobre carga perigosa (termos: "carga perigosa", "ANTT", "classe 1 a 6") só podem citar documentos formais (`POL-001`, `PROC-042`, `PROC-042-v2`, `PROC-043`). Se o único chunk recuperado for do `FAQ-Atendimento`, substituir por: *"Não localizei documentação formal sobre este procedimento. Recomendo contato com o Compliance."*  
> **HITL:** Respostas sobre carga perigosa com confiança alta baseadas em FAQ devem ser sinalizadas para revisão humana antes de chegar ao atendente.

---

## 5. Síntese dos Ajustes Propostos

| Tipo de Ajuste | Respostas | Prioridade |
|----------------|-----------|-----------|
| Ajuste de prompt — completude de procedimento | #1, #2 | Média |
| Ajuste de prompt — citação de subseção correta | #1 | Baixa |
| Ajuste de prompt — enriquecimento de chunk | #3 | Baixa |
| **Ajuste de pipeline — rejeitar resposta sem fonte** | **#4** | **Alta** |
| **Ajuste de pipeline — bloquear fonte informal para temas sensíveis** | **#6** | **Alta** |

---

## 6. Reflexão sobre a Comparação com o Claude

As avaliações convergiram nas respostas **#1, #2, #4, #5 e #6**, com divergência leve apenas na **#3**:

- **#4 — Alucinação:** Identifiquei corretamente como incorreta e classifiquei como alucinação. Não havia fonte citada e não existe documento formal sobre política de carga danificada na base — o modelo inventou conteúdo específico com confiança alta. A distinção entre alucinação e fonte não confiável é relevante para o ajuste: alucinação exige bloqueio determinístico via pipeline; fonte não confiável pode ser tratada com filtro de retrieval.

- **#5 — Resposta correta reconhecida:** Identifiquei corretamente como adequada. O assistente demonstrou o comportamento ideal: reconhecer o limite do próprio conhecimento em vez de inventar uma resposta. Baixa confiança foi apropriada. Este padrão deve ser preservado e incentivado no produto.

- **#6 — Fonte não confiável:** Identifiquei corretamente como problemática com base na classificação informal do FAQ-Atendimento. O cabeçalho do documento avisa explicitamente que não foi validado por Compliance. Para temas de alto risco como carga perigosa, a origem da informação é tão importante quanto o conteúdo — uma resposta plausível baseada em fonte não autorizada é igualmente problemática.

- **#3 — Divergência leve:** Avaliei como correta com informação incompleta (falta o ramal 4500); o Claude considera plenamente adequada. A menção a "escalar para o supervisor" é funcionalmente equivalente ao encaminhamento correto conforme POL-001.

---

## Avaliação do Exercício 3.1

### Resumo
O entregável demonstra estrutura completa e avaliação independente sólida, com identificação correta das duas armadilhas centrais do exercício. A análise própria (pré-Claude) classificou corretamente a resposta #4 como alucinação e a #6 como problemática por fonte não confiável — os pontos mais críticos do conjunto. As propostas de ajuste distinguem adequadamente intervenções de prompt (probabilísticas) e de pipeline (determinísticas). A comparação com o Claude é convergente e honesta.

### Scores por Dimensão

| Dimensão | Score | Justificativa |
|----------|-------|---------------|
| D1 — Domínio Conceitual | 2 | Demonstra compreensão da distinção prompt vs. pipeline, do conceito de alucinação e de fonte não confiável como categorias distintas. Propostas de ajuste mostram clareza sobre quando cada intervenção se aplica. Ponto de crescimento: aprofundar structured outputs como mecanismo preventivo de alucinação |
| D2 — Uso de Ferramentas | 2 | Comparação com segundo avaliador realizada e documentada com evidência clara. O exercício exige uso direto do Claude (chat); a mediação via ferramenta atende à evidência de comparação, mas não substitui a interação direta iterativa |
| D3 — Qualidade do Entregável | 2 | Artefato completo com todas as seções exigidas: avaliação própria, avaliação do Claude, comparação, propostas de ajuste, síntese e reflexão. Propostas são concretas e acionáveis com distinção prompt/pipeline |
| D4 — Pensamento Crítico | 3 | Armadilha #4 (alucinação): identificada corretamente de forma independente ✓. Armadilha #6 (fonte não confiável): identificada corretamente, citando o aviso do cabeçalho do FAQ ✓. Resposta #5 reconhecida como adequada ✓. As três armadilhas do exercício foram identificadas na análise própria antes do Claude |
| D5 — Aplicabilidade ao Projeto | 2 | Conectado ao contexto NovaTech com referência correta a POL-001, SLA-2024 e FAQ-Atendimento. Não referencia explicitamente os guardrails do cenário 2 (DEVE/NÃO DEVE/QUANDO EM DÚVIDA) nas propostas de ajuste |

**Score do exercício: 2.2**

### Verificação de Armadilhas

| Armadilha | Esperado | Identificado na avaliação própria | Resultado |
|-----------|----------|------------------------------------|-----------|
| Resposta #4 — Alucinação | Identificar como alucinação: modelo inventou política inexistente com confiança alta e sem fonte | Identificada como **Incorreta — Alucinação** ✓. Justificativa: não existe documento formal sobre carga danificada na base | ✅ Identificada |
| Resposta #6 — Fonte não confiável | Identificar como problemática: FAQ informal para tema de carga perigosa (alto risco) | Identificada como **Problemática — Fonte não confiável** ✓. Citou aviso explícito do cabeçalho do FAQ | ✅ Identificada |
| Respostas #1, #2, #3, #5 como adequadas | Reconhecer as respostas corretas | #1 ✓ parcialmente correta, #2 ✓ parcialmente correta, #3 ✓ correta, #5 ✓ correta | ✅ Todas reconhecidas |

### Pontos Fortes
1. **Identificação independente das armadilhas centrais:** As respostas #4 (alucinação) e #6 (fonte não confiável) foram corretamente classificadas na avaliação própria, antes do Claude — o principal critério de D4 neste exercício.
2. **Propostas de ajuste concretas e bem diferenciadas:** A distinção entre ajuste de prompt (probabilístico) e ajuste de pipeline (determinístico) está correta e demonstra maturidade técnica de produto.
3. **Estrutura completa e convergência honesta com o Claude:** Todas as seções exigidas estão presentes. A comparação registra concordâncias e a única divergência leve (#3) é explicada com justificativa.

### Pontos de Melhoria
1. **Conectar as propostas de ajuste aos guardrails do cenário 2** — o harness de produto deve preservar os guardrails DEVE/NÃO DEVE/QUANDO EM DÚVIDA já formalizados. Mencionar essa conexão fortaleceria o D5 e demonstraria visão sistêmica do produto.
2. **Aprofundar o mecanismo de structured output como prevenção de alucinação** — a proposta de pipeline para #4 está correta, mas poderia detalhar como o schema Zod com `source_document` obrigatório impede especificamente esse tipo de resposta.
3. **Usar o Claude diretamente (chat interativo)** para a etapa de segundo avaliador — permite iterar sobre divergências em tempo real e registrar a evolução da análise.

### Classificação
**Aprovado (2.2)**

### Tópicos da Trilha para Reforço
- **Harness Engineering — Structured Outputs:** aprofundar como o schema obrigatório (`source_document`, `confidence_score`) atua como barreira determinística contra alucinação — complementando o que os guardrails de prompt fazem de forma probabilística.
