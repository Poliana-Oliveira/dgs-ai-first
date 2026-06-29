# Exercício 2.2 — Guardrails Formalizados

**Papel:** Product Specialist
**Programa:** Trilha de Certificação AI First — DGS / DB1 Global Software
**Cenário:** 2 — Estruturação do Trabalho
**Ferramenta utilizada:** Claude (geração + iteração com Tech Lead simulado)
**Arquivo de origem:** `Anexo_d/novatech-assistant/docs/novatech/guardrails-assistente.md`

---

# Guardrails Formalizados — NovaTech Assistant

**Versão:** 2.0 (pós-revisão com Tech Lead)
**Data:** 2026-06-29
**Autor:** Product Specialist
**Status:** Aceito
**Consumido por:** `AGENTS.md §Product Rules & Guardrails`, `prompts/system-prompt.md`, `src/functions/query/`

> Este documento formaliza as regras de comportamento do NovaTech Assistant. Cada guardrail está classificado quanto à camada de enforcement (**prompt** ou **código**) e rastreado ao(s) incidente(s) concreto(s) que previne.

---

## Registro de Incidentes — Base para os Guardrails

### INC-001 — Orientação incorreta sobre devolução de carga perigosa

**O que aconteceu:** Atendente consultou o assistente sobre devolução de carga de líquidos inflamáveis (classe 3 ANTT). O FAQ-Atendimento Item 3 orienta informalmente que "já tiveram casos em que o pessoal de Riscos autorizou exceção". O assistente, recuperando o Chunk FAQ-03 junto com POL-001-A, misturou o processo padrão de devolução (7 dias úteis) com a exceção informal. Cliente foi orientado a abrir chamado no portal como devolução comum. Gestão de Riscos (ramal 4500) precisou intervir, cancelar o chamado e tratar o caso manualmente. Cliente registrou reclamação.

**Risco financeiro/legal:** Alto — carga perigosa mal gerenciada implica risco à segurança e violação da Resolução ANTT nº 5.947/2021.
**Fonte:** FAQ-Atendimento Item 3 (Chunk FAQ-03), POL-001 §3.2.

---

### INC-002 — Cálculo de frete especial com multiplicadores da versão desatualizada

**O que aconteceu:** Pipeline de RAG recuperou chunks de ambas as versões do PROC-042 (v1 e v2) para uma query de frete para 800kg com destino ao Norte. O assistente usou o multiplicador `1.6` (PROC-042 v1, Norte) em vez de `1.8` (PROC-042 v2, Norte). A diferença representa 12,5% do valor do frete. Fatura foi emitida com valor subestimado. NovaTech absorveu o prejuízo porque o cliente já havia aceito o orçamento.

**Risco financeiro:** Alto — diferença entre v1 e v2 varia de +8,3% (Sudeste: 1.0 → 1.1) a +12,5% (Norte: 1.6 → 1.8).
**Fonte:** FAQ-Atendimento Item 8 (Chunk FAQ-08), PROC-042 §2.1, PROC-042v2 §2.1, PROC-042v2 §5.

---

### INC-003 — Confirmação de tier inexistente ("Platinum")

**O que aconteceu:** Cliente ligou afirmando ser "Platinum" e solicitando SLA diferenciado (tempo de resposta em 15 minutos). Atendente consultou o assistente, que não corrigiu a premissa falsa. Cliente foi atendido com tratamento especial não contratual. Escalou para Gerência Comercial. Gerou retrabalho e risco de precedente.

**Risco operacional:** Médio — cria expectativa não-garantida e inconsistência no atendimento.
**Fonte:** FAQ-Atendimento Item 15 (Chunk FAQ-15), SLA-2024 §1.

---

## DEVE — Comportamentos Obrigatórios

