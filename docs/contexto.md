# Contexto e Motivação

## O que foi feito nos Mini Trabalhos anteriores

Nos MTs 4, 5 e 6 foi desenvolvido um modelo de **regressão** para prever o `FRP_log` (Fire Radiative Power em escala logarítmica) de focos de incêndio já detectados pelo satélite. O melhor resultado foi:

- **Random Forest:** R² = 0.7282 no conjunto holdout (após otimização no MT6)

## Limitação identificada

O modelo de regressão tem valor limitado em produção porque o BDqueimadas/INPE já fornece o FRP medido pelo satélite junto com todas as outras colunas usadas como features. Ou seja, quando você tem os dados para rodar o modelo, já tem a resposta que ele prevê.

## Redefinição do problema

Para o PBL, o problema foi redefinido para algo genuinamente útil:

> **"Dado o histórico de queimadas e as condições climáticas atuais, quais áreas de Goiás têm maior probabilidade de registrar focos de incêndio nos próximos dias?"**

Isso é **classificação binária** por unidade geográfica por dia, **antes** de qualquer foco existir.

## Por que dois estágios

Os dois modelos rodam de forma **independente** com as mesmas features climáticas. Ambos são executados diariamente pelo cron job. O "dois estágios" é uma decisão de **apresentação ao usuário**, não de pipeline em cascata — a saída do Estágio 1 não entra no Estágio 2.

```
Cron job diário
    │
    ├─► Modelo 1 (municipio_full.pkl)
    │     Features climáticas por município → P(fogo) × 244 municípios
    │     Saída: ranking — "NIQUELÂNDIA 98%, CAVALCANTE 95%..."
    │
    └─► Modelo 2 (grade_full.pkl)
          Features climáticas por célula → P(fogo) × 2.976 células
          Saída: probabilidade para cada célula 0.1° de Goiás

Frontend (apresentação em dois estágios):
  Estágio 1 → ranking de municípios (visão geral, alerta regional)
  Estágio 2 → usuário clica num município → zoom no mapa
               células daquele município coloridas por risco relativo
```

| Aspecto | Município | Grade 0.1° |
|---|---|---|
| AUC validação 2026 | 0.816 | 0.710 |
| Unidades | 244 | 2.976 |
| Resolução geográfica | ~50km | ~11km |
| Custo computacional | Baixo | Alto |
| Uso em produção | Alerta diário | Visualização, planejamento |

O modelo de município é mais preciso porque cada município agrega muitos focos históricos (sinal forte). O modelo de grade é menos preciso mas oferece localização dentro do município — crucial para operações de campo.
