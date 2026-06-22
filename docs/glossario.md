# Glossário de Termos e Siglas

Referência rápida para termos técnicos utilizados nesta documentação.

---

## Métricas de avaliação de modelos

| Termo | Significado |
|---|---|
| **AUC-ROC** | *Area Under the ROC Curve* — área sob a curva ROC. Mede a capacidade do modelo de **ordenar** corretamente exemplos positivos acima dos negativos, independentemente do limiar escolhido. Varia de 0 a 1; 0,5 equivale a chute aleatório. AUC de 0,811 significa que em 81,1% das comparações entre um município com fogo e um sem fogo, o modelo classifica corretamente qual tem maior risco. |
| **PR-AUC** | *Area Under the Precision-Recall Curve* — área sob a curva precisão × recall. Métrica mais adequada que AUC-ROC quando as classes são muito desbalanceadas (ex.: 2% de positivos). Penaliza mais falsos positivos e é mais informativa sobre o desempenho real em classes raras. |
| **Brier score** | Erro quadrático médio entre a probabilidade prevista e o resultado real (0 ou 1). Varia de 0 (perfeito) a 1 (pior possível). Um Brier score alto indica que as probabilidades não estão bem calibradas, mesmo que o ranqueamento (AUC) seja bom. |
| **Recall** | Também chamado de *Sensibilidade* ou *Taxa de Verdadeiros Positivos*. Dos exemplos que realmente são positivos (dias com fogo), qual fração o modelo detectou. Fórmula: `TP / (TP + FN)`. Recall alto → poucos fogos passam despercebidos. |
| **Precisão** | Dos exemplos que o modelo classificou como positivos (alertas), qual fração realmente era positiva (tinha fogo). Fórmula: `TP / (TP + FP)`. Precisão alta → poucos alarmes falsos. |
| **F1** | Média harmônica entre Precisão e Recall. Útil quando se quer equilibrar os dois, especialmente com classes desbalanceadas. Fórmula: `2 × (Precisão × Recall) / (Precisão + Recall)`. |
| **Limiar** (*threshold*) | Valor de corte aplicado à probabilidade prevista para decidir se um exemplo é classificado como positivo (fogo) ou negativo (sem fogo). Limiar 0,3 → tudo com probabilidade ≥ 0,3 é alertado. Mudar o limiar não exige retreinar o modelo. |
| **Curva de captura** | Métrica operacional que responde: "se eu monitorar os X% de maior risco previstos, que fração dos fogos reais estará coberta?" Ex.: top 10% captura 22,3% dos fogos. Mede o valor prático do ranqueamento. |
| **Lift** | Razão entre a captura do modelo e a captura de uma seleção aleatória. Lift = 2,2 no top 10% significa que o modelo concentra 2,2× mais fogos reais do que uma escolha aleatória de municípios. |
| **TP / FP / TN / FN** | *True Positives* (verdadeiros positivos), *False Positives* (falsos positivos), *True Negatives* (verdadeiros negativos), *False Negatives* (falsos negativos). Compõem a matriz de confusão para um dado limiar. |

---

## Conceitos de aprendizado de máquina

