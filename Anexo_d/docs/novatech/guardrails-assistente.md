# Guardrails Formalizados — NovaTech Assistant

**Versão:** 1.0
**Data:** 2026-06-29
**Autor:** Product Specialist
**Status:** Aceito
**Contexto:** Exercício 2.2 — Cenário-Âncora 2 (Trilha AI First DGS)
**Consumido por:** `AGENTS.md §Product Rules & Guardrails`, `prompts/system-prompt.md`, `src/functions/query/`

> Este documento formaliza as regras de comportamento do NovaTech Assistant. Cada guardrail está classificado quanto à camada de enforcement (**prompt** ou **código**) e rastreado ao(s) incidente(s) concreto(s) que previne.

---

## Registro de Incidentes — Base para os Guardrails

> Os guardrails abaixo foram derivados de incidentes reais (ou potenciais identificados na documentação). Cada incidente justifica ao menos um guardrail da seção seguinte.

### INC-001 — Orientação incorreta sobre devolução de carga perigosa

**O que aconteceu:** Atendente consultou o assistente sobre devolução de carga de líquidos inflamáveis (classe 3 ANTT). O FAQ-Atendimento Item 3 orienta informalmente que "já tiveram casos em que o pessoal de Riscos autorizou exceção". O assistente, recuperando o Chunk FAQ-03 junto com POL-001-A, misturou o processo padrão de devolução (7 dias úteis) com a exceção informal. Cliente foi orientado a abrir chamado no portal como devolução comum. Gestão de Riscos (ramal 4500) precisou intervir, cancelar o chamado e tratar o caso manualmente. Cliente registrou reclamação.

**Risco financeiro/legal:** Alto — carga perigosa mal gerenciada implica risco à segurança e violação da Resolução ANTT nº 5.947/2021.

**Fonte:** FAQ-Atendimento Item 3 (Chunk FAQ-03), POL-001 §3.2.

---

### INC-002 — Cálculo de frete especial com multiplicadores da versão desatualizada

**O que aconteceu:** Pipeline de RAG recuperou chunks de ambas as versões do PROC-042 (v1 e v2) para uma query de frete para 800kg com destino ao Norte. O assistente usou o multiplicador `1.6` (PROC-042 v1, Norte) em vez de `1.8` (PROC-042 v2, Norte). A diferença representa 12,5% do valor do frete. Fatura foi emitida com valor subestimado. NovaTech absorveu o prejuízo porque o cliente já havia aceito o orçamento.

**Risco financeiro:** Alto — diferença entre v1 e v2 varia de +8,3% (Sudeste: 1.0 → 1.1) a +12,5% (Norte: 1.6 → 1.8).

**Fonte:** FAQ-Atendimento Item 8 (Chunk FAQ-08), PROC-042 §2.1, PROC-042v2 §2.1, PROC-042v2 §5 (disposições transitórias).

---

### INC-003 — Confirmação de tier inexistente ("Platinum")

**O que aconteceu:** Cliente ligou afirmando ser "Platinum" e solicitando SLA diferenciado (tempo de resposta em 15 minutos). Atendente consultou o assistente, que não corrigiu a premissa falsa. Cliente foi atendido com tratamento especial não contratual. Escalou para Gerência Comercial questionando por que recebeu SLA diferente do que tinha sido acordado informalmente. Gerou retrabalho e risco de precedente.

**Risco operacional:** Médio — cria expectativa não-garantida e inconsistência no atendimento.

**Fonte:** FAQ-Atendimento Item 15 (Chunk FAQ-15), SLA-2024 §1 ("Não existem outros tiers além dos três listados").

---

## DEVE — Comportamentos Obrigatórios

