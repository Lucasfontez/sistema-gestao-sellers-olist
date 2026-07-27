# 05 — Páginas e análise

O dashboard tem quatro páginas, estruturadas como contexto, conflito e resolução. A capa apresenta o modelo, o diagnóstico estabelece o problema, a segmentação resolve, e a análise qualitativa testa as hipóteses secundárias.

Todos os números deste documento são do período completo, sem filtro aplicado.

---

## Página 1 — Capa

Estática, sem nenhum visual do Power BI. Apresenta a tese numa frase e os quatro quadrantes com critério e ação recomendada.

A decisão de fazer uma capa sem indicadores foi deliberada. Ela funciona como página de entrada: quem abre entende o modelo antes de ver qualquer número. Os botões de navegação e os ícones de contato são elementos transparentes sobre o fundo.

O elemento gráfico do lado direito é o emblema dos quatro quadrantes — os mesmos blocos de cor que aparecem em escala reduzida na marca da barra lateral. Não é um gráfico com dado; é o modelo desenhado.

Considerei colocar ali uma miniatura da matriz com pontos representando vendedores, e descartei: seriam dados inventados numa capa de projeto de dados. Num portfólio, isso é risco que não compensa.

---

## Página 2 — Diagnóstico geral

Estabelece que existe um problema de concentração.

**Indicadores:** R$ 15,42 milhões de faturamento, ticket médio de R$ 159,83, 96.478 pedidos entregues e 1.794 vendedores elegíveis.

### Curva de Pareto

O visual mais importante da página, e o que abre o argumento do projeto.

| Recorte | Vendedores | Faturamento acumulado |
|---|---|---|
| D01 (10% maiores) | 297 | 67,2% |
| D01 a D02 (20%) | 594 | 82,5% |
| D01 a D05 (50%) | 1.485 | 96,8% |
| D06 a D10 (metade inferior) | 1.485 | 3,2% |

**E daí?** A regra 80/20 diz que 20% geram 80%. Aqui, 20% geram 82,5% — mas o número que importa é outro: bastam **10% para 67%**. A concentração é mais agressiva que o padrão clássico.

Isso significa que ordenar a base por faturamento não separa nada. Quase todo mundo está na cauda: 1.485 vendedores movimentam R$ 493 mil somados, menos que os três maiores individualmente. Uma régua que coloca 90% da base na mesma categoria não é régua de gestão.

### Ticket médio mensal

Linha temporal do ticket ao longo dos meses.

O ticket fica estável em torno de R$ 155 a R$ 170, sem tendência de alta.

**E daí?** Se o ticket é estável e o faturamento cresce, o crescimento vem de volume. Mais pedidos com o mesmo valor médio significa mais operação — mais embalagem, mais envio, mais chance de erro. Isso é um indício a favor da hipótese de que escala e qualidade brigam, que é o que a página 3 vai mostrar de forma direta.

### Tempo de casa

Distribuição dos vendedores pelo ano do primeiro pedido: 129 em 2016, 1.586 em 2017, 1.255 em 2018.

**E daí?** Esse gráfico existe para desambiguar a Cauda Longa. Um vendedor com pouco volume pode ser novo em rampa ou antigo estagnado — e a ação recomendada é diferente em cada caso.

Como mais da metade da base entrou em 2017, quem está com volume baixo já teve tempo. Não é rampa, é estagnação. Isso sustenta a recomendação de baixa prioridade para esse quadrante.

### Ticket médio por decil

Ticket médio de cada faixa da curva de Pareto.

A queda de D01 para D10 é gradual, sem salto.

**E daí?** Os grandes vendedores não vendem produtos muito mais caros — vendem muito mais. Combinado com o ticket mensal estável, a leitura fica consistente: o que separa um vendedor grande de um pequeno neste marketplace é quantidade de operação, não valor unitário.

Esse card substituiu um Top 10 de vendedores que contava a mesma história do Pareto com menos precisão.

---

## Página 3 — Matriz de segmentação

O coração do projeto.

### Os quatro quadrantes

| Quadrante | Vendedores | % do faturamento | Média por vendedor |
|---|---|---|---|
| Risco Crítico | 496 | 52% | R$ 16,2 mil |
| Estrelas | 422 | 36% | R$ 13,2 mil |
| Cauda Longa | 389 | 5% | R$ 2,0 mil |
| Cavalos de Batalha | 486 | 4% | R$ 1,3 mil |

**E daí?** Este é o achado central, e ele prova a tese com número.

Os vendedores de Risco Crítico faturam mais por cabeça que as Estrelas — R$ 16,2 mil contra R$ 13,2 mil. Se a Olist ordenar a base por faturamento, esses 496 aparecem no topo da lista. São os melhores pelo critério errado.

Cruzando com nota, eles viram o maior passivo do marketplace: mais da metade da receita rodando com avaliação abaixo da mediana. Cada real que entra por eles traz junto uma probabilidade maior de comprador insatisfeito.

Dois recortes complementares. Estrelas e Risco Crítico somam 918 vendedores — 51% da base elegível — e 88% da receita: volume é o que manda em faturamento. E a Cauda Longa, com 389 vendedores e 5%, confirma que não é ali que está o problema. O barulho está em cima.

### Distribuição dos vendedores elegíveis

Gráfico de dispersão com volume no eixo horizontal, nota no vertical, faturamento no tamanho da bolha e quadrante na cor. Linhas tracejadas marcam as medianas.

O eixo horizontal está em escala logarítmica. O volume vai de 5 a 1.854 pedidos; em escala linear, 95% das bolhas se amontoariam à esquerda e o eixo inteiro existiria para acomodar meia dúzia de gigantes.

