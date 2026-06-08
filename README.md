# dgs-ai-first
# Cenário 2 — Design de Critérios de Aceitação para Respostas de IA

> **Branch:** `cenario-2`
> **Papel:** QA

## Objetivo

Definir quando uma resposta do assistente de IA (RAG) é "boa o suficiente", criando uma rubrica de avaliação, um template reutilizável e aplicando-os a um lote de 5 pares de pergunta/resposta simulados da **NovaTech**.

## Fonte de Verdade

Anexo A — Documentação Simulada: `POL-001`, `PROC-042 v1`, `PROC-042 v2`, `SLA-2024`, `FAQ-Atendimento`.

## Guardrails

1. Sempre citar a fonte.
2. Nunca inventar prazos ou valores.
3. Dizer explicitamente quando não souber.
4. Responder em português formal.

## Entregáveis

1. **Avaliação manual** das 5 respostas com base direta no Anexo A (antes do uso de IA).
2. **Rubrica de avaliação** com 4 dimensões (escala 1–3 cada; total de 4 a 12 pontos).
3. **Template reutilizável** de avaliação para qualquer resposta do assistente.
4. **Pontuações aplicadas** às 5 respostas, com detalhamento das reprovadas.

## Rubrica — 4 Dimensões

| Dimensão | O que avalia |
|---|---|
| D1 — Precisão Factual | Conteúdo correto e sem contradição com a documentação. |
| D2 — Citação de Fonte | Fonte presente, correta e verificável (documento + seção). |
| D3 — Aderência aos Guardrails | Respeito aos 4 guardrails do projeto. |
| D4 — Completude | Cobertura das informações necessárias, sem omissões de risco. |

**Classificação:** Aprovada (10–12) · Revisão (7–9) · Reprovada (até 6).

## Resultado do Lote (5 respostas)

| Classificação | Qtd | % |
|---|---|---|
| Aprovada | 1 | 20% |
| Revisão | 2 | 40% |
| Reprovada | 2 | 40% |

> 40% de reprovação indica risco operacional alto, concentrado em alucinação (tier "Platinum" inexistente) e inversão de regra crítica de segurança (devolução de carga perigosa).

## Conteúdos Abordados

- Engenharia de Prompt
- Engenharia de Contexto
- RAG e MCP

## Ferramentas e Restrições

- Exercícios feitos com **Claude (chat)** e **Claude Cowork** (usar a licença disponível; também válidos GitHub Copilot ou Claude Code).

## Arquivos

- [cenario-2-criterios-aceitacao-respostas-ia.md](cenario-2-criterios-aceitacao-respostas-ia.md) — detalhamento completo (avaliação manual, rubrica, template e pontuações aplicadas).
