# AGENTS.md — NovaTech Assistant

> Constitution do projeto. Todo agente de IA (Copilot, Claude Code) lê este arquivo antes de gerar qualquer artefato.
> As seções abaixo são preenchidas por papéis diferentes nos exercícios do Cenário 2.

## Project Overview
<!-- TODO (Tech Lead — Ex. 2.1) -->

## Tech Stack & Architecture
<!-- TODO (Tech Lead — Ex. 2.1): inclui regras de gerenciamento de contexto da ADR-0002 -->

## Coding Standards (Tech Lead)
<!-- TODO (Tech Lead — Ex. 2.1) -->

## Product Rules & Guardrails (Product Specialist)

<!--
  EXERCÍCIO 2.3 — Cenário-Âncora 2 (Trilha AI First DGS)
  Autor: Product Specialist | Versão: 1.0 | Data: 2026-06-29
  Fonte: docs/novatech/guardrails-assistente.md
  Spec:  specs/query-endpoint/requirements.md
-->

### bounded-contexts

```yaml
contexts:
  - id: atendimento
    description: Consultas sobre SLA, tiers de cliente e incidentes críticos
    source_documents:
      - docs/novatech/SLA-2024-tabela-sla-clientes.md
    tiers_canonicos: [Gold, Silver, Standard]
    tiers_invalidos: [Platinum, Bronze, Diamond]

  - id: frete-especial
    description: Cálculo de frete para cargas acima de 500kg
    source_documents:
      - docs/novatech/PROC-042-frete-especial-v1.md        # válido apenas para chamados < 2023-12-01
      - docs/novatech/PROC-042-v2-frete-especial-revisado.md  # vigente para chamados >= 2023-12-01
    version_cutoff: "2023-12-01"
    active_version: "PROC-042v2"

  - id: devolucoes
    description: Prazos, elegibilidade, custos e procedimento de coleta reversa
    source_documents:
      - docs/novatech/POL-001-politica-devolucao.md
    prazo_padrao_dias_uteis: 7

  - id: cargas-especiais
    description: Cargas perigosas (ANTT classes 1-6), refrigeradas e lacre violado
    source_documents:
      - docs/novatech/POL-001-politica-devolucao.md  # §3.2
    escalation_contact: "Gestão de Riscos — ramal 4500"
    antt_classes_perigosas: [1, 2, 3, 4, 5, 6]
```

---

### glossary

