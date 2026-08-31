[README.md](https://github.com/user-attachments/files/31664971/README.md)
# MVP2 SSD - Dimensionamento e Faseamento de uma Nova Unidade

**Disciplina:** Sistemas de Suporte à Decisão
**Universidade de Brasília (UnB)**
**Aluno:** Gabriel

MVP de apoio à decisão para a implantação de uma nova unidade de uma empresa de
armazenamento de dados em nuvem. A decisão de construir já está tomada. O que este
trabalho responde é **como se programar**: quanta potência elétrica contratar, em quantas
fases implantar e quando reinvestir.

## Base de dados

Hyperscale Data Center Dataset (Kaggle), arquivo `green_ai_datacenter.csv`, com 100.000
registros de servidores e 35 colunas.

As métricas ambientais da base são estimadas por modelagem, não medidas em campo. Os
números absolutos servem para comparar alternativas, não para embasar contrato real.

## O problema de decisão

| Eixo | Pergunta | Modelo |
|---|---|---|
| Custo | Quanta potência elétrica contratar? | Regressão sobre `power_consumption_kw` |
| Risco | A unidade nasce eficiente ou ineficiente? | Classificação de `energy_efficiency_class` |
| Tempo | Quando a unidade sai do padrão aceitável? | Curva de classe prevista por idade |

Apenas variáveis conhecidas **antes** da construção entram como preditoras: região, tipo de
servidor, refrigeração, fonte de energia, idade do equipamento e intensidade de carga
prevista. Telemetria operacional é excluída por não existir no momento da decisão. `pue`,
`wue`, `carbon_emission_kg` e `compute_cost_usd` são excluídas por serem consequências do
consumo, não causas.

## Modelos e resultados

**Potência elétrica (regressão)**

| Modelo | R² teste | Gap treino-teste | MAE (kW) | Monotônico |
|---|---|---|---|---|
| Baseline (média) | 0,000 | - | 73,05 | - |
| Regressão Linear | 0,916 | 0,004 | 13,04 | sim |
| Random Forest (livre) | 0,920 | 0,057 | 9,35 | **não** |
| **Random Forest (podada)** | **0,937** | **0,005** | **7,69** | **sim** |

**Eficiência energética (classificação)**

| Modelo | Acurácia | F1 Macro |
|---|---|---|
| Baseline (classe frequente) | 0,340 | 0,169 |
| **Regressão Logística** | **0,882** | **0,883** |
| Random Forest | 0,855 | 0,855 |

## Três achados

### 1. Acurácia agregada não basta para uma ferramenta de decisão

A Random Forest sem poda teve o segundo melhor erro e foi descartada. Varrendo a
intensidade de carga com todo o resto constante, ela prevê 76 kW a 20% de carga, 148 kW a
60% e 99 kW a 80%. A resposta não é monotônica, o que é fisicamente impossível.

O modelo seria consultado exatamente com perguntas desse tipo no simulador de fases, e
nenhuma métrica agregada revelaria o problema. O notebook implementa um teste de sanidade
explícito e o usa como critério eliminatório antes do erro.

A versão podada (`max_depth=10`, `min_samples_leaf=50`) passa no teste e ainda reduz o MAE,
porque folhas maiores diminuem a variância.

### 2. A região não afeta o dimensionamento elétrico, mas decide a eficiência

Em regime pleno, a diferença de demanda entre a melhor e a pior região é de 0,03 MW, ou
0,1% da média. Já a classe de eficiência prevista varia: EU entrega High, enquanto APAC, ME
e NA entregam Medium.

Não existe, portanto, o trade-off esperado entre eficiência e custo de infraestrutura
elétrica nesta configuração. Escolher a região mais eficiente sai praticamente de graça.

### 3. A queda de consumo com a idade é composição, não degradação

O consumo médio cai de 175 kW no ano 3 para 112 kW no ano 4. A causa é a mudança de
composição da frota: servidores GPU representam 51% dos registros nos três primeiros anos e
17% a partir do quarto. Dentro de um único tipo de servidor a curva é estável e crescente.

## Entregável de decisão

Plano de implantação em três fases para 200 racks, com reserva de contingência dimensionada
pelo percentil 90 do erro absoluto do modelo (16,1 kW por rack).

| Fase | Racks | Carga | Potência a contratar |
|---|---|---|---|
| Fase 1 - Piloto | 40 | 30% | 4,02 MW |
| Fase 2 - Expansão | 120 | 60% | 13,47 MW |
| Fase 3 - Regime | 200 | 90% | 24,50 MW |

Configuração recomendada: EU / Compute / Liquid / Grid, que entrega classe High sem custo
elétrico adicional relevante.

Gatilho de renovação: a classe prevista cai para Medium no ano 7. O planejamento de
renovação deve começar no ano 5, para dar prazo a orçamento e homologação de fornecedores.

## Limitações

1. As métricas ambientais são sintéticas. Servem para comparação, não para contrato.
2. A base descreve servidores em operação, não obras. O eixo tempo aqui é ciclo de vida do
   equipamento, não cronograma de construção. Prazo de licenciamento, obra civil e conexão
   à rede ficam fora do alcance destes dados.
3. Configurações fora da distribuição de treino produzem previsões sem validade.

## Como executar

1. Abra `MVP2_Nova_Unidade.ipynb` no Google Colab.
2. Faça o upload de `green_ai_datacenter.csv` quando solicitado, ou preencha a constante
   `URL_REPOSITORIO` na célula de carga com o link bruto deste repositório.
3. Execute todas as células.

## Dependências

```
pandas
numpy
matplotlib
scikit-learn
```
