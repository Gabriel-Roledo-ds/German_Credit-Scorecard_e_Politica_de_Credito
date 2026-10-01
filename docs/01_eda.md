# EDA Crédito - German Credit

Análise exploratória realizada em 01/10/2026.

A base tem 1000 clientes, 30% de inadimplência e nenhum problema de qualidade; `month_duration`, `credit_history` e `status_account` são os sinais mais fortes de risco, e vários resultados contraintuitivos pedem validação do negócio.

## Descrição do dataset

Base com 1000 clientes que solicitaram crédito, com 21 colunas: 7 numéricas, 13 categóricas e 1 target. Os valores monetários estão em marcos alemães (DM), o que indica o German Credit Dataset (UCI, 1994). O objetivo é prever a inadimplência para construir um scorecard tradicional e uma política de crédito, e comparar com uma regressão logística.

**Qualidade e estrutura**

- Base limpa: 0 nulos, 0 duplicadas, nenhum valor escondido nas categóricas ('?', 'N/A', espaços ou inconsistências de escrita) e nenhum zero ou valor negativo nas numéricas.
- Não há ID do cliente nem data de contratação. Isso impede análise temporal e validação out-of-time (OOT), uma limitação do estudo.
- `unknown/ no savings account` em `status_savings` e `none` em `secondary_obligor`, `collateral`, `other_installment_plans` e `telephone` são níveis válidos, não nulos, e foram mantidos como categorias próprias. Por exemplo, `unknown/ no savings account` tem 17,5% de bad, ou seja, carrega informação.

**Target**

- `target` tem as classes `good` (700, 70%) e `bad` (300, 30%). Foi criada a coluna `target_bin` (1 = bad, evento de interesse).
- Desbalanceamento moderado. Será usado split estratificado e, como são só 300 eventos, validação cruzada em vez de um único split.

## Variáveis numéricas

`month_duration` é a numérica mais informativa: o bad passa de 21% (até 12 meses) para 48% (acima de 30 meses). As contínuas foram divididas em 5 faixas por quantis, e as discretas foram avaliadas por valor.

- `credit_amount` (921 valores distintos) e `month_duration` (33): média acima da mediana e valores altos legítimos (até 18.424 DM e 72 meses), com assimetria à direita confirmada nos gráficos. Não serão removidos.
- `age` (53 valores): 19 a 75 anos, mediana 33. Candidata a faixas no scorecard.
- `payment_to_income_ratio`, `residence_since`, `n_credits` e `n_guarantors`: cardinalidade de 2 a 4, tratadas como ordinais/categóricas. Em `n_credits`, as categorias 3 e 4 somam só 34 casos e serão agrupadas em "3+".

### Taxa de inadimplência por variável numérica

| Variável | Menor % bad | Maior % bad | Amplitude | Comportamento |
| --- | --- | --- | --- | --- |
| `month_duration` | 18,1% (12 a 15 meses) | 48,0% (> 30 meses) | ~30 p.p. | Sobe com o prazo; leve não monotonicidade entre 12 e 15 meses |
| `credit_amount` | 24,1% (1.262 a 1.907 DM) | 42,5% (> 4.720 DM) | ~18 p.p. | Não linear: o risco só se destaca na faixa mais alta |
| `age` | 25,3% (> 45 anos) | 39,2% (até 26 anos) | ~14 p.p. | Cai até os 30 anos e estabiliza depois |
| `payment_to_income_ratio` | 25,0% (faixa 1) | 33,4% (faixa 4) | ~8 p.p. | Monotônica crescente, de baixa amplitude |
| `residence_since` | 27,7% (faixa 1) | 31,5% (faixa 2) | ~4 p.p. | Sem tendência |
| `n_guarantors` | 29,7% (2) | 30,1% (1) | ~0,4 p.p. | Não discrimina |
| `n_credits` | 21,4% (3) | 33,3% (4, só 6 casos) | ~12 p.p. | Leve queda com mais créditos; o extremo é ruído |

- Em `credit_amount`, as quatro primeiras faixas ficam entre 24% e 30%, sem tendência clara. Agrupá-las e separar a faixa mais alta deve bastar.
- Em `age`, a diferença se concentra nos mais jovens (39,2% até 26 anos, contra cerca de 26% a partir dos 30). Duas ou três faixas devem bastar.
- Em `n_credits`, agrupar 3 e 4 em "3+" (34 casos) resulta em cerca de 23,5% de bad. A categoria 4 isolada tem 6 casos (2 inadimplentes), portanto é ruído.

## Variáveis categóricas mais discriminantes

