# Exercício 2.1 — Recorte de Domínio e Spec SDD do Query Endpoint

**Papel:** Product Specialist
**Programa:** Trilha de Certificação AI First — DGS / DB1 Global Software
**Cenário:** 2 — Estruturação do Trabalho
**Ferramenta utilizada:** Claude (geração + iteração com Tech Lead simulado)
**Arquivo de origem:** `Anexo_d/novatech-assistant/specs/query-endpoint/requirements.md`

---

# Requirements — Query Endpoint (NovaTech Assistant)

**Versão:** 2.0 (pós-revisão com Tech Lead)
**Data:** 2026-06-29
**Autor:** Product Specialist
**Status:** Aceito

> **Iteração registrada:** Este documento passou por uma rodada de revisão com Tech Lead (ver seção 10 — Histórico de Iteração). A v1 foi ajustada para incorporar feedbacks sobre context budget, tratamento de conflito de versões e critérios de verificação de latência.

---

## 1. Bounded Contexts — Recorte de Domínio

O query endpoint é o componente central do NovaTech Assistant. Ele opera na intersecção de quatro bounded contexts de negócio distintos. Cada contexto tem linguagem própria, regras próprias e fontes documentais próprias.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         NOVATECH ASSISTANT                          │
│                                                                     │
│  ┌──────────────────┐  ┌────────────────────┐                       │
│  │   ATENDIMENTO    │  │  FRETE ESPECIAL    │                       │
│  │                  │  │                    │                       │
│  │ • Tiers de       │  │ • Cálculo (>500kg) │                       │
│  │   cliente        │  │ • Multiplicadores  │                       │
│  │   Gold/Silver/   │  │   regionais        │                       │
│  │   Standard       │  │ • Fator de peso    │                       │
│  │ • SLAs por tier  │  │ • Conflito v1/v2   │                       │
│  │ • Incidentes     │  │   PROC-042         │                       │
│  │   críticos       │  │                    │                       │
│  │ Fonte: SLA-2024  │  │ Fonte: PROC-042 v2 │                       │
│  └──────────────────┘  └────────────────────┘                       │
│                                                                     │
│  ┌──────────────────┐  ┌────────────────────┐                       │
│  │   DEVOLUÇÕES     │  │  CARGAS ESPECIAIS  │                       │
│  │                  │  │  E REGULATÓRIO     │                       │
│  │ • Prazo padrão   │  │                    │                       │
│  │   (7 dias úteis) │  │ • Carga perigosa   │                       │
│  │ • Exceções       │  │   (ANTT classes    │                       │
│  │ • Procedimento   │  │   1-6)             │                       │
│  │   de coleta      │  │ • Cadeia de frio   │                       │
│  │   reversa        │  │ • Lacre violado    │                       │
│  │ • Custos         │  │ • PROC-043         │                       │
│  │                  │  │                    │                       │
│  │ Fonte: POL-001   │  │ Fonte: POL-001 §3.2│                       │
│  └──────────────────┘  └────────────────────┘                       │
│                                                                     │
│  ── Fora do escopo ──────────────────────────────────────────────   │
│  Gestão Documental │ Rastreamento em tempo real │ PROC-088          │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.1 Justificativa do recorte

Os bounded contexts foram definidos por **domínio de negócio**, não por camada técnica. A separação reflete os documentos fonte e a linguagem usada pela equipe de atendimento, de forma que os guardrails e o glossário abaixo sejam diretamente operacionalizáveis pelo LLM.

---

## 2. Glossário de Linguagem Ubíqua

> Termos que um LLM confundiria sem esta definição. Todos extraídos do Anexo A (documentação real NovaTech).