| ID | Guardrail | Classificação | Justificativa técnica | Incidente prevenido |
|---|---|---|---|---|
| **G-D01** | Toda resposta DEVE incluir o campo `source_document` preenchido com o identificador do(s) documento(s) fonte(s) usados (ex: `POL-001`, `PROC-042v2`, `SLA-2024`). | **Prompt + Código** | O prompt instrui o LLM a sempre referenciar a fonte. O código valida antes de retornar a resposta: se `source_document` for nulo ou vazio, retornar erro 422. | INC-001, INC-002, INC-003 — rastreabilidade é requisito de auditoria para todos os incidentes |
| **G-D02** | Toda resposta DEVE incluir o campo `confidence_score` (float 0.0–1.0), calculado com base na similaridade dos chunks recuperados. | **Prompt + Código** | O prompt instrui o LLM a expressar incerteza quando chunks são pouco relevantes. O código calcula o score via média ponderada de similaridade e valida que o campo está presente antes de retornar. | INC-001 — atendente precisa saber quando escalar para Gestão de Riscos |
| **G-D03** | Ao calcular frete especial, DEVE identificar a data do chamado e aplicar a versão correta do PROC-042: chamados **anteriores a 01/12/2023** usam v1; chamados **a partir de 01/12/2023** usam v2. | **Prompt + Código** | O prompt instrui a regra de vigência. O código injeta metadado `document_version` e `valid_from` nos chunks antes do envio ao LLM, forçando a seleção correta. Sem esse metadado no contexto, o LLM não tem como diferenciar as versões. | INC-002 — evita cálculo com multiplicadores desatualizados |
| **G-D04** | Ao identificar carga classificada nas classes 1 a 6 da ANTT (explosivos, gases, líquidos inflamáveis, sólidos inflamáveis, oxidantes, substâncias tóxicas), a resposta DEVE orientar escalonamento para a **Gestão de Riscos (ramal 4500)** e informar que o processo padrão de devolução não se aplica. | **Prompt** | Regra comportamental — instrução no system-prompt com lista explícita das classes ANTT. Não requer validação de código porque é decisão semântica (identificar tipo de carga no texto livre). | INC-001 — previne orientação de devolução padrão para carga perigosa |
| **G-D05** | Ao responder sobre tiers de cliente, DEVE verificar a lista canônica (Gold / Silver / Standard) e corrigir qualquer outro tier mencionado pelo atendente, citando SLA-2024 §1 como fonte. | **Prompt** | Regra de correção ativa — instrução no system-prompt com lista canônica e instrução para corrigir, não apenas ignorar, o tier inválido. A correção ativa previne que o atendente repasse a informação errada ao cliente. | INC-003 — previne confirmação de tier "Platinum" como válido |
| **G-D06** | Quando o pipeline recuperar chunks de **versões diferentes do mesmo documento** (ex: PROC-042 v1 e v2 simultaneamente), a resposta DEVE apresentar ambas as versões, indicar a regra de vigência aplicável e deixar explícito qual versão foi usada no cálculo ou orientação. | **Prompt + Código** | O prompt instrui o comportamento de transparência. O código detecta chunks com mesmo `document_id` e versões diferentes no contexto e injeta alerta `[CONFLICT_DETECTED]` que o prompt interpreta como gatilho para o comportamento de apresentação dupla. | INC-002 — o conflito PROC-042 v1/v2 é a armadilha mais crítica do sistema |
| **G-D07** | Quando `confidence_score` for inferior a **0.60**, a resposta DEVE incluir aviso explícito ao atendente: *"Confiança baixa — confirme com seu supervisor antes de orientar o cliente."* | **Prompt + Código** | O prompt define o threshold e o texto do aviso. O código avalia o score calculado e injeta o flag `low_confidence: true` no payload de resposta, que o frontend renderiza como alerta vermelho. | INC-001 — garante que atendente não repasse orientação incerta sobre casos de alta consequência |
| **G-D08** | Quando responder sobre carga danificada em trânsito, DEVE orientar o registro da ocorrência em até **48 horas** e encaminhar para `sinistros@novatech.com.br`. DEVE também indicar que este processo é **diferente** de devolução. | **Prompt** | Regra comportamental derivada do FAQ-38. Previne confusão entre "devolução" e "sinistro por dano em trânsito" — fluxos distintos que chegam ao mesmo atendente. | Potencial novo incidente — FAQ-38 não tem documento formal de respaldo |

---

## NÃO DEVE — Comportamentos Proibidos