`credit_history` e `status_account` têm as maiores variações de % bad entre categorias (~45 e ~38 p.p.), seguidas por `purpose`, `status_savings` e `years_employment`.

| Variável | Menor % bad | Maior % bad | Amplitude |
| --- | --- | --- | --- |
| `credit_history` | 17,1% (conta crítica) | 62,5% (sem créditos) | ~45 p.p. |
| `status_account` | 11,7% (sem conta) | 49,3% (< 0 DM) | ~38 p.p. |
| `purpose` | 11,1% (retraining) | 44,0% (education) | ~33 p.p. |
| `status_savings` | 12,5% (>= 1000 DM) | 36,0% (< 100 DM) | ~23 p.p. |
| `years_employment` | 22,4% (4 a 7 anos) | 40,7% (< 1 ano) | ~18 p.p. |

São as candidatas a maior IV, e a confirmação vem na etapa de WoE/IV. Atenção: em `purpose`, parte da amplitude vem de categorias com poucos casos.

## Análise visual

Os gráficos confirmam as tabelas e trazem um achado novo: os outliers de valor e prazo têm cerca do dobro da inadimplência da base.

### Histogramas e boxplots (numéricas contínuas)

- `credit_amount`: assimetria à direita. A média (~3.271 DM) fica acima da mediana (~2.320 DM), com maior concentração entre 1.000 e 3.000 DM. O boxplot mostra muitos outliers acima de ~7.900 DM, com cauda até 18.424 DM.
- `month_duration`: distribuição multimodal, com picos em 12 e 24 meses e picos menores em 36, 48 e 60. Os prazos são padronizados em múltiplos de 12, e a variável não é realmente contínua. A assimetria é moderada (média 20,9 e mediana 18). Os outliers começam acima de 42 meses (45, 48, 54, 60 e 72).
- `age`: leve assimetria à direita, com maior concentração entre 23 e 35 anos (mediana 33, média 35,5). Poucos outliers, acima de ~64 anos.

### Barras das categóricas e discretas (univariada)

- Categoria dominante: `is_foreign_worker` (96,3% `yes`), `secondary_obligor` (90,7% `none`), `other_installment_plans` (81,4% `none`), `n_guarantors` (84,5% com 1 avalista) e `housing` (71,3% `own`).
- Categorias raras (menos de 3% da base): em `purpose`, `retraining` (0,9%), `others` e `domestic appliances` (1,2% cada) e `repairs` (2,2%); em `job`, `unemployed/ unskilled - non-resident` (2,2%); em `n_credits`, as categorias 3 (2,8%) e 4 (0,6%). Suas taxas de bad são instáveis e elas serão agrupadas.
- Mais equilibradas: `status_account`, `credit_history` (53% em uma categoria), `years_employment` e `collateral`.
- `payment_to_income_ratio` e `residence_since` concentram 47,6% e 41,3% da base na faixa 4.

### KDE por target (numéricas contínuas)

- `month_duration`: é o gráfico que mais separa as classes. Os bons pagadores têm pico forte em ~12 meses, e os maus têm curva mais achatada e deslocada para a direita, com mais massa entre 30 e 50 meses.
- `credit_amount`: os maus têm cauda mais pesada acima de ~5.000 DM, e os bons se concentram em valores baixos (pico em ~1.500 DM). Separação moderada.
- `age`: a curva dos maus se concentra entre os mais jovens (cerca de 22 a 30 anos), e a dos bons é deslocada para a direita. Separação fraca a moderada, restrita à faixa jovem.
- O KDE suaviza a distribuição, então a multimodalidade de `month_duration` só aparece no histograma.

### Barras 100% empilhadas por target

As barras estão ordenadas por % bad, e a linha tracejada marca a proporção de bons da base (70%).

- Maior poder de separação: `status_account`, `credit_history`, `purpose` e `status_savings`.
- Em `status_account`, `status_savings` e `years_employment`, o % bad varia de forma ordenada entre as categorias, o que favorece a monotonicidade do WoE.
- `payment_to_income_ratio` mostra progressão suave e coerente (o bad aumenta com a faixa), mas com baixa amplitude.
- Sem separação visível: `n_guarantors`, `residence_since`, `telephone` e `job`, com barras quase idênticas e próximas da linha de 70%.
- Barras que se afastam muito da linha mas têm `n` pequeno, como `is_foreign_worker = no` (n=37), `retraining` (n=9), `others` (n=12) e `domestic appliances` (n=12), refletem ruído amostral e não devem ser interpretadas isoladamente.
- Os padrões contraintuitivos de `credit_history`, `status_account` e `collateral` aparecem de forma clara nesses gráficos.