```yaml
# Termos que um LLM confundiria sem esta definição.
# Fonte: specs/query-endpoint/requirements.md §2 + docs/novatech/

terms:
  frete-especial:
    definition: "Modalidade de frete aplicável EXCLUSIVAMENTE a cargas com peso > 500kg"
    formula: "Valor base × multiplicador_regional × fator_peso"
    source: "docs/novatech/PROC-042-v2-frete-especial-revisado.md §2"

  carga-perigosa:
    definition: "Carga classificada nas classes 1 a 6 da ANTT (Resolução 5.947/2021)"
    classes:
      1: explosivos
      2: gases
      3: liquidos-inflamaveis
      4: solidos-inflamaveis
      5: oxidantes-e-peroxidos
      6: substancias-toxicas-e-infectantes
    source: "docs/novatech/POL-001-politica-devolucao.md §3.2"

  tier-Gold:
    definition: "Cliente com contrato anual > R$500.000 OU > 200 operações/mês"
    sla_chamado_geral_resposta: "2h úteis"
    sla_incidente_critico_resposta: "30min"
    source: "docs/novatech/SLA-2024-tabela-sla-clientes.md §1-2"

  tier-Silver:
    definition: "Cliente com contrato entre R$100.000 e R$500.000 OU 50-200 operações/mês"
    sla_chamado_geral_resposta: "4h úteis"
    sla_incidente_critico_resposta: "1h"
    source: "docs/novatech/SLA-2024-tabela-sla-clientes.md §1-2"

  tier-Standard:
    definition: "Todos os demais clientes (critério residual)"
    sla_chamado_geral_resposta: "8h úteis"
    sla_incidente_critico_resposta: "2h"
    source: "docs/novatech/SLA-2024-tabela-sla-clientes.md §1-2"

  tier-Platinum:
    definition: "TIER INEXISTENTE. Programa descontinuado em 2022."
    action: "Corrigir o interlocutor e referenciar SLA-2024 §1"
    source: "docs/novatech/FAQ-atendimento.md item 15"

  incidente-critico:
    definition: "Chamado que atende a AO MENOS UM dos critérios abaixo"
    criterios:
      - "Carga com valor > R$100.000 com status desconhecido há > 6h"
      - "Carga perigosa com qualquer irregularidade de documentação ou rastreamento"
      - "Mais de 5 chamados do mesmo cliente em 24h sobre o mesmo problema"
      - "Qualquer situação com risco à segurança de pessoas"
    source: "docs/novatech/SLA-2024-tabela-sla-clientes.md §3"

  multiplicador-regional:
    definition: "Fator numérico por região de destino aplicado no cálculo de frete especial"
    version_vigente: "PROC-042v2 (chamados >= 2023-12-01)"
    valores_v2:
      Sul: 1.3
      Sudeste: 1.1
      Centro-Oeste: 1.4
      Nordeste: 1.5
      Norte: 1.8
    valores_v1_obsoletos:   # usar apenas para chamados anteriores a 2023-12-01
      Sul: 1.2
      Sudeste: 1.0
      Centro-Oeste: 1.3
      Nordeste: 1.4
      Norte: 1.6
    source: "docs/novatech/PROC-042-v2-frete-especial-revisado.md §2.1"

  fator-peso:
    definition: "Multiplicador aplicado sobre o valor base conforme a faixa de peso da carga"
    version_vigente: "PROC-042v2"
    valores_v2:
      "500kg-1000kg": 1.0
      "1001kg-3000kg": 1.15
      ">3000kg": 1.4
    source: "docs/novatech/PROC-042-v2-frete-especial-revisado.md §2"

  CT-e:
    definition: "Conhecimento de Transporte Eletrônico — documento OBRIGATÓRIO para abrir chamado de devolução"
    source: "docs/novatech/POL-001-politica-devolucao.md §3.3"

  coleta-reversa:
    definition: "Recolhimento de mercadoria devolvida no endereço do cliente"
    prazo_apos_aprovacao_dias_uteis: 2
    source: "docs/novatech/POL-001-politica-devolucao.md §3.3"

  cadeia-de-frio:
    definition: "Controle de temperatura em cargas refrigeradas"
    ruptura: "Temperatura fora da faixa por > 30 minutos contínuos (sensor IoT)"
    impacto: "Carga com ruptura NÃO é elegível para devolução padrão"
    source: "docs/novatech/POL-001-politica-devolucao.md §3.2"

  source-document:
    definition: "Campo obrigatório em toda resposta — identifica o(s) documento(s) fonte(s)"
    exemplos: ["POL-001", "PROC-042v2", "SLA-2024"]
    constraint: "CÓDIGO — retornar HTTP 422 se ausente ou vazio"

  confidence-score:
    definition: "Float 0.0–1.0 indicando a confiança do assistente na resposta"
    thresholds:
      alta: ">= 0.80 — resposta direta"
      media: "0.60–0.79 — aviso amarelo ao atendente"
      baixa: "< 0.60 — aviso vermelho, confirmação com supervisor obrigatória"
    constraint: "CÓDIGO — campo obrigatório; calcular via similaridade média dos chunks"
```

---

### must-do

