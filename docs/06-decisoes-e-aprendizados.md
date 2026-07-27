# 06 — Decisões e aprendizados

Este documento tem duas partes: o log das decisões de tratamento, e os erros que cometi no caminho.

A segunda parte é a que me parece mais útil. Resultado limpo qualquer projeto mostra; o que diferencia é entender onde as coisas quebraram e por quê.

---

## Log de decisões

Cada decisão registra contexto, alternativas consideradas e escolha. Numerei na ordem em que apareceram.

### D-01 — Filtro por status de pedido

O dataset tem oito status. O profiling mostrou 2.965 pedidos sem data de entrega.

Considerei manter todos, manter `delivered` mais `shipped`, ou só `delivered`.

**Escolhi só `delivered`.** Análise de performance de vendedor exige transação completa. Pedido cancelado enviesa faturamento e nota simultaneamente — o comprador pode avaliar mal justamente porque cancelou.

### D-02 — Múltiplas avaliações por pedido

551 pedidos (0,6%) têm mais de uma avaliação.

Considerei média das notas, avaliação mais recente, ou primeira avaliação.

**Escolhi a mais recente.** Reflete a percepção final do comprador sobre o pedido. Volume baixo demais para justificar tratamento mais elaborado.

### D-03 — Categoria em pedidos com produtos de categorias diferentes

O grão do fato é vendedor por pedido, então cada linha precisa de uma categoria.

Considerei categoria do item de maior valor, categoria com mais itens, ou concatenar todas.

**Escolhi maior valor (preço mais frete).** Um pedido com um tênis de R$ 500 e um livro de R$ 50 é um pedido de calçados. Concatenar inviabilizaria a análise de especialização.

### D-04 — Nota em pedidos com múltiplos vendedores

1,30% dos pedidos envolvem mais de um vendedor. A avaliação é por pedido, não por vendedor.

Considerei nota espelhada para todos, atribuição ao vendedor de maior valor, ou descartar esses pedidos.

**Escolhi nota espelhada.** Defini o critério antes de ver o dado: até 15% de multi-vendedor, regra simples. Deu 1,30% — um décimo do limite. Atribuição ponderada seria complexidade desproporcional.

### D-05 — Grão da tabela fato

**Uma linha por combinação de vendedor e pedido.** Permite contar volume, somar faturamento e cruzar com nota sem duplicação. O grão de item seria fino demais e inflaria o fato sem ganho analítico.

### D-06 — Produtos sem categoria

610 produtos (1,9%) com categoria nula.

**Substituí por "Não Informado".** Descartar perderia parte do catálogo na análise de especialização. Isolar mantém o volume e sinaliza a lacuna.

### D-07 — Piso mínimo de pedidos

Um vendedor com dois pedidos e duas notas cinco tem média 5,0, e isso não é performance.

**Piso de 5 pedidos**, definido pela distribuição observada: 42% da base tem menos que isso. Remove 1.176 vendedores da matriz e preserva 1.794 com histórico suficiente.

A análise agregada — Pareto, geografia — continua usando a base completa. Só a matriz de segmentação aplica o piso.

### D-08 — Pedidos sem itens

775 pedidos existem em `orders` mas não têm nenhum registro em `order_items`.

**Descartei via junção interna.** Um pedido sem item não é pedido para efeito de análise de vendedor.

### D-09 — Região como coluna, não dimensão

**Coluna em `dVendedor`.** Cinco regiões numa hierarquia rasa não justificam uma tabela separada. Star schema com dimensão de cinco linhas é complexidade sem retorno.

### D-10 — Vendedores sem pedido entregue

125 vendedores ficaram sem data de primeiro pedido depois dos merges — existem no cadastro mas nenhum pedido deles foi entregue.

Considerei manter com flag de inativo ou remover.

**Removi.** A base caiu de 3.095 para 2.970. Coerente com D-01: vendedor sem transação completa não tem performance para medir. Análise de ativação está fora do escopo.

### D-11 — Volume como eixo da classificação, não faturamento

A primeira versão da coluna `Quadrante` usava faturamento no eixo horizontal.

**Troquei para volume de pedidos.** A tese do projeto diz que a régua correta não é faturamento absoluto — usar faturamento na classificação seria contradizer a tese dentro do código.

Faturamento virou o tamanho da bolha no gráfico de dispersão, que é onde ele agrega: mostra quanto dinheiro está em jogo em cada quadrante.

### D-12 — Mediana como corte, não média

**Mediana.** Volume e faturamento têm distribuição muito assimétrica — o maior vendedor tem 1.854 pedidos, a mediana é 17. A média seria puxada pelos gigantes e classificaria quase todo mundo como baixo volume.

### D-13 — Quadrante como coluna, não medida

**Coluna calculada.** Medida não pode ser usada em legenda de gráfico, e o quadrante precisa colorir o gráfico de dispersão.

A contrapartida é que a classificação não recalcula quando se filtra por ano. Aceitável conceitualmente — quadrante é diagnóstico estrutural do vendedor, não recorte do mês — mas produz o comportamento estranho de a contagem por quadrante ficar fixa enquanto os percentuais de faturamento variam com o filtro.

---

## Os erros

### O bug de escala de 100×

O mais grave, e o que mais me ensinou.

Cheguei na etapa de DAX com faturamento total de R$ 1,54 bilhão e ticket médio de R$ 15.980. Números plausíveis se o contexto fosse outro — mas a Olist é marketplace popular, vendendo produtos de cama e mesa, perfumaria e brinquedos. Ticket de R$ 16 mil é impossível.