### Taxa de bad por faixa (numéricas contínuas)

- `credit_amount`: relativamente estável nas quatro primeiras faixas (24% a 30%) e salta para 42,5% na última (acima de ~4.720 DM). A relação é não linear.
- `month_duration`: cresce com o prazo, de ~21% (até 12 meses) a 48% (acima de 30 meses), com leve queda na faixa de 12 a 15 meses (18,1%). O binning deve corrigir essa não monotonicidade.
- `age`: cai de 39,2% (até 26 anos) para ~26% a partir dos 30 e estabiliza depois.

### Correlação de Pearson

- A única correlação elevada entre as numéricas é `credit_amount` x `month_duration` (0,62), a monitorar na regressão logística por risco de multicolinearidade.
- Correlações moderadas: `credit_amount` x `payment_to_income_ratio` (-0,27) e `age` x `residence_since` (0,27).
- Com o target, as maiores correlações lineares são `month_duration` (0,21) e `credit_amount` (0,15). `age` (-0,09) e `payment_to_income_ratio` (0,07) são fracas, e `n_guarantors` (-0,00) e `residence_since` (0,00) são nulas.
- O Pearson mede apenas relação linear. `credit_amount` tem correlação baixa com o target, mas efeito não linear relevante na faixa mais alta, então seu poder preditivo é subestimado por essa métrica.

### Outliers (critério do IQR)

| Variável | Limite superior | Outliers | % da base | % bad nos outliers | % bad da base |
| --- | --- | --- | --- | --- | --- |
| `credit_amount` | 7.882 DM | 72 | 7,2% | 54,2% | 30% |
| `month_duration` | 42 meses | 70 | 7,0% | 57,1% | 30% |
| `age` | 64,5 anos | 23 | 2,3% | 26,1% | 30% |

- Os outliers de valor e prazo têm inadimplência quase o dobro da base. Não são erro de dado, e sim o segmento de maior risco, por isso foram mantidos. O efeito da cauda será tratado por binning e, na regressão logística, avaliando transformação logarítmica.
- Em `age`, a taxa dos outliers (26,1%) é próxima à da base, mas com apenas 23 casos (cerca de 6 inadimplentes), então a leitura é pouco confiável.

## Comportamentos que contrariam a intuição

Quatro resultados vão contra o senso comum. São padrões observados na base, não regras de negócio, e cada explicação provável precisa ser validada com o time de negócio antes de virar conclusão.

| Variável | Achado | Explicação provável | Confiança |
| --- | --- | --- | --- |
| `credit_history` | Histórico "limpo" tem mais bad | Viés de seleção | Média |
| `status_account` | Sem conta é o grupo mais seguro | Possível viés de seleção | Baixa |
| `is_foreign_worker` | Estrangeiros têm menos bad | Amostra pequena (ruído) | Não concluir |
| `collateral` | Com garantia tem mais bad | Causalidade invertida | Média |

### `credit_history` (histórico de crédito)

- **O que a base mostra:** quem nunca tomou crédito ou quitou todos os créditos tem as piores taxas de inadimplência (62,5% e 57,1%). Já "conta crítica / outros créditos existentes" tem a melhor (17,1%).
- **Por que é contraintuitivo:** o esperado seria o contrário, com bom histórico associado a menos inadimplência.
- **Explicação provável:** viés de seleção. A base parece conter apenas clientes aprovados. Para liberar crédito a alguém com conta crítica, o banco provavelmente foi muito rigoroso e aprovou só os casos que pareciam seguros. Clientes com histórico "limpo" passavam com mais facilidade, inclusive perfis frágeis em outros aspectos.
- **Implicação:** não interpretar como "histórico ruim reduz o risco". O resultado reflete a política de aprovação vigente. Sem clientes recusados na base, um modelo treinado nela pode errar ao avaliar o público que a política antiga barrava. Técnicas de *reject inference* (estimar o comportamento dos recusados) podem ajudar, mas dependem de dados que a base não tem.

### `status_account` (situação da conta corrente)

- **O que a base mostra:** clientes sem conta corrente são o grupo mais seguro (11,7% de bad), contra 49,3% de quem tem saldo negativo.
- **Por que é contraintuitivo:** a ausência de conta poderia indicar menor vínculo com o banco e, portanto, maior risco.
- **Explicação provável:** possivelmente o mesmo viés de seleção, em que clientes sem histórico bancário só foram aprovados se tinham perfil muito forte em outros aspectos. Também pode haver um tipo diferente de relacionamento com o banco. São hipóteses, sem confirmação.
- **Implicação:** a variável é muito discriminante (12% a 49% de bad) e deve ser considerada no modelo, mesmo com a explicação incerta.

