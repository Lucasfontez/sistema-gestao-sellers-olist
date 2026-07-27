# 03 — ETL e modelagem

Todo o ETL foi feito no Power Query dentro do Power BI. O Excel serviu só para exploração; nada do que foi analisado lá entrou no modelo.

## Arquitetura em camadas

Organizei as consultas em três camadas:

**`stg_*`** — as seis tabelas cruas, importadas direto dos CSVs originais. Recebem tratamento de tipo, renomeação de colunas e filtros técnicos.

**`aux_*`** — consultas intermediárias de agregação. Existem só para alimentar as dimensões e não vão para o modelo.

**`dVendedor`, `dCategoria`, `dCalendario`, `fVendas`** — o modelo dimensional final.

As camadas `stg_` e `aux_` ficam com carga desabilitada. Elas continuam vivas no Power Query alimentando o resto, mas não aparecem no painel de campos.

A separação importa porque uma dimensão não é uma cópia de uma tabela de origem. `dVendedor` tem sete colunas, e quatro delas não existem em `olist_sellers_dataset` — vêm de agregações sobre outras tabelas.

## Uma regra que custou caro aprender

Toda `stg_` importa direto do CSV em `dados/raw/`. Nenhuma importa de arquivo intermediário.

Eu quebrei essa regra sem perceber. Em algum momento consolidei os CSVs num Excel e passei a importar de lá. O resultado foi um bug de escala de 100× que só apareceu duas etapas depois. A história completa está em [06-decisoes-e-aprendizados](06-decisoes-e-aprendizados.md); o resumo é que o Excel interpretou `58.90` como `5890` e o Power Query importou o inteiro sem reclamar.

O código correto, importando do CSV e declarando os tipos numéricos como `type number` em vez de `Int64.Type`:

```m
let
    Fonte = Csv.Document(
        File.Contents("...\dados\raw\olist_order_items_dataset.csv"),
        [Delimiter=",", Columns=7, Encoding=65001, QuoteStyle=QuoteStyle.None]
    ),
    #"Cabeçalhos Promovidos" = Table.PromoteHeaders(Fonte, [PromoteAllScalars=true]),
    TipoAlterado = Table.TransformColumnTypes(#"Cabeçalhos Promovidos", {
        {"order_id", type text},
        {"order_item_id", Int64.Type},
        {"product_id", type text},
        {"seller_id", type text},
        {"shipping_limit_date", type datetime},
        {"price", type number},
        {"freight_value", type number}
    }),
    ColunasRenomeadas = Table.RenameColumns(TipoAlterado, {
        {"order_id", "ID Pedido"},
        {"seller_id", "ID Vendedor"},
        {"price", "Preço"},
        {"freight_value", "Frete"}
    })
in
    ColunasRenomeadas
```

O `Encoding=65001` é UTF-8. Sem ele, "beleza_saude" vira "beleza_saÃºde".

## Tratamentos por tabela

**`stg_pedidos`** — filtro `Status Pedido = "delivered"`. Reduz de 99.441 para cerca de 96.478 pedidos. Também removi as quatro datas de logística e o ID de cliente: são o eixo do projeto operacional, não deste.

**`stg_avaliacoes`** — ordenação decrescente por data e remoção de duplicatas por `ID Pedido`. Fica a avaliação mais recente de cada pedido. De 99.224 para 98.673 linhas.

**`stg_produtos`** — substituição de categoria nula por "Não Informado".

**`stg_vendedores`** — sem tratamento. Só renomeação.

## Região a partir da UF

A coluna `Região` não existe no dataset. Criei mapeando a UF:

```m
#"Região Adicionada" = Table.AddColumn(Fonte, "Região", each 
    if List.Contains({"SP","RJ","MG","ES"}, [UF Vendedor]) then "Sudeste"
    else if List.Contains({"RS","SC","PR"}, [UF Vendedor]) then "Sul"
    else if List.Contains({"BA","PE","CE","MA","PB","RN","AL","PI","SE"}, [UF Vendedor]) then "Nordeste"
    else if List.Contains({"GO","MT","MS","DF"}, [UF Vendedor]) then "Centro-Oeste"
    else if List.Contains({"PA","AM","TO","RO","AP","AC","RR"}, [UF Vendedor]) then "Norte"
    else "Não Informado", type text)
```

