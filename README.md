# vigIA — Previsão de Risco de Incêndio em Goiás

> Acompanhe o risco de focos de incêndio nos municípios de Goiás nos próximos 5 dias, com mapa interativo atualizado diariamente.

**Acesse:** [vigIA](https://vig-ia.vercel.app/)

![Tela principal do vigIA](./assets/telaPadrao.png)

---

## O que é

O vigIA é um sistema de previsão de risco de queimadas para o estado de Goiás. A cada dia, modelos de aprendizado de máquina analisam dados climáticos e históricos de focos de incêndio para estimar a probabilidade de ocorrência de fogo nos próximos 5 dias — por município e por célula espacial de ~11 km².

## Como usar

No mapa, cada município é colorido de acordo com o risco relativo do dia — a posição de cada área no ranking comparado a todos os municípios de Goiás:

- **Vermelho** — alto risco (top 10%)
- **Âmbar** — atenção (top 10–30%)
- **Verde** — calmo (demais 70%)

O ranking lateral exibe cada município com sua posição relativa ("top 5%", "top 30%", etc.) e reordena ao trocar o dia. Clique em qualquer município para ver o detalhamento espacial interno (células de ~11 km²).

## Atualização

O boletim é gerado automaticamente todos os dias às 03h (horário de Brasília) com base nos dados climáticos mais recentes. Nenhuma ação é necessária.

## Escopo e limitações

- Cobre 244 municípios de Goiás no bioma Cerrado
- Goiás possui 246 municípios; Gouvelândia e São Simão (bioma Mata Atlântica) estão fora do escopo do modelo e aparecem sinalizados no mapa
- O risco exibido é **relativo** — indica a posição no ranking do dia, não uma probabilidade absoluta de ocorrência
- Modelos treinados com dados de 2015 a 2025 (INPE BDqueimadas + Open-Meteo)

---

*FGA0083 — Aprendizado de Máquina | UnB 2026-1 | Turma 01 | Grupo 4*  
*Felipe Rodrigues · João Paulo Cristo · Guilherme Vilela · Luiz Guilherme Faria*
