# Cenário 2 — Design de Critérios de Aceitação para Respostas de IA

> **Papel:** QA
> **Ferramentas utilizadas:** Claude (chat) + Claude Cowork
> **Fonte de verdade:** Anexo A — Documentação Simulada da NovaTech (POL-001, PROC-042 v1, PROC-042 v2, SLA-2024, FAQ-Atendimento)
> **Guardrails do projeto:** (1) Sempre citar fonte. (2) Nunca inventar prazos ou valores. (3) Quando não souber, dizer explicitamente. (4) Responder em português formal.

---

## Exercício 1.2 — Enunciado

**Contexto:** O time precisa definir quando uma resposta do assistente é "boa o suficiente".

### Inputs: 5 pares de pergunta/resposta simulados

| # | Pergunta | Resposta do Assistente | Fonte Citada |
|---|----------|----------------------|--------------|
| 1 | "Qual o prazo de devolução?" | "O prazo é de 7 dias úteis, exceto para cargas perigosas classes 1 a 6 da ANTT." | POL-001, seção 3.2 |

| 2 | "Quanto custa frete para 600kg para Manaus?" | "O frete especial para cargas acima de 500kg para a região Norte tem multiplicador de 1.8 sobre o valor base." | PROC-042-v2, seção 2 |
| 3 | "Qual o SLA do cliente Platinum?" | "O cliente Platinum tem resposta em até 1h e resolução em até 12h." | SLA-2024 |
| 4 | "Posso devolver carga perigosa?" | "Sim, cargas perigosas podem ser devolvidas em até 7 dias úteis." | POL-001, seção 3.2 |
| 5 | "Qual o multiplicador de frete para o Sudeste?" | "O multiplicador regional para o Sudeste é 1.1." | PROC-042-v2, seção 2 |

---

## Entregável 1 — Avaliação Manual [feita antes da rubrica]

> Avaliação realizada com base direta nos documentos do Anexo A, antes de qualquer uso de ferramenta de IA.

| # | Veredicto | Justificativa com base no Anexo A |
|---|---|---|
| **1** | **Parcialmente correta** | O prazo de 7 dias está correto (POL-001, seção 3.1). A exceção para carga perigosa está correta (seção 3.2). Porém a resposta omite as outras duas exceções da mesma seção: carga refrigerada com cadeia de frio rompida e carga com lacre de segurança violado. Além disso, a fonte citada (seção 3.2) é imprecisa para o prazo — o prazo de 7 dias vem da seção 3.1, não da 3.2. |
| **2** | **Correta e ao mesmo tempo incorreta** | O multiplicador 1.8 para a região Norte está correto conforme PROC-042-v2, seção 2.1. Porém a resposta omite o fator de peso — para 600kg (faixa 500–1.000kg) o fator é 1.0, o que não altera o resultado neste caso específico, mas a fórmula completa é: `valor base × multiplicador regional × fator de peso`. A resposta é incompleta para uma pergunta de custo. |
| **3** | **Incorreta — alucinação dupla** | O tier "Platinum" não existe na NovaTech. A SLA-2024, seção 1, declara explicitamente: *"Não existem outros tiers além dos três listados acima"* (Gold, Silver e Standard). O assistente inventou o tier e fabricou os valores de SLA. Falha dupla: entidade inexistente + valores não documentados. |
| **4** | **Incorreta — contradição direta** | A POL-001, seção 3.2, afirma explicitamente que cargas perigosas **NÃO são elegíveis** para devolução pelo processo padrão. A resposta do assistente diz o contrário. A resposta correta seria: orientar o cliente a contatar Gestão de Riscos (ramal 4500) para tratamento individual. |
| **5** | **Correta** | O multiplicador 1.1 para o Sudeste está correto conforme PROC-042-v2, seção 2.1. Fonte citada corretamente. Resposta direta e sem omissões relevantes para a pergunta feita. |

---

## Entregável 2 — Rubrica de Avaliação [gerada com Claude]

> Rubrica com 4 dimensões, escala 1–3 cada. Pontuação total: mínimo 4, máximo 12.

---

### Dimensão 1 — Precisão Factual

*Verifica se o conteúdo da resposta está correto e não contradiz a documentação oficial (Anexo A).*

