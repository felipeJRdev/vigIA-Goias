# Decisões Técnicas e Resultados

## Resultados consolidados

| | Estágio 1 — Município | Estágio 2 — Grade 0.1° |
|---|---|---|
| Modelo | LightGBM | LightGBM |
| Dataset treino | ~980k amostras | 18,4M amostras |
| AUC Teste (2024-25) | **0.835** | 0.831 |
| AUC Validação 2026 | **0.816** | 0.710 |
| Limiar operacional | 0.3 | 0.6 |
| Recall (limiar) | **79,2%** | 70,3% |
| Precisão (limiar) | **95,3%**¹ | 42,7% |
| Top 10% captura | **43,6%** fogos | 30,3% fogos |
| Top 20% captura | **62,7%** fogos | 48,5% fogos |

¹ _Inflada: município considerado correto se teve ao menos um foco em 6 meses. A métrica honesta é Captura top N%._

## Decisões técnicas

| Decisão | Motivo |
|---|---|
| Classificação em vez de regressão | FRP já é medido — regressão não agrega valor em produção |
| Ausência = negativo (BDqueimadas) | Satélite cobre Goiás continuamente |
| Split temporal | Random split vaza dados futuros no treino |
| Subsample 200k para busca | RandomizedSearchCV inviável em 720k+ amostras |
| `n_jobs=1` no estimador interno | Nested parallelism travou processo por 30+ minutos |
| Negativos naturais completos (grade) | NEG_RATIO=4 inflou positivos para 20%, causou AUC 0,69 |
| Grade climática 0,5° | 148 pontos vs 2.976: sem rate limit, erro máx ~35km |
| Limiar E1 = 0,3 | Recall 79,2%, Precisão 95,3% — equilíbrio para alerta regional |
| Limiar E2 = 0,6 | Recall 70,3%, Precisão 42,7% — equilíbrio para drill-down geográfico |
| Limiar ≠ retreino | O limiar é aplicado pós-previsão; mudar limiar não requer retreinar |
| Dois estágios complementares | Município: triagem confiável; grade: localização operacional |
| Lookup tables (85KB + 937KB) | Substituem datasets de treino de 1,7GB no pipeline de produção |
| `past_days=30` (chamada única) | Substitui Archive + Forecast separados; mesma janela, metade das chamadas |
| ThreadPoolExecutor (max_workers=8) | 244 chamadas paralelas vs sequencial: ~6 min vs ~150 min |
| `sys.exit(1)` em erro fatal | Impede commit de previsão incompleta no GitHub Actions |
| Percentil diário em vez de probabilidade bruta | `class_weight="balanced"` inflata probabilidades (Brier pior que baseline ingênuo); o modelo ranqueia bem (AUC 0,816), então exibir posição relativa ("top X%") é mais honesto |
| LightGBM para produção E2 (vs Random Forest) | Diferença de AUC dentro do ruído (~0,005); LightGBM tem inferência mais rápida, suporte nativo a NaN e consistência com E1 |
| Escrita progressiva em `fase1b_clima.py` | Batch write perdia todo o progresso em Ctrl+C; escrita por município garante retomada incremental |

## Arquivos de modelos e dados

### Modelos — `modelos/`

| Arquivo | Descrição |
|---|---|
| `municipio_full.pkl` | LightGBM 2015–2025, AUC 0.816 (**produção estágio 1**) |
| `grade_full.pkl` | LightGBM grade 2015–2025, AUC 0.710 (**produção estágio 2**) |

### Dados — `dados/`

| Arquivo | Descrição |
|---|---|
| `mapeamento_municipio.csv` | 244 municípios — freq + centroide lat/lon |
| `mapeamento_grade.csv` | 2.976 células — freq + município proxy |
| `lookup_municipio.csv` | Média histórica de focos por (município, mês) — 85 KB |
| `lookup_grade.csv` | Média histórica de focos por (célula, mês) — 937 KB |

### Resultados — `resultados/`

| Arquivo | Descrição |
|---|---|
| `resultados_municipio_fase2.csv` | Métricas dos 3 modelos (município) |
| `resultados_grade_fase2.csv` | Métricas dos 3 modelos (grade) |
| `validacao_municipio_2026.csv` | Métricas de validação 2026 (município) |
| `validacao_grade_2026.csv` | Métricas de validação 2026 (grade) |
| `previsao_municipio_<data>.csv` | Ranking 5 dias por município |
| `previsao_grade_<data>.csv` | Ranking 5 dias por célula |