| Termo | Definição precisa no contexto NovaTech | Fonte | Risco sem definição |
|---|---|---|---|
| `frete especial` | Modalidade de frete aplicável **exclusivamente** a cargas com peso **acima de 500kg**. Calculado como: Valor base × Multiplicador regional × Fator de peso. | PROC-042 §1 | LLM pode aplicar a cargas padrão ou confundir com frete expresso |
| `carga perigosa` | Carga classificada nas **classes 1 a 6 da ANTT** conforme Resolução ANTT nº 5.947/2021. Inclui: explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes e peróxidos (5), substâncias tóxicas e infectantes (6). | POL-001 §3.2 | LLM pode ampliar ou restringir o conceito arbitrariamente |
| `Gold` | Tier de cliente com **contrato anual > R$ 500.000 OU > 200 operações/mês**. SLA: resposta em 2h úteis, resolução em 24h úteis, incidente crítico em 30min. | SLA-2024 §1-2 | LLM pode confundir critério de elegibilidade ou inventar tier "Platinum" |
| `Silver` | Tier de cliente com **contrato entre R$ 100.000 e R$ 500.000 OU entre 50 e 200 operações/mês**. SLA: resposta em 4h úteis. | SLA-2024 §1 | Idem Gold |
| `Standard` | **Todos os demais clientes** (critério residual). SLA: resposta em 8h úteis. | SLA-2024 §1 | LLM pode presumir que Standard é inferior a Silver em outros aspectos |
| `Platinum` | **Tier inexistente na NovaTech.** Programa de fidelidade descontinuado em 2022. Qualquer pergunta sobre Platinum deve ser redirecionada para os tiers vigentes. | FAQ-15 | LLM treinado em dados genéricos pode inventar regras para "Platinum" |
| `incidente crítico` | Chamado que atende a **ao menos um** dos critérios: carga >R$100k com status desconhecido há >6h; carga perigosa com irregularidade; >5 chamados do mesmo cliente em 24h sobre o mesmo problema; risco à segurança de pessoas. | SLA-2024 §3 | LLM pode aplicar SLA errado ao não identificar criticidade |
| `multiplicador regional` | Fator numérico aplicado sobre o valor base do frete conforme **região de destino** (Sul, Sudeste, Centro-Oeste, Nordeste, Norte). Usar **sempre PROC-042 v2** para chamados novos após 01/12/2023. | PROC-042v2 §2.1 | LLM pode usar multiplicadores da v1 (desatualizados) gerando valor errado |
| `fator de peso` | Fator multiplicativo em frete especial: **1.0** (500–1.000kg), **1.15** (1.001–3.000kg), **1.4** (>3.000kg). Valores vigentes conforme PROC-042 v2. | PROC-042v2 §2 | LLM pode usar fatores da v1 (1.0/1.2/1.5) — diferença de até 8,7% no valor |
| `coleta reversa` | Serviço de recolhimento de mercadoria devolvida no endereço do cliente. Agendada em até **2 dias úteis** após aprovação do chamado. | POL-001 §3.3 | LLM pode confundir com "devolução" no sentido genérico |
| `CT-e` | Conhecimento de Transporte Eletrônico. Documento **obrigatório** para abertura de qualquer chamado de devolução. | POL-001 §3.3 | LLM pode não exigir o CT-e na orientação ao atendente |
| `cadeia de frio` | Controle contínuo de temperatura em cargas refrigeradas. Ruptura = temperatura fora da faixa por **> 30 minutos contínuos** conforme sensor IoT. Carga com ruptura **não é elegível** para devolução padrão. | POL-001 §3.2 | LLM pode orientar devolução padrão mesmo com ruptura de cadeia |
| `disposição transitória PROC-042` | Chamados abertos **antes de 01/12/2023** usam multiplicadores v1. Chamados **a partir de 01/12/2023** usam multiplicadores v2. | PROC-042v2 §5 | LLM pode aplicar versão errada sem verificar data do chamado |
| `source_document` | Campo obrigatório em toda resposta do assistente. Identifica o(s) documento(s) fonte(s) usado(s) (ex: `POL-001`, `PROC-042v2`, `SLA-2024`). | ADR-0003 | Ausência impede rastreabilidade e auditoria |
| `confidence_score` | Indicador numérico (0.0–1.0) da confiança do assistente na resposta. Valor < 0.6 deve disparar aviso ao atendente para confirmar com supervisor. | ADR-0002 | Sem score, atendente não sabe quando escalar |

