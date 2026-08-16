# Clusterização de Clientes

Análise não supervisionada que segmenta a base de clientes de um e-commerce em personas de comportamento de compra, combinando **RFM (Recency, Frequency, Monetary)** com **K-Means**. O projeto cobre o pipeline completo: da limpeza de dados transacionais brutos até a definição dos segmentos de clientes com recomendações de ação para cada um.

## Sobre o projeto

Empresas de varejo geralmente têm uma base de clientes com diversidade de padrões, mas tratam todos de forma semelhante. O objetivo deste projeto é usar dados históricos de transações de produtos para responder perguntas de negócio como:

- Quem são os clientes mais valiosos da base e o quanto eles concentram de receita?
- Quais clientes estão em risco de abandono (churn) e precisam de ação de reativação?
- Como priorizar investimento de CRM entre aquisição, retenção e upsell?

A resposta é orientada a dados, sem regras de negócio pré-definidas: os grupos são obtidos a partir da mineração dos dados estudando o comportamento de compra (valor gasto, frequência de compras e quão recente foi a última compra).

## Dataset

[Online Retail II — UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/502/online+retail+ii): transações de um varejista online do Reino Unido, de produtos de presentes para todas as ocasiões. O arquivo original contém duas abas (2009–2010 e 2010–2011); esta análise utiliza a aba **"Year 2009-2010"**.

| Item | Valor |
|---|---|
| Registros brutos (transações/linhas) | 525.461 |
| Período analisado | Dez/2009 – Dez/2010 |
| Colunas originais | Invoice, StockCode, Description, Quantity, InvoiceDate, Price, Customer ID, Country |
| Valores ausentes | 107.927 sem `Customer ID` (~20,5%) · 2.928 sem `Description` |
| Registros após limpeza | 406.309 (**77,3%** de aproveitamento) |
| Clientes únicos identificados | 4.285 |

## Metodologia

O pipeline segue 5 etapas principais, cada uma documentada e validada no notebook:

**1. Exploração e diagnóstico de qualidade de dados**
Identificação de inconsistências reais do dataset:  `StockCode` fora do padrão de 5 dígitos (ex.: taxas de postagem, ajustes manuais), preços zerados e negativos, e ausência de `Customer ID` em ~20% das transações.

**2. Limpeza de dados**
Regras de exclusão definidas com regex e filtros de negócio: apenas invoices numéricas de 6 dígitos (remove cancelamentos/ajustes), apenas StockCodes válidos, remoção de registros sem `Customer ID` e remoção de preço unitário ≤ 0. Resultado: 77,3% dos registros originais mantidos.

**3. Engenharia de features (RFM)**
Agregação a nível de cliente (`groupby Customer ID`) para calcular:
- **Monetary**: soma de `Quantity × Price` por cliente
- **Frequency**: número de invoices (pedidos) únicos por cliente
- **Recency**: dias desde a última compra até a data mais recente do dataset

**4. Tratamento de outliers**
As distribuições de Monetary e Frequency são fortemente assimétricas (poucos clientes concentram compras muito acima da média). Em vez de descartar esses pontos, o projeto isola outliers via **IQR** separadamente para Monetary e Frequency e os trata como **segmentos de negócio próprios**. Os clientes restantes seguem para o clustering padrão.

**5. Clusterização com K-Means**
- Padronização das features.
- Varredura de `k = 2` a `12`, com **método do cotovelo (inertia)** e **silhouette score** para escolher o número ideal de clusters.
- Modelo final: **K-Means com k = 4** sobre os clientes não-outliers, combinado aos 3 segmentos de outliers definidos na etapa anterior, totalizando **7 personas de cliente** que cobrem 100% da base.

## Métricas do modelo

| k | Inertia | Silhouette Score |
|---|---|---|
| 2 | 6.174,79 | 0,444 |
| 3 | 3.606,65 | 0,459 |
| **4 (escolhido)** | **2.751,55** | **0,416** |
| 5 | 2.380,78 | 0,403 |
| 6 | 2.211,12 | 0,382 |

`k=4` foi escolhido por equilibrar bem as duas métricas.

## Segmentação final e principais insights

| Segmento | Clientes | % da base | Receita | % da receita | Ticket médio | Frequência média | Recência média (dias) |
|---|---:|---:|---:|---:|---:|---:|---:|
| VIP | 226 | 5,3% | £3.875.372 | 44,7% | £17.147,66 | 25,9 pedidos | 14,5 |
| Alto Valor | 197 | 4,6% | £1.280.195 | 14,8% | £6.498,45 | 7,2 pedidos | 47,9 |
| Leal | 494 | 11,5% | £1.203.429 | 13,9% | £2.436,09 | 7,2 pedidos | 33,9 |
| Regular | 914 | 21,3% | £1.196.083 | 13,8% | £1.308,62 | 3,9 pedidos | 49,7 |
| Recente | 1.499 | 35,0% | £626.513 | 7,2% | £417,95 | 1,6 pedidos | 54,1 |
| Em Risco | 902 | 21,1% | £346.852 | 4,0% | £384,54 | 1,4 pedidos | 251,2 |
| Frequente | 53 | 1,2% | £144.938 | 1,7% | £2.734,69 | 15,0 pedidos | 23,1 |

**Principal conclusão (efeito Pareto na base):** os segmentos **VIP + Alto Valor + Leal** representam apenas **21,4% dos clientes**, mas concentram **73,3% de toda a receita** do período analisado. 

Outras leituras de negócio extraídas da segmentação:

- **Em Risco (21% dos clientes)** não compram há em média 251 dias. É o segmento com maior urgência de reativação.
- **Recente (35% da base, o maior segmento em volume)** compra pouco e gasta pouco, mas comprou há pouco tempo (54 dias em média). Indica clientes novos ou ocasionais com potencial de conversão para via engajamento.
- **Frequente (apenas 1,2% da base)** compra com alta regularidade (15 pedidos em média) mas com ticket médio moderado. É uma oportunidade de aumentar valor via cross-sell/upsell, não de reter (já é engajado).
- **VIP** é o segmento mais raro (5,3%) e mais recente em comportamento (14,5 dias desde a última compra).

## Tecnologias utilizadas

| Categoria | Ferramentas |
|---|---|
| Linguagem | **Python**
| Manipulação de dados | pandas, numpy |
| Machine Learning | scikit-learn (`KMeans`, `StandardScaler`, `silhouette_score`) |
| Visualização | matplotlib, seaborn, plotly (gráficos 3D interativos) 

## Como rodar

**Pré-requisitos:** Python 3.12+ e [uv](https://docs.astral.sh/uv/) instalado.

```bash
# 1. Clonar
git clone <url-do-repositorio>
cd customer-clustering

# 2. Instalar as dependências
uv sync
```

## Possíveis próximos passos

- Estender a análise para a segunda aba do dataset (`Year 2010-2011`) para validar a estabilidade dos segmentos ao longo de um período mais longo.
- Testar algoritmos alternativos (DBSCAN, Gaussian Mixture) para comparar com o K-Means, especialmente para tratar os outliers de forma nativa em vez de segmentá-los manualmente por IQR.