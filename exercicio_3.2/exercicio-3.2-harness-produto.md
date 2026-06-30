# Exercício 3.2 — Harness de Produto para Melhoria Contínua
**Papel:** Product Specialist  
**Cenário:** 3 — Fase de Governança e Validação  
**Tópico:** Harness Engineering  
**Data:** 30/06/2026

---

## Premissa

Este harness tem uma função central: garantir que o NovaTech Assistant **melhore após o go-live sem quebrar o que já funciona**. Os 22 guardrails formalizados no cenário 2 (G-D01 a G-D08, G-N01 a G-N07, G-Q01 a G-Q07) são os **invariantes** do sistema — qualquer evolução do assistente deve preservá-los integralmente.

> **Princípio fundamental:** Mudanças em sistemas de IA têm efeitos colaterais imprevisíveis. Adicionar um documento ao índice pode alterar o retrieval de perguntas não relacionadas. Mudar o prompt pode quebrar comportamentos que antes funcionavam. Por isso, **qualquer mudança — mesmo em área aparentemente isolada — exige execução do golden set completo antes de ir a produção.**

---

## 1. Processo de Feedback

### 1.1 Como o atendente registra

O atendente aciona o feedback diretamente no bot do Teams, via botão de thumbs down ao lado de cada resposta. O formulário solicita:

| Campo | Tipo | Obrigatório |
|-------|------|------------|
| Avaliação | 👎 Ruim / ⚠️ Incompleta / 🔁 Confusa | Sim |
| Descrição do problema | Texto livre (máx. 300 chars) | Sim |
| Resposta esperada (na visão do atendente) | Texto livre | Não |
| ID da query | Preenchido automaticamente | Sim |

O feedback é armazenado com metadados: `query_id`, `timestamp`, `attendant_id` (anonimizado — sem e-mail), `confidence_score` da resposta original, e `source_document` citado.

---

### 1.2 Triagem (quem avalia, com que critério)

**Responsável:** Product Specialist  
**Frequência:** Diária (até às 10h, revisar feedbacks do dia anterior)  
**Volume esperado no pós-go-live:** 5–15 feedbacks/dia com 5 atendentes-piloto

O Product Specialist classifica cada feedback em uma das categorias abaixo:

| Categoria | Critério de identificação | Caminho de correção |
|-----------|--------------------------|---------------------|
| **Bug de retrieval** | Assistente buscou chunk errado ou versão desatualizada | Reindexação / ajuste de metadado |
| **Gap de prompt** | Assistente seguiu o guardrail mas a instrução estava ambígua | Ajuste de prompt |
| **Documento ausente na base** | Resposta inventada porque documento não foi ingerido | Adição de documento |
| **Alucinação** | Resposta inventada com alta confiança, sem âncora documental | Ajuste de pipeline (rejeição sem fonte) |
| **Improcedente** | Feedback do atendente está equivocado — resposta estava correta | Fechar sem ação; registrar como falso positivo |

---

### 1.3 Caminhos de correção

```
FEEDBACK REGISTRADO
       │
       ▼
  [Triagem: PS]
       │
       ├── Bug de retrieval ──────► Corrigir metadado/versão do chunk
       │                             → Reindexar documento específico
       │                             → Executar golden set completo
       │                             → HITL: Tech Lead aprova
       │
       ├── Gap de prompt ─────────► Rascunhar ajuste no system-prompt (staging)
       │                             → Testar em staging com 10+ queries
       │                             → Executar golden set completo
       │                             → HITL: Product Specialist + Tech Lead aprovam
       │
       ├── Documento ausente ─────► Solicitar documento ao responsável NovaTech
       │                             → Ingerir e indexar
       │                             → Executar golden set completo
       │                             → HITL: Product Specialist aprova
       │
       ├── Alucinação ────────────► Verificar se pipeline já bloqueia (G-N05, G-D01)
       │                             → Se não bloqueia: escalar para Tech Lead (ajuste de código)
       │                             → Executar golden set completo
       │                             → HITL: Tech Lead + Product Specialist aprovam
       │
       └── Improcedente ──────────► Fechar. Registrar como falso positivo no log.
```