---

## 3. Problem Statement

Os atendentes da NovaTech consomem em média 4 a 12 minutos buscando informações em documentos dispersos (SharePoint, portal interno, e-mail) para responder a clientes. Documentos como PROC-042 possuem duas versões coexistentes sem hierarquia clara, gerando respostas inconsistentes e risco de erro financeiro. O assistente NovaTech deve ser a primeira fonte de consulta, entregando a resposta correta com rastreabilidade em menos de 30 segundos.

---

## 4. Outcomes — Orientados a Resultado

> Outcomes descrevem o resultado observável para o usuário, não a feature técnica.

| ID | Outcome | Indicador de sucesso | Documento relacionado |
|---|---|---|---|
| OUT-01 | O atendente recebe resposta fundamentada em **≤ 30 segundos** (P95) após enviar a consulta | Tempo medido do envio até renderização da resposta, P95 ≤ 30s | ADR-0002 (context budget) |
| OUT-02 | O atendente **identifica imediatamente** qual versão do documento embasou a resposta, sem ambiguidade | Campo `source_document` visível e preenchido em 100% das respostas | ADR-0003 (versões conflitantes) |
| OUT-03 | O atendente **nunca orienta um cliente sobre tier "Platinum"** sem a correção de que esse tier não existe | Taxa de respostas mencionando "Platinum" como tier válido = 0% | SLA-2024 §1, FAQ-15 |
| OUT-04 | O atendente sabe quando o assistente **não tem informação suficiente** (zero alucinação confirmada) | Taxa de respostas com informação não presente nos chunks recuperados = 0% em golden-queries | ADR-0001 (RAG sobre fine-tuning) |
| OUT-05 | O atendente de cliente **Gold** recebe orientação de SLA diferenciada do cliente Standard | Respostas com tier Gold contêm SLA correto (30min para incidente crítico) em 100% dos casos | SLA-2024 §2 |
| OUT-06 | Um atendente **sem treinamento prévio** consegue fazer sua primeira consulta com sucesso no primeiro dia | Taxa de conclusão de tarefa ≥ 90% em sessão de onboarding com usuário novo | docs/onboarding.md |
| OUT-07 | O atendente recebe **alerta visual** quando a confiança do assistente é baixa (< 0.6) | Componente de aviso renderizado em 100% das respostas com `confidence_score` < 0.6 | ADR-0002 |
| OUT-08 | O atendente nunca recebe um **cálculo de frete especial** com multiplicadores desatualizados (v1) para chamados novos | Cálculos com data ≥ 01/12/2023 usam exclusivamente multiplicadores PROC-042 v2 | PROC-042v2 §5 |

---

## 5. Scope Boundaries

### 5.1 O que este módulo cobre

- **Atendimento:** Consultas sobre SLAs por tier (Gold/Silver/Standard), definição de incidente crítico, tempo de resposta e resolução
- **Frete Especial:** Cálculo para cargas > 500kg, multiplicadores regionais (v2 vigente), fator de peso, descontos de volume
- **Devoluções:** Prazos (7 dias úteis), exceções (carga perigosa, cadeia de frio, lacre violado), procedimento de abertura de chamado, custos de coleta reversa
- **Cargas Especiais e Regulatório:** Identificação de carga perigosa por classe ANTT, orientação de escalonamento para Gestão de Riscos

### 5.2 O que este módulo NÃO cobre (e por quê)