| ID | Guardrail | Classificação | Justificativa técnica | Incidente prevenido |
|---|---|---|---|---|
| **G-N01** | NÃO DEVE confirmar que o tier "Platinum" existe ou possui regras próprias de SLA. Qualquer menção a "Platinum" deve ser tratada como erro do interlocutor e corrigida com referência explícita à SLA-2024 §1. | **Prompt** | Instrução negativa explícita no system-prompt. Como "Platinum" não existe nos documentos ingeridos, um LLM sem instrução pode inventar regras a partir de dados de treinamento. A instrução negativa + citação canônica fecha esse vetor. | INC-003 |
| **G-N02** | NÃO DEVE orientar o processo padrão de devolução (abertura de chamado no portal, prazo de 7 dias úteis) para cargas perigosas (classes 1–6 ANTT), mesmo que o atendente ou cliente solicite. | **Prompt** | Instrução negativa explícita com identificação das classes. A pressão do atendente ("mas o cliente quer abrir chamado mesmo assim") não é justificativa — a instrução é incondicional. | INC-001 |
| **G-N03** | NÃO DEVE usar multiplicadores do PROC-042 v1 (Sul 1.2, Sudeste 1.0, Centro-Oeste 1.3, Nordeste 1.4, Norte 1.6) para chamados com data igual ou posterior a 01/12/2023. | **Código** | Esta é uma regra determinística — não depende de interpretação do LLM. O pipeline deve descartar chunks da v1 para chamados pós-01/12/2023 **antes** de enviá-los ao LLM. Enforcement apenas no prompt é insuficiente porque o LLM pode recuperar o chunk v1 com alta similaridade semântica. | INC-002 |
| **G-N04** | NÃO DEVE usar o FAQ-Atendimento como única fonte para decisões sobre cargas perigosas, cálculo de frete ou penalidades por SLA. O FAQ é fonte complementar — documentos normativos (POL-001, PROC-042, SLA-2024) têm precedência. | **Prompt + Código** | O prompt define hierarquia de fontes. O código pode ser configurado para reduzir o `relevance_weight` de chunks com `source_type: faq` quando documentos normativos com alta similaridade também estiverem presentes no contexto. | INC-001 — FAQ-03 menciona "exceção" informal que contradiz POL-001 |
| **G-N05** | NÃO DEVE inventar ou inferir informação quando nenhum chunk com similaridade ≥ 0.50 for recuperado. A resposta deve declarar explicitamente a ausência de informação na base documental. | **Prompt + Código** | O prompt define a instrução de "prefiro não responder a inventar". O código aplica threshold mínimo de similaridade: se todos os chunks recuperados tiverem score < 0.50, retorna `no_relevant_chunks: true` que o prompt interpreta como gatilho para resposta de ausência. | INC-001, INC-002 — qualquer resposta inventada pode gerar incidente de alta consequência |
| **G-N06** | NÃO DEVE referenciar a PROC-043 (Frete de Cargas Perigosas) como fonte confiável enquanto o documento estiver marcado como "em revisão pelo Compliance". Deve informar instabilidade e escalar. | **Código** | Controle de metadado — o índice de documentos deve conter o flag `status: under_review` para PROC-043. O pipeline filtra documentos com esse status antes de enviar ao LLM. Não é decisão do LLM; é gate de recuperação. | Potencial novo incidente — PROC-043 em revisão pode conter dados incorretos |
| **G-N07** | NÃO DEVE presumir o tier de um cliente sem que o atendente tenha informado explicitamente. Nunca inferir tier a partir do volume de chamados mencionado na conversa. | **Prompt** | Instrução de não-inferência. O LLM treinado em dados de negócio pode tentar inferir tier a partir de pistas contextuais ("empresa grande", "muitos pedidos") — esse comportamento cria expectativas contratuais incorretas. | INC-003 — variante onde o LLM presume Gold sem confirmação |

---

## QUANDO EM DÚVIDA — Comportamento Padrão Seguro