| Nível | Critério |
|---|---|
| **3 — Correto** | Todos os fatos estão alinhados com a documentação. Nenhuma informação fabricada, contraditória ou desatualizada. |
| **2 — Parcialmente correto** | O conteúdo principal está certo, mas há omissão de informação relevante ou imprecisão que não inverte o sentido da resposta. |
| **1 — Incorreto** | Contém fato errado, inventado ou que contradiz diretamente a documentação. Inclui alucinação de entidades inexistentes (ex: tier Platinum). |

---

### Dimensão 2 — Citação de Fonte

*Verifica se a fonte está presente, correta e específica o suficiente para ser verificada independentemente.*

| Nível | Critério |
|---|---|
| **3 — Completa e correta** | Cita documento e seção corretamente. A citação permite localizar a informação sem ambiguidade. |
| **2 — Parcial** | Cita o documento mas a seção está errada ou ausente. Ou a fonte existe mas não é a mais precisa para o trecho citado. |
| **1 — Ausente ou inválida** | Não cita fonte, ou cita um documento que não contém a informação afirmada, ou cita documento inexistente na base. |

---

### Dimensão 3 — Aderência aos Guardrails

*Verifica se os 4 guardrails do projeto foram respeitados: (1) citar fonte, (2) não inventar valores/prazos, (3) dizer quando não souber, (4) português formal.*

| Nível | Critério |
|---|---|
| **3 — Total** | Todos os guardrails aplicáveis à pergunta foram respeitados sem exceção. |
| **2 — Parcial** | Um guardrail foi violado de forma leve (ex: linguagem informal pontual) ou a violação não causou dano direto à informação entregue. |
| **1 — Violação grave** | Inventou prazo, valor ou entidade inexistente (guardrail 2), ou respondeu com confiança quando deveria ter dito "não encontrei" (guardrail 3). |

---

### Dimensão 4 — Completude

*Verifica se a resposta abrange todas as informações necessárias para a pergunta feita, sem omissões que causem risco no atendimento.*

| Nível | Critério |
|---|---|
| **3 — Completa** | A resposta cobre todos os elementos relevantes para a pergunta, sem omissões que possam induzir erro no atendimento. |
| **2 — Incompleta** | Responde a parte da pergunta corretamente, mas omite informação que o atendente precisaria para agir corretamente. |
| **1 — Insuficiente** | A omissão é tão relevante que a resposta pode induzir ação errada, ou a pergunta não foi efetivamente respondida. |

---

## Entregável 3 — Template Reutilizável de Avaliação [modelo Claude Cowork]

> Template genérico para avaliação de qualquer resposta do assistente de IA da NovaTech. Não é específico para as 5 respostas deste exercício.

```
═══════════════════════════════════════════════════════════════
  TEMPLATE DE AVALIAÇÃO DE RESPOSTA — ASSISTENTE IA NOVATECH
  Pipeline RAG | Versão do template: 1.0 | Data: ___/___/______
═══════════════════════════════════════════════════════════════

IDENTIFICAÇÃO DA AVALIAÇÃO
  Avaliador:            _______________________________________
  ID do chamado/lote:   _______________________________________
  Data da avaliação:    _______________________________________
  Ambiente:             [ ] DEV   [ ] HMG   [ ] PRD

───────────────────────────────────────────────────────────────
  RESPOSTA AVALIADA
───────────────────────────────────────────────────────────────

  Pergunta feita pelo atendente:
  _______________________________________________________________

  Resposta gerada pelo assistente:
  _______________________________________________________________
  _______________________________________________________________

  Fonte(s) citada(s) pelo assistente:
  _______________________________________________________________

───────────────────────────────────────────────────────────────
  PONTUAÇÃO — marque de 1 a 3 em cada dimensão
───────────────────────────────────────────────────────────────

  D1 — Precisão Factual          [ 1 — Incorreto  ]
                                 [ 2 — Parcial    ]
                                 [ 3 — Correto    ]

  D2 — Citação de Fonte          [ 1 — Ausente/inválida ]
                                 [ 2 — Parcial          ]
                                 [ 3 — Completa/correta ]

  D3 — Aderência aos Guardrails  [ 1 — Violação grave ]
                                 [ 2 — Parcial        ]
                                 [ 3 — Total          ]

  D4 — Completude                [ 1 — Insuficiente ]
                                 [ 2 — Incompleta   ]
                                 [ 3 — Completa     ]

  PONTUAÇÃO TOTAL: _______ / 12

───────────────────────────────────────────────────────────────
  CLASSIFICAÇÃO FINAL
───────────────────────────────────────────────────────────────

  [ ] APROVADA     10 a 12 pontos — pode ser usada no atendimento
  [ ] REVISÃO       7 a 9 pontos  — usar com cautela, sinalizar
  [ ] REPROVADA    até 6 pontos   — não usar, acionar correção

───────────────────────────────────────────────────────────────
  JUSTIFICATIVA (obrigatória quando D1 ≤ 2 ou D3 = 1)
───────────────────────────────────────────────────────────────

  Descrição da falha identificada:
  _______________________________________________________________
  _______________________________________________________________

  Documento de referência usado para verificação:
  _______________________________________________________________

───────────────────────────────────────────────────────────────
  AÇÃO NECESSÁRIA
───────────────────────────────────────────────────────────────

  [ ] Nenhuma — resposta aprovada sem ressalvas
  [ ] Registrar como falha leve — monitorar frequência
  [ ] Ajuste no system prompt ou guardrail
  [ ] Atualização ou curadoria do documento-fonte
  [ ] Revisão da estratégia de chunking/retrieval
  [ ] Escalar para Tech Lead
  [ ] Outro: ________________________________________________

═══════════════════════════════════════════════════════════════
```