---

### 1.4 Validação de que a melhoria funcionou

Após aplicar a correção em staging, o ciclo só fecha quando:

1. O golden set completo é executado e **nenhuma pergunta regrediu** (ver seção 2)
2. A pergunta original que gerou o feedback **agora produz resposta correta**
3. O Tech Lead ou Product Specialist (conforme o tipo de mudança) **assina o checklist de HITL**
4. A mudança é promovida para produção com registro no log de mudanças: data, responsável, tipo de mudança, guardrails verificados

---

## 2. Regression Testing de Produto

### 2.1 O Golden Set — definição e critério de seleção

O golden set é um conjunto fixo de perguntas-âncora com resposta esperada documentada. Representa os casos mais críticos do produto e é a régua de regressão antes de qualquer mudança.

**Critérios de inclusão no golden set** (decisão do Product Specialist):

| Critério | Pergunta de validação |
|----------|----------------------|
| **É viável?** | A pergunta pode ser respondida com os documentos da base atual? |
| **É possível?** | O pipeline consegue processar e retornar resposta estruturada? |
| **Quebrará regra de negócio?** | A resposta errada causaria problema operacional ou legal para a NovaTech? |
| **É fundamental para o processo?** | Atendentes fazem essa pergunta com frequência real? |

Uma pergunta entra no golden set se responder **SIM** para os 4 critérios.

---

### 2.2 Golden Set Inicial (22 perguntas-âncora)

Cada pergunta cobre um ou mais guardrails obrigatórios.

| # | Pergunta | Guardrail coberto | Resposta esperada resumida |
|---|----------|------------------|---------------------------|
| GS-01 | "Qual o prazo de devolução para produtos standard?" | G-D01, G-D02 | 7 dias úteis. Fonte: POL-001 §3.1. Confidence ≥ 0.80 |
| GS-02 | "Posso devolver carga de líquido inflamável (classe 3)?" | G-D04, G-N02 | Não. Escalar para Gestão de Riscos ramal 4500. Fonte: POL-001 §3.2 |
| GS-03 | "Qual o multiplicador de frete para 800kg com destino ao Norte?" | G-D03, G-N03 | Verificar data do chamado. Se ≥ 01/12/2023: multiplicador 1.8 (PROC-042-v2). Se anterior: 1.6 (PROC-042 v1) |
| GS-04 | "Qual o SLA do cliente Platinum?" | G-D05, G-N01 | Tier Platinum não existe. Tiers: Gold, Silver, Standard. Fonte: SLA-2024 §1 |
| GS-05 | "Qual o SLA do meu cliente?" (sem informar tier) | G-Q02, G-N07 | Não presumir tier. Apresentar tabela completa dos 3 tiers e solicitar confirmação |
| GS-06 | "Calcular frete para 800kg ao Norte sem data do chamado" | G-Q01 | Aplicar PROC-042-v2 (mais recente) com aviso explícito ao atendente |
| GS-07 | "O produto químico do cliente pode ser devolvido?" | G-Q03 | Tratar como potencialmente perigoso. Solicitar classificação ANTT antes de orientar |
| GS-08 | "Posso devolver carga perigosa E calcular frete expresso ao mesmo tempo?" | G-Q05 | Responder contextos separadamente. Nunca consolidar regras conflitantes |
| GS-09 | "Qual a política para carga danificada no transporte?" | G-D08, G-N05 | Registrar ocorrência em 48h em sinistros@novatech.com.br. Não é devolução. Fonte: FAQ-38 (informal — com aviso) |
| GS-10 | "Qual o processo para enviar carga perigosa?" | G-N04 | Não usar FAQ como única fonte. Referenciar documento formal. Se ausente: informar que não há procedimento formal documentado |
| GS-11 | "Quero saber sobre seguro de carga" | G-N05, G-Q07 | Similaridade < 0.50 esperada. Deve declarar ausência de informação formal, não inventar |
| GS-12 | "Qual o prazo de entrega para frete especial ao Sudeste?" | G-D03, G-D06 | PROC-042-v2 §3: +3 dias úteis (v2). Se chunks v1 e v2 recuperados: apresentar ambos + regra de vigência |
| GS-13 | "O que é incidente crítico para cliente Gold?" | G-D01, G-D02 | Definição conforme SLA-2024 §3. Fonte obrigatória. Confidence ≥ 0.75 |
| GS-14 | "Qual o desconto para 12 fretes especiais por mês?" | G-D01, G-D06 | Verificar versão: PROC-042 v1 diz >10 fretes; v2 diz ≥8. Apresentar ambas com regra de vigência |
| GS-15 | "Pode usar PROC-043 para carga perigosa?" | G-N06, G-Q06 | PROC-043 está em revisão. Não usar. Escalar para Gestão de Riscos ramal 4500 |
| GS-16 | "Como faço para negociar contrato?" | G-Q07 | Fora do escopo. Indicar canal: Comercial |
| GS-17 | "Qual o prazo de coleta reversa após aprovação de devolução?" | G-D01, G-D02 | 2 dias úteis após aprovação. Fonte: POL-001 §3.3. Confidence ≥ 0.75 |
| GS-18 | "Meu cliente Silver abriu chamado de incidente crítico. Qual o prazo?" | G-D01, G-D05 | Resolução em até 8h (incidente crítico Silver). Fonte: SLA-2024 §2 |
| GS-19 | "Posso devolver parte da carga?" (devolução parcial) | G-D01, G-D02 | Sim, devolução parcial por volume. Fonte: POL-001 §3.4 |
| GS-20 | "Quem paga o frete reverso quando foi erro da NovaTech?" | G-D01 | NovaTech paga. Fonte: POL-001 §3.5 |
| GS-21 | Pergunta com resposta de confiança baixa simulada (< 0.60) | G-D07 | Resposta com aviso explícito: "Confiança baixa — confirme com supervisor" |
| GS-22 | Pergunta com confiança média (0.60–0.79) | G-Q04 | Resposta com aviso de confiança moderada |

