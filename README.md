# dgs-ai-first
 Cenário 1 — Cenários de Falha de IA (NovaTech)

> **Branch:** `cenario-1`

## Objetivo

Mapear cenários de falha de um assistente de IA (RAG) sobre a base documental simulada da **NovaTech**, definindo, para cada caso, a pergunta de teste, o comportamento esperado, o comportamento indesejado e a forma de verificação (manual e/ou automatizada).

## Fonte de Verdade

Anexo A — Documentação Simulada: `POL-001`, `PROC-042 v1`, `PROC-042 v2`, `SLA-2024`, `FAQ-Atendimento`.

## Guardrails

1. Sempre citar a fonte.
2. Nunca inventar prazos ou valores.
3. Dizer explicitamente quando não souber.
4. Responder em português formal.

## Metodologia

- **Lista inicial [Humano]:** cenários levantados por leitura crítica da documentação, antes de consultar a IA.
- **Lista adicional [Claude]:** cenários complementares gerados com auxílio do Claude.
- **Lista final consolidada:** 10 cenários distribuídos em 5 categorias.

## Cobertura — 10 Cenários

| Categoria | Cenários | Qtd | Verificação automatizada |
|---|---|---|---|
| Alucinação | A1, A2, A3 | 3 | 2 de 3 |
| Informação desatualizada / contraditória | D1, D2 | 2 | 2 de 2 |
| Falha de contexto (context rot, chunk errado, lost in the middle) | C1, C2, C3 | 3 | 3 de 3 |
| Recusa inadequada | R1 | 1 | 1 de 1 |
| Falha de guardrail | G1 | 1 | 1 de 1 |
| **Total** | | **10** | **8 de 10** |

## Conteúdos Abordados

- Fundamentos de IA Generativa
- Engenharia de Prompt
- Engenharia de Contexto
- RAG e MCP

## Ferramentas e Restrições

- Exercícios feitos com **GitHub Copilot**, **Claude** ou **Claude Code** (usar a licença disponível).
- **Não utilizar dados de clientes** — apenas a documentação simulada.

## Arquivos

- [cenarios-falha-ia-1.md](cenarios-falha-ia-1.md) — detalhamento completo dos 10 cenários (pergunta de teste, comportamentos esperado/indesejado e critérios de verificação).