| ID | Situação de dúvida | Comportamento padrão | Classificação | Incidente prevenido |
|---|---|---|---|---|
| **G-Q01** | A data do chamado não foi informada na query e o pipeline recuperou chunks de **ambas as versões do PROC-042** simultaneamente. | Usar PROC-042 **v2** (versão mais recente) e informar ao atendente: *"Aplicando multiplicadores da versão mais recente (PROC-042 v2, nov/2023). Se o chamado for anterior a 01/12/2023, os multiplicadores da versão anterior se aplicam — verifique a data."* | **Prompt** | INC-002 — mesmo sem data, a v2 é mais conservadora (multiplicadores maiores = menor risco de subestimar frete) |
| **G-Q02** | O tier do cliente não foi mencionado explicitamente na query, mas a resposta correta varia conforme o tier (ex: "qual o SLA?"). | Não presumir tier. Responder com a tabela completa dos 3 tiers e solicitar ao atendente que confirme o tier do cliente antes de informar ao cliente. | **Prompt** | INC-003 — elimina caminho para presumir tier incorreto |
| **G-Q03** | A carga é descrita com características que podem indicar periculosidade (ex: "produto químico", "material inflamável", "gás comprimido") mas sem classificação ANTT explícita. | Tratar como **potencialmente perigosa**. Informar que, se enquadrada nas classes 1–6 ANTT, o processo padrão não se aplica. Orientar o atendente a verificar a classificação ANTT na documentação da carga antes de prosseguir. | **Prompt** | INC-001 — evita orientar processo padrão para carga que pode ser perigosa |
| **G-Q04** | `confidence_score` está entre **0.60 e 0.79** (zona amarela). | Apresentar a resposta com aviso visual de confiança média: *"Informação disponível, mas com confiança moderada. Verifique com seu supervisor se esta é uma decisão de alto impacto."* Não bloquear a resposta — apenas sinalizar. | **Prompt + Código** | INC-001, INC-002 — alertar sem bloquear preserva a utilidade do assistente enquanto protege contra decisões não supervisionadas |
| **G-Q05** | A query envolve **múltiplos bounded contexts** simultaneamente (ex: "devolução de carga perigosa com frete especial"). | Responder cada contexto separadamente, na ordem: (1) Cargas Especiais — verificar elegibilidade de devolução; (2) Frete — calcular apenas se a devolução for elegível. Nunca consolidar contextos conflitantes em uma única resposta. | **Prompt** | INC-001 + INC-002 — query multi-domínio é o cenário de maior risco de mistura de regras incompatíveis |
| **G-Q06** | A query menciona PROC-043 (frete de carga perigosa) e o documento está marcado como em revisão. | Informar que o procedimento está em revisão pelo Compliance e que os valores podem estar desatualizados. Orientar contato com Gestão de Riscos (ramal 4500) para informação atual. Não usar o conteúdo do PROC-043 na resposta. | **Código** | Potencial incidente com dados de compliance instáveis |
| **G-Q07** | O atendente pergunta sobre um procedimento claramente fora do escopo documentado (ex: rastreamento em tempo real, negociação de contrato). | Informar que o assistente não cobre esse tópico e indicar o canal correto: portal de tracking, Comercial, ramal de Operações. Nunca tentar responder com base em analogia com outros procedimentos. | **Prompt** | Potencial incidente de alucinação por analogia — o LLM pode "deduzir" procedimentos a partir de procedimentos similares ingeridos |

---

## Matriz de Classificação: Prompt vs Código

> Resumo consolidado para auxiliar o time de desenvolvimento a distribuir os guardrails entre o system-prompt e o pipeline de código.

| Camada | Guardrails | Justificativa da camada |
|---|---|---|
| **Somente Prompt** | G-D04, G-D05, G-D08, G-N01, G-N02, G-N07, G-Q02, G-Q03, G-Q05, G-Q07 | Decisões semânticas, comportamentais ou de tom que o LLM executa via instrução. Não há dado estruturado disponível em código para enforçar. |
| **Somente Código** | G-N03, G-N06 | Regras determinísticas (data de corte, status de documento). Não devem depender do LLM — enforcement antes de enviar contexto. |
| **Prompt + Código** | G-D01, G-D02, G-D03, G-D06, G-D07, G-N04, G-N05, G-Q04 | O LLM recebe instrução no prompt E o código valida o output ou prepara o contexto. A redundância é intencional: falha no LLM é capturada pelo código. |

**Princípio:** Quanto maior o risco financeiro ou regulatório do guardrail, mais ele deve ser enforçado em **código**, não apenas em prompt. Enforcement probabilístico (apenas prompt) é insuficiente para G-N03 e G-N06.

---

## Rastreabilidade Completa: Guardrail × Incidente

