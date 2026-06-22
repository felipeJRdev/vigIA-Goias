# Fonte de Dados

## BDqueimadas / INPE

Sistema de monitoramento de queimadas por satélite.

- **Escopo:** Estado de Goiás | 2015–2025
- **Registros brutos:** 1.390.827 focos detectados
- **Após filtro (Goiás + remoção Mata Atlântica):** 1.354.326 registros | 244 municípios

Goiás possui 246 municípios. Dois foram excluídos do escopo do modelo: **Gouvelândia** e **São Simão**, que pertencem ao bioma Mata Atlântica. O modelo foi treinado exclusivamente com dados do bioma Cerrado — incluir municípios de outro bioma introduziria padrões de vegetação e sazonalidade incompatíveis com os 244 municípios do conjunto de treino, degradando a qualidade das previsões. No frontend, esses dois municípios aparecem sinalizados separadamente no mapa.

## Open-Meteo API

Dados climáticos históricos e de previsão, gratuitos (sem autenticação).

- Precipitação diária por coordenada geográfica
- Período: 2015–2025 (treino) + Janeiro–Junho 2026 (validação) + uso diário em produção

Em produção, uma única chamada por município retorna 30 dias de histórico + 6 dias de previsão (`past_days=30, forecast_days=6`), eliminando a necessidade de combinar Archive API e Forecast API separadamente.

## Estratégia de negativos (exemplos sem fogo)

O BDqueimadas registra **apenas** dias com foco detectado. Para treinar um classificador binário são necessários exemplos de dias **sem** fogo. A premissa adotada:

> **Ausência de registro no BDqueimadas = ausência de foco detectado naquele município naquele dia.**

Isso é válido porque o sistema de satélites cobre Goiás continuamente.

### Dataset de município

```
244 municípios × 4.018 dias (2015-2025) = 980.392 combinações
980.392 − 143.067 positivos = 837.325 negativos (naturais, sem amostragem)
```

| Classe | Registros | % |
|---|---|---|
| fogo = 1 | 143.067 | 14,6% |
| fogo = 0 | 837.325 | 85,4% |
| **Total** | **980.392** | 100% |

### Dataset de grade

Na grade espacial, a taxa de positivos é muito menor — células pequenas têm poucos focos históricos:

| Classe | Registros | % |
|---|---|---|
| fogo = 1 | 361.516 | 2,0% |
| fogo = 0 | 18.032.888 | 98,0% |
| **Total** | **18.394.404** | 100% |

Uma tentativa anterior com `NEG_RATIO=4` inflou a taxa de positivos para 20%, causando calibração errada na validação com 2026 (0,5% de positivos na chuva) → AUC 0,69. A solução foi usar os negativos naturais completos.