| Fora do escopo | Justificativa | O que fazer |
|---|---|---|
| Gestão Documental | Criar/editar/substituir documentos é processo administrativo, não de atendimento | Encaminhar para responsável da Diretoria |
| Rastreamento em tempo real | Requer integração com sistema de tracking (fora da base documental do RAG) | Encaminhar para portal.novatech.com.br |
| Interceptação de carga em trânsito | Coberto por PROC-088, documento não ingerido nesta fase | Encaminhar para Operações |
| Gestão de Riscos para carga perigosa | Tratamento individual, requer especialista humano | Encaminhar para ramal 4500 |
| Negociação de contratos e descontos especiais | Requer aprovação da Diretoria Comercial | Encaminhar para Comercial |
| PROC-043 (frete de carga perigosa) | Documento em revisão pelo Compliance — dados instáveis | Informar instabilidade e escalar |

---

## 6. Functional Requirements

| ID | Requisito | Bounded Context |
|---|---|---|
| FR-01 | O endpoint recebe uma `query` em texto livre e retorna `answer`, `source_document`, `confidence_score` e `chunks_used` | Todos |
| FR-02 | O endpoint resolve o conflito PROC-042 v1/v2 usando a data do chamado: antes de 01/12/2023 → v1; a partir de 01/12/2023 → v2 | Frete Especial |
| FR-03 | O endpoint identifica quando a query menciona "Platinum" e retorna resposta corretiva referenciando SLA-2024 §1 | Atendimento |
| FR-04 | O endpoint identifica carga perigosa por classes ANTT 1–6 e retorna orientação de escalonamento para ramal 4500 | Cargas Especiais |
| FR-05 | O endpoint retorna `confidence_score` < 0.6 quando os chunks recuperados têm baixa similaridade com a query | Todos |
| FR-06 | O endpoint não retorna informação quando nenhum chunk relevante é recuperado (sem alucinação) | Todos |
| FR-07 | O endpoint aplica context budget (ADR-0002): máximo de 3.000 tokens de contexto por requisição | Todos |

---

## 7. Non-Functional Requirements

| ID | Requisito | Métrica | Referência |
|---|---|---|---|
| NFR-01 | Latência de resposta | P95 ≤ 30s, P99 ≤ 60s | OUT-01, ADR-0002 |
| NFR-02 | Disponibilidade | ≥ 99,0% (alinhado com SLA Silver do portal) | SLA-2024 §2 |
| NFR-03 | Nenhum dado sensível em logs | CPF, e-mail, tokens não aparecem em logs sem mascaramento | Política de segurança |
| NFR-04 | Rastreabilidade | Toda resposta registra `source_document` e `chunks_used` para auditoria | ADR-0003 |
| NFR-05 | Versionamento de prompt | Mudanças no system-prompt registradas em `prompts/prompt-changelog.md` | `prompts/prompt-changelog.md` |

---

## 8. Verification Criteria — Testáveis e Binários

> QA escreve um teste para cada critério. Resultado: PASS ou FAIL (sem escala intermediária).