Fiz por código em vez da interface de coluna condicional porque o diálogo do Power Query exige uma cláusula por valor — seriam 27 cláusulas. Com `List.Contains` são cinco linhas, e o código fica legível no repositório.

O `else "Não Informado"` no fim é rede de segurança. Se aparecesse UF fora do padrão, ela seria isolada em vez de quebrar. Não apareceu nenhuma.

Optei por região como coluna de `dVendedor` e não como dimensão separada. Cinco regiões numa hierarquia rasa não justificam uma tabela — seria complexidade sem ganho analítico.

## Agregações auxiliares

`dVendedor` precisa saber quantos pedidos cada vendedor teve e quando foi o primeiro. Esses números vivem em `order_items` e `orders`, não em `sellers`.

Não dá para fazer merge direto: `order_items` tem 112 mil linhas contra 3 mil de `sellers`. Um merge de muitos-para-muitos explodiria a dimensão e destruiria o modelo. A dimensão precisa ter uma linha por vendedor.

A solução é agregar antes de juntar.

**`aux_pedidos_por_vendedor`** — agrupa `order_items` por vendedor contando pedidos distintos:

```m
Agrupado = Table.Group(Fonte, {"ID Vendedor"}, {
    {"Qtd Pedidos Total", each List.Count(List.Distinct([ID Pedido])), Int64.Type}
})
```

Escrevi em M porque a opção "Contagem Distinta" não aparece no diálogo de agrupamento da versão em português. O que aparece é "Contar Linhas Distintas", que é outra coisa: conta combinações únicas de todas as colunas. Como `Item Sequência` varia dentro do mesmo pedido, praticamente nada é deduplicado e o resultado conta itens em vez de pedidos.

Descobri porque o vendedor do topo apareceu com 2.033 pedidos quando o Excel tinha dado 1.854. A diferença de 179 era exatamente a inflação por itens repetidos.

**`aux_primeiro_pedido_vendedor`** — junta `order_items` com `orders` para trazer a data, depois agrupa por vendedor pegando o mínimo. A junção é interna, o que descarta automaticamente itens de pedidos não entregues.

## As dimensões

**`dVendedor`** — 2.970 linhas, 7 colunas.

Construída por referência a `stg_vendedores`, enriquecida com região, quantidade de pedidos, data do primeiro pedido e flag de elegibilidade:

```m
ColunaCondicionalMatriz = Table.AddColumn(#"Colunas Renomeadas", "Elegível Matriz", 
    each if [Qtd Pedidos Total] >= 5 then true else false)
```

Depois dos merges, 125 vendedores ficaram sem data de primeiro pedido. São vendedores que existem no cadastro mas não têm nenhum pedido entregue no período — todos os pedidos deles foram cancelados ou não chegaram. Removi.

A base caiu de 3.095 para 2.970. É coerente com o filtro de status: vendedor sem transação completa não tem performance para medir.

**`dCategoria`** — 72 linhas.

As 71 categorias oficiais mais "Não Informado", inserida manualmente porque ela existe em `stg_produtos` depois do tratamento mas não na tabela de referência:

```m
#"Linha Não Informado Adicionada" = Table.InsertRows(Fonte, 0, 
    {[Nome Categoria = "Não Informado"]})
```

Sem essa linha, a categoria ficaria órfã no relacionamento.

**`dCalendario`** — cerca de 1.096 linhas.

Gerada por código, com a janela derivada dos próprios dados:

```m
let
    Fonte = stg_pedidos[Data Compra],
    DataMin = Date.StartOfYear(List.Min(Fonte)),
    DataMax = Date.EndOfYear(List.Max(Fonte)),
    QtdeDias = Duration.Days(DataMax - DataMin) + 1,
    ListDates = List.Dates(DataMin, QtdeDias, #duration(1,0,0,0)),
    ParaTabela = Table.FromList(ListDates, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    NomeMesInserido = Table.AddColumn(ParaTabela, "NomeMes", each Date.MonthName([Column1]), type text),
    NumMesInserido = Table.AddColumn(NomeMesInserido, "NumMes", each Date.Month([Column1]), Int64.Type),
    AnoInserido = Table.AddColumn(NumMesInserido, "Ano", each Date.Year([Column1]), Int64.Type)
in
    AnoInserido
```