**E daí?** O gráfico mostra uma diagonal — mais volume tende a mais nota. Mas na ponta direita, onde estão as bolhas maiores, o vermelho domina. Os vendedores de altíssimo volume são majoritariamente Risco Crítico.

Isso é o que os cartões não conseguem mostrar: a relação entre tamanho e problema. E é o que sugere que escala e qualidade estão em tensão.

Sendo honesto sobre esse visual: com 1.793 pontos, a região densa fica difícil de ler mesmo com transparência aplicada. Ele agrega a relação entre as variáveis, mas os números precisos vêm dos cartões acima.

### Vendedores em risco crítico

Tabela com os vendedores do quadrante, ordenada por faturamento, com cidade, UF, região, volume, nota e ticket médio.

**E daí?** É a única parte do dashboard que desce ao vendedor individual. Os cartões dizem "496 vendedores"; a tabela mostra quem são os que mais pesam. Sem ela, o diagnóstico fica abstrato e a recomendação não tem onde aterrissar.

O maior deles: 1.132 pedidos, nota 4,13, R$ 148,9 mil de faturamento. Nota 4,13 não é catastrófica em termos absolutos — só está abaixo da mediana de 4,14 da base elegível. Isso mostra que o corte é apertado e que a classificação é relativa ao conjunto, não a um padrão de qualidade externo.

---

## Página 4 — Análise qualitativa

Testa as duas hipóteses secundárias e mostra a distribuição por categoria.

### Especialistas × generalistas

Vendedores com até duas categorias contra os demais.

Especialistas: 4,15. Generalistas: 4,10.

**E daí?** Hipótese refutada. Eu esperava que quem se concentra numa categoria dominasse melhor o processo — estoque, embalagem, expectativa do comprador — e tivesse nota melhor. A diferença de 0,05 está dentro do ruído.

Especialização não prediz qualidade neste dataset. Isso elimina um caminho de recomendação: não adianta incentivar vendedores a se especializarem esperando ganho de nota.

O eixo do gráfico é truncado. Com escala de zero a cinco, as duas barras ficariam idênticas.

### Categoria × quadrante

Matriz das 15 maiores categorias por faturamento, com a distribuição percentual dos quatro quadrantes em cada linha, colorida como mapa de calor.

| Categoria | Risco Crítico |
|---|---|
| Cama_Mesa_Banho | 47,3% |
| Informatica_Acessorios | 45,8% |
| Moveis_Decoracao | 45,5% |
| Cool_Stuff | 45,6% |
| Relogios_Presentes | 43,8% |
| Bebes | 43,0% |
| Automotivo | 39,7% |
| Esporte_Lazer | 37,9% |
| Beleza_Saude | 30,8% |
| Moveis_Escritorio | 27,8% |
| Perfumaria | 27,9% |

**E daí?** A variação é grande: Cama_Mesa_Banho tem quase o dobro de concentração de Risco Crítico que Móveis_Escritório.

Quando a diferença entre segmentos é dessa ordem, o problema deixa de ser individual e passa a ser estrutural. Um vendedor de cama e mesa opera com produtos volumosos, frete caro e expectativa alta de acabamento — condições que tornam o erro mais provável independentemente de quem ele seja.

Isso muda a ação recomendada. Em vez de intervir vendedor por vendedor nos 496, faz mais sentido investigar o que as categorias críticas têm em comum e atacar a causa.

Uma nota sobre a construção: a primeira versão usava contagem absoluta, e isso escondia o padrão. Categorias grandes ficavam escuras em todas as colunas só por serem grandes. Trocar para percentual da linha foi o que tornou os segmentos comparáveis.

### Quadrante por região

Barras empilhadas em 100% com a composição de quadrantes de cada região.

**E daí?** Sudeste, Sul e Centro-Oeste têm distribuição parecida, com Risco Crítico entre 20% e 32%. A composição da base é homogênea geograficamente.

Combinado com o resultado da nota por região — que variava de 4,20 a 4,00 —, a conclusão é que geografia não explica qualidade. O problema é do modelo de gestão, não da localização.

A região Norte aparece com 50% e 50% em dois quadrantes. São dois vendedores e R$ 630 de faturamento total. Estatisticamente irrelevante, e mantenho no gráfico apenas para não esconder parte da base.

### Ticket médio e faturamento por região

Ticket: Nordeste R$ 240,85, Sul R$ 201,15, Sudeste R$ 151,00, Centro-Oeste R$ 148,45, Norte R$ 126,00.

Faturamento: Sudeste R$ 6,6 milhões, Sul R$ 1,4 milhão, Nordeste R$ 251 mil, Centro-Oeste R$ 112 mil, Norte R$ 630.

**E daí?** O ranking se inverte entre as duas métricas. Nordeste tem o maior ticket e o terceiro menor faturamento; Sudeste tem o terceiro maior ticket e domina em faturamento com folga.

Regiões periféricas concentram operação em produtos de maior valor unitário — provavelmente porque o frete inviabiliza itens baratos. O Sudeste, com a maior densidade logística, sustenta volume alto com ticket baixo.

Isso não é achado sobre vendedores, é sobre estrutura de mercado. Mas contextualiza por que a base elegível é majoritariamente do Sudeste.

---

## Cobertura das perguntas

| Pergunta | Onde responde | Resultado |
|---|---|---|
| 1. Existe Pareto? | Página 2 | Sim, mais agressivo que 80/20 |
| 2. Distribuição na matriz | Página 3 | 422 / 496 / 486 / 389 |
| 3. Quem está em Risco Crítico | Página 3 | Tabela com os maiores |
| 4. Especialização prediz nota? | Página 4 | Não, hipótese refutada |
| 5. Diferença regional de qualidade? | Página 4 | Não, hipótese refutada |
| 6. Cruzamento com o operacional | — | Não realizado |