```yaml
# Comportamentos OBRIGATÓRIOS do assistente.
# Ref: docs/novatech/guardrails-assistente.md §DEVE

guardrails:
  - id: G-D01
    rule: "Toda resposta DEVE incluir campo `source_document` não-vazio"
    enforcement: prompt+code
    code_constraint: "Validar source_document antes de retornar — HTTP 422 se ausente"
    prevents: [INC-001, INC-002, INC-003]

  - id: G-D02
    rule: "Toda resposta DEVE incluir campo `confidence_score` (float 0.0–1.0)"
    enforcement: prompt+code
    code_constraint: "Calcular via média ponderada de similaridade dos chunks; campo obrigatório no schema de resposta"
    prevents: [INC-001]

  - id: G-D03
    rule: "Cálculo de frete especial DEVE usar PROC-042v2 para chamados >= 2023-12-01 e PROC-042v1 para chamados < 2023-12-01"
    enforcement: prompt+code
    code_constraint: "Pipeline injeta metadados `document_version` e `valid_from` nos chunks antes de enviar ao LLM"
    prevents: [INC-002]
    adr: ADR-0003

  - id: G-D04
    rule: "Carga perigosa (ANTT classes 1-6) DEVE ter resposta com escalonamento para ramal 4500 — processo padrão NÃO se aplica"
    enforcement: prompt
    prevents: [INC-001]

  - id: G-D05
    rule: "Resposta sobre tier DEVE verificar lista canônica [Gold, Silver, Standard] e CORRIGIR qualquer tier inválido citando SLA-2024 §1"
    enforcement: prompt
    prevents: [INC-003]

  - id: G-D06
    rule: "Quando pipeline detectar chunks de versões diferentes do mesmo documento, DEVE apresentar ambas as versões com regra de vigência explícita"
    enforcement: prompt+code
    code_constraint: "Pipeline injeta flag [CONFLICT_DETECTED] quando document_id coincide mas versions diferem no contexto"
    prevents: [INC-002]
    adr: ADR-0003

  - id: G-D07
    rule: "confidence_score < 0.60 DEVE disparar aviso: 'Confiança baixa — confirme com seu supervisor'"
    enforcement: prompt+code
    code_constraint: "Campo `low_confidence: true` injetado no payload quando score < 0.60; frontend renderiza alerta vermelho"
    prevents: [INC-001]
    adr: ADR-0002
```

---

### must-not

```yaml
# Comportamentos PROIBIDOS do assistente.
# Ref: docs/novatech/guardrails-assistente.md §NÃO DEVE

guardrails:
  - id: G-N01
    rule: "NÃO confirmar que tier 'Platinum' existe ou possui SLA próprio"
    enforcement: prompt
    prevents: [INC-003]

  - id: G-N02
    rule: "NÃO orientar devolução padrão (portal, 7 dias úteis) para cargas perigosas ANTT classes 1-6"
    enforcement: prompt
    prevents: [INC-001]

  - id: G-N03
    rule: "NÃO usar multiplicadores PROC-042v1 para chamados com data >= 2023-12-01"
    enforcement: code   # determinístico — não depende do LLM
    code_constraint: "Pipeline descarta chunks PROC-042v1 para chamados pós-cutoff ANTES de enviar ao LLM"
    prevents: [INC-002]

  - id: G-N04
    rule: "NÃO usar FAQ-Atendimento como única fonte para cargas perigosas, frete ou penalidades de SLA"
    enforcement: prompt+code
    code_constraint: "Reduzir relevance_weight de chunks source_type=faq quando docs normativos com alta similaridade estiverem presentes"
    prevents: [INC-001]

  - id: G-N05
    rule: "NÃO inventar informação quando nenhum chunk com similaridade >= 0.50 for recuperado"
    enforcement: prompt+code
    code_constraint: "Se todos os chunks tiverem score < 0.50, retornar no_relevant_chunks=true — gatilho para resposta de ausência"
    prevents: [INC-001, INC-002]
    adr: ADR-0001

  - id: G-N06
    rule: "NÃO referenciar PROC-043 enquanto status=under_review no índice de documentos"
    enforcement: code   # controle de metadado — gate de recuperação pré-LLM
    code_constraint: "Pipeline filtra documentos com status=under_review antes de recuperar chunks"
    prevents: [incidente-potencial-PROC-043]

  - id: G-N07
    rule: "NÃO presumir tier de cliente sem confirmação explícita do atendente"
    enforcement: prompt
    prevents: [INC-003]
```

