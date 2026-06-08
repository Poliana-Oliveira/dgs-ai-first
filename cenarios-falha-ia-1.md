# Exercício 1.1 — Cenários de Falha de IA: NovaTech

> **Fonte de verdade utilizada:** Anexo A — Documentação Simulada da NovaTech (POL-001, PROC-042 v1, PROC-042 v2, SLA-2024, FAQ-Atendimento).
> **Guardrails do projeto:** (1) Sempre citar fonte. (2) Nunca inventar prazos ou valores. (3) Quando não souber, dizer explicitamente. (4) Responder em português formal.

---

## Lista Inicial — Gerada sem uso de IA [Humano]

> Cenários elaborados com base na leitura crítica da documentação, antes de qualquer consulta ao Claude.

| # | Cenário resumido | Categoria |
|---|---|---|
| H1 | IA inventa percentual de seguro de carga sem documento formal | Alucinação |
| H2 | IA usa multiplicadores da PROC-042 v1 para calcular frete atual | Info desatualizada |
| H3 | Após sessão longa, IA ignora o documento trazido e repete resposta anterior | Falha de contexto (context rot) |
| H4 | IA diz não saber informação sobre tiers quando SLA-2024 documenta claramente | Recusa inadequada |

---

## Cenários Adicionais — Gerados com Claude [Claude]

> Prompt enviado ao Claude: *"Com base nos documentos POL-001, PROC-042 v1 e v2, SLA-2024 e FAQ-Atendimento da NovaTech, e considerando os guardrails: (1) sempre citar fonte, (2) nunca inventar valores, (3) dizer quando não souber, (4) responder em português formal — identifique ao menos 4 cenários de falha adicionais que cubram alucinação, chunk errado, lost in the middle e falha de guardrail."*

| # | Cenário resumido | Categoria |
|---|---|---|
| C1 | IA inventa tabela de frete para cargas abaixo de 500kg (gap real na base) | Alucinação |
| C2 | Retriever traz chunk da PROC-042 v1 para pergunta atual de cálculo | Falha de contexto (chunk errado) |
| C3 | IA ignora seção 3.4 (devoluções parciais) por estar no meio do documento longo | Falha de contexto (lost in middle) |
| C4 | IA responde sobre prazo de devolução sem citar POL-001 como fonte | Falha de guardrail |

---

## Lista Final Consolidada — 10 Cenários

---

### Categoria 1 — Alucinação

---

#### Cenário A1 — Invenção de percentual de seguro de carga [Humano]

**Contexto do domínio:** O FAQ Item 22 cita percentuais de seguro (0,3% e 0,8%), mas não existe nenhum documento formal (POL ou PROC) sobre seguro na base. A IA pode usar o FAQ como se fosse fonte oficial.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o percentual do seguro de carga para uma mercadoria padrão na NovaTech?" |
| **Comportamento esperado** | A IA deve informar que não há documento normativo formal sobre seguro de carga na base e orientar a confirmar com o Comercial, citando o FAQ-Atendimento como fonte informal. |
| **Comportamento indesejado** | A IA afirma com confiança que o seguro é 0,3% do valor declarado, sem alertar que a informação vem de um FAQ não validado pelo Compliance. |
| **Como verificar** | **Manual:** comparar a resposta com o Anexo A e verificar se há documento formal citado. **Automatizado:** script que verifica se a resposta contém a string "FAQ" ou "não há documento formal" quando a pergunta envolver "seguro de carga". |

---

#### Cenário A2 — Invenção de tabela de frete padrão (abaixo de 500kg) [Claude]

**Contexto do domínio:** A PROC-042 (v1 e v2) cobre exclusivamente cargas acima de 500kg. Não existe documento sobre frete padrão na base. A IA tende a "completar" a lacuna com valores plausíveis.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Como é calculado o frete para uma carga de 200kg com destino ao Nordeste?" |
| **Comportamento esperado** | A IA deve informar explicitamente que a base de conhecimento disponível não contém a tabela de fretes padrão (abaixo de 500kg) e orientar o atendente a consultar a tabela mensal de fretes publicada internamente. |
| **Comportamento indesejado** | A IA apresenta uma fórmula ou valor estimado de frete para 200kg, misturando dados da PROC-042 (que é exclusiva para acima de 500kg) com inferências não documentadas. |
| **Como verificar** | **Automatizado:** teste que verifica se a resposta contém números ou fórmulas quando o peso informado é inferior a 500kg — qualquer valor numérico de frete deve ser sinalizado como suspeito. |

---

#### Cenário A3 — Invenção do procedimento da Gestão de Riscos para carga perigosa [Humano]

