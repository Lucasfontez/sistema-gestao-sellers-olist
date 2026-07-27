# 04 — Medidas e colunas DAX

O modelo tem 17 medidas e 4 colunas calculadas. Documento aqui as que carregam lógica de negócio; as aritméticas simples ficam na tabela do fim.

## A distinção que importa: medida ou coluna

Levei um tempo para internalizar isso, e o projeto me forçou a aprender na prática.

**Medida** é avaliada no contexto do visual. Muda quando você filtra. Não pode ser usada em eixo, legenda ou fatia — só em valores.

**Coluna calculada** é avaliada uma vez, no refresh, e fica gravada na tabela. Não responde a filtro. Pode ser usada em qualquer lugar.

Criei `Quadrante do Vendedor` como medida e ela funcionou nos cartões. Quando fui montar o gráfico de dispersão e precisei dela na legenda, o Power BI recusou. Tive que reescrever como coluna.

A consequência dessa mudança é que a classificação passou a ser fixa: um vendedor classificado como Risco Crítico continua Risco Crítico mesmo com filtro de 2018 aplicado. Isso é defensável — quadrante é diagnóstico estrutural do vendedor, não recorte do mês — mas precisa estar declarado, porque produz um comportamento estranho no dashboard: a contagem de vendedores por quadrante não muda com o filtro, enquanto os percentuais de faturamento mudam.

## As medidas base

```dax
Faturamento Total = SUM(fVendas[Receita Item])
```

```dax
Qtd Pedidos = DISTINCTCOUNT(fVendas[ID Pedido])
```

Contagem distinta, não contagem de linhas. Um pedido com dois vendedores aparece em duas linhas do fato; contar linhas inflaria o número.

```dax
Ticket Médio = DIVIDE([Faturamento Total], [Qtd Pedidos])
```

`DIVIDE` em vez do operador `/` porque protege contra divisão por zero sem precisar de `IF`.

```dax
Nota Média Vendedor = AVERAGE(fVendas[Nota Avaliação])
```

`AVERAGE` ignora nulos automaticamente. Cerca de 1% dos pedidos não tem avaliação, e eles simplesmente não entram na média.

```dax
Qtd Vendedores Elegíveis = 
CALCULATE(
    DISTINCTCOUNT(dVendedor[ID Vendedor]),
    dVendedor[Elegível Matriz] = TRUE()
)
```

Escrevi a primeira versão comparando com a string `"Sim"`. A coluna é booleana. A comparação não deu erro — retornou zero silenciosamente. Erro de tipo em DAX raramente grita.

## A coluna de quadrante

É o centro do projeto.

```dax
Quadrante = 
VAR MedVolume =
    MEDIANX(
        FILTER( ALL( dVendedor ), dVendedor[Elegível Matriz] = TRUE() ),
        dVendedor[Qtd Pedidos Total]
    )
VAR MedNota =
    MEDIANX(
        FILTER( ALL( dVendedor ), dVendedor[Elegível Matriz] = TRUE() ),
        CALCULATE( [Nota Média Vendedor] )
    )
VAR Volume = dVendedor[Qtd Pedidos Total]
VAR Nota = CALCULATE( [Nota Média Vendedor] )
RETURN
    SWITCH(
        TRUE(),
        NOT dVendedor[Elegível Matriz], BLANK(),
        ISBLANK( Nota ), BLANK(),
        Volume >= MedVolume && Nota >= MedNota, "Estrela",
        Volume >= MedVolume && Nota <  MedNota, "Risco Crítico",
        Volume <  MedVolume && Nota >= MedNota, "Cavalo de Batalha",
        "Cauda Longa"
    )
```

Três decisões dentro dessas vinte linhas.

**Mediana, não média.** Faturamento e volume têm distribuição muito assimétrica — o maior vendedor tem 1.854 pedidos, a mediana é 17. A média seria puxada pelos gigantes e classificaria quase todo mundo como "baixo volume". A mediana corta a base ao meio por construção.

**Volume, não faturamento.** Essa é a decisão que quase passou batido. Minha primeira versão usava `[Faturamento Total]` no eixo horizontal, e as duas correlacionam bastante, então a distribuição não ficava absurda.