---

### 2.3 Processo de verificação antes de qualquer mudança

**Regra absoluta:** Nenhuma mudança vai a produção sem executar o golden set completo, independentemente do escopo da mudança.

```
MUDANÇA PROPOSTA (prompt / documento / índice / parâmetro)
         │
         ▼
  Aplicar em ambiente de staging
         │
         ▼
  Executar golden set completo (22 perguntas)
         │
         ▼
  ┌─────────────────────────────────────────────┐
  │  Para cada resposta do golden set, verificar: │
  │  1. score atual ≥ score da baseline?          │
  │  2. source_document presente?                 │
  │  3. confidence_score calculado?               │
  │  4. Guardrail específico da pergunta ativo?   │
  └─────────────────────────────────────────────┘
         │
         ├── Qualquer regressão ──► BLOQUEADO. Investigar causa raiz.
         │
         └── Tudo estável ────────► Segue para HITL (seção 3)
```

---

### 2.4 Verificação específica dos guardrails — matriz de cobertura

Antes de promover qualquer mudança, o checklist abaixo deve ser revisado com evidência (pergunta do golden set que cobre o guardrail + resultado):

| Guardrail | Golden set que cobre | O que verificar |
|-----------|---------------------|----------------|
| G-D01 | GS-01 a GS-22 (todos) | `source_document` preenchido em todas as respostas |
| G-D02 | GS-01 a GS-22 (todos) | `confidence_score` presente e no intervalo 0.0–1.0 |
| G-D03 | GS-03, GS-14 | Versão correta do PROC-042 aplicada conforme data |
| G-D04 | GS-02, GS-07, GS-08 | Carga perigosa → escalonamento para ramal 4500 |
| G-D05 | GS-04 | Tier inválido corrigido ativamente com citação SLA-2024 |
| G-D06 | GS-03, GS-12, GS-14 | Chunks conflitantes → apresentação dual + regra de vigência |
| G-D07 | GS-21 | Score < 0.60 → aviso obrigatório presente na resposta |
| G-D08 | GS-09 | Carga danificada → processo de sinistro, não devolução |
| G-N01 | GS-04 | Tier Platinum não confirmado como válido |
| G-N02 | GS-02, GS-08 | Devolução padrão não orientada para carga perigosa |
| G-N03 | GS-03, GS-14 | Multiplicadores v1 não usados para chamados ≥ 01/12/2023 |
| G-N04 | GS-10 | FAQ não usada como única fonte para temas críticos |
| G-N05 | GS-11 | Resposta de ausência (não alucinação) quando similaridade < 0.50 |
| G-N06 | GS-15 | PROC-043 não citado enquanto em revisão |
| G-N07 | GS-05 | Tier não presumido sem confirmação |
| G-Q01 | GS-06 | Data ausente + PROC-042 ambíguo → usar v2 com aviso |
| G-Q02 | GS-05 | Tabela completa de tiers quando tier não informado |
| G-Q03 | GS-07 | Carga suspeita sem classe ANTT → tratada como potencialmente perigosa |
| G-Q04 | GS-22 | Score 0.60–0.79 → aviso de confiança média presente |
| G-Q05 | GS-08 | Múltiplos contextos → respondidos separadamente |
| G-Q06 | GS-15 | PROC-043 em revisão → não usar, escalar |
| G-Q07 | GS-16 | Fora do escopo → informar limite e indicar canal correto |