---

### when-in-doubt

```yaml
# Comportamento padrão seguro para situações ambíguas.
# Ref: docs/novatech/guardrails-assistente.md §QUANDO EM DÚVIDA

guardrails:
  - id: G-Q01
    situation: "Data do chamado ausente na query com chunks PROC-042 v1 e v2 simultaneamente no contexto"
    default_behavior: "Usar PROC-042v2 e avisar o atendente sobre a regra de vigência"
    enforcement: prompt
    prevents: [INC-002]

  - id: G-Q02
    situation: "Tier do cliente não informado na query e resposta varia por tier"
    default_behavior: "Responder com tabela completa dos 3 tiers e solicitar confirmação do tier antes de orientar"
    enforcement: prompt
    prevents: [INC-003]

  - id: G-Q03
    situation: "Carga descrita com características de periculosidade sem classificação ANTT explícita"
    default_behavior: "Tratar como potencialmente perigosa e orientar verificação da classificação ANTT"
    enforcement: prompt
    prevents: [INC-001]

  - id: G-Q04
    situation: "confidence_score entre 0.60 e 0.79 (zona amarela)"
    default_behavior: "Apresentar resposta com aviso de confiança moderada — não bloquear, apenas alertar"
    enforcement: prompt+code
    prevents: [INC-001, INC-002]

  - id: G-Q05
    situation: "Query envolve múltiplos bounded contexts simultaneamente (ex: devolução de carga perigosa com frete especial)"
    default_behavior: "Responder cada contexto separadamente na ordem: (1) cargas-especiais, (2) devolucoes, (3) frete-especial"
    enforcement: prompt
    prevents: [INC-001, INC-002]

  - id: G-Q06
    situation: "Query sobre PROC-043 com documento marcado status=under_review"
    default_behavior: "Informar instabilidade e orientar contato com ramal 4500"
    enforcement: code
    prevents: [incidente-potencial-PROC-043]

  - id: G-Q07
    situation: "Query sobre tópico fora do escopo documental (tracking em tempo real, negociação de contrato)"
    default_behavior: "Informar ausência de cobertura e indicar canal correto — NÃO responder por analogia"
    enforcement: prompt
    prevents: [incidente-potencial-alucinacao-por-analogia]
```

---

### response-schema

```yaml
# Schema obrigatório para toda resposta do query endpoint.
# Ref: specs/query-endpoint/requirements.md §6 FR-01
# Code constraint: validar antes de retornar — HTTP 422 se campos obrigatórios ausentes

response:
  required:
    - answer          # string — resposta gerada pelo LLM
    - source_document # string[] — ex: ["POL-001", "PROC-042v2"]
    - confidence_score # float 0.0–1.0
    - chunks_used     # string[] — IDs dos chunks recuperados
  optional:
    - low_confidence  # boolean — true se confidence_score < 0.60
    - conflict_detected # boolean — true se [CONFLICT_DETECTED] injetado pelo pipeline
    - no_relevant_chunks # boolean — true se todos os chunks tiverem score < 0.50
```

---

### scope-boundaries

```yaml
# O que este assistente cobre e o que está fora do escopo.
# Ref: specs/query-endpoint/requirements.md §5
# Usar para responder queries de out-of-scope com redirecionamento correto.

in_scope:
  - atendimento        # SLAs, tiers, incidentes críticos
  - frete-especial     # cálculo para cargas > 500kg
  - devolucoes         # prazos, elegibilidade, procedimento
  - cargas-especiais   # carga perigosa, refrigerada, lacre violado

out_of_scope:
  - tracking-tempo-real:
      redirect: "portal.novatech.com.br"
  - interceptacao-carga-em-transito:
      redirect: "Operações — consultar PROC-088"
  - gestao-riscos-carga-perigosa:
      redirect: "ramal 4500"
  - negociacao-contratos-e-descontos:
      redirect: "Diretoria Comercial"
  - gestao-documental:
      redirect: "Responsável de cada Diretoria"
  - PROC-043-frete-carga-perigosa:
      status: "under_review — não usar até liberação pelo Compliance"
      redirect: "ramal 4500"
```

