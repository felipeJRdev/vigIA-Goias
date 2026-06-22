# Estratégia de Manutenção e Atualização — vigIA

**Projeto:** vigIA — Previsão de Risco de Incêndio em Goiás
**Disciplina:** FGA0083 — Aprendizado de Máquina · UnB 2026-1 · Turma 01 · Grupo 4
**Equipe:** Felipe Rodrigues · João Paulo Cristo · Guilherme Vilela · Luiz Guilherme Faria
**Documento:** Critério 3 — Estratégias de manutenção e atualização contínua do modelo

---

## 1. Princípios

A manutenção do vigIA segue três premissas: **minimalismo operacional** (uma solução batch com dados públicos não exige infraestrutura sofisticada), **versionamento explícito** (toda mudança em modelo ou pipeline é rastreável via git e artefato nomeado) e **qualidade verificável** (nenhum modelo novo vai a produção sem passar pela suíte de testes e superar ou igualar o desempenho do predecessor).

## 2. Gatilhos de retreino

O retreino do modelo não é feito em calendário fixo, mas acionado por qualquer um dos eventos abaixo:

| Gatilho | Fonte de detecção | Criticidade |
|---|---|---|
| Hit-rate@10 < 0,15 por dois ciclos semanais consecutivos | Monitor de produção (backtest) | Alta — acionar imediatamente |
| AUC ou Brier sustentados abaixo da baseline sazonal por um mês | Revisão mensal | Alta |
| Anomalia de distribuição: > 70% dos municípios em ALTO por > 5 dias | Monitor de produção (camada 3) | Média — investigar antes de retreinar |
| Mudança estrutural nos dados de entrada (novo formato Open-Meteo, mudança no BDqueimadas) | Falha no pipeline de coleta | Alta — retreino após adaptação do pipeline |
| Novo ano completo de observações disponível | Calendário (janeiro) | Baixa — retreino preventivo anual |

O retreino **não** é acionado por métricas operacionais (boletim atrasado, cobertura baixa); esses são erros de pipeline, não de modelo.

## 3. Procedimento de retreino

### 3.1 Preparação

1. Coletar os dados do novo período (BDqueimadas + Open-Meteo) para o intervalo ausente desde o último treino.
2. Executar `fase1_coleta_clima.py` e `fase1b_coleta_focos.py` para o período novo.
3. Re-executar a limpeza e engenharia de features (`fase2_feature_engineering.py`).
4. Verificar que a distribuição das features principais (precipitação, dias secos, temperatura máxima) não mudou estruturalmente; se mudou, registrar como drift de dado antes de prosseguir.

### 3.2 Treinamento e validação

1. Executar o treinamento com os novos dados: `fase3_modelo_municipio.py` (E1) e `fase3_modelo_grade.py` (E2).
2. Comparar as métricas do novo modelo com o modelo em produção usando o mesmo conjunto de validação temporal (janela out-of-sample dos últimos 12 meses):

| Métrica | Critério de aprovação |
|---|---|
| AUC-ROC | ≥ AUC do modelo atual |
| PR-AUC | ≥ PR-AUC do modelo atual |
| Captura top 10% | ≥ Captura atual |
| Recall (limiar 0,3) | ≥ Recall atual − 0,02 (tolerância) |

3. Se aprovado, o novo modelo substitui o anterior. Se reprovado, manter o modelo atual e investigar causa raiz.

### 3.3 Deploy do novo modelo

1. Nomear os novos artefatos com data de corte dos dados: ex. `municipio_full_2026.pkl`, `grade_full_2026.pkl`.
2. Atualizar as referências em `pbl/previsao_leve.py` para apontar para os novos arquivos.
3. Executar manualmente o pipeline (`workflow_dispatch`) e confirmar que o `forecast.json` gerado passa na suíte `pytest` integral.
4. Fazer commit com mensagem descritiva (ex.: `model: retreino com dados até dez/2026`) e tag de versão (`v2.0.0`).
5. Guardar o modelo anterior com sufixo `_deprecated` por pelo menos 30 dias antes de remover, para rollback de emergência.

## 4. Estratégia de rollback de modelo

Se o novo modelo causar regressão detectada no monitoramento após o deploy:

1. Reverter o commit de atualização dos artefatos (`git revert`).
2. O pipeline do dia seguinte usará automaticamente o modelo anterior (revertido).
3. Abrir issue no repositório registrando a causa da regressão antes de tentar um novo retreino.

