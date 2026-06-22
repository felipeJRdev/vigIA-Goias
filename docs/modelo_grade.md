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
| AUC-ROC 2026 | **0,811** | 0,711 |
| PR-AUC 2026 | **0,232** | 0,015 |
| Brier score | 0,094 | 0,079 |
| Recall (limiar 0,5) | **46,9%** | 26,7% |
| Recall (limiar 0,3) | **73,8%** | 60,8% |

O gap em relação ao município é estrutural: células 0,1° têm menos histórico individual e o problema é intrinsecamente mais difícil (11km vs município inteiro). O PR-AUC de 0,015 reflete o desbalanceamento extremo da grade (~2% positivos).

## Métricas por (célula, dia) — Jan-Jun 2026

Métricas calculadas para cada par (célula, dia), avaliando se a probabilidade prevista corresponde à ocorrência real de foco:

| Métrica | Limiar 0,5 |
|---|---|
| Verdadeiros positivos | 613 |
| Falsos positivos | 37.611 |
| Verdadeiros negativos | 418.400 |
| Falsos negativos | 1.680 |
| Recall | 26,7% |
| Precisão | 1,6% |
| F1 | 0,030 |

Com limiar 0,3 o recall sobe para **60,8%**. A precisão baixa é esperada no nível de célula — com 2.976 células × 5 dias e apenas 2% de positivos, mesmo um modelo preciso gera muitos falsos positivos em valor absoluto. O uso correto do Estágio 2 é **dentro de um município já sinalizado** pelo Estágio 1, reduzindo o espaço de busca de ~2.976 para ~10–50 células.

> **Nota:** o limiar não faz parte do modelo — é aplicado após a previsão e pode ser ajustado sem retreinar.

## Previsão 5 dias (produção)

O Estágio 2 reutiliza os dados climáticos já coletados para o Estágio 1 — cada célula 0,1° herda o `DiaSemChuva` e `Precipitacao` do município proxy mais próximo. Custo adicional em produção: **zero chamadas extras à API**.