---

## Entregável 4 — Pontuações Aplicadas às 5 Respostas

| # | D1 Precisão | D2 Fonte | D3 Guardrails | D4 Completude | **Total** | Classificação |
|---|---|---|---|---|---|---|
| 1 | 2 | 2 | 2 | 2 | **8/12** | Revisão |
| 2 | 2 | 3 | 2 | 2 | **9/12** | Revisão |
| 3 | 1 | 1 | 1 | 1 | **4/12** | Reprovada |
| 4 | 1 | 2 | 1 | 1 | **5/12** | Reprovada |
| 5 | 3 | 3 | 3 | 3 | **12/12** | Aprovada |

### Detalhamento das reprovadas

**Resposta 3 — "Qual o SLA do cliente Platinum?"**
- D1 = 1: Tier Platinum não existe na NovaTech (SLA-2024, seção 1). Valores de SLA completamente fabricados.
- D2 = 1: Cita SLA-2024, mas essa fonte não contém nenhuma informação sobre Platinum — a fonte citada contradiz a própria resposta.
- D3 = 1: Violação grave do guardrail 2 (inventou valores) e guardrail 3 (deveria ter dito "tier não encontrado na base").
- D4 = 1: Responde uma premissa falsa. A informação correta seria: informar que o tier não existe e listar os 3 tiers disponíveis.
- **Ação:** Ajuste no system prompt para instruir o modelo a verificar se a entidade existe antes de atribuir propriedades a ela.

**Resposta 4 — "Posso devolver carga perigosa?"**
- D1 = 1: Contradição direta com POL-001, seção 3.2, que proíbe explicitamente a devolução de carga perigosa pelo processo padrão.
- D2 = 2: Cita POL-001 seção 3.2, que é o documento correto, mas o conteúdo da resposta contradiz exatamente o que essa seção diz.
- D3 = 1: Violação grave do guardrail 3 — respondeu com confiança afirmando algo que a documentação proíbe, sem indicar restrição ou escalonamento.
- D4 = 1: Além de incorreta, omite a orientação correta: contatar Gestão de Riscos no ramal 4500.
- **Ação:** Revisar chunking e retrieval para garantir que a seção 3.2 (exceções) seja recuperada junto com a seção 3.1 (regra geral), evitando contexto parcial.

### Resumo geral do lote

| Classificação | Qtd | % |
|---|---|---|
| Aprovada | 1 | 20% |
| Revisão | 2 | 40% |
| Reprovada | 2 | 40% |

> **Conclusão:** 40% de reprovação em um lote de 5 respostas indica risco operacional alto. As falhas concentram-se em alucinação (resposta 3) e inversão de regra crítica de segurança (resposta 4). Recomenda-se priorizar ajuste de system prompt e revisão da estratégia de retrieval antes de qualquer go-live.