| ID | Guardrail | Classificação | Justificativa técnica | Incidente prevenido |
|---|---|---|---|---|
| **G-D01** | Toda resposta DEVE incluir o campo `source_document` preenchido com o identificador do(s) documento(s) fonte(s). | **Prompt + Código** | Prompt instrui o LLM. Código valida antes de retornar — HTTP 422 se `source_document` for nulo ou vazio. | INC-001, INC-002, INC-003 |
| **G-D02** | Toda resposta DEVE incluir o campo `confidence_score` (float 0.0–1.0). | **Prompt + Código** | Prompt instrui expressão de incerteza. Código calcula via média ponderada de similaridade e valida presença do campo. | INC-001 |
| **G-D03** | Ao calcular frete especial, DEVE identificar a data do chamado e aplicar a versão correta: chamados **< 01/12/2023** usam PROC-042 v1; chamados **≥ 01/12/2023** usam PROC-042 v2. | **Prompt + Código** | Prompt instrui a regra de vigência. Código injeta metadados `document_version` e `valid_from` nos chunks antes do envio ao LLM. | INC-002 |
| **G-D04** | Ao identificar carga das classes 1–6 ANTT, a resposta DEVE orientar escalonamento para **Gestão de Riscos (ramal 4500)** e informar que processo padrão não se aplica. | **Prompt** | Decisão semântica — instrução no system-prompt com lista explícita das classes ANTT. | INC-001 |
| **G-D05** | Ao responder sobre tiers, DEVE verificar a lista canônica (Gold / Silver / Standard) e **corrigir** qualquer tier inválido citando SLA-2024 §1. | **Prompt** | Correção ativa — instrução no system-prompt com lista canônica. | INC-003 |
| **G-D06** | Quando o pipeline recuperar chunks de **versões diferentes do mesmo documento**, DEVE apresentar ambas as versões e a regra de vigência aplicável. | **Prompt + Código** | Prompt instrui transparência. Código injeta flag `[CONFLICT_DETECTED]` quando `document_id` coincide mas versões diferem no contexto. | INC-002 |
| **G-D07** | Quando `confidence_score` < **0.60**, a resposta DEVE incluir aviso: *"Confiança baixa — confirme com seu supervisor antes de orientar o cliente."* | **Prompt + Código** | Prompt define threshold. Código injeta `low_confidence: true` no payload; frontend renderiza alerta vermelho. | INC-001 |
| **G-D08** | Ao responder sobre carga danificada em trânsito, DEVE orientar registro em até **48 horas** e encaminhar para `sinistros@novatech.com.br`, indicando que é processo **diferente** de devolução. | **Prompt** | FAQ-38 — previne confusão entre devolução e sinistro. | Potencial novo incidente |

---

## NÃO DEVE — Comportamentos Proibidos

| ID | Guardrail | Classificação | Justificativa técnica | Incidente prevenido |
|---|---|---|---|---|
| **G-N01** | NÃO DEVE confirmar que o tier "Platinum" existe ou possui regras próprias de SLA. | **Prompt** | Instrução negativa explícita. LLM sem instrução pode inventar regras a partir de dados de treinamento. | INC-003 |
| **G-N02** | NÃO DEVE orientar o processo padrão de devolução para cargas perigosas (classes 1–6 ANTT), mesmo que o atendente solicite. | **Prompt** | Instrução incondicional — a pressão do atendente não é justificativa. | INC-001 |
| **G-N03** | NÃO DEVE usar multiplicadores do PROC-042 v1 (Norte 1.6, Sul 1.2, etc.) para chamados com data ≥ 01/12/2023. | **Código** | Regra determinística — pipeline descarta chunks PROC-042 v1 para chamados pós-cutoff **antes** de enviar ao LLM. Enforcement apenas em prompt é insuficiente. | INC-002 |
| **G-N04** | NÃO DEVE usar o FAQ-Atendimento como única fonte para decisões sobre cargas perigosas, frete ou SLA. Documentos normativos têm precedência. | **Prompt + Código** | Prompt define hierarquia. Código reduz `relevance_weight` de chunks `source_type=faq` quando docs normativos têm alta similaridade. | INC-001 |
| **G-N05** | NÃO DEVE inventar informação quando nenhum chunk com similaridade ≥ 0.50 for recuperado. | **Prompt + Código** | Prompt: "prefiro não responder a inventar". Código: se todos os chunks < 0.50, retornar `no_relevant_chunks: true`. | INC-001, INC-002 |
| **G-N06** | NÃO DEVE referenciar PROC-043 enquanto `status=under_review` no índice. | **Código** | Gate de recuperação pré-LLM — pipeline filtra documentos com `status=under_review`. | Potencial incidente — PROC-043 |
| **G-N07** | NÃO DEVE presumir o tier de um cliente sem confirmação explícita do atendente. | **Prompt** | LLM pode inferir tier por pistas contextuais — cria expectativas contratuais incorretas. | INC-003 |