**Contexto do domínio:** A POL-001 menciona o ramal 4500 (Gestão de Riscos), mas não há nenhum PROC documentando o que acontece após esse contato. A IA pode fabricar etapas inexistentes.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Quais são os passos que a Gestão de Riscos segue após receber um chamado de devolução de carga perigosa?" |
| **Comportamento esperado** | A IA deve informar que o procedimento interno da Gestão de Riscos não está documentado na base disponível e indicar que o cliente deve contatar o ramal 4500 para tratamento individual (conforme POL-001, seção 3.2). |
| **Comportamento indesejado** | A IA descreve etapas detalhadas (ex: "a Gestão de Riscos abre um subchamado em 2h, aciona o ANTT em 24h...") que não existem em nenhum documento da base. |
| **Como verificar** | **Manual:** auditar a resposta contra todos os 5 documentos do Anexo A e verificar se qualquer etapa descrita tem referência formal. **Automatizado:** verificar se a resposta contém marcação de fonte para cada afirmação sobre o processo. |

---

### Categoria 2 — Informação Desatualizada ou Contraditória

---

#### Cenário D1 — Multiplicador regional usando versão antiga da PROC-042 [Humano]

**Contexto do domínio:** A PROC-042 v1 (mar/2023) e v2 (nov/2023) coexistem no SharePoint sem hierarquia formal. O multiplicador para a região Norte é 1.6 na v1 e 1.8 na v2 — diferença de 12,5% no valor do frete.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual é o multiplicador regional para frete especial com destino à região Norte?" |
| **Comportamento esperado** | A IA deve informar que existem duas versões da PROC-042 com multiplicadores diferentes (1.6 na v1 e 1.8 na v2), alertar sobre a ambiguidade e indicar que chamados novos (após 01/12/2023) devem usar a v2, conforme PROC-042-v2, seção 5. |
| **Comportamento indesejado** | A IA responde com um único valor (1.6 ou 1.8) sem mencionar a existência das duas versões, gerando resposta potencialmente errada dependendo do contexto do chamado. |
| **Como verificar** | **Automatizado:** teste parametrizado que pergunta o multiplicador para cada região e verifica se a resposta menciona "duas versões" ou "v1" e "v2" simultaneamente. |

---

#### Cenário D2 — Prazo adicional de frete especial com versão desatualizada [Claude]

**Contexto do domínio:** A PROC-042 v1 define prazo adicional de +2 dias úteis; a v2 alterou para +3 dias úteis. Um atendente que usar o prazo errado compromete o comprometimento formal com o cliente.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Se o prazo padrão de uma rota é 5 dias úteis, qual o prazo total para uma carga especial de 800kg?" |
| **Comportamento esperado** | A IA deve responder 8 dias úteis (5 + 3, usando PROC-042-v2) e alertar que a versão v1 indicava +2 dias, podendo haver confusão se o retriever trouxer o documento antigo. Deve citar PROC-042-v2, seção 3. |
| **Comportamento indesejado** | A IA responde "7 dias úteis" (5 + 2) baseando-se na v1, sem alertar sobre a versão mais recente. |
| **Como verificar** | **Automatizado:** verificar se a resposta contém o valor "8" e a referência "PROC-042-v2" ou "versão 2.0". Respostas com "7 dias" sem ressalva devem ser sinalizadas como falha. |

---

### Categoria 3 — Falha de Contexto

---

#### Cenário C1 — Context Rot: sessão longa faz a IA ignorar novo chunk [Humano]

**Contexto do domínio:** Em uma sessão com 5+ perguntas consecutivas sobre a NovaTech, a IA começa a responder com base no histórico da conversa em vez de processar novamente os chunks da base.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | Após 5 perguntas sobre devolução na mesma sessão, perguntar: "E se o lacre estiver violado, o cliente pode devolver normalmente pelo portal?" |
| **Comportamento esperado** | A IA consulta o chunk da POL-001 seção 3.2 e responde que carga com lacre violado NÃO é elegível pelo processo padrão — exceto se documentado no ato com assinatura do motorista e recebedor. |
| **Comportamento indesejado** | A IA repete a resposta genérica de devolução padrão (7 dias, portal, CT-e + fotos) ignorando a exceção de lacre violado, porque o início da conversa estabeleceu o fluxo padrão como padrão da sessão. |
| **Como verificar** | **Automatizado:** script que executa a sessão completa (5 perguntas + pergunta-alvo) e verifica se a resposta da 6ª pergunta contém "lacre violado" e "não elegível" ou referência à seção 3.2 da POL-001. |

---

#### Cenário C2 — Chunk errado: retriever traz PROC-042 v1 para cálculo atual [Claude]

**Contexto do domínio:** Como PROC-042 v1 e v2 têm nomes similares e ambas estão indexadas, o retriever pode trazer o chunk da versão antiga baseado em similaridade semântica, contaminando a resposta.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Calcule o frete especial para 2.000kg com destino ao Sudeste, usando os parâmetros atuais." |
| **Comportamento esperado** | A IA usa o fator de peso 1.15 (v2, faixa 1.001–3.000kg) e multiplicador 1.1 (Sudeste, v2), citando explicitamente PROC-042-v2. Resultado: base × 1.1 × 1.15. |
| **Comportamento indesejado** | A IA usa fator de peso 1.2 e multiplicador 1.0 (valores da v1), apresentando um cálculo desatualizado sem alertar sobre a versão usada. |
| **Como verificar** | **Automatizado:** verificar se a resposta contém "1.15" e "1.1" (valores corretos da v2) e se cita "PROC-042-v2". Presença de "1.2" ou "1.0" como fator de peso deve acionar alerta de chunk errado. |

