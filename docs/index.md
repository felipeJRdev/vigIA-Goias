# vigIA — Previsão de Risco de Incêndio em Goiás

> Sistema de aprendizado de máquina para previsão diária de risco de queimadas nos municípios de Goiás, com mapa interativo atualizado automaticamente.

**Acesse:** [vig-ia.vercel.app](https://vig-ia.vercel.app/)

---

## O que é

O vigIA combina dois modelos LightGBM em estágios complementares para estimar, **antes da detecção por satélite**, a probabilidade de focos de incêndio nos próximos 5 dias:

- **Estágio 1 — Município:** ranqueia os 244 municípios de Goiás por risco relativo diário (AUC 0,811)
- **Estágio 2 — Grade 0,1°:** detalha a localização dentro de cada município com resolução de ~11 km² (AUC 0,711)

O boletim é gerado automaticamente todo dia às 03h Brasília pelo GitHub Actions e publicado no Vercel sem intervenção humana.

## Resultados principais

| | Estágio 1 — Município | Estágio 2 — Grade 0,1° |
|---|---|---|
| AUC Validação 2026 | **0,811** | 0,711 |
| PR-AUC Validação 2026 | **0,232** | 0,015 |
| Recall (limiar 0,3, dia a dia) | **73,8%** | 60,8% |
| Top 10% captura | **22,3%** dos fogos | — |
| Ganho sobre melhor baseline | **+0,066 AUC** | — |

## Equipe

**FGA0083 — Aprendizado de Máquina | UnB 2026-1 | Turma 01 | Grupo 4**

| Membro | Matrícula |
|---|---|
| Felipe de Jesus Rodrigues | 211062867 |
| João Paulo Barros de Cristo | 202023805 |
| Guilherme Aguera de la Fuente Vilela | 190088168 |
| Luiz Guilherme Morais da Costa Faria | 231011696 |

## Organização desta documentação

| Seção | Conteúdo |
|---|---|
| [Contexto e Motivação](contexto.md) | Redefinição do problema dos MTs 4-6 para previsão antecipada |
| [Fonte de Dados](dados.md) | BDqueimadas/INPE + Open-Meteo — escopo, período, volumes |
| [Estágio 1 — Municípios](modelo_municipio.md) | Dataset, modelagem, validação 2026 e análise de captura |
| [Estágio 2 — Grade Espacial](modelo_grade.md) | Grade 0,1°, clima 0,5°, comparação com Estágio 1 |
| [Arquitetura e Pipeline](arquitetura.md) | Pipeline leve, lookup tables, GitHub Actions, Vercel |
| [Frontend Interativo](frontend.md) | Mapa Leaflet, risco relativo (percentil), filtro top X% |
| [Decisões e Resultados](decisoes_tecnicas.md) | Tabela de decisões técnicas e resultados consolidados |
| [Prontidão para Produção](relatorio_prontidao.md) | Estratégia de lançamento, testes, segurança, Go/No-Go |
| [Plano de Monitoramento](plano_de_monitoramento.md) | Métricas, limiares, alertas e cadência de revisão |
| [Estratégia de Manutenção](estrategia_de_manutencao.md) | Retreino, runbook de incidentes, roadmap de evolução |
| [Glossário](glossario.md) | Definição de termos técnicos e siglas utilizados nesta documentação |
