# German Credit — Análise de Risco e Política de Crédito

> **Aviso — análise em revisão.** Este projeto recebeu um feedback técnico relevante após a publicação (ver a seção [Feedback recebido](#feedback-recebido)). Em breve será feita uma correção na análise, adicionando o que for necessário para responder aos pontos levantados. Enquanto isso, as conclusões sobre o benchmark devem ser lidas com as ressalvas descritas em [Limitações](#limitações).
> **Atualização Análise exploratória realizada dia 01/10**

Análise exploratória e construção de uma política de crédito baseada em regras (scorecard), usando o dataset [Statlog (German Credit Data)](https://archive.ics.uci.edu/dataset/144/statlog), do UCI Machine Learning Repository. Projeto estruturado seguindo a metodologia **CRISP-DM**.
O objetivo desta análise é entender como um scorecard tradicional (baseado em regras) se diferencia de modelos estatísticos mais robustos, como a regressão logística — tanto em poder discriminante quanto em custo esperado da política de crédito. Nesta base, as métricas de discriminação (AUC, Gini e KS) ficaram próximas entre as duas abordagens, mas a comparação tem limites importantes (ver seção Limitações): o teste tem só 300 clientes, o benchmark usou 6 das 20 variáveis e o custo no ponto de corte não foi calculado para a regressão.

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

Para verificar se o scorecard de 3 variáveis deixa sinal na mesa, foi treinado um modelo de regressão logística com 6 das 20 variáveis da base: as 3 do scorecard mais `purpose`, `housing` e `years_employment`:

| Métrica (teste) | Regressão logística | Scorecard de regras |
|------------------|----------------------|----------------------|
| AUC              | 0,743                | 0,741                |
| Gini             | 0,485                | 0,482                |
| KS               | 0,408                | 0,371                |

Adicionar `purpose`, `housing` e `years_employment` não melhorou o AUC nem o Gini nesta amostra (Gini +0,004). Com 300 clientes no teste, essa diferença está dentro da margem de erro, então o resultado indica ausência de ganho detectável, não equivalência entre os modelos. Com desempenho semelhante, o scorecard tem a vantagem de ser simples de explicar e auditar.

## Limitações

- **Benchmark parcial:** a regressão usou 6 das 20 variáveis. O sinal das demais (idade, valor do crédito, duração, etc.) não foi testado, então não é possível afirmar que o scorecard esgota o sinal da base
- **Teste pequeno:** 300 clientes (~90 maus pagadores). Diferenças de AUC, Gini e KS estão dentro da margem de erro
- **Métricas de ranking não mostram custo nem erros:** AUC, Gini e KS medem ordenação. O custo esperado 5:1 no ponto de corte foi calculado só para o scorecard, não para a regressão, então a comparação de custo entre os modelos não foi feita
- **Um único split treino/teste**, sem validação cruzada nem intervalo de confiança
- Pesos por faixa definidos por ranking de taxa de default, não por WOE/IV formal ou otimização — próximo passo natural
- **Nenhuma análise de fairness foi feita.** Variáveis sensíveis (`status_and_sex`, `is_foreign_worker`) não entraram no modelo, mas viés indireto via variáveis correlacionadas (`housing`, `job`, `purpose`) não foi descartado
- Dataset sintético/histórico (1994), sem dimensão temporal — não é possível avaliar estabilidade da política ao longo do tempo
- Amostra pequena (1.000 clientes): categorias como "retraining" (9 clientes) e scores extremos (0 e 9, <16 clientes cada) têm taxas pouco confiáveis isoladamente

## Possíveis extensões

- Comparar o custo esperado 5:1 da regressão em seu corte ótimo com o do scorecard
- Adicionar uma regressão só com as 3 variáveis do scorecard, para separar o efeito do método do efeito das variáveis extras, e outra com as 20 variáveis
- Validação cruzada repetida e/ou bootstrap para obter intervalos de confiança das diferenças de AUC, Gini e KS
- Análise de fairness (taxa de aprovação e de erro por subgrupo sensível) antes de qualquer uso em produção
- Calibrar pesos do scorecard via WOE/IV

## Feedback recebido

Após a publicação do projeto no LinkedIn, um comentário de uma cientista de dados apontou limites na conclusão do benchmark. Os pontos levantados foram:

1. **A conclusão de que as 3 variáveis "capturam quase todo o sinal disponível" era forte demais.** AUC de 0,741 contra 0,743 não é suficiente para concluir que não há informação relevante nas demais variáveis.
2. **AUC, Gini e KS avaliam essencialmente a capacidade de ordenação (discriminação).** Em problemas de inadimplência, podem continuar razoáveis mesmo quando o modelo ainda erra uma quantidade relevante de casos, e não mostram onde estão os erros, como eles se distribuem no ponto de corte ou quanto custam.
3. **A análise de custo e de inadimplência no ponto de corte traz informação que essas métricas não capturam**, e ela foi feita só para o scorecard, não para a regressão.

Na revisão do feedback, foram confirmados também outros limites da análise: a regressão usou 6 das 20 variáveis, o teste tem apenas 300 clientes e foi usado um único split treino/teste, sem validação cruzada.

**Status:** a conclusão foi reescrita neste README como ausência de ganho detectável, não como equivalência entre os modelos. A correção da análise (ver [Possíveis extensões](#possíveis-extensões)) será feita em breve, e este documento será atualizado com os resultados.

## Tecnologias

- Python (pandas, numpy)
- Visualização: matplotlib, seaborn
- Modelagem: scikit-learn (regressão logística, métricas de classificação)
- Ambiente: Google Colab

## Fonte dos dados

[Statlog (German Credit Data) — UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/144/statlog), compilado pelo Prof. Dr. Hans Hofmann (Institut für Statistik und Ökonometrie, Universität Hamburg).

## Autor

Gabriel Guaitolini Roledo — [GitHub](https://github.com/Gabriel-Roledo-ds)