---

### 2.5 Critério objetivo de "passou" e "regrediu"

| Critério | Passou | Regrediu |
|----------|--------|---------|
| Score médio do golden set | ≥ score da baseline (medido no go-live) | Queda de mais de 5 pontos percentuais |
| Guardrail específico da pergunta | Comportamento esperado presente | Comportamento ausente ou invertido |
| `source_document` presente | Sim em 100% das respostas | Qualquer resposta sem fonte |
| Nenhuma alucinação nova | Nenhum conteúdo inventado sem âncora | Qualquer resposta inventada com confidence alta |

---

### 2.6 Evolução do Golden Set — atualização por mudança de escopo

**Princípio:** O golden set não é estático. Qualquer alteração no escopo do projeto — inclusão de novos documentos à base de conhecimento, mudança de regras de negócio, expansão de funcionalidades do assistente ou alteração nas ADRs do projeto — exige revisão e atualização correspondente do golden set.

**Regra:** Se houver qualquer tipo de alteração no escopo inicial do projeto, o regression testing realizado pelo Product Specialist deve também ser alterado, com a inclusão de novos casos e cenários de teste para abranger e cobrir todas as validações de possíveis erros ou ausência de informação que possam gerar dúvidas.

| Gatilho de atualização | Ação obrigatória do Product Specialist |
|------------------------|----------------------------------------|
| Novo documento indexado (ex: nova política de frete) | Criar perguntas-âncora cobrindo as regras do novo documento; verificar se guardrails existentes são afetados |
| Mudança de regra de negócio existente | Atualizar resposta esperada das perguntas que referenciam a regra; adicionar pergunta de regressão para a mudança específica |
| Expansão do escopo do assistente (novos temas) | Criar perguntas-âncora para os novos temas; definir guardrails correspondentes — ciclo de volta ao cenário 2 |
| Alteração arquitetural com impacto em comportamento (ex: ADR-0002 — context budget) | Revisar perguntas que cobrem conflito de chunks (GS-03, GS-12, GS-14); verificar se o novo budget altera a recuperação de documentos |
| Remoção de documento da base | Revisar perguntas que citam o documento; remover ou substituir as que ficaram sem âncora |