Derivar a janela dos dados em vez de fixar datas significa que, se a base for atualizada, o calendário acompanha sozinho.

As colunas de nome de mês precisam de coluna de ordenação. Sem isso, o Power BI ordena alfabeticamente e abril vem antes de janeiro. Configurei `NomeMesAbreviado` para classificar por `NumMes` em Ferramentas de Coluna.

## A tabela fato

**`fVendas`** — 97.819 linhas, 7 colunas.

Grão: uma linha por combinação de vendedor e pedido. Se um vendedor vendeu três itens no mesmo pedido, aparece uma vez com a receita somada.

A construção tem três etapas.

**Enriquecer** — merge com `stg_pedidos` (junção interna, que já filtra por entregue) para trazer a data, e com `stg_produtos` (junção externa esquerda) para trazer a categoria de cada item. Resulta em cerca de 110 mil linhas no grão de item.

**Agregar** — agrupamento por vendedor e pedido, resolvendo a categoria dominante:

```m
ValorItemAdicionado = Table.AddColumn(Anterior, "Valor Item", 
    each [Preço] + [Frete], type number),

Agrupado = Table.Group(
    ValorItemAdicionado, 
    {"ID Vendedor", "ID Pedido"}, 
    {
        {"Data Compra", each List.First([Data Compra]), type date},
        {"Nome Categoria", each 
            let 
                OrdenadoPorValor = Table.Sort(_, {{"Valor Item", Order.Descending}}),
                PrimeiraLinha = Table.First(OrdenadoPorValor)
            in 
                PrimeiraLinha[Nome Categoria], 
            type text},
        {"Receita Item", each List.Sum([Valor Item]), type number},
        {"Qtd Itens", each Table.RowCount(_), Int64.Type}
    }
)
```

A categoria dominante é a do item de maior valor no pedido. Um pedido com um tênis de R$ 500 e um livro de R$ 50 é um pedido de calçados, não de livros. A alternativa seria concatenar as categorias, o que inviabilizaria a análise de especialização.

Esse trecho não tem equivalente na interface — o "Agrupar Por" não faz "o valor da coluna X na linha de maior Y".

**Trazer a nota** — merge com `stg_avaliacoes` por pedido, junção externa esquerda. Left join porque nem todo pedido tem avaliação, e perder esses pedidos distorceria o faturamento.

## O modelo

```
        dVendedor              dCalendario
            │                       │
            │ 1                   1 │
            │                       │
            └──────── fVendas ──────┘
                         │
                       * │
                         │
                    dCategoria
```

| Relacionamento | Cardinalidade | Direção |
|---|---|---|
| `dVendedor[ID Vendedor]` → `fVendas[ID Vendedor]` | 1:N | única |
| `dCategoria[Nome Categoria]` → `fVendas[Nome Categoria]` | 1:N | única |
| `dCalendario[Data]` → `fVendas[Data Compra]` | 1:N | única |

Star schema puro. Todas as direções únicas, das dimensões para o fato. Filtro bidirecional não é necessário aqui e criaria ambiguidade.

`dCalendario` está marcada como Tabela de Datas, o que habilita as funções de time intelligence do DAX.

## Validação

O primeiro cross-check entre Excel e Power BI foi na contagem de pedidos por vendedor. O Excel tinha dado 1.854 para o maior vendedor; a agregação do Power Query, depois de corrigida, deu 1.854. Os três primeiros bateram: 1.854, 1.806, 1.706.

O segundo foi indireto e mais interessante. A metade inferior da curva de Pareto (decis 6 a 10) responde por 3,2% do faturamento. A soma dos quatro quadrantes fecha em 97% do total — a diferença de 3% são os vendedores não elegíveis. Duas medidas construídas de formas completamente diferentes chegando no mesmo número.
