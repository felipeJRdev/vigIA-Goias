# Estágio 2 — Modelo de Grade Espacial

## Motivação

O modelo de município diz *qual município* está em risco, mas não *onde dentro do município*. Um município goiano pode ter 3.000–15.000 km² — informação insuficiente para operações de campo.

A grade espacial divide Goiás em células de **0,1° × 0,1° (~11km × 11km)**, com 2.976 células que tiveram ao menos 5 focos históricos.

## Features

| Feature | Tipo | Diferença do município |
|---|---|---|
| `Mes`, `DiaSemana`, `Estacao_Seca` | Temporal | Igual |
| `Cell_Lat`, `Cell_Lon` | Geográfica | Centro da célula 0,1° |
| `Cell_Freq` | Histórica | Focos da célula / total Goiás |
| `DiaSemChuva`, `Precipitacao` | Climática | Open-Meteo no ponto 0,5° mais próximo |
| `media_focos_mes_hist` | Histórica | Média de focos por célula+mês |

## Clima por grade 0,5°

Em vez de baixar clima para as 2.976 células (2h+, rate limit inviável), é usada uma grade climática intermediária de 148 pontos únicos que cobrem todo Goiás:

| Estratégia | Chamadas API | Erro máximo | Tempo |
|---|---|---|---|
| Município proxy | 244 | ~80km | 6 min |
| **Grade 0.5° (adotada)** | **148** | **~35km** | **~10 min** |
| Por célula 0.1° | 2.976 | ~0km | 2h+ (rate limit) |

Cada célula 0,1° usa o ponto 0,5° mais próximo. O dataset final ficou com 18.394.404 linhas porque o clima exato permitiu mapear mais células sem valores nulos.

## Comparação de modelos

| Modelo | AUC Val | AUC Teste | Recall Teste |
|---|---|---|---|
| XGBoost (GPU) | 0,8086 | **0,8314** | 0,770 |
| **LightGBM** | 0,8045 | 0,8285 | **0,761** |
| Random Forest | 0,8051 | 0,8288 | 0,753 |

LightGBM selecionado por recall superior e por não requerer CUDA em produção.

```python
LGBMClassifier(
    n_estimators=200, learning_rate=0.01, num_leaves=31,
    max_depth=10, subsample=0.8, colsample_bytree=1.0,
    class_weight='balanced', random_state=42
)
```

## Validação com dados de 2026

Retreinado em 18.394.404 amostras (2015–2025) → `modelos/grade_full.pkl`

| Métrica | Município | Grade (clima 0,5°) |
|---|---|---|
| AUC-ROC 2026 | **0,816** | 0,710 |
| Recall (limiar 0,5) | **47,4%** | 41,6% |
| Recall (limiar 0,3) | **73,8%** | 66,6% |

O gap em relação ao município é estrutural: células 0,1° têm menos histórico individual e o problema é intrinsecamente mais difícil (11km vs município inteiro).

## Análise célula-nível (limiar 0.6, Jan-Jun 2026)

Uma célula é "alertada" se tiver ao menos um dia com probabilidade ≥ 0,6 no período.

| Métrica | Valor |
|---|---|
| Células alertadas | 1.825 de 2.976 (61%) |
| Recall | **70,3%** — 780 de 1.109 células com fogo foram alertadas |
| Precisão | **42,7%** — 780 de 1.825 alertas eram células que realmente queimaram |
| Falsos negativos | 329 células com fogo não alertadas |
| Falsos positivos | 1.045 células alertadas sem fogo confirmado |

**Curva de captura:**

- Top 10% → captura **30,3%** dos fogos reais (3× melhor que aleatório)
- Top 20% → captura **48,5%** dos fogos reais

> **Nota:** o limiar não faz parte do modelo — é aplicado após a previsão e pode ser ajustado sem retreinar.

## Previsão 5 dias (produção)

O Estágio 2 reutiliza os dados climáticos já coletados para o Estágio 1 — cada célula 0,1° herda o `DiaSemChuva` e `Precipitacao` do município proxy mais próximo. Custo adicional em produção: **zero chamadas extras à API**.