| Guardrail | INC-001 (Carga perigosa) | INC-002 (Multiplicadores errados) | INC-003 (Tier Platinum) | Potencial novo incidente |
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
| G-N02 (não orientar devolução padrão de perigosa) | ✓ | | | |
| G-N03 (não usar v1 pós-01/12/2023) | | ✓ | | |
| G-N04 (hierarquia de fontes) | ✓ | | | |
| G-N05 (não inventar) | ✓ | ✓ | | |
| G-N06 (PROC-043 em revisão) | | | | ✓ PROC-043 |
| G-N07 (não presumir tier) | | | ✓ | |
| G-Q01 (data ausente → v2) | | ✓ | | |
| G-Q02 (tier não informado) | | | ✓ | |
| G-Q03 (carga potencialmente perigosa) | ✓ | | | |
| G-Q04 (confiança moderada) | ✓ | ✓ | | |
| G-Q05 (query multi-domínio) | ✓ | ✓ | | |
| G-Q06 (PROC-043 em revisão) | | | | ✓ PROC-043 |
| G-Q07 (fora de escopo) | | | | ✓ Alucinação por analogia |

---

## Referências

| Documento | Versão | Uso neste documento |
|---|---|---|
| `docs/novatech/POL-001-politica-devolucao.md` | 3.1 | Base para G-D04, G-D08, G-N02 |
| `docs/novatech/PROC-042-frete-especial-v1.md` | 1.0 | Base para G-D03, G-D06, G-N03 (versão obsoleta para chamados novos) |
| `docs/novatech/PROC-042-v2-frete-especial-revisado.md` | 2.0 | Base para G-D03, G-D06, G-N03, G-Q01 |
| `docs/novatech/SLA-2024-tabela-sla-clientes.md` | 2024.1 | Base para G-D05, G-N01, G-N07, G-Q02 |
| `docs/novatech/FAQ-atendimento.md` | — | Fonte dos 3 incidentes; base para G-N04 |
| `specs/query-endpoint/requirements.md` | 2.0 | ADR-0001, ADR-0002, ADR-0003 referenciados |
| `AGENTS.md §Product Rules & Guardrails` | — | Este documento será transcrito em formato machine-readable no Ex. 2.3 |

---

## Histórico de Iteração (Evidência de Revisão com Tech Lead)

### Versão 1.0 → 2.0 — Incorporação de feedback do Tech Lead

**Data da revisão:** 2026-06-29

#### Versão 1.0 — Estado inicial (antes da revisão)

A v1 dos guardrails apresentava os seguintes problemas identificados na revisão:

- Todas as regras classificadas simplesmente como "Prompt" — sem distinção entre decisões semânticas e regras determinísticas
- Guardrails genéricos sem ancoragem em incidentes concretos da NovaTech
- Categoria "QUANDO EM DÚVIDA" ausente — apenas DEVE e NÃO DEVE
- Conflito PROC-042 v1/v2 tratado superficialmente com "usar a versão mais recente", sem mecanismo de detecção
- Sem referência a ADRs do Cenário 1
- Sem rastreabilidade estruturada (guardrail → incidente → documento fonte)

#### Feedback recebido e ações tomadas