Mas a tese do projeto diz explicitamente que a régua correta não é faturamento absoluto. Usar faturamento na classificação seria contradizer a própria tese dentro do código. Reescrevi para `Qtd Pedidos Total`. Faturamento virou o tamanho da bolha no gráfico de dispersão, que é onde ele agrega valor: mostra quanto dinheiro está em jogo em cada quadrante.

**Mediana calculada sobre os elegíveis.** O `FILTER(ALL(dVendedor), ...)` restringe o cálculo aos 1.794 com cinco ou mais pedidos. Sem isso, os 1.176 inelegíveis — todos com volume baixíssimo — puxariam a mediana para baixo e distorceriam a classificação.

Os cortes resultantes: **17 pedidos** e **nota 4,14**.

O `ISBLANK(Nota)` na segunda cláusula existe por causa de um caso de borda: há um vendedor elegível que nunca recebeu avaliação nenhuma. Sem nota, não dá para posicionar no eixo vertical. Por isso a soma dos quatro quadrantes dá 1.793 e não 1.794.

## Contagem por quadrante

```dax
Qtd Estrelas = CALCULATE( COUNTROWS( dVendedor ), dVendedor[Quadrante] = "Estrela" )
```

Uma para cada quadrante, mudando só a string. Simples porque `Quadrante` é coluna — se fosse medida, precisaria de `FILTER(VALUES(...))` e ficaria mais lento.

A validação é somar as quatro e conferir contra 1.794. Se alguma string divergir do que a coluna produz, a medida retorna zero em silêncio.

```dax
% Fat Risco Crítico = 
DIVIDE(
    CALCULATE( [Faturamento Total], dVendedor[Quadrante] = "Risco Crítico" ),
    [Faturamento Total]
)
```

O denominador é o faturamento total, não o dos elegíveis. As quatro somam cerca de 97% — a diferença são os vendedores fora da matriz. Isso é informação, não erro de arredondamento: os 1.176 inelegíveis movimentam 3% da receita.

## A curva de Pareto

Um Pareto clássico colocaria cada vendedor no eixo. São 2.970 — em 754 pixels, vira mancha. Agrupei em decis.

```dax
Decil Faturamento = 
VAR Posicao =
    RANKX( ALL(dVendedor), CALCULATE([Faturamento Total]), , DESC, Dense )
VAR TotalVendedores = COUNTROWS( ALL(dVendedor) )
RETURN
    "D" & FORMAT( CEILING( DIVIDE(Posicao, TotalVendedores) * 10, 1 ), "00" )
```

Retorna `D01` a `D10`. O `FORMAT(..., "00")` põe zero à esquerda para o texto ordenar sozinho, sem precisar de coluna auxiliar de classificação.

```dax
% Faturamento Acumulado = 
VAR DecilAtual = MAX( dVendedor[Decil Faturamento] )
VAR FatTotal =
    CALCULATE( [Faturamento Total], REMOVEFILTERS( dVendedor[Decil Faturamento] ) )
VAR FatAcum =
    CALCULATE(
        [Faturamento Total],
        REMOVEFILTERS( dVendedor[Decil Faturamento] ),
        dVendedor[Decil Faturamento] <= DecilAtual
    )
RETURN
    DIVIDE( FatAcum, FatTotal )
```

O `REMOVEFILTERS` é o que faz funcionar. Dentro de cada coluna do gráfico, o contexto natural filtra aquele decil específico; removendo esse filtro e reaplicando como "menor ou igual ao atual", chega-se ao acumulado.

A validação é direta: `D10` tem que fechar em 100,0%. Se fechar em outro valor, o `REMOVEFILTERS` não está pegando.

Minha primeira versão usava `TOPN` sobre vendedores individuais, calculando o acumulado ponto a ponto. Funcionava, mas era lenta — `TOPN` com 2.970 vendedores em cada célula do visual. A versão por decil é mais rápida e responde melhor à pergunta de negócio, que é sobre percentuais e não sobre vendedores específicos.

## Perfil de especialização