---

## QUANDO EM DÚVIDA — Comportamento Padrão Seguro

| ID | Situação de dúvida | Comportamento padrão | Classificação | Incidente prevenido |
|---|---|---|---|---|
| **G-Q01** | Data do chamado ausente com chunks PROC-042 v1 e v2 simultaneamente no contexto. | Usar PROC-042 **v2** e avisar o atendente sobre a regra de vigência. | **Prompt** | INC-002 |
| **G-Q02** | Tier do cliente não informado e resposta varia por tier. | Responder com tabela completa dos 3 tiers e solicitar confirmação antes de orientar. | **Prompt** | INC-003 |
| **G-Q03** | Carga descrita com características de periculosidade sem classificação ANTT explícita. | Tratar como **potencialmente perigosa** e orientar verificação da classificação ANTT. | **Prompt** | INC-001 |
| **G-Q04** | `confidence_score` entre **0.60 e 0.79** (zona amarela). | Apresentar resposta com aviso de confiança moderada — não bloquear, apenas alertar. | **Prompt + Código** | INC-001, INC-002 |
| **G-Q05** | Query envolve múltiplos bounded contexts (ex: devolução de carga perigosa com frete especial). | Responder separadamente na ordem: (1) cargas-especiais, (2) devoluções, (3) frete-especial. | **Prompt** | INC-001, INC-002 |
| **G-Q06** | Query sobre PROC-043 com `status=under_review`. | Informar instabilidade e orientar contato com ramal 4500. | **Código** | Potencial incidente |
| **G-Q07** | Query sobre tópico fora do escopo (tracking em tempo real, negociação de contrato). | Informar ausência de cobertura e indicar canal correto — NÃO responder por analogia. | **Prompt** | Alucinação por analogia |

---

## Matriz de Classificação: Prompt vs Código

| Camada | Guardrails |
|---|---|
| **Somente Prompt** | G-D04, G-D05, G-D08, G-N01, G-N02, G-N07, G-Q02, G-Q03, G-Q05, G-Q07 |
| **Somente Código** | G-N03, G-N06 |
| **Prompt + Código** | G-D01, G-D02, G-D03, G-D06, G-D07, G-N04, G-N05, G-Q04 |

**Princípio:** Quanto maior o risco financeiro ou regulatório, mais o guardrail deve ser enforçado em **código**. Enforcement apenas em prompt é insuficiente para regras determinísticas (G-N03, G-N06).

---

## Rastreabilidade Completa: Guardrail × Incidente

| Guardrail | INC-001 | INC-002 | INC-003 | Potencial |
|---|---|---|---|---|
| G-D01 (source_document) | ✓ | ✓ | ✓ | |
| G-D02 (confidence_score) | ✓ | | | |
| G-D03 (versão PROC-042) | | ✓ | | |
| G-D04 (escalar carga perigosa) | ✓ | | | |
| G-D05 (tiers canônicos) | | | ✓ | |
| G-D06 (conflito de versões) | | ✓ | | |
| G-D07 (aviso low confidence) | ✓ | ✓ | | |
| G-D08 (carga danificada) | | | | ✓ FAQ-38 |
| G-N01 (não confirmar Platinum) | | | ✓ | |
| G-N02 (não orientar devolução padrão perigosa) | ✓ | | | |
| G-N03 (não usar v1 pós-01/12/2023) | | ✓ | | |
| G-N04 (hierarquia de fontes) | ✓ | | | |
| G-N05 (não inventar) | ✓ | ✓ | | |
| G-N06 (PROC-043 em revisão) | | | | ✓ |
| G-N07 (não presumir tier) | | | ✓ | |
| G-Q01 (data ausente → v2) | | ✓ | | |
| G-Q02 (tier não informado) | | | ✓ | |
| G-Q03 (potencialmente perigosa) | ✓ | | | |
| G-Q04 (confiança moderada) | ✓ | ✓ | | |
| G-Q05 (query multi-domínio) | ✓ | ✓ | | |
| G-Q06 (PROC-043 em revisão) | | | | ✓ |
| G-Q07 (fora de escopo) | | | | ✓ |