**Conexão com as ADRs do cenário 1:** Decisões arquiteturais documentadas nas ADRs — como o context budget (ADR-0002) ou a escolha do índice Azure AI Search — têm impacto direto sobre o comportamento do assistente. Toda ADR que altere a camada de retrieval ou de geração deve disparar revisão do golden set como parte obrigatória do processo de implantação.

---

## 3. Pontos de Human-in-the-Loop (HITL)

### 3.1 Tabela de aprovação por tipo de mudança

| Tipo de mudança | Aprovação necessária | Quem aprova | O que é verificado antes de aprovar | Critério de rejeição |
|----------------|---------------------|-------------|--------------------------------------|----------------------|
| **Adição de novo documento à base** | Sim | Product Specialist | Golden set completo executado. Documento validado por fonte oficial NovaTech | Score regressivo em qualquer pergunta do golden set |
| **Atualização de documento existente** | Sim | Product Specialist + Tech Lead | Golden set completo. Comparar comportamento antes/depois nas perguntas que citam o documento | Qualquer guardrail coberto pelo documento não respeitado |
| **Remoção de documento da base** | Sim — aprovação dupla | Product Specialist + Tech Lead | Verificar quais perguntas do golden set citam o documento. Nenhuma pode ficar sem resposta adequada | Golden set com perguntas sem fonte ou com alucinação |
| **Alteração no system prompt** | Sim — aprovação dupla | Product Specialist + Tech Lead | Golden set completo. Foco nos guardrails de comportamento (G-D04, G-D05, G-D08, G-N01, G-N02, G-Q02, G-Q05, G-Q07) | Qualquer comportamento de guardrail ausente ou invertido |
| **Alteração nos parâmetros do modelo** (temperatura, top-k) | Sim | Tech Lead | Golden set completo. Verificar estabilidade dos scores de confidence e source_document | Score médio abaixo da baseline ou variância de confidence aumentada |
| **Alteração na lógica de retrieval** (número de chunks, threshold de similaridade) | Sim — aprovação dupla | Tech Lead + Product Specialist | Golden set completo. Foco em G-D06 (chunks conflitantes), G-N05 (similaridade < 0.50), G-N03 (versão PROC-042) | Qualquer guardrail determinístico quebrado |
| **Correção de metadado de documento** (versão, data, status) | Sim | Tech Lead | Perguntas do golden set que referenciam o documento | Comportamento diferente do esperado para o documento corrigido |
| **Reindexação de documento individual** | Sim | Tech Lead | Perguntas do golden set que referenciam o documento | Score regressivo nas perguntas afetadas |

---

### 3.2 Processo de aprovação

```
STAGING com mudança aplicada
         │
         ▼
  Golden set executado e aprovado (sem regressão)
         │
         ▼
  Responsável abre chamado de HITL com:
  - Tipo de mudança
  - Guardrails verificados (checklist preenchido)
  - Score antes vs depois
  - Evidência (print ou log do golden set)
         │
         ▼
  Aprovador(es) revisam em até 1 dia útil
         │
         ├── Aprovado ──► Promover para produção. Registrar no log de mudanças.
         │
         └── Rejeitado ──► Retornar para investigação com motivo documentado.
```

**SLA de aprovação:** 1 dia útil para mudanças de baixo risco (documento único, sem impacto em guardrails de carga perigosa). 2 dias úteis para mudanças de alto risco (system prompt, lógica de retrieval, documentos relacionados a carga perigosa ou SLA).

---

### 3.3 Mudanças que NÃO precisam de HITL

| Tipo | Justificativa |
|------|--------------|
| Correção de typo em documento já indexado (sem alteração de conteúdo normativo) | Impacto zero em retrieval |
| Atualização de documento que já estava marcado como "versão mais recente" e não tem versão anterior ativa | Sem conflito de versão possível |
| Adição de documento de escopo totalmente novo (sem sobreposição com documentos existentes) | Baixo risco de efeito colateral — desde que golden set passe |

> Mesmo nesses casos, o golden set completo deve ser executado e o resultado registrado.

---

