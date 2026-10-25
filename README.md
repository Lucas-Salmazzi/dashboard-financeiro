# Dashboard Financeiro: por que a margem está caindo?

Projeto de portfólio em **Power BI** que simula um pedido urgente do diretor financeiro (CFO) de uma empresa B2B: entender, em uma página, por que a margem está caindo.

> Os dados são \*\*fictícios\*\*, gerados para estudo. Nenhuma empresa real é representada.

!\[Dashboard financeiro](imagens/dashboard.png)

\---

## O cenário

> "A diretoria suspeita que a margem está caindo, mas ninguém sabe o porquê. Quero que o dashboard mostre onde está o problema, não só os números."

A empresa vende assinaturas, serviços e consultoria, treinamentos e licenças de software. O período analisado vai de **jan/2025 a set/2026** (21 meses).

\---

## O que o dashboard mostra

|Achado|Número|
|-|-|
|A margem caiu ao longo de 2026|**13,0%** no 1º trimestre, **9,4%** no 2º e **6,4%** no 3º (13,3% em 2025 e 9,5% em 2026)|
|A nuvem está consumindo a receita|De **5,9%** para **10,4%** da receita; **36,5% acima do orçado** em 2026|
|Marketing estourou o orçamento no inverno|Cerca de R$ 104 a 112 mil por mês em jun, jul e ago/2026 contra R$ 72 mil orçados (+16,0% no ano)|
|A inadimplência está concentrada no Varejo|De **0,6%** (2025) para **14,3%** (2026); o Varejo responde por 65% dos R$ 813 mil em atraso|
|Despesa cresce, receita não|Em 2026, despesa **+6,0%** e receita **-1,2%** em relação ao orçado|
|Dependência de poucos clientes|Os 3 maiores clientes são 32% da receita|

**Recomendações que fecham o dashboard:** renegociar contratos de nuvem, congelar o aumento de marketing até medir o retorno, cobrar os títulos do Varejo com mais de 90 dias e reduzir a dependência dos maiores clientes.

\---

## Os dados

Arquivo: [`dados/dataset\_financeiro.xlsx`](dados/dataset_financeiro.xlsx), com duas abas.

**`lancamentos`** (1.766 linhas, 13 colunas): cada linha é uma receita ou uma despesa.

|Coluna|Descrição|
|-|-|
|`id\_lancamento`|Identificador único|
|`data\_competencia`|Mês a que o lançamento pertence (base de receita, despesa, resultado e margem)|
|`data\_vencimento`|Data limite de pagamento ou recebimento|
|`data\_pagamento`|Data em que o dinheiro entrou ou saiu. Vazio = ainda não pago|
|`tipo`|`Receita` ou `Despesa`|
|`categoria`|Assinaturas, Serviços e consultoria, Treinamentos, Licenças, Folha de pagamento, Impostos, Infraestrutura/Cloud, Marketing, Aluguel, Software e licenças, Comissões, Viagens, Jurídico e contábil, Equipamentos|
|`centro\_custo`|Área responsável|
|`contraparte`|Cliente (receitas) ou fornecedor (despesas)|
|`segmento\_cliente`|Varejo, Indústria, Serviços, Saúde, Educação, Financeiro. Só nas receitas|
|`valor`|Valor em R$, **sempre positivo**|
|`forma\_pagamento`|PIX, Boleto, Transferência, Cartão ou Débito automático|
|`status`|`Pago`, `Pendente` (ainda não venceu) ou `Atrasado` (venceu e não foi pago até 30/09/2026)|
|`dias\_atraso`|Dias de atraso no pagamento, ou até 30/09/2026 se ainda aberto|

**`orcamento`** (294 linhas, 4 colunas): valor planejado por mês e categoria (`mes`, `tipo`, `categoria`, `valor\_orcado`).

### Pontos de atenção nos dados

* Valores sempre positivos: o sinal vem da coluna `tipo`, então nunca se soma a coluna `valor` inteira.
* Vazios têm significado: `data\_pagamento` vazio é "não pago" e `segmento\_cliente` vazio é despesa.
* **Competência x caixa:** o resultado usa `data\_competencia`; o fluxo de caixa usaria `data\_pagamento`.

\---

## Modelo de dados

```
Calendario\[Date] ──1:\*──> lancamentos\[data\_competencia]   (ativo)
Calendario\[Date] ──1:\*──> lancamentos\[data\_pagamento]     (inativo, para caixa)
Calendario\[Date] ──1:\*──> lancamentos\[data\_vencimento]    (inativo)
Calendario\[Date] ──1:\*──> orcamento\[mes]                  (ativo)
dim\_categoria\[categoria] ──1:\*──> lancamentos\[categoria]
dim\_categoria\[categoria] ──1:\*──> orcamento\[categoria]
```

A tabela `dim\_categoria` existe porque `lancamentos` e `orcamento` compartilham a categoria, e ligar as duas direto geraria uma relação muitos para muitos. Uma tabela de dimensão no meio resolve isso.

\---

## Principais medidas DAX

```DAX
Receita = CALCULATE(SUM(lancamentos\[valor]); lancamentos\[tipo] = "Receita")
Despesa = CALCULATE(SUM(lancamentos\[valor]); lancamentos\[tipo] = "Despesa")
Resultado = \[Receita] - \[Despesa]
Margem % = DIVIDE(\[Resultado]; \[Receita])

Var Despesa vs Orcado % = DIVIDE(\[Despesa] - \[Despesa Orcada]; \[Despesa Orcada])

Cloud % Receita =
DIVIDE(CALCULATE(\[Despesa]; lancamentos\[categoria] = "Infraestrutura/Cloud"); \[Receita])

Inadimplencia % = DIVIDE(\[Inadimplencia R$]; \[Receita Vencida])

Receita % Total =
DIVIDE(\[Receita]; CALCULATE(\[Receita]; ALL(lancamentos\[segmento\_cliente])))
```

Conceitos praticados: `CALCULATE`, `DIVIDE`, `ALL`, `VAR`, relacionamentos inativos, tabela de calendário e dimensão auxiliar.

\---

## Decisões de projeto

* **Inadimplência sobre a receita vencida**, e não sobre a total: receita que ainda nem venceu não tem como estar atrasada, e dividir pela total esconderia o problema.
* **Competência para o resultado.** A visão de caixa fica como evolução futura.
* **Uma página principal e uma de explicação** ("Sobre os dados"), para o dashboard se explicar sozinho.
* **Cor de alerta única** (laranja e vermelho) só no que está fora do esperado; azul-marinho e azul para o restante.

\---

## Como abrir

1. Baixe `dashboard\_financeiro.pbix`.
2. Abra no **Power BI Desktop** (gratuito).
3. Se o Power BI pedir, atualize o caminho da fonte para `dados/dataset\_financeiro.xlsx`.

\---

## Próximos passos

* Visão de **fluxo de caixa** (por `data\_pagamento`) com alternância entre competência e caixa.
* Previsão de margem para os próximos trimestres.
* Análise de rentabilidade por cliente.

\---

**Autor:** Lucas Salmazzi · www.linkedin.com/in/lucas-leonardo-bertolini-salmazzi-5a6146312