---

### prior-decisions

```yaml
# ADRs do Cenário 1 que constraingem este módulo.
# Ref: specs/query-endpoint/requirements.md §9

adrs:
  - id: ADR-0001
    decision: "RAG sobre fine-tuning"
    impact: "Assistente só responde com base nos chunks recuperados. Sem chunk relevante = resposta de ausência."

  - id: ADR-0002
    decision: "Context budget: máximo 3.000 tokens por requisição"
    impact: "Limita número de chunks enviados ao LLM. Define threshold de latência P95 <= 30s."

  - id: ADR-0003
    decision: "Resolução de conflito entre versões de documentos via metadado de vigência"
    impact: "Pipeline injeta document_version e valid_from em cada chunk. Regra de corte: 2023-12-01 para PROC-042."
```

---

### autoavaliacao-ex-2-3

```yaml
# Autoavaliação D1–D5 — Exercício 2.3 (Trilha AI First DGS, Cenário 2)

dimensoes:
  D1-dominio-conceitual:
    score: 3
    justificativa: >
      Demonstra compreensão de AGENTS.md como artefato machine-readable consumido por agentes de IA:
      regras em YAML estruturado com IDs, campos tipados e referências a paths reais do repo.
      Glossário parsável (não narrativo). Schema de resposta com campos obrigatórios e constraints de código.
      Seções nomeadas consumíveis individualmente por agentes (must-do, must-not, when-in-doubt, response-schema).

  D2-uso-de-ferramentas:
    score: 3
    justificativa: >
      Claude usado para gerar a seção e verificar consistência com guardrails-assistente.md (Ex. 2.2)
      e requirements.md (Ex. 2.1). Copilot é o consumidor direto deste artefato — as regras foram
      escritas em formato que o Copilot consegue seguir ao gerar código em src/functions/query/.
      Iteração implícita: toda regra foi revisada contra os 3 incidentes do Ex. 2.2.

  D3-qualidade-do-entregavel:
    score: 3
    justificativa: >
      Machine-readable: YAML estruturado com IDs, tipos, constraints e referências a paths.
      Completo: 7 seções cobrindo bounded-contexts, glossary, must-do, must-not, when-in-doubt,
      response-schema, scope-boundaries e prior-decisions. Sem texto narrativo — cada campo é
      parsável. Referências a docs/novatech/, specs/ e ADRs são caminhos reais do repositório.

  D4-pensamento-critico:
    score: 3
    justificativa: >
      G-N03 e G-N06 classificados como enforcement=code (não prompt) com justificativa técnica explícita.
      response-schema inclui campos opcionais de sinalização (low_confidence, conflict_detected,
      no_relevant_chunks) que não foram pedidos no enunciado mas são necessários para que o frontend
      e o Copilot consigam reagir ao estado interno do pipeline.
      scope-boundaries inclui PROC-043 com status=under_review — reconhece limitação de documento instável.

  D5-aplicabilidade-ao-projeto:
    score: 3
    justificativa: >
      Totalmente conectado à NovaTech: glossário com termos reais (frete-especial, carga-perigosa,
      tier-Platinum, multiplicador-regional, CT-e, cadeia-de-frio), multiplicadores das duas versões
      do PROC-042 com data de corte 2023-12-01, ramal 4500 de Gestão de Riscos, paths reais do repo.
      Consistente com guardrails-assistente.md (Ex. 2.2): todos os 22 guardrails estão refletidos.
      ADR-0001, ADR-0002, ADR-0003 do Cenário 1 referenciados com impacto concreto.

score-final: "3.0 — Aprovado com distinção"
```

## Testing Standards (QA)
<!-- TODO (QA — Ex. 2.1) -->

## Project Management Rules (Delivery Manager)
<!-- TODO (Delivery Manager — Ex. 2.3) -->

## Build & Deploy
<!-- TODO (Tech Lead — Ex. 2.1) -->
