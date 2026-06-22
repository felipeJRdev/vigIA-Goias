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
| AUC-ROC | **0,811** |
| PR-AUC | **0,232** |
| Brier score | 0,094 |
| Recall (limiar 0,5) | 46,9% |
| Recall (limiar 0,3) | 73,8% |
| Precisão (limiar 0,5) | 20,0% |

O recall cai em Jan–Jun porque é a estação chuvosa — focos são raros (4,9% dos dias vs 14% no treino). O **AUC de 0,811** confirma que a ordenação dos municípios permanece correta mesmo na chuva. O PR-AUC de 0,232 reflete o forte desbalanceamento de classes (~85% negativos).

## Métricas por (município, dia) — Jan-Jun 2026

Métricas calculadas para cada par (município, dia), avaliando se a probabilidade prevista naquele dia corresponde à ocorrência real de fogo:

| Métrica | Limiar 0,5 |
|---|---|
| Verdadeiros positivos | 880 |
| Falsos positivos | 3.520 |
| Verdadeiros negativos | 32.181 |
| Falsos negativos | 995 |
| Recall | 46,9% |
| Precisão | 20,0% |
| F1 | 0,280 |

Com limiar 0,3 o recall sobe para **73,8%** (mais fogos detectados, mais falsos alarmes). O limiar operacional é escolhido conforme o trade-off de custo de alerta vs custo de miss para o uso em campo.

## Curva de captura

Priorizando pares (município, dia) com maior probabilidade prevista:

- Top 10% → captura **22,3%** dos fogos reais (2,2× melhor que aleatório)
- Top 20% → captura **33,5%** dos fogos reais
- Top 30% → captura **42,9%** dos fogos reais

## Comparação com baselines de climatologia

| Ranqueador | AUC | Captura top 10% | Captura top 20% |
|---|---|---|---|
| `media_focos_mes_hist` | 0,721 | 12,3% | 20,8% |
| `Municipio_Freq` | 0,745 | 19,2% | 28,2% |
| `media + DiaSemChuva` | 0,723 | 12,3% | 20,9% |
| **Modelo LightGBM** | **0,811** | **22,3%** | **33,5%** |

Delta sobre o melhor baseline (`Municipio_Freq`): **+0,066 AUC** (limiar mínimo adotado: 0,02). A frequência histórica já carrega sinal forte, mas o modelo adiciona sazonalidade climática que os baselines ignoram.

## Previsão 5 dias

Para cada previsão diária, uma **única chamada à API** por município retorna 30 dias de histórico + 6 dias de previsão:

1. Open-Meteo Forecast (`past_days=30, forecast_days=6`) → 36 valores de precipitação por município
2. Dias 1–30: calculam `DiaSemChuva` acumulado atual
3. Dias 32–36 (amanhã → hoje+5): projetam `DiaSemChuva` para cada dia futuro
4. `municipio_full.pkl.predict_proba()` → probabilidade por (município, dia)
5. Ranking final dos 244 municípios para os próximos 5 dias