### `is_foreign_worker` (trabalhador estrangeiro)

- **O que a base mostra:** estrangeiros têm 10,8% de bad, contra 30,7% dos demais.
- **Por que não é possível concluir:** são apenas 37 casos, cerca de 4 inadimplentes. Com tão poucos registros, a taxa oscila bastante por acaso.
- **Implicação:** não tirar conclusões sobre esse grupo. Usar nacionalidade como critério de decisão de crédito também envolve risco regulatório e de discriminação.

### `collateral` (garantia)

- **O que a base mostra:** clientes sem garantia têm a menor taxa de bad (21,3%), e os que oferecem poupança vinculada ou seguro de vida têm a maior (43,5%).
- **Por que é contraintuitivo:** ter garantia deveria reduzir o risco de calote, não aumentá-lo.
- **Explicação provável:** causalidade invertida. O banco tende a exigir garantia justamente dos perfis já considerados mais arriscados, então o grupo "com garantia" já é de maior risco antes de a garantia entrar na análise.
- **Implicação:** a variável pode ser útil no modelo, mas seu sinal não deve ser lido como "garantia aumenta o risco".

Todas as explicações acima são hipóteses. Para confirmá-las, é necessário o dicionário de dados e a validação do time de negócio, por exemplo sobre se a base inclui apenas clientes aprovados e como a garantia era exigida.

## Categorias raras e baixo poder discriminativo

Com 9 casos, 11% de bad significa 1 inadimplente: é ruído, não padrão. As categorias abaixo serão agrupadas por taxa de bad parecida ou por afinidade antes do WoE/IV.

**Categorias raras (instáveis)**

- `purpose`: retraining (9), others (12), domestic appliances (12), repairs (22).
- `job`: unemployed/unskilled - non-resident (22).
- `secondary_obligor`: co-applicant (41) e guarantor (52).
- `is_foreign_worker`: no (37).
- `n_credits`: categorias 3 (28) e 4 (6).

**Candidatas a IV baixo**

- `n_guarantors` (30,1% vs 29,7%) e `residence_since` (27,7% a 31,5%, sem tendência): sem diferença relevante de risco, candidatas a descarte.
- `telephone` (31,4% vs 28,0%) e `job` (28% a 34,5%): variam pouco entre categorias.
- `payment_to_income_ratio`: comportamento coerente (monotônico), mas de baixa amplitude (~8 p.p.).
- `is_foreign_worker` e `secondary_obligor`: muito concentradas (96,3% e 90,7% em uma categoria), com minorias pequenas.

## Atenção ética e regulatória

O uso de `status_and_sex` no modelo precisa de decisão consciente e documentada.

- `status_and_sex` mistura sexo e estado civil, com risco de discriminação e restrição legal.
- `age` e `is_foreign_worker` merecem a mesma análise de fairness.

## Observações para o scorecard

- Várias variáveis são ordinais naturais (`status_savings`, `years_employment`, `status_account`), o que ajuda a manter a monotonicidade do WoE.
- Categorias com % bad parecido e volume baixo serão agrupadas antes do cálculo do IV.
- Binning previsto para as contínuas: `credit_amount` (agrupar as quatro faixas inferiores e separar a mais alta), `age` (duas ou três faixas) e `month_duration` (faixas que corrijam a leve não monotonicidade entre 12 e 15 meses, podendo seguir os prazos naturais de 12, 24 e 36 meses).
- Na regressão logística, monitorar a multicolinearidade entre `credit_amount` e `month_duration` (0,62) e avaliar `log(credit_amount)`.
- Os valores de `credit_amount` estão em DM de 1994 e não são comparáveis com valores atuais em R$.

**Próximos passos**

1. Split estratificado treino/teste (ou validação cruzada) antes de calcular WoE e IV, para evitar vazamento.
2. Agrupamento das categorias raras e binning das contínuas, usando só o treino.
3. WoE e IV, ranking e seleção de variáveis.
4. Scorecard, política de crédito e comparação com a regressão logística.

## Perguntas ao time de negócio

1. Os dados incluem apenas clientes aprovados? Existem recusados disponíveis?
2. Qual é a definição de "bad" (atraso de quantos dias)?
3. Qual é a codificação de `payment_to_income_ratio`, `residence_since` e `n_guarantors`?
4. Há data de concessão para validação temporal?
