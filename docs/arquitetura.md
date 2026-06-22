# Arquitetura e Pipeline

## Visão geral

A arquitetura é deliberadamente simples — o problema é **batch** (uma previsão por dia), então não há necessidade de servidor de aplicação, banco de dados ou inferência em tempo real.

```
┌─────────────────────────────────────────────────────────┐
│  ESTÁGIO 1 — Modelo de Município (triagem regional)     │
│  Unidade: 244 municípios de Goiás (bioma Cerrado)       │
│  Modelo:  modelos/municipio_full.pkl (LightGBM)         │
│  AUC:     0.816 (validação Jan-Jun 2026)                │
│  Pergunta: "QUAIS municípios estão em risco?"           │
└─────────────────────────────────────────────────────────┘
                    roda em paralelo ↕ (independente)
┌─────────────────────────────────────────────────────────┐
│  ESTÁGIO 2 — Modelo de Grade Espacial (drill-down)      │
│  Unidade: 2.976 células 0.1° × 0.1° (~11km × 11km)     │
│  Modelo:  modelos/grade_full.pkl (LightGBM)             │
│  AUC:     0.710 (validação 2026 com clima proxy)        │
│  Pergunta: "ONDE dentro do município?"                  │
└─────────────────────────────────────────────────────────┘
```

## Fluxo completo de produção

```
GitHub Actions (cron, 03h Brasília)
    │
    ├─► Open-Meteo Forecast (past_days=30, 244 municípios, ~6 min)
    │       ↓ DiaSemChuva projetado para os próximos 5 dias
    │
    ├─► modelos/municipio_full.pkl  →  P(fogo) × 244 municípios × 5 dias
    │
    ├─► modelos/grade_full.pkl  →  P(fogo) × 2.976 células × 5 dias
    │   (usa clima dos municípios proxy — 0 chamadas extras à API)
    │
    ├─► exportar_json.py  →  forecast.json
    │
    └─► git commit + push
            │
            └─► Vercel redeploy automático
                    │
                    └─► https://vig-ia.vercel.app/  (atualizado em ~30s)
```

## Problema: datasets pesados no pipeline

Os scripts de treinamento dependem de dois datasets pesados:

| Arquivo | Tamanho |
|---|---|
| `dataset_municipio.csv` | 1,6 GB |
| `dataset_grade.csv` | 1,1 GB |

Distribuir esses arquivos no repositório ou baixá-los a cada execução no CI é inviável.

## Solução: Lookup Tables

O script `gerar_lookups.py` extrai de cada dataset apenas a coluna que não pode ser calculada em tempo real: a média histórica de focos por (unidade geográfica, mês).

| Arquivo | Tamanho | Linhas |
|---|---|---|
| `dados/lookup_municipio.csv` | 85 KB | 2.964 |
| `dados/lookup_grade.csv` | 937 KB | 29.375 |

Todas as outras features são calculadas em tempo real a partir das chamadas à API climática.

## Script de previsão leve (`previsao_leve.py`)

Pipeline completo E1 + E2 sem dependência dos datasets de treino:

```
Entrada:
  modelos/municipio_full.pkl  (1,1 MB)
  modelos/grade_full.pkl      (314 KB)
  dados/mapeamento_municipio.csv + mapeamento_grade.csv
  dados/lookup_municipio.csv  + lookup_grade.csv
  Open-Meteo API (244 chamadas paralelas, ~6 min)

Saída:
  resultados/previsao_municipio_<hoje>.csv  (inclui prob_fogo, percentil, risco)
  resultados/previsao_grade_<hoje>.csv      (inclui prob_fogo, percentil, risco)
```

**Estratégia de chamadas à API:**

- `ThreadPoolExecutor(max_workers=8)` — 244 municípios em paralelo
- Retry automático com backoff exponencial por chamada (4 tentativas)
- Retry sequencial global (3 rounds, 30s entre rounds) para municípios que falharam
- `sys.exit(1)` se algum município permanecer sem dados — impede commit de previsão incompleta

## GitHub Actions (`previsao_diaria.yml`)

```
Cron: 0 6 * * *  →  03h00 Brasília (UTC-3)
  1. actions/checkout@v4
  2. pip install -r pbl/requirements.txt
  3. python pbl/previsao_leve.py          → previsao_municipio_*.csv + previsao_grade_*.csv
  4. python pbl/exportar_json.py          → pbl/forecast.json
  5. cp pbl/forecast.json frontend/       → atualiza frontend
  6. git commit + push                    → aciona redeploy automático no Vercel
```

Disparo manual disponível via `workflow_dispatch` para reprocessamento sob demanda.

## Deploy — Vercel

- Root Directory: `frontend/` (sem build command — arquivos estáticos)
- Cada push com novo `forecast.json` aciona redeploy automático em ~30s
- Branches `docs` e `gh-pages` estão excluídas do deploy automático via `vercel.json`

## Arquivos do projeto

### Scripts de desenvolvimento — `estagio1_municipio/`

| Arquivo | Descrição |
|---|---|
| `fase1_dataset.py` | Constrói dataset_municipio.csv |
| `fase1b_clima.py` | Baixa Open-Meteo histórico (244 municípios, escrita progressiva) |
| `fase1d_clima_2026.py` | Baixa Open-Meteo de Jan–Jun 2026 para validação |
| `fase2_modelagem.py` | Treina RF, XGBoost, LightGBM com busca de hiperparâmetros |
| `fase3_validacao_2026.py` | Retreino completo + validação 2026 |
| `fase3c_baseline.py` | Compara modelo LightGBM vs baselines de climatologia |
| `fase4_previsao_5dias.py` | Previsão 5 dias por município (desenvolvimento) |

### Scripts de desenvolvimento — `estagio2_grade/`

| Arquivo | Descrição |
|---|---|
| `fase1_dataset_grade.py` | Constrói dataset_grade.csv (18,4M linhas) |
| `fase1b_clima_05graus.py` | Baixa clima 0,5° para 148 pontos (~10 min) |
| `fase1c_aplicar_clima.py` | Substitui clima proxy pelo clima 0,5° no dataset |
| `fase2_modelagem.py` | Treina modelos na grade |
| `fase3_validacao_2026.py` | Retreino completo + validação 2026 grade |
| `fase4_previsao_offline.py` | Previsão 5 dias sem API (desenvolvimento/demo) |

### Scripts de produção — `pbl/`

| Arquivo | Descrição |
|---|---|
| `gerar_lookups.py` | Extrai lookup tables dos datasets de treino (execução única) |
| `previsao_leve.py` | Pipeline E1+E2 completo sem datasets pesados (~6 min) |
| `exportar_json.py` | Converte CSVs de previsão em forecast.json para o frontend |
| `requirements.txt` | Dependências do pipeline de produção |
