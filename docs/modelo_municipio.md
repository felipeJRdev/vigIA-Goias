# Estágio 1 — Modelo de Município

## Features

| Feature | Tipo | Fonte |
|---|---|---|
| `Mes` | Temporal | Data |
| `DiaSemana` | Temporal | Data |
| `Estacao_Seca` | Temporal | Mes ∈ {6,7,8,9,10} → 1 |
| `Latitude` | Geográfica | Centroide do município |
| `Longitude` | Geográfica | Centroide do município |
| `Municipio_Freq` | Histórica | Focos do município / total Goiás |
| `DiaSemChuva` | Climática | Dias consecutivos < 0.1mm (Open-Meteo) |
| `Precipitacao` | Climática | Precipitação diária em mm (Open-Meteo) |
| `media_focos_mes_hist` | Histórica | Média histórica de focos por município+mês |

## Split temporal

Random split causaria data leakage — o split é obrigatoriamente temporal:

| Conjunto | Período | Registros |
|---|---|---|
| Treino | 2015–2022 | ~712.000 |
| Validação | 2023 | ~89.000 |
| Teste | 2024–2025 | ~179.000 |

## Comparação de modelos

Busca de hiperparâmetros: RandomizedSearchCV em subsample de 200k + retreino no conjunto completo.

| Modelo | AUC Val | AUC Teste | Recall Teste | Precisão Teste |
|---|---|---|---|---|
| **LightGBM** | 0,8412 | **0,8350** | **0,750** | 0,329 |
| XGBoost | 0,8411 | 0,8343 | 0,710 | 0,350 |
| Random Forest | 0,8382 | 0,8297 | 0,740 | 0,326 |

**Modelo selecionado: LightGBM** — maior AUC + maior Recall.

```python
LGBMClassifier(
    n_estimators=200, learning_rate=0.05, num_leaves=50,
    max_depth=-1, subsample=1.0, colsample_bytree=0.8,
    class_weight='balanced', random_state=42
)
```

## Validação com dados de 2026

Retreinado nas amostras completas (2015–2025) → `modelos/municipio_full.pkl`

| Métrica | Valor |
|---|---|
| AUC-ROC | **0,816** |
| Recall (limiar 0,5) | 47,4% |
| Recall (limiar 0,3) | 73,8% |
| Precisão (limiar 0,3) | 12,7% |

O recall cai em Jan–Jun porque é a estação chuvosa — focos são raros (4,9% dos dias vs 14% no treino). O **AUC de 0,816** confirma que a ordenação dos municípios permanece correta mesmo na chuva.

## Análise município-nível (limiar 0.3, Jan-Jun 2026)

Um município é considerado "alertado" se tiver ao menos um dia com probabilidade ≥ 0,3 no período.

| Métrica | Valor |
|---|---|
| Municípios alertados | 172 de 244 (70%) |
| Recall | **79,2%** — 164 de 207 municípios com fogo foram alertados |
| Precisão | **95,3%** — 164 de 172 alertas eram municípios que realmente queimaram |
| Falsos negativos | 43 municípios com fogo não alertados |
| Falsos positivos | 8 municípios alertados sem fogo |

> **Nota sobre a precisão de 95,3%:** este valor está inflado — um município é considerado "correto" se teve ao menos um foco em 6 meses de período. A métrica honesta é a curva de captura (PR-AUC e Captura top N%), que mede dia a dia.

## Curva de captura

Priorizando pares (município, dia) com maior probabilidade:

- Top 10% → captura **43,6%** dos fogos reais (4,4× melhor que aleatório)
- Top 20% → captura **62,7%** dos fogos reais

## Comparação com baselines de climatologia

| Ranqueador | AUC | Captura top 10% | Captura top 20% |
|---|---|---|---|
| `media_focos_mes_hist` | ~0,73 | — | — |
| `Municipio_Freq` | ~0,749 | — | — |
| `media + DiaSemChuva` | ~0,75 | — | — |
| **Modelo LightGBM** | **0,816** | **43,6%** | **62,7%** |

Delta sobre o melhor baseline: **+0,067** (threshold mínimo adotado: 0,02). O `Municipio_Freq` é o baseline mais forte; a frequência histórica já carrega muito sinal, mas o modelo adiciona sazonalidade climática que os baselines ignoram.

## Previsão 5 dias

Para cada previsão diária, uma **única chamada à API** por município retorna 30 dias de histórico + 6 dias de previsão:

1. Open-Meteo Forecast (`past_days=30, forecast_days=6`) → 36 valores de precipitação por município
2. Dias 1–30: calculam `DiaSemChuva` acumulado atual
3. Dias 32–36 (amanhã → hoje+5): projetam `DiaSemChuva` para cada dia futuro
4. `municipio_full.pkl.predict_proba()` → probabilidade por (município, dia)
5. Ranking final dos 244 municípios para os próximos 5 dias
