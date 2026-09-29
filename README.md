# German Credit — Análise de Risco e Política de Crédito

Análise exploratória e construção de uma política de crédito baseada em regras (scorecard), usando o dataset [Statlog (German Credit Data)](https://archive.ics.uci.edu/dataset/144/statlog), do UCI Machine Learning Repository. Projeto estruturado seguindo a metodologia **CRISP-DM**.
O objetivo desta análise é entender como um scorecard tradicional (baseado em regras) se diferencia de modelos estatísticos mais robustos, como a regressão logística — tanto em poder discriminante quanto em custo esperado da política de crédito. Devido ao tamanho pequeno da base (1.000 clientes), as métricas avaliadas não mostraram grande diferença entre as duas abordagens; a próxima etapa é repetir essa comparação em uma base maior para ver se essa vantagem se confirma fora de uma amostra pequena.

## Problema de negócio

O dataset reúne 1.000 clientes de crédito com 20 variáveis (7 numéricas, 13 categóricas), compilado pelo Prof. Hans Hofmann (Universität Hamburg). A documentação não identifica uma instituição financeira específica — é um dataset acadêmico usado como benchmark de risco de crédito —, mas o cenário de negócio permanece válido: uma instituição que concede crédito ao consumidor quer reduzir a inadimplência da carteira sem comprometer o volume de aprovações.

A própria documentação do dataset (seção 8, `german.doc`) define uma **matriz de custo assimétrica** obrigatória para avaliar qualquer modelo construído sobre esses dados: aprovar um mau pagador custa **5x mais** do que reprovar um bom pagador. Essa assimetria é a métrica de sucesso central do projeto — não apenas reduzir a taxa de inadimplência, mas reduzir o **custo esperado da carteira**.

## Estrutura do notebook (CRISP-DM)

1. **Business Understanding** — problema, hipótese de trabalho e métrica de sucesso
2. **Data Understanding** — análise exploratória: distribuição do target, estatísticas descritivas, taxa de inadimplência por categoria, correlações
3. **Data Preparation** — split treino/teste estratificado (70/30), sem vazamento de dados
4. **Modeling** — construção do scorecard de regras e definição da política de crédito por faixa de risco
5. **Evaluation** — validação da política em dados de teste (nunca vistos na construção) e matriz de custo
6. **Deployment** — benchmark com regressão logística, limitações e resumo executivo

## Principais achados

- **Taxa de inadimplência geral:** 30% (linha de base da carteira)
- **Variáveis mais discriminantes:** `status_account` (situação da conta corrente), `credit_history` (histórico de crédito) e `status_savings` (conta poupança)
- Dois padrões **contraintuitivos** identificados: clientes sem conta corrente e com "critical account" no histórico performam melhor do que o esperado — reforça a importância de validar hipóteses com dado, não só intuição
- `credit_amount` e `month_duration` são assimétricas à direita e moderadamente correlacionadas entre si (0,62); sem sinal de multicolinearidade grave entre as demais variáveis numéricas

## Scorecard e política de crédito

Cada uma das 3 variáveis mais discriminantes recebe pontos de 0 a 3 por faixa, calibrados pela taxa de default real de cada categoria (menor taxa = mais pontos), calculada apenas em `df_treino`. A soma dos pontos (score bruto de 0 a 9) é agrupada em 3 faixas de risco:

| Faixa de risco | Taxa de default (treino) | Decisão               |
|----------------|---------------------------|------------------------|
| Alto Risco     | 54,2%                      | Reprovado             |
| Médio Risco    | 26,1%                      | Aprovado com restrição |
| Baixo Risco    | 8,6%                       | Aprovado              |

### Resultado da política

| Métrica                              | Treino | Teste | Diferença |
|---------------------------------------|--------|-------|-----------|
| Taxa de aprovação                     | 69,4%  | 69,3% | -0,1 p.p. |
| Maus pagadores entre aprovados        | 19,3%  | 18,8% | -0,5 p.p. |
| Bons pagadores entre reprovados       | 45,8%  | 44,6% | -1,2 p.p. |
| Custo médio por cliente (vs. 1,500 aprovando todos) | 0,811 (-45,9%) | 0,787 (-47,6%) | — |

Diferenças abaixo de 1,5 p.p. entre treino e teste em todas as métricas — a política generaliza bem, não está sobreajustada.

## Benchmark: regressão logística

Para verificar se o scorecard de 3 variáveis deixa sinal na mesa, foi treinado um modelo de regressão logística com todas as variáveis candidatas (incluindo `purpose`, `housing` e `years_employment`, que o scorecard não usa):

| Métrica (teste) | Regressão logística | Scorecard de regras |
|------------------|----------------------|----------------------|
| AUC              | 0,743                | 0,741                |
| Gini             | 0,485                | 0,482                |
| KS               | 0,408                | 0,371                |

A diferença é pequena (Gini +0,004) — o scorecard de 3 variáveis já captura quase todo o sinal disponível nesta base, e a simplicidade/explicabilidade compensa a complexidade extra do modelo estatístico.

## Limitações

- Scorecard usa apenas 3 variáveis; o benchmark de regressão logística mostra que o espaço de melhoria é pequeno, mas não nulo
- Pesos por faixa definidos por ranking de taxa de default, não por WOE/IV formal ou otimização — próximo passo natural
- **Nenhuma análise de fairness foi feita.** Variáveis sensíveis (`status_and_sex`, `is_foreign_worker`) não entraram no modelo, mas viés indireto via variáveis correlacionadas (`housing`, `job`, `purpose`) não foi descartado
- Dataset sintético/histórico (1994), sem dimensão temporal — não é possível avaliar estabilidade da política ao longo do tempo
- Amostra pequena (1.000 clientes): categorias como "retraining" (9 clientes) e scores extremos (0 e 9, <16 clientes cada) têm taxas pouco confiáveis isoladamente

## Próximos passos

- Repetir a comparação scorecard vs. modelo estatístico em uma base maior (ex: [Give Me Some Credit](https://www.kaggle.com/c/GiveMeSomeCredit), ~150 mil registros) para ver se a vantagem do modelo cresce fora de uma amostra pequena
- Análise de fairness (taxa de aprovação e de erro por subgrupo sensível) antes de qualquer uso em produção
- Calibrar pesos do scorecard via WOE/IV

## Tecnologias

- Python (pandas, numpy)
- Visualização: matplotlib, seaborn
- Modelagem: scikit-learn (regressão logística, métricas de classificação)
- Ambiente: Google Colab

## Fonte dos dados

[Statlog (German Credit Data) — UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/144/statlog), compilado pelo Prof. Dr. Hans Hofmann (Institut für Statistik und Ökonometrie, Universität Hamburg).

## Autor

Gabriel Guaitolini Roledo — [GitHub](https://github.com/Gabriel-Roledo-ds)