```dax
Qtd Categorias = 
CALCULATE(
    DISTINCTCOUNT( fVendas[Nome Categoria] ),
    ALLEXCEPT( dVendedor, dVendedor[ID Vendedor] )
)
```

```dax
Perfil Vendedor = 
IF( dVendedor[Qtd Categorias] <= 2, "Especialista", "Generalista" )
```

O `ALLEXCEPT` remove todos os filtros de `dVendedor` menos o próprio vendedor — é o que permite contar categorias distintas por vendedor dentro de uma coluna calculada.

O corte em duas categorias é arbitrário. Testei mentalmente 1, 2 e 3 e fiquei com 2 por distribuir melhor a base, mas não rodei um teste formal. É uma escolha frágil que declaro como tal.

Também criei as duas primeiro como medidas, e o Power BI recusou `Perfil Vendedor` no eixo. Mesmo erro do quadrante, cometido duas vezes.

## Medidas que apaguei

Terminei a etapa de DAX com 23 medidas e apaguei seis. Vale registrar por quê, porque a tentação de acumular é real.

`Faturamento por Vendedor` — retornava exatamente `[Faturamento Total]`. Medida responde ao contexto do visual; se o eixo é vendedor, o cálculo já é por vendedor. A medida existia por não entender isso.

`Volume por Vendedor` — somava `Qtd Itens`, mas "volume" na tese do projeto é pedidos, não itens. Nome enganoso.

`Ranking Vendedor` — o Top N do visual faz o mesmo trabalho sem medida.

`Mediana Faturamento` — sobra da versão em que a classificação usava faturamento.

`Nota Média Geral` — era para um cartão na capa, que depois virou estática.

`Cor por Quadrante` — retornava hex por quadrante. Quando `Quadrante` virou coluna, a cor passou a ser configurada direto na formatação do visual.

`Mediana Nota` — essa merece atenção. Ela existia e retornava 4,22, enquanto a coluna `Quadrante` classificava usando 4,14. Dois cortes diferentes convivendo no mesmo modelo.

A causa: medida responde ao contexto de filtro do visual, coluna não. Com 2018 selecionado, a medida calculava a mediana das notas daquele ano; a coluna sempre calcula sobre a base inteira.

Isso gerou um bug visual que demorei a diagnosticar — a linha de referência do gráfico de dispersão estava em 4,22 e a classificação em 4,14, então vendedores entre esses dois valores apareciam do lado errado da linha. Apaguei a medida. Manter no modelo um número que contradiz a classificação é armadilha para mim mesmo daqui a dois meses.

## Inventário

| Nome | Tipo | Papel |
|---|---|---|
| `Faturamento Total` | Medida | Soma da receita |
| `Qtd Pedidos` | Medida | Contagem distinta de pedidos |
| `Ticket Médio` | Medida | Receita por pedido |
| `Nota Média Vendedor` | Medida | Média das avaliações |
| `Qtd Vendedores Ativos` | Medida | Contagem distinta de vendedores |
| `Qtd Vendedores Elegíveis` | Medida | Vendedores com 5+ pedidos |
| `Mediana Volume` | Medida | Corte horizontal da matriz |
| `Qtd Estrelas` | Medida | Contagem do quadrante |
| `Qtd Risco Crítico` | Medida | Contagem do quadrante |
| `Qtd Cavalos de Batalha` | Medida | Contagem do quadrante |
| `Qtd Cauda Longa` | Medida | Contagem do quadrante |
| `% Fat Estrelas` | Medida | Participação no faturamento |
| `% Fat Risco Crítico` | Medida | Participação no faturamento |
| `% Fat Cavalos` | Medida | Participação no faturamento |
| `% Fat Cauda Longa` | Medida | Participação no faturamento |
| `% Faturamento Acumulado` | Medida | Curva de Pareto |
| `Quadrante` | Coluna | Classificação do vendedor |
| `Decil Faturamento` | Coluna | Agrupamento para o Pareto |
| `Qtd Categorias` | Coluna | Categorias distintas por vendedor |
| `Perfil Vendedor` | Coluna | Especialista ou generalista |
