# Experimentação e Validação de Modelos em Fraud Analytics

## Visão Geral

Este projeto demonstra um fluxo de experimentação aplicado a **Fraud Analytics**, partindo de uma hipótese de negócio até a avaliação de uma possível mudança em produção.

O objetivo não é simplesmente encontrar um modelo com boas métricas, mas responder a uma pergunta mais importante:

> **Existe evidência suficiente de que uma nova feature melhora o sistema atual e justifica uma mudança em produção?**

O estudo percorre:

**EDA → Hipótese → Feature Engineering → Baseline → Modelo Candidato → Avaliação Offline → Simulação A/B → Decisão de Negócio**

---

## Problema

Em sistemas antifraude, determinados comportamentos podem indicar maior risco, como:

- valores de transação elevados;
- contas mais recentes;
- várias tentativas em curto período;
- utilização de dispositivo novo;
- compras realizadas durante a madrugada.

Entretanto, um comportamento anômalo não significa necessariamente fraude.

Por isso, antes de recomendar qualquer alteração em produção, é necessário transformar observações em hipóteses e testar se elas realmente acrescentam poder preditivo ao modelo.

---

## Dataset

Foi utilizado um dataset sintético com **3.000 transações**, contendo informações comportamentais e transacionais.

Principais variáveis:

- `valor_transacao`
- `ticket_medio_cliente`
- `idade_conta_dias`
- `tentativas_24h`
- `dispositivo_novo`
- `compra_madrugada`
- `fraude_real`

A distribuição da variável alvo foi:

- **90,6%** transações legítimas
- **9,4%** transações fraudulentas

Esse desbalanceamento reforça a importância de métricas como **Precision, Recall e PR-AUC**, em vez de utilizar apenas Accuracy.

---

## Análise Exploratória

A EDA mostrou diferenças relevantes entre transações legítimas e fraudulentas.

| Indicador | Legítima | Fraude |
|---|---:|---:|
| Valor médio da transação | 229,03 | 366,82 |
| Ticket médio do cliente | 179,51 | 192,25 |
| Idade média da conta | 1.034,88 dias | 729,63 dias |
| Tentativas em 24h | 1,92 | 2,65 |
| Dispositivo novo | 17,77% | 41,84% |
| Compra na madrugada | 13,65% | 29,08% |

As fraudes apresentaram, em média, transações maiores, contas mais recentes, maior número de tentativas, maior utilização de dispositivos novos e maior ocorrência durante a madrugada.

Esses resultados indicam associação, mas não são suficientes para determinar individualmente se uma transação é fraudulenta.

---

## Hipótese

A análise sugeriu que o valor absoluto de uma transação talvez não fosse suficiente para representar adequadamente o comportamento do cliente.

Foi formulada a hipótese:

> **Transações cujo valor está muito acima do comportamento histórico de gasto do cliente podem apresentar maior risco de fraude.**

Para testar essa hipótese foi criada uma nova feature:

```python
razao_valor_ticket = valor_transacao / ticket_medio_cliente
```

Essa variável contextualiza a transação atual em relação ao padrão histórico do próprio cliente.

---

## Feature Engineering

A média da nova variável apresentou diferença entre as classes:

| Classe | Razão média |
|---|---:|
| Transação legítima | 1,88 |
| Fraude | 2,95 |

Fraudes apresentaram, em média, valores aproximadamente **2,95 vezes o ticket médio histórico**, contra **1,88** nas transações legítimas.

A análise exploratória indicou potencial discriminativo.

Entretanto:

> **Associação exploratória não garante ganho preditivo incremental.**

Por isso, a feature foi submetida a uma avaliação offline.

---

## Modelo Baseline

Foi treinado um modelo de **Regressão Logística** sem a nova feature.

A matriz de confusão obtida foi:

```text
[[810   5]
 [ 72  13]]
```

Principais resultados para fraude:

- **Precision:** ~72,2%
- **Recall:** ~15,3%
- **PR-AUC:** ~0,425

O modelo apresentou comportamento conservador: poucos falsos positivos, porém deixou passar grande quantidade de fraudes.

---

## Modelo Candidato

Um segundo modelo foi treinado utilizando as mesmas variáveis do baseline, acrescentando:

```text
razao_valor_ticket
```

Para a classe fraude, foram obtidos:

- **Precision:** 65,0%
- **Recall:** 15,29%
- **F1-score:** 24,76%

O Recall permaneceu praticamente inalterado, enquanto a Precision caiu.

A nova feature apresentou associação com fraude durante a EDA, mas **não produziu ganho incremental suficiente no modelo candidato**.

---

## Decisão Offline

Com base nos resultados, o modelo candidato **não seria recomendado como substituto do baseline**.

A redução da Precision representa aumento de falsos positivos sem melhoria relevante na captura das fraudes.

Em ambiente real, falsos positivos podem representar:

- bloqueios de transações legítimas;
- aumento de análises manuais;
- maior demanda de atendimento;
- pior experiência do cliente;
- aumento de custos operacionais;
- possível perda de clientes.

Portanto, uma alteração em produção precisa ser justificada não apenas por métricas de Machine Learning, mas também pelo impacto econômico e operacional.

---

## Avaliação Online e Teste A/B

Em um cenário no qual um modelo candidato apresente evidências offline suficientes, o próximo passo pode ser uma validação controlada em produção.

Exemplo:

```text
Tráfego
   |
Randomização
 /          \
Grupo A     Grupo B
Baseline    Candidato
```

Além de métricas de Machine Learning, uma avaliação online poderia acompanhar:

- fraude capturada;
- falsos positivos;
- chargeback;
- taxa de aprovação;
- bloqueios indevidos;
- reclamações;
- custo operacional;
- latência.

---

## Guardrail Metrics

Reduzir fraude não significa necessariamente melhorar o sistema.

Um modelo pode reduzir perdas financeiras e simultaneamente aumentar excessivamente o bloqueio de clientes legítimos.

Por isso, métricas de proteção — **guardrails** — devem acompanhar o experimento.

Exemplos:

**Métrica principal**

- redução das perdas com fraude.

**Guardrails**

- taxa de falsos positivos;
- taxa de aprovação;
- clientes legítimos bloqueados;
- reclamações;
- latência.

Uma melhoria na métrica principal não deve justificar automaticamente uma mudança caso métricas críticas de proteção sejam deterioradas.

---

## Significância Estatística × Relevância de Negócio

Resultados de experimentos também precisam ser avaliados considerando a possibilidade de variação aleatória.

Testes estatísticos e intervalos de confiança ajudam a avaliar se uma diferença observada entre grupos possui evidência estatística.

Entretanto:

> **Significância estatística não significa necessariamente relevância econômica.**

Uma mudança pode ser estatisticamente detectável e, ainda assim, pequena demais para justificar custo, risco ou complexidade de implementação.

---

## Principal Aprendizado

O projeto mostrou uma situação importante em Ciência de Dados:

> Uma feature pode parecer promissora durante a análise exploratória e ainda assim não gerar ganho incremental suficiente quando incorporada ao modelo.

Esse resultado não representa falha do experimento.

Representa exatamente sua função:

**testar uma hipótese antes de assumir o risco de alterar produção.**

---

## Stack

- Python
- Pandas
- NumPy
- Matplotlib
- Scikit-learn
- Logistic Regression
- Precision / Recall
- PR-AUC
- Feature Engineering
- A/B Testing
- Experimentação

---

## Próximos Passos

Possíveis evoluções incluem:

- otimização do threshold;
- criação de novas features comportamentais;
- análise temporal;
- validação estatística formal do experimento A/B;
- estimativa monetária do custo de falsos positivos;
- monitoramento de drift;
- avaliação de modelos alternativos.

Uma evolução adicional seria incorporar **features relacionais baseadas em grafos**, permitindo representar relações entre clientes, dispositivos, contas, transações e outros elementos do ecossistema antifraude.

---

## Conclusão

Modelos antifraude não devem ser avaliados apenas pela capacidade de classificar transações.

Uma solução em produção precisa equilibrar:

**detecção de fraude + impacto financeiro + custo operacional + experiência do cliente.**

A pergunta final deixa de ser:

> “O novo modelo possui uma métrica melhor?”

e passa a ser:

> **“Existe evidência suficiente de que essa mudança melhora o sistema como um todo e justifica o risco de colocá-la em produção?”**

---

**Alexandre Lotti <!LOTTI>**

*Fraud Analytics | Graph Intelligence | Machine Learning | Neo4j | Python*