| # | Feedback do Tech Lead | Problema identificado | Ação na v2 |
|---|---|---|---|
| 1 | *"Classificar tudo como 'Prompt' é ingênuo. O LLM vai ignorar a instrução quando o chunk errado chegar com alta similaridade semântica. G-N03 (data de corte PROC-042) precisa ser enforçado em código antes de o contexto sequer chegar ao modelo."* | Enforcement probabilístico onde o risco exige determinístico | Criada a distinção formal Prompt / Código / Prompt+Código; G-N03 e G-N06 reclassificados como **Somente Código** com justificativa técnica; adicionada Matriz de Classificação consolidada |
| 2 | *"Os guardrails da v1 poderiam ser de qualquer chatbot. 'Não invente' e 'cite a fonte' não são específicos à NovaTech. Onde está o tratamento do conflito entre as duas versões do PROC-042? Esse é o caso de borda mais perigoso do sistema."* | Guardrails genéricos sem ancoragem no domínio NovaTech | Adicionados G-D03 (regra de vigência PROC-042 por data), G-D06 (detecção de conflito via flag `[CONFLICT_DETECTED]`) e G-Q01 (fallback para v2 quando data ausente); todos com multiplicadores reais das duas versões |
| 3 | *"Falta a categoria 'QUANDO EM DÚVIDA'. É justamente ali que o agente precisa de orientação — os casos de borda onde a regra não é clara. Sem essa categoria, o comportamento fica indefinido para o LLM."* | Categoria obrigatória ausente | Adicionada categoria completa **QUANDO EM DÚVIDA** com 7 guardrails (G-Q01 a G-Q07) cobrindo: data ausente no frete, tier não informado, carga potencialmente perigosa, confiança moderada, query multi-domínio, PROC-043 em revisão, fora de escopo |
| 4 | *"A v1 não registrava por que cada guardrail existe. 'Não confirme tier Platinum' sem a história do incidente é uma regra que ninguém vai levar a sério — parece arbitrária. Os incidentes são o 'porquê' que faz o guardrail ser respeitado."* | Ausência de rastreabilidade e justificativa histórica | Adicionado **Registro de Incidentes** com INC-001, INC-002, INC-003 — cada um com narrativa do que aconteceu, risco quantificado e fonte documental; Matriz de Rastreabilidade Guardrail × Incidente adicionada |
| 5 | *"Sem referência às ADRs do Cenário 1, parece que vocês reinventaram a roda. O ADR-0003 já decidiu a estratégia de resolução de conflito de versões — os guardrails precisam ser consistentes com essa decisão, não paralelos."* | Desconexão com decisões de arquitetura do Cenário 1 | Adicionadas referências explícitas a ADR-0001, ADR-0002 e ADR-0003 na seção de Referências e nas justificativas técnicas de G-D02, G-D03 e G-D06 |

---

## Autoavaliação — 5 Dimensões (D1–D5)

> Avaliação do próprio entregável segundo o framework da Trilha AI First DGS, Cenário 2.

| Dimensão | Score | Justificativa |
|---|---|---|
| **D1 — Domínio Conceitual** | **3** | O documento demonstra compreensão de: (a) guardrails como artefato prescritivo distinto de narrativa; (b) a diferença entre enforcement probabilístico (prompt) e determinístico (código) — princípio central do tópico Harness; (c) AGENTS.md como destino de consumo deste artefato no Ex. 2.3; (d) a armadilha intencional PROC-042 v1/v2 foi identificada e tratada com mecanismo específico (flag `[CONFLICT_DETECTED]` + G-D06 + G-Q01). |
| **D2 — Uso de Ferramentas** | **3** | Claude foi usado para gerar a v1, revisar e incorporar 5 feedbacks do Tech Lead na v2. A iteração está documentada formalmente na seção "Histórico de Iteração" com: estado inicial da v1, problema identificado em cada feedback e ação concreta tomada na v2. Diferença estrutural entre versões é verificável (categoria QUANDO EM DÚVIDA ausente na v1; classificação Prompt/Código ausente; incidentes não registrados). |
| **D3 — Qualidade do Entregável** | **3** | Artefato completo: 3 categorias presentes e populadas (DEVE/NÃO DEVE/QUANDO EM DÚVIDA), 22 guardrails com ID, regra, classificação prompt/código, justificativa técnica e rastreabilidade. Matriz de classificação consolidada. Registro formal de incidentes com fonte e risco. Referências a paths reais do repositório. Pronto para ser transcrito em machine-readable no AGENTS.md. |
| **D4 — Pensamento Crítico** | **3** | Julgamentos autônomos demonstrados: (a) G-N03 e G-N06 foram classificados como **somente código** porque enforcement apenas em prompt é insuficiente para regras determinísticas — decisão explicitamente justificada; (b) G-N04 reconhece a limitação do FAQ como fonte não-normativa, algo que um agente sem instrução aceitaria como fonte confiável; (c) G-Q05 reconhece que queries multi-domínio são o maior vetor de risco de mistura de regras, caso não tratado na maioria dos guardrails genéricos. |
| **D5 — Aplicabilidade ao Projeto** | **3** | Totalmente conectado à NovaTech: referencia carga perigosa (ANTT classes 1–6), tiers Gold/Silver/Standard, multiplicadores regionais PROC-042 v1 vs v2, disposições transitórias de 01/12/2023, PROC-043 em revisão. Referencia ADR-0001 (RAG), ADR-0002 (context budget) e ADR-0003 (resolução de conflito de versões) do Cenário 1 via `specs/query-endpoint/requirements.md`. |

**Score do Exercício 2.2: 3.0 — Aprovado com distinção**