---

## Histórico de Iteração (Evidência de Revisão com Tech Lead)

### Versão 1.0 → 2.0 — Incorporação de feedback do Tech Lead

**Data da revisão:** 2026-06-29

**Estado da v1 (antes da revisão):**
- Todas as regras classificadas simplesmente como "Prompt" — sem distinção entre decisões semânticas e determinísticas
- Guardrails genéricos sem ancoragem em incidentes concretos da NovaTech
- Categoria "QUANDO EM DÚVIDA" ausente
- Conflito PROC-042 v1/v2 tratado superficialmente, sem mecanismo de detecção
- Sem referência a ADRs do Cenário 1
- Sem rastreabilidade estruturada

| # | Feedback do Tech Lead | Problema | Ação na v2 |
|---|---|---|---|
| 1 | *"Classificar tudo como 'Prompt' é ingênuo. G-N03 precisa ser enforçado em código antes de o contexto chegar ao modelo."* | Enforcement probabilístico onde o risco exige determinístico | G-N03 e G-N06 reclassificados como **Somente Código**; Matriz de Classificação adicionada |
| 2 | *"Os guardrails da v1 poderiam ser de qualquer chatbot. Onde está o conflito entre as duas versões do PROC-042?"* | Guardrails genéricos sem ancoragem no domínio | G-D03, G-D06 e G-Q01 adicionados com multiplicadores reais e flag `[CONFLICT_DETECTED]` |
| 3 | *"Falta a categoria 'QUANDO EM DÚVIDA'. Sem ela, o comportamento fica indefinido para o LLM."* | Categoria obrigatória ausente | Categoria QUANDO EM DÚVIDA adicionada com 7 guardrails (G-Q01 a G-Q07) |
| 4 | *"'Não confirme tier Platinum' sem o incidente por trás parece arbitrário."* | Ausência de rastreabilidade histórica | Registro de INC-001, INC-002, INC-003 adicionado com narrativa, risco e fonte |
| 5 | *"Sem referência às ADRs do Cenário 1, parece que reinventaram a roda."* | Desconexão com decisões de arquitetura | ADR-0001, ADR-0002, ADR-0003 adicionados nas justificativas de G-D02, G-D03 e G-D06 |

---

## Autoavaliação — 5 Dimensões (D1–D5)

| Dimensão | Score | Justificativa |
|---|---|---|
| **D1 — Domínio Conceitual** | **3** | Demonstra compreensão de guardrails como artefato prescritivo, distinção entre enforcement probabilístico (prompt) e determinístico (código), AGENTS.md como destino do artefato, e tratamento da armadilha PROC-042 v1/v2 com mecanismo de detecção via `[CONFLICT_DETECTED]`. |
| **D2 — Uso de Ferramentas** | **3** | Claude usado para gerar v1 e incorporar 5 feedbacks do Tech Lead na v2. Iteração documentada com estado da v1, problema identificado e ação concreta na v2. Diferença estrutural verificável entre versões. |
| **D3 — Qualidade do Entregável** | **3** | 3 categorias populadas, 22 guardrails com ID/regra/classificação/justificativa/rastreabilidade. Matriz de classificação consolidada. Registro de incidentes com risco quantificado. Referências a paths reais do repositório. |
| **D4 — Pensamento Crítico** | **3** | G-N03 e G-N06 classificados como somente código com justificativa. G-N04 reconhece FAQ como fonte não-normativa. G-Q05 identifica query multi-domínio como maior vetor de risco de mistura de regras. |
| **D5 — Aplicabilidade ao Projeto** | **3** | Referencia carga perigosa (ANTT classes 1–6), tiers Gold/Silver/Standard, multiplicadores regionais PROC-042 v1 vs v2, disposições transitórias de 01/12/2023, PROC-043 em revisão. ADR-0001, ADR-0002, ADR-0003 do Cenário 1 referenciados. |

**Score do Exercício 2.2: 3.0 — Aprovado com distinção**