O que aconteceu: em algum momento consolidei os CSVs num Excel intermediário e passei a importar dali. O Excel, com locale português, interpretou `58.90` como `5890` — tratou o ponto decimal como separador de milhar e o removeu.

E o Power Query aceitou sem reclamar porque eu tinha declarado as colunas de preço e frete como `Int64.Type`:

```m
{"price", Int64.Type}, {"freight_value", Int64.Type}
```

Inteiro não tem casa decimal. A conversão descartou a informação silenciosamente. Uma etapa seguinte convertia para `type number`, mas aí já era tarde — `5890` continua `5890`.

A correção foi voltar a importar do CSV original e declarar `type number` desde a primeira conversão de tipo.

**O que aprendi:** dois princípios que virei regra.

Primeiro, sempre importar da fonte original. Arquivo intermediário herda todo bug de quem o gerou, e o bug fica invisível porque a origem parece limpa.

Segundo, verificar plausibilidade de ordem de grandeza antes de construir qualquer coisa em cima. Eu tinha o número errado desde a etapa de ETL, mas só notei duas etapas depois, quando ele apareceu num cartão. Se tivesse feito uma conta de guardanapo — 97 mil pedidos vezes um ticket típico de e-commerce popular — teria pego na hora.

### Contar linhas distintas não é contagem distinta

Ao agrupar pedidos por vendedor, o topo apareceu com 2.033 pedidos. O Excel tinha dado 1.854.

A causa: usei a operação "Contar Linhas Distintas" do diálogo de agrupamento, que conta combinações únicas considerando todas as colunas. Como `Item Sequência` varia dentro do mesmo pedido, quase nada era deduplicado — contei itens, não pedidos.

O que eu queria era contagem distinta de uma coluna específica, opção que não aparece no diálogo em português. Resolvi escrevendo M direto.

**O que aprendi:** o cross-check com o Excel foi o que pegou o erro. Sem ele, 2.033 passaria batido — é um número plausível. Ter calculado a mesma coisa duas vezes, por caminhos diferentes, foi o que permitiu a comparação.

### Duas medianas convivendo no mesmo modelo

Esse foi o mais sutil.

Eu tinha uma medida `Mediana Nota` que retornava 4,22 e usei esse valor na linha de referência do gráfico de dispersão. Mas a coluna `Quadrante` calculava a própria mediana internamente, e ela dava 4,14.

Resultado: vendedores com nota entre 4,14 e 4,22 apareciam do lado errado da linha. Passei um bom tempo achando que a classificação estava quebrada.

A causa é a diferença entre medida e coluna. Medida responde ao contexto de filtro do visual — com 2018 selecionado, ela calculava a mediana daquele ano. Coluna ignora filtro e calcula sobre a base inteira.

Dois números corretos, medindo coisas diferentes, usados como se fossem o mesmo.

**O que aprendi:** quando uma coluna calculada faz um cálculo internamente, esse cálculo precisa estar exposto em algum lugar para ser referenciado. Manter uma medida paralela que "deveria" dar o mesmo número é convite para divergência. Apaguei a medida.

### Medida onde precisava de coluna, duas vezes

Criei `Quadrante do Vendedor` como medida. Funcionou nos cartões. Quando fui usar na legenda do gráfico de dispersão, o Power BI recusou — legenda só aceita coluna.

Reescrevi como coluna. Semanas depois, cometi o mesmo erro com `Perfil Vendedor`, e o Power BI recusou no eixo pelo mesmo motivo.

**O que aprendi:** a pergunta a fazer antes de escrever não é "como calculo isso" e sim "onde isso vai ser usado". Se for eixo, legenda, segmentação ou filtro, tem que ser coluna. Se for valor de visual, medida.

### Nomenclatura decidida tarde demais

Comecei importando as tabelas com nomes em inglês, depois renomeei para português, depois decidi manter nomes de coluna em português mas nomes de tabela conforme a origem. Cada mudança quebrou referências em código M já escrito.

**O que aprendi:** convenção de nomenclatura é decisão de arquitetura e tem que ser tomada antes da primeira linha de ETL. Mudar no meio custa retrabalho em cascata, e o custo cresce a cada etapa.

### Documentação em excesso no começo

Montei uma estrutura de dezenas de arquivos de documentação antes de ter qualquer resultado. Todos ficaram como template vazio, e a maioria foi descartada.

**O que aprendi:** documentação intermediária de projeto individual é burocracia. O que precisa existir durante a execução é o log de decisões técnicas — sem ele, você esquece por que filtrou algo. O resto se escreve no fim, quando o contexto inteiro está na cabeça e você sabe o que realmente importou.

---

## O que eu faria diferente

**Validação sistemática entre etapas.** Os três bugs sérios foram encontrados por acaso, quando um número apareceu estranho num visual. Se eu tivesse uma rotina de conferência ao fim de cada etapa — ordem de grandeza, contagem de linhas, cross-check com o Excel — teria pego todos mais cedo e mais barato.

**Teste de sensibilidade no piso de 5 pedidos.** Escolhi 5 pela distribuição, o que é razoável, mas não medi o que aconteceria com 3 ou 10. Um teste simples daria robustez a um parâmetro que define toda a segmentação.

**A ponte com o projeto operacional.** Era a pergunta 6 e teria sido o diferencial do par de projetos. Exigia preparar uma tabela auxiliar com os resultados do projeto irmão num formato importável — trabalho que não fiz e que fica como continuação.

**O gráfico de dispersão.** Investi bastante tempo nele e o resultado ainda é difícil de ler com 1.793 pontos sobrepostos. Ele mostra uma relação que os cartões não mostram, então tem valor, mas eu deveria ter reconhecido antes o limite do formato em vez de tentar resolver por configuração.
