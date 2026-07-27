# 02 — Dados e profiling

Antes de construir qualquer coisa, carreguei as seis tabelas no Excel e fiz profiling coluna a coluna. O objetivo não era conhecer os dados em abstrato — era responder duas perguntas que travavam o projeto inteiro.

## As tabelas

| Tabela | Linhas | Papel |
|---|---|---|
| `olist_sellers_dataset` | 3.095 | Cadastro de vendedores |
| `olist_products_dataset` | 32.951 | Produtos e categorias |
| `olist_orders_dataset` | 99.441 | Pedidos e status |
| `olist_order_items_dataset` | 112.650 | Itens, preço, frete, vínculo vendedor-pedido |
| `olist_order_reviews_dataset` | 99.224 | Avaliações |
| `product_category_name_translation` | 71 | Lista de categorias |

## Como fiz o profiling

Montei uma matriz por tabela com uma linha por coluna e sete métricas: tipo, total de linhas, nulos, percentual de nulos, valores distintos, percentual de distintos e uma amostra.

Usei `INDIRETO` para que a fórmula lesse o nome da coluna da própria célula, o que transformou a planilha num template reutilizável — bastava trocar o nome da tabela para replicar nas outras cinco.

```excel
Linhas totais:  =LINS(INDIRETO("stg_vendedores[" & A2 & "]"))
Nulos:          =CONTAR.VAZIO(INDIRETO("stg_vendedores[" & A2 & "]"))
Distintos:      =SOMARPRODUTO((INDIRETO(...)<>"")/CONT.SE(INDIRETO(...);INDIRETO(...)&""))
```

O `<>""` no numerador e o `&""` no `CONT.SE` existem porque a versão clássica dessa fórmula quebra com `#DIV/0!` quando há células vazias na coluna.

Essa abordagem tem um custo que descobri na prática: `INDIRETO` é função volátil e `SOMARPRODUTO` com `CONT.SE` é O(n²). Em `order_items`, com 112 mil linhas, isso são bilhões de comparações a cada recálculo. O arquivo travou. Resolvi mudando o cálculo para manual, congelando os resultados como valores e salvando em `.xlsb`.

## O que o profiling revelou

**610 produtos sem categoria (1,9%).** Não dava para descartar sem perder parte do catálogo na análise de especialização. Virou categoria "Não Informado".

**551 pedidos com mais de uma avaliação (0,6%).** São 99.224 reviews para 98.673 pedidos distintos. Precisava de um critério de desempate.

**2.965 pedidos sem data de entrega (3%).** São pedidos cancelados, extraviados ou ainda em trânsito. Confirmou a necessidade de filtrar por status.

**775 pedidos que existem em `orders` mas não têm nenhum item em `order_items` (0,78%).** Um pedido sem item é um pedido inexistente para efeito de análise de vendedor.

**Oito status distintos de pedido.** `delivered`, `shipped`, `canceled`, `unavailable`, `invoiced`, `processing`, `approved`, `created`.

**Todos os 3.095 vendedores cadastrados aparecem em pelo menos um item.** Não há vendedor fantasma no cadastro.

## Análise 1 — Qual o piso mínimo de pedidos?

Um vendedor com dois pedidos e duas notas cinco tem média 5,0. Isso não é performance, é ruído. Precisava de um corte, mas queria que ele viesse do dado e não de palpite.

Fiz uma tabela dinâmica sobre `order_items` com `seller_id` nas linhas e contagem distinta de `order_id` nos valores, depois agrupei em faixas:

| Faixa | Vendedores | % |
|---|---|---|
| 1 pedido | 571 | 18,4% |
| 2 a 4 pedidos | 730 | 23,6% |
| 5 ou mais | 1.795 | 58,0% |

A distribuição decide sozinha. Quarenta e dois por cento da base tem menos de cinco pedidos — é cauda de vendedor ocasional, gente que abriu, vendeu meia dúzia e sumiu. Os 58% restantes têm histórico suficiente para leitura.

Piso definido em **5 pedidos**.

Um detalhe importante: contagem **distinta** de `order_id`, não contagem de linhas. Um vendedor pode ter três itens no mesmo pedido — contar linhas infla o número. Perdi tempo com isso, e conto o erro em [06-decisoes-e-aprendizados](06-decisoes-e-aprendizados.md).

## Análise 2 — Quantos pedidos têm mais de um vendedor?

A tabela de avaliações é por pedido, não por vendedor. Se um pedido tem dois vendedores e uma nota, a nota é de quem?

Antes de olhar o dado, defini a regra de decisão: se o volume de pedidos multi-vendedor fosse até 15%, aplicaria a nota igualmente a todos e declararia como limitação. Acima disso, teria que atribuir ao vendedor de maior valor no pedido.

Tabela dinâmica com `order_id` nas linhas e contagem distinta de `seller_id` nos valores:

| Vendedores no pedido | Pedidos | % |
|---|---|---|
| 1 | 97.388 | 98,70% |
| 2 | 1.219 | 1,24% |
| 3 ou mais | 60 | 0,06% |

**1,30%.** Um décimo do limite que eu tinha estabelecido. A decisão caiu sozinha: nota espelhada, limitação declarada.

Definir o critério antes de ver o número foi o que tornou essa decisão defensável. Se eu tivesse olhado primeiro, qualquer justificativa depois seria racionalização.

## O que ficou para o ETL

Do profiling saíram cinco tratamentos obrigatórios:

- filtrar `order_status = 'delivered'`
- deduplicar avaliações mantendo a mais recente
- substituir categoria nula por "Não Informado"
- descartar pedidos sem itens via junção interna
- aplicar o piso de 5 pedidos como flag de elegibilidade

Todos estão documentados com justificativa em [06-decisoes-e-aprendizados](06-decisoes-e-aprendizados.md).