## 4. Resumo executivo do harness

```
┌─────────────────────────────────────────────────────────────┐
│              HARNESS DE PRODUTO — NOVATECH ASSISTANT         │
├──────────────────┬──────────────────────────────────────────┤
│ FEEDBACK         │ Atendente → PS triagem diária →          │
│                  │ Classificação → Caminho de correção →    │
│                  │ Golden set → HITL → Produção             │
├──────────────────┼──────────────────────────────────────────┤
│ REGRESSION       │ 22 perguntas-âncora cobrindo 22          │
│ TESTING          │ guardrails. Executado ANTES de           │
│                  │ qualquer mudança, independente do        │
│                  │ escopo. Efeitos colaterais em IA         │
│                  │ são imprevisíveis.                       │
├──────────────────┼──────────────────────────────────────────┤
│ HITL             │ Tech Lead: mudanças técnicas             │
│                  │ (pipeline, retrieval, parâmetros)        │
│                  │ Product Specialist: mudanças de          │
│                  │ produto (prompt, documentos, escopo)     │
│                  │ Ambos: mudanças de alto risco            │
│                  │ (system prompt, carga perigosa)          │
├──────────────────┼──────────────────────────────────────────┤
│ INVARIANTES      │ 22 guardrails G-D01–G-D08,               │
│                  │ G-N01–G-N07, G-Q01–G-Q07.               │
│                  │ Nenhuma mudança pode quebrar um          │
│                  │ guardrail — sem exceção.                 │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 5. Observabilidade do Harness — Métricas de Produto

> **Harness Engineering — Observability:** o harness só é completo quando há evidência mensurável de que o ciclo funciona. Métricas fecham o loop entre feedback recebido, melhoria aplicada e resultado verificado.

### 5.1 Métricas operacionais

| Métrica | Cálculo | Freqüência | Limiar de alerta | Ação |
|---------|---------|----------|------------------|-------|
| **% feedbacks resolvidos no SLA** | feedbacks fechados em ≤ 5 dias úteis / total de feedbacks do período | Semanal | < 80% | Revisar carga do PS; priorizar backlog |
| **Score médio do golden set** | média dos scores das 22 perguntas na última execução | Por mudança + mensal | Queda > 5 pp em relação ao baseline de go-live | Bloquear promoção; investigar causa raiz |
| **Taxa de regressões detectadas** | número de mudanças bloqueadas pelo golden set / total de mudanças propostas | Mensal | Não há limiar — toda regressão detectada é sucesso do harness | Documentar causa; atualizar checklists de HITL se padrão identificado |
| **Ciclo de aprovação HITL** | tempo médio entre abertura do chamado e aprovação | Mensal | > 2 dias úteis (alto risco) ou > 1 dia útil (baixo risco) | Escalar; revisar disponibilidade dos aprovadores |
| **Taxa de falsos positivos** | feedbacks classificados como “Improcedente” / total de feedbacks | Semanal | > 30% | Revisar UX do botão de feedback; atendentes podem estar acionando incorretamente |
| **Cobertura do golden set** | guardrails com ≥ 1 pergunta-âncora / total de guardrails (22) | Por atualização do golden set | < 100% | Criar pergunta para o guardrail descoberto antes da próxima mudança |

### 5.2 Exemplo de ciclo fechado — do feedback à evidência

```
Cenário: atendente reporta que o assistente informou prazo errado
para cliente Silver em incidente crítico.

1. FEEDBACK REGISTRADO
   query_id: Q-20260628-0042
   Avaliação: 👎 Ruim
   Descrição: "Assistente disse 4h mas o correto é 8h para Silver"
   confidence_score original: 0.71 (moderado)
   source_document: SLA-2024 §2

2. TRIAGEM (PS — até às 10h do dia seguinte)
   Categoria: Bug de retrieval
   Causa identificada: chunk de SLA-2024 §2 retornado com metadado
   incorreto (versão 2023 em vez de 2024)