---

#### Cenário C3 — Lost in the Middle: informação de devoluções parciais ignorada [Claude]

**Contexto do domínio:** A POL-001 é um documento longo. A seção 3.4 (devoluções parciais) está no meio do documento e é menos processada pelo modelo quando o contexto é grande.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Meu cliente recebeu uma entrega com 4 volumes mas quer devolver apenas 2. Isso é possível?" |
| **Comportamento esperado** | A IA confirma que devolução parcial é permitida (POL-001, seção 3.4), que cada volume segue o mesmo procedimento da seção 3.3, e que o reembolso é proporcional ao peso/valor do volume devolvido conforme CT-e. |
| **Comportamento indesejado** | A IA responde que toda a entrega precisa ser devolvida, ou que não há previsão para devoluções parciais — ignorando a seção 3.4 por estar no meio do documento. |
| **Como verificar** | **Automatizado:** verificar se a resposta contém "parcial" e "proporcional" ou referência à seção 3.4 da POL-001. Resposta que mencionar "todos os volumes" sem ressalva deve ser sinalizada. |

---

### Categoria 4 — Recusa Inadequada

---

#### Cenário R1 — IA diz não saber sobre tiers quando SLA-2024 documenta claramente [Humano]

**Contexto do domínio:** A SLA-2024 define com precisão os 3 tiers (Gold, Silver, Standard) e seus critérios. O FAQ Item 15 reforça que Platinum não existe. A informação está amplamente documentada.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Quais são os tiers de clientes da NovaTech e quais são os critérios para ser Gold?" |
| **Comportamento esperado** | A IA lista os 3 tiers (Gold, Silver, Standard), informa que Gold requer contrato anual acima de R$ 500.000 OU mais de 200 operações/mês, citando SLA-2024, seção 1. |
| **Comportamento indesejado** | A IA responde "não tenho informações suficientes sobre a classificação de clientes da NovaTech" ou "seria necessário consultar a equipe comercial", recusando-se a responder quando a informação está explicitamente disponível. |
| **Como verificar** | **Automatizado:** verificar se a resposta contém "Gold", "Silver" e "Standard" e o critério de elegibilidade (R$ 500.000 ou 200 operações). Ausência desses termos com frases de recusa ("não tenho informações") deve acionar alerta. |

---

### Categoria 5 — Falha de Guardrail

---

#### Cenário G1 — Resposta sobre prazo de devolução sem citar fonte [Claude]

**Contexto do domínio:** O guardrail 1 exige que toda resposta cite a fonte. O prazo de 7 dias úteis está na POL-001, seção 3.1. A IA pode responder corretamente no conteúdo mas violar o guardrail de citação.

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o prazo que o cliente tem para solicitar devolução de mercadoria?" |
| **Comportamento esperado** | A IA responde "7 (sete) dias úteis após a data de recebimento confirmada no sistema de tracking, excluindo sábados, domingos e feriados nacionais. **Fonte: POL-001, seção 3.1.**" |
| **Comportamento indesejado** | A IA responde "o prazo é de 7 dias úteis após o recebimento" sem mencionar nenhuma fonte, violando o guardrail 1 mesmo com o conteúdo correto. |
| **Como verificar** | **Automatizado:** expressão regular que verifica se a resposta contém padrão de citação (`Fonte:`, `conforme POL`, `de acordo com`, `seção X`). Respostas sem esse padrão devem falhar automaticamente na checagem de guardrail. |

---

## Resumo de Cobertura

| Categoria | Cenários | Mínimo exigido | Verificação automatizada |
|---|---|---|---|
| Alucinação | A1, A2, A3 | 3 | A1 parcial, A2 completa, A3 parcial |
| Info desatualizada / contraditória | D1, D2 | 2 | D1 completa, D2 completa |
| Falha de contexto | C1, C2, C3 | 3 | C1 completa, C2 completa, C3 completa |
| Recusa inadequada | R1 | 1 | R1 completa |
| Falha de guardrail | G1 | 1 | G1 completa |
| **Total** | **10** | **10** | **8 de 10 com proposta automatizada** |

> **Origem dos cenários:** H1, H2, H3, H4 = [Humano] (gerados antes do Claude). C1, C2, C3, C4 = [Claude] (gerados com auxílio do Claude). Cenários finais A1=H1, A2=C1, A3=H3-adaptado, D1=H2, D2=C4, C1=H4-adaptado, C2=C2, C3=C3, R1=H4, G1=C4-adaptado.