## 5. Manutenção do pipeline de dados

### 5.1 Open-Meteo

A API pública do Open-Meteo não exige autenticação, mas pode mudar parâmetros ou formato de resposta. Sinais de problema: falha no job de coleta, campos climáticos ausentes no boletim, cobertura de municípios < 90%.

Ação: verificar o changelog da API; atualizar os parâmetros de requisição em `fase1_coleta_clima.py`; reprocessar o dia afetado via `workflow_dispatch`.

### 5.2 INPE / BDqueimadas

O histórico de focos é a base de treinamento e também serve ao backtest. Se o formato do CSV exportado mudar: atualizar o parser em `fase1b_coleta_focos.py` e re-executar o pipeline de coleta para o período afetado.

### 5.3 Dependências de software

As dependências são fixadas em `pbl/requirements.txt`. Manutenção semestral:

1. Testar atualização das bibliotecas principais (LightGBM, pandas, scikit-learn) em branch separada.
2. Rodar a suíte de testes e comparar métricas do modelo com versões atualizadas.
3. Atualizar `requirements.txt` somente após a suíte passar e as métricas permanecerem estáveis.

## 6. Runbook de incidentes

### Falha total do pipeline (boletim não gerado)

1. Verificar os logs do job no GitHub Actions.
2. Se erro de coleta climática: testar acesso manual à API Open-Meteo; aguardar 1h e re-acionar via `workflow_dispatch`.
3. Se erro de inferência: verificar que os arquivos `.pkl` estão presentes no repositório; rodar `previsao_leve.py` localmente.
4. Se erro de commit/push: verificar permissões do `GITHUB_TOKEN`; re-acionar.
5. Se persistir por mais de 24h: notificar o grupo e manter o boletim anterior visível no frontend (o fallback sintético já está ativo).

### Boletim publicado com dados incorretos

1. Corrigir a causa raiz no pipeline.
2. Reprocessar via `workflow_dispatch`.
3. Se o `forecast.json` errado foi commitado: `git revert` do commit do boletim, forçar novo deploy na Vercel.

### Degradação silenciosa do modelo (detectada pelo backtest)

1. Registrar a queda nas métricas no histórico e abrir issue.
2. Analisar se é drift sazonal (esperado na estação chuvosa) ou queda estrutural.
3. Se sazonal: anotar, revisar os limiares se necessário, não retreinar.
4. Se estrutural: acionar o procedimento de retreino (seção 3).

## 7. Versionamento e rastreabilidade

Toda mudança significativa recebe uma tag semântica no repositório:

- `v1.x` — modelo inicial treinado com dados 2015–2025.
- `v2.x` — próximo retreino (com dados 2026+) ou mudança de features.
- `v3.x` — mudança de arquitetura de modelo ou de contrato de dados.

O arquivo `forecast.json` inclui o campo `gerado_em`, que serve como rastreador de versão operacional. O histórico de boletins pode ser recuperado via `git log`.

## 8. Roadmap de evolução

| Horizonte | Melhoria planejada | Justificativa |
|---|---|---|
| Curto prazo (< 6 meses) | Adicionar dados de índice de vegetação (NDVI) como feature | Reduzir erros na estação chuvosa, onde a umidade atual subestima o risco em cerrado degradado |
| Médio prazo (6–12 meses) | Retreino com dados de 2026 completo | Cobrir o ciclo climático do ano de produção |
| Médio prazo | Gate automático de testes no workflow de geração do boletim (bloquear commit se `pytest` falhar) | Formalizar o critério Go/No-Go descrito no Relatório de Prontidão |
| Longo prazo (> 1 ano) | Avaliação de modelo de previsão climática local (NWP) como substituto ao Open-Meteo | Reduzir dependência de API externa de terceiro |

## 9. Responsabilidades

| Atividade | Frequência | Responsável |
|---|---|---|
| Revisão dos logs do pipeline | Diária | Plantonista da semana (rodízio) |
| Revisão do hit-rate@10 | Semanal | Plantonista da semana |
| Decisão de retreino | Conforme gatilho | Equipe (consenso) |
| Atualização de dependências | Semestral | Membro técnico designado |
| Reavaliação de limiares de alerta | Antes da estação seca | Equipe |