| Termo | Significado |
|---|---|
| **Desbalanceamento de classes** | Quando uma classe é muito mais frequente que a outra no dataset. No vigIA, ~85% dos pares (município, dia) não têm fogo — o modelo precisa de estratégias especiais (ex.: `class_weight='balanced'`) para não ignorar a classe positiva. |
| **`class_weight='balanced'`** | Parâmetro que ajusta automaticamente o peso dos exemplos para compensar o desbalanceamento. Efeito colateral: infla as probabilidades previstas acima da frequência real — por isso o vigIA exibe percentil relativo em vez de probabilidade bruta. |
| **Split temporal** | Divisão do dataset em treino/validação/teste respeitando a ordem cronológica. Obrigatório em séries temporais para evitar *data leakage* — se dados futuros vazam para o treino, as métricas ficam infladas e não refletem o desempenho real em produção. |
| **Data leakage** | Contaminação do treino com informações que não estariam disponíveis no momento da previsão real. Causa métricas de validação otimistas que não se repetem em produção. |
| **Validação out-of-sample** | Avaliação do modelo em dados completamente fora do período de treino. No vigIA, validação com Jan–Jun 2026, dados nunca vistos durante o desenvolvimento. |
| **Overfitting** | Quando o modelo aprende padrões específicos do conjunto de treino que não generalizam para novos dados, resultando em métricas de treino muito melhores que as de teste/validação. |
| **Retreino** | Processo de treinar novamente o modelo com dados mais recentes ou corrigidos. O vigIA define gatilhos objetivos para retreino (ex.: AUC abaixo da baseline sazonal por um mês). |
| **Baseline** | Modelo simples de referência, sem ML, usado para avaliar se o modelo treinado agrega valor real. No vigIA, o melhor baseline é `Municipio_Freq` (frequência histórica de focos por município), com AUC 0,745. O modelo LightGBM supera em +0,066. |
| **Hiperparâmetros** | Configurações do modelo definidas antes do treino (ex.: número de árvores, profundidade máxima). Otimizados por busca (*RandomizedSearchCV*) em um subconjunto dos dados. |
| **Percentil** | Posição relativa de um valor dentro de uma distribuição. No vigIA, o percentil diário de um município indica sua posição entre os 244 municípios naquele dia — percentil 0,95 = top 5% de maior risco. |
| **Pipeline batch** | Execução do modelo em lotes, periodicamente (ex.: uma vez por dia), sem necessidade de resposta em tempo real. O vigIA é inteiramente batch: gera previsões às 03h BRT e publica o resultado. |

---

## Algoritmos e ferramentas

| Termo | Significado |
|---|---|
| **LightGBM** | *Light Gradient Boosting Machine* — algoritmo de gradient boosting otimizado para velocidade e eficiência em memória. Modelo escolhido para produção nos dois estágios do vigIA por combinar boa performance preditiva com inferência rápida e suporte nativo a valores ausentes (NaN). |
| **XGBoost** | *Extreme Gradient Boosting* — outro algoritmo de gradient boosting, avaliado como alternativa ao LightGBM. Obteve AUC similar, mas requer GPU para a grade e é mais lento em inferência. |
| **Random Forest** | Ensemble de árvores de decisão treinadas de forma independente. Avaliado como alternativa; AUC ligeiramente inferior ao LightGBM no dataset de município. |
| **Gradient Boosting** | Técnica de ensemble que constrói árvores sequencialmente, cada uma corrigindo os erros da anterior. Base dos algoritmos LightGBM e XGBoost. |
| **`predict_proba()`** | Método que retorna a probabilidade de cada classe, em vez de apenas a classe predita. No vigIA, retorna P(fogo) para cada par (município, dia). |

---

## Termos específicos do projeto

| Termo | Significado |
|---|---|
| **BDqueimadas** | Banco de Dados de Queimadas do INPE (*Instituto Nacional de Pesquisas Espaciais*). Fonte de dados históricos de focos de incêndio detectados por satélite no Brasil. |
| **INPE** | Instituto Nacional de Pesquisas Espaciais — órgão brasileiro responsável pelo monitoramento de queimadas via satélite. |
| **Open-Meteo** | API pública e gratuita de dados climáticos históricos e de previsão. Fonte das variáveis climáticas (precipitação, temperatura, vento) usadas no vigIA. |
| **FRP** | *Fire Radiative Power* — potência radiativa do fogo, medida em Megawatts, disponível no BDqueimadas. Usado nos MTs 4–6 (regressão); substituído por classificação binária no PBL. |
| **Risco relativo** | Risco expresso como posição relativa no ranking do dia, não como probabilidade absoluta. "Top 5%" significa que o município está entre os 5% de maior risco estimado em Goiás naquele dia. |
| **Forecast.json** | Arquivo gerado diariamente pelo pipeline do vigIA contendo as previsões dos próximos 5 dias para municípios e células. Funciona como contrato de dados entre o backend e o frontend. |
| **Lookup table** | Tabela pré-calculada com médias históricas, substituindo os datasets de treino completos (1,7 GB) no pipeline de produção. Permite executar a inferência com apenas ~1 MB de dados auxiliares. |
| **Degradação graciosa** | Comportamento do sistema quando um componente falha: em vez de travar, exibe um estado alternativo aceitável. O frontend do vigIA exibe previsão sintética de contingência se o `forecast.json` real estiver indisponível. |
| **CDN** | *Content Delivery Network* — rede de servidores distribuídos geograficamente que entrega conteúdo estático (HTML, JS, JSON) com baixa latência. O vigIA usa a CDN da Vercel para servir o frontend. |
