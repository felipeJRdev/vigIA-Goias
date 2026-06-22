# Frontend Interativo

O frontend é uma aplicação estática hospedada no **Vercel**, disponível em **https://vig-ia.vercel.app/**.

**Tecnologias:** Leaflet.js + GeoJSON + Esri World Imagery (satélite, sem API key)

**Arquivos:** `index.html`, `styles.css`, `app.js`, `data.js`, `forecast.json`

## Funcionamento

- `forecast.json` é gerado diariamente pelo GitHub Actions e commitado no repositório
- O Vercel redeploia automaticamente a cada novo commit, mantendo o site sempre atualizado
- `app.js` lê `forecast.json` e renderiza o mapa com Leaflet

## Funcionalidades

- Mapa coroplético dos 244 municípios colorido por **risco relativo** (percentil dentro do dia)
- Cores: **ALTO** = top 10% (percentil ≥ 0,90, vermelho) · **Atenção** = top 10–30% (percentil 0,70–0,90, âmbar) · **Calmo** = demais 70% (verde)
- Barra de navegação para selecionar entre os próximos 5 dias de previsão
- Ranking lateral com municípios ordenados por risco relativo decrescente — exibe "top X%" (posição relativa entre os 244 municípios no dia)
- **Clique no município:** drill-down mostrando células 0,1° internas coloridas por risco relativo
- Hover com tooltip: posição relativa ("top X%"), categoria de risco, dias sem chuva
- **Filtro de zonas:** slider "mostrar apenas top X%" — oculta células abaixo do threshold escolhido

## Por que exibir percentil em vez de probabilidade bruta

O `class_weight="balanced"` inflata as probabilidades (Brier score piora em relação ao baseline ingênuo). O modelo é excelente em **ordenar** municípios por risco (AUC 0,816), mas os valores absolutos de probabilidade não devem ser lidos como frequência real de ocorrência.

Exibir a posição relativa ("top 5%") é honesto: informa onde concentrar atenção sem fazer promessas sobre calibração.

## Ranking por risco relativo

Em vez de um limiar binário fixo ("vai ter fogo? sim/não"), o sistema usa os modelos como **ferramentas de ranqueamento**:

- **Estágio 1:** ordena os 244 municípios por probabilidade → o percentil diário determina a posição relativa exibida ("top 5%", "top 30%", etc.)
- **Estágio 2:** ordena as células dentro de cada município → identifica subáreas críticas

**Percentil diário:** calculado como `rank(pct=True)` dentro de cada dia — cada município recebe sua posição relativa entre os 244 municípios de Goiás naquele dia. Isso torna a visualização estável entre estações: no período seco todos os valores absolutos sobem, mas a distribuição relativa permanece interpretável.

**Vantagem do ranking:** o AUC garante que a ordenação está correta independentemente da calibração. AUC = 0,816 significa que em 81,6% das comparações (município com fogo vs sem fogo), o modelo classifica corretamente qual tem maior risco.

## Degradação graciosa

Caso o `forecast.json` real esteja ausente ou inválido, o frontend recorre a uma previsão sintética de contingência, garantindo que o usuário nunca veja uma tela vazia. Esse é o principal mecanismo de tolerância a falhas das fontes externas.