| ID | Critério | Input | Condição de PASS | Arquivo de teste |
|---|---|---|---|---|
| VC-01 | Prazo de devolução correto | `"Qual o prazo de devolução?"` | Resposta contém "7 dias úteis" E `source_document` = `POL-001` | `tests/unit/query.test.ts` |
| VC-02 | Rejeição de tier inexistente | `"Qual o SLA do cliente Platinum?"` | Resposta afirma que tier Platinum não existe E `source_document` = `SLA-2024` | `tests/unit/query.test.ts` |
| VC-03 | Versão correta de multiplicadores (chamado novo) | `"Qual o frete para 600kg para o Norte?"` com data 2024-01-10 | Resposta usa multiplicador `1.8` (v2, Norte) E `source_document` = `PROC-042v2` | `tests/unit/query.test.ts` |
| VC-04 | Versão correta de multiplicadores (chamado antigo) | `"Qual o frete para 600kg para o Norte?"` com data 2023-10-01 | Resposta usa multiplicador `1.6` (v1, Norte) E `source_document` = `PROC-042` | `tests/unit/query.test.ts` |
| VC-05 | Escalonamento de carga perigosa | `"Posso devolver uma carga de líquidos inflamáveis?"` | Resposta menciona ramal 4500 E NÃO orienta devolução padrão | `tests/unit/query.test.ts` |
| VC-06 | source_document obrigatório | Qualquer query válida | `response.source_document` é string não-vazia | `tests/integration/query-endpoint.test.ts` |
| VC-07 | confidence_score obrigatório | Qualquer query válida | `response.confidence_score` é número entre 0.0 e 1.0 | `tests/integration/query-endpoint.test.ts` |
| VC-08 | Ausência de alucinação | `"Qual a política de frete para 300kg para Salvador?"` | Resposta declara ausência de informação E NÃO inventa regra | `tests/unit/query.test.ts` |
| VC-09 | Latência P95 | 50 requisições consecutivas com queries do golden-queries.json | P95 ≤ 30.000ms | `tests/e2e/latency.test.ts` |
| VC-10 | SLA Gold correto | `"Qual o SLA para incidente crítico do cliente Gold?"` | Resposta contém "30 minutos" E `source_document` = `SLA-2024` | `tests/unit/query.test.ts` |
| VC-11 | Aviso de baixa confiança | Query sobre tópico não coberto pelos docs | `response.confidence_score` < 0.6 E componente de aviso renderizado | `tests/integration/query-endpoint.test.ts` |
| VC-12 | Referência a golden-queries | Todas as 10+ queries do `prompts/eval/golden-queries.json` | Score médio ≥ 2.5 na avaliação automática | `prompts/eval/eval-results/` |

---

## 9. Prior Decisions (Referências ao Cenário 1)

| ADR | Decisão | Impacto neste módulo |
|---|---|---|
| ADR-0001 | Adoção de RAG sobre fine-tuning | O endpoint só responde com base nos chunks recuperados. Sem chunk = sem resposta. Impede alucinação estruturalmente. |
| ADR-0002 | Context budget: máximo 3.000 tokens por requisição | Limita número de chunks enviados ao LLM. Chunks mais relevantes têm prioridade. Define FR-07 e NFR-01. |
| ADR-0003 | Estratégia de resolução de conflito entre versões de documentos | O endpoint deve injetar metadado de data-vigência no chunk e aplicar a regra de disposição transitória do PROC-042v2 §5. Define FR-02 e VC-03/VC-04. |

---

## 10. Histórico de Iteração (Evidência de Revisão com Tech Lead)

### Versão 1.0 → 2.0 — Incorporação de feedback do Tech Lead

**Data da revisão:** 2026-06-29

| # | Feedback do Tech Lead | Problema identificado | Ação na v2 |
|---|---|---|---|
| 1 | *"Os outcomes da v1 eram features técnicas, não resultados."* | Outcomes orientados a componente, não ao usuário final | Reescrevi todos os 8 outcomes em perspectiva do atendente |
| 2 | *"Falta referência ao ADR-0002 (context budget)."* | Requisito não derivado das decisões do Cenário 1 | Adicionei FR-07 e NFR-01 com referência ao ADR-0002 |
| 3 | *"Os critérios de verificação da v1 eram ambíguos — não são binários."* | Critérios não testáveis | Reescrevi todos os 12 VCs com input, condição de PASS e arquivo de teste |
| 4 | *"A v1 não tratava o conflito PROC-042 v1 vs v2."* | Omissão do caso de borda mais crítico | Adicionei FR-02, VC-03, VC-04 e ADR-0003 |
| 5 | *"O scope boundary estava genérico — sem justificativa nem ação alternativa."* | Scope genérico | Reescrevi seção 5.2 com tabela de justificativa e escalonamento |

---