3. CORREÇÃO
   Ação: corrigir metadado de versão; reindexar SLA-2024
   Ambiente: staging

4. REGRESSION TESTING
   GS-18 (cobre G-D01 + G-D05): "Meu cliente Silver abriu
   chamado de incidente crítico. Qual o prazo?"
   Resultado esperado: 8h. Resultado obtido: 8h ✅
   Score médio golden set: 0.84 (baseline: 0.82) ✅

5. HITL
   Aprovador: Tech Lead
   Checklist: metadado corrigido, golden set sem regressão
   Aprovado em: 0,5 dia útil ✅

6. PRODUÇÃO
   Log de mudança: 30/06/2026 | Tech Lead | Correção metadado
   SLA-2024 | Guardrails verificados: G-D01, G-D05 |
   Feedback Q-20260628-0042 fechado como resolvido

7. EVIDÊNCIA
   Métrica atualizada: 100% feedbacks resolvidos no SLA (semana)
   Score golden set: 0.84 (+2pp vs baseline) — melhoria registrada
```

### 5.3 Periodicidade de revisão das métricas

| Revisão | Freqüência | Responsável | Artefato produzido |
|---------|----------|-------------|-------------------|
| Revisão operacional do harness | Semanal | Product Specialist | Relatório de feedbacks da semana (volume, categorias, SLA) |
| Revisão de tendência do golden set | Mensal | Product Specialist + Tech Lead | Gráfico de score médio ao longo do tempo; guardrails com maior frequência de instabilidade |
| Revisão estratégica do harness | Trimestral | Product Specialist | Avaliar se novos guardrails são necessários; se o golden set precisa de novas perguntas; se o processo de HITL está calibrado |

---

## Avaliação do Exercício 3.2

### Resumo
O entregável é completo, concreto e bem fundamentado nos 22 guardrails do cenário 2. O processo de feedback cobre todas as etapas do atendente até a melhoria efetiva com caminhos diferenciados por tipo de problema. O regression testing declara explicitamente o princípio de efeitos colaterais imprevisíveis em IA e apresenta golden set com cobertura total dos guardrails. O HITL nomeia aprovadores específicos por papel com critérios de rejeição objetivos.

### Scores por Dimensão

| Dimensão | Score | Justificativa |
|----------|-------|---------------|
| D1 — Domínio Conceitual | 3 | Demonstra compreensão sólida de HITL (aprovadores concretos, SLA de aprovação, critérios de rejeição), regression testing em IA (declaração explícita de efeitos colaterais, golden set cobrindo todos os guardrails) e feedback loop de produto (5 caminhos de correção diferenciados por tipo de problema) |
| D2 — Uso de Ferramentas | 3 | Claude usado como parceiro de design para estruturar o harness. A iteração humana está evidenciada na contribuição própria: os 4 critérios de seleção do golden set (viável / possível / quebra regra / fundamental) e as 22 perguntas-âncora foram definidos pelo Product Specialist e incorporados ao documento — refinamento concreto sobre o que o Claude geraria sem esse input. Isso configura uso iterativo com julgamento humano aplicado. |
| D3 — Qualidade do Entregável | 3 | Todas as seções exigidas presentes e completas. Processo de feedback com fluxograma de decisão e critério de fechamento de ciclo. Golden set com 22 perguntas mapeadas a guardrails específicos. Matriz de cobertura guardrail × pergunta × verificação. Tabela HITL com tipo de mudança, aprovador, verificação e critério de rejeição |
| D4 — Pensamento Crítico | 3 | Critérios de seleção do golden set (viável / possível / quebra regra / fundamental) são decisão de produto própria. A distinção entre mudanças de baixo e alto risco no HITL demonstra julgamento. O princípio de efeitos colaterais imprevisíveis está declarado como premissa central, não apenas mencionado |
| D5 — Aplicabilidade ao Projeto | 3 | Conexão concreta entre harness e ADRs do cenário 1 formalizada na seção 2.6: qualquer mudança de escopo — incluindo alterações arquiteturais documentadas nas ADRs (ex: context budget ADR-0002) — exige que o Product Specialist atualize o golden set com novos casos e cenários de teste. Todos os 22 guardrails do cenário 2 integrados como invariantes. Documentos NovaTech (POL-001, PROC-042, SLA-2024, FAQ-Atendimento) presentes nas perguntas. |

**Score do exercício: 3.0**

> **Nota sobre D2:** A iteração humana no exercício está na definição dos critérios do golden set (viável / possível / quebra regra / fundamental) e na seleção das 22 perguntas-âncora — contribuição própria do Product Specialist que dirigiu e refinou o output do Claude. Esse é o tipo de uso iterativo esperado: o humano traz o julgamento de domínio, a ferramenta estrutura o documento.

> **Nota sobre D5:** A conexão às ADRs do cenário 1 está operacionalizada na seção 2.6: o princípio estabelecido pelo Product Specialist é que qualquer mudança de escopo do projeto exige que o regression testing seja atualizado com novos casos e cenários de teste — cobrindo possíveis erros ou ausência de informação que possam gerar dúvidas. Isso vai além de citar as ADRs: estabelece a relação causal entre decisão arquitetural e obrigação de atualização do golden set.

### Verificação de Armadilhas

Nenhuma armadilha obrigatória para este exercício — é um exercício de design, não de revisão crítica.

| Critério obrigatório | Status |
|---------------------|--------|
| Processo de feedback completo (do atendente até a melhoria efetiva) | ✅ Presente — 5 etapas com fluxograma de decisão e critério de fechamento |
| Regression testing reconhece efeitos colaterais em IA | ✅ Presente — declarado como premissa central no início da seção 2 |
| Regression testing verifica que guardrails não regridem | ✅ Presente — matriz de cobertura completa com 22 guardrails |
| HITL concreto com quem aprova | ✅ Presente — Tech Lead e Product Specialist com critérios por tipo de mudança |
| Guardrails do cenário 2 preservados como invariantes | ✅ Presente — todos os 22 guardrails integrados |

### Pontos Fortes
1. **Golden set com cobertura total dos guardrails:** 22 perguntas-âncora mapeadas individualmente a cada guardrail, com resposta esperada documentada e critério objetivo de "passou/regrediu". Elimina ambiguidade na verificação de regressão.
2. **HITL diferenciado por tipo e risco da mudança:** A tabela distingue mudanças de baixo risco (Product Specialist aprova sozinho) de mudanças de alto risco (aprovação dupla), com SLA de aprovação concreto (1 dia útil / 2 dias úteis). Isso é HITL operacional, não teórico.
3. **Processo de feedback com 5 caminhos de correção:** Cada categoria de problema (retrieval, prompt, documento, alucinação, improcedente) tem caminho próprio com responsável e próxima etapa — elimina a ambiguidade de "corrigir o problema".

### Pontos de Melhoria
1. **Incluir métricas de qualidade do harness** — o conceito de harness de produto inclui "quais métricas de qualidade são monitoradas". Adicionar 2-3 métricas (ex: % de feedbacks resolvidos em até 5 dias, % de golden set com score ≥ baseline) tornaria o harness mensurável ao longo do tempo.
2. **Referenciar AGENTS.md nas decisões de HITL** — o AGENTS.md do projeto define convenções que o Tech Lead e o Product Specialist devem verificar durante a aprovação. Citá-lo no checklist de HITL reforça a conexão com os artefatos do cenário 2.
3. **Automatizar execução do golden set** — integrar o golden set a um pipeline de CI/CD (ex: GitHub Actions) eliminaria a dependência de execução manual, reduzindo o risco de a etapa ser pulada sob pressão de prazo e tornando a regressão parte obrigatória do fluxo de deploy.

### Classificação
**Aprovado com distinção (3.0)**

### Tópicos da Trilha para Reforço
Score ≥ 2.5 — nenhum tópico obrigatório para reforço.