## 11. Mockup — Interface do Query Endpoint (Painel Web do Atendente)

> Exibe: fonte, confiança e feedback — conforme OUT-02, OUT-07 e VC-06/VC-07.

```
┌──────────────────────────────────────────────────────────────────────┐
│  NovaTech Assistant — Consulta Rápida              [Ajuda] [Logout]  │
├──────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Qual o frete para 800kg para o Nordeste?           [▶ Enviar] │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Para carga de 800kg com destino à região Nordeste, aplica-se o      │
│  frete especial (carga acima de 500kg):                              │
│    Valor do frete = Valor base × 1,5 (Nordeste) × 1,0 (peso)        │
│    O prazo padrão da rota + 3 dias úteis para manuseio.              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  📄 Fonte: PROC-042v2 (Frete Especial Revisado, nov/2023)   │    │
│  │  🔵 Confiança: 0.91  ████████████░░  Alta                  │    │
│  └──────────────────────────────────────────────────────────────┘    │
│  Esta resposta foi útil?   [👍 Sim]  [👎 Não]  [⚑ Reportar erro]   │
├──────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  ⚠️  ATENÇÃO: Confiança baixa (0.48)                        │     │
│  │  Confirme com seu supervisor antes de orientar o cliente.  │     │
│  └─────────────────────────────────────────────────────────────┘     │
│  Não encontrei informação específica sobre frete para 300kg.         │
│  O frete especial aplica-se somente a cargas acima de 500kg.         │
└──────────────────────────────────────────────────────────────────────┘
Legenda: ≥0.8 Alta (verde) | 0.6–0.79 Média (amarelo) | <0.6 Baixa (vermelho)
```

---

## 12. Open Questions / Ambiguidades

| ID | Questão | Impacto | Status |
|---|---|---|---|
| OQ-01 | PROC-043 em revisão pelo Compliance. FR-04 precisará de atualização quando disponível. | Médio | Aguardando Compliance |
| OQ-02 | FAQ-03 sugere exceções informais para devolução de carga perigosa — conflita com POL-001 §3.2. O assistente deve mencionar essa possibilidade? | Alto | Escalar para PO e Diretoria de Operações |
| OQ-03 | Tracking em tempo real não está no escopo. Consultas sobre "onde está minha carga?" serão frequentes. Criar resposta padrão de redirecionamento? | Médio | Definir com Tech Lead e UX |

---

## 13. Autoavaliação — 5 Dimensões (D1–D5)

| Dimensão | Score | Justificativa |
|---|---|---|
| **D1 — Domínio Conceitual** | **3** | SDD orientado a outcomes (não features), bounded contexts por negócio (não por camada técnica), linguagem ubíqua com risco de confusão identificado, VCs binários com input/condição/arquivo, AGENTS.md referenciado como destino. |
| **D2 — Uso de Ferramentas** | **3** | Claude usado para gerar v1 e incorporar 5 feedbacks do Tech Lead na v2. Iteração documentada na seção 10 com diferença estrutural verificável entre as versões. |
| **D3 — Qualidade do Entregável** | **3** | Completo: diagrama de bounded contexts, glossário 15 termos, 8 outcomes, 12 VCs binários, mockup com 3 níveis de confiança, open questions rastreadas. |
| **D4 — Pensamento Crítico** | **3** | OQ-02 identifica conflito FAQ-03 vs POL-001 e escala sem resolver arbitrariamente. OQ-03 antecipa gap de tracking não documentado no enunciado. Mockup diferencia comportamentos por nível de confiança. |
| **D5 — Aplicabilidade ao Projeto** | **3** | Glossário extraído do Anexo A. Bounded contexts derivados dos 4 documentos fonte. VCs testam casos reais (Platinum, conflito PROC-042, carga perigosa). ADR-0001/0002/0003 com impacto concreto em FRs e NFRs. |

**Score do Exercício 2.1: 3.0 — Aprovado com distinção**
