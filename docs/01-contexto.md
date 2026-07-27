# 01 — Contexto

## O problema

Um marketplace intermedia a relação entre vendedores e compradores. A receita dele vem de comissão sobre o que os vendedores vendem, mas a reputação dele depende do que os compradores experimentam. Esses dois interesses nem sempre apontam para o mesmo lado.

A forma mais natural de olhar para uma base de vendedores é ordenar por faturamento. Quem vende mais está no topo, quem vende menos está embaixo, e a gestão prioriza de cima para baixo.

O problema desse critério é que ele é cego para qualidade. Um vendedor pode faturar alto e entregar mal — e continua aparecendo no topo da lista. A plataforma acaba protegendo e expandindo justamente quem está corroendo a experiência do comprador.

## A tese

A régua correta não é faturamento absoluto. É volume cruzado com qualidade.

Volume porque é o que gera receita e escala. Qualidade porque é o que sustenta a base de compradores no longo prazo. Cruzando os dois, uma base que parecia uma lista ordenada vira quatro grupos com naturezas distintas:

**Estrelas** — alto volume e nota alta. São o ativo da plataforma. A ação é proteger e expandir.

**Risco Crítico** — alto volume e nota baixa. Geram receita hoje e passivo amanhã. A ação é intervenção urgente.

**Cavalos de Batalha** — baixo volume e nota alta. Entregam bem mas vendem pouco. A ação é investigar o gargalo e investir.

**Cauda Longa** — baixo volume e nota baixa. Pouco impacto em qualquer direção. A ação é baixa prioridade.

O ponto não é que os quatro grupos existem — isso é trivial. O ponto é medir quanto cada um pesa, e descobrir se a régua tradicional está enganando.

## O dataset

Brazilian E-Commerce Public Dataset by Olist, publicado no Kaggle. São aproximadamente 100 mil pedidos reais feitos entre setembro de 2016 e outubro de 2018, com dados anonimizados de vendedores, produtos, itens de pedido, avaliações e geolocalização.

Usei seis das nove tabelas. As três que ficaram de fora — clientes, geolocalização e pagamentos — não têm relação com o ângulo B2B.

O dataset tem uma característica que define o escopo: os vendedores são identificados por hash, sem nome, categoria de contrato ou histórico de relacionamento. Isso significa que toda a análise é sobre comportamento observado, não sobre perfil cadastral.

## Escopo

**Dentro:** segmentação por volume e nota, concentração de faturamento, distribuição por categoria e região, tempo de casa dos vendedores.

**Fora:** análise preditiva, modelagem de churn, análise de custo e margem (não há dados), processamento de texto das avaliações, e vendedores que nunca tiveram pedido entregue.

A restrição de stack foi deliberada: Excel e Power BI, sem SQL, Python ou nuvem. O objetivo era demonstrar profundidade nessas duas ferramentas — Power Query para ETL, DAX para lógica de negócio, modelagem dimensional para estrutura.

## O par com o projeto operacional

Este projeto é metade de um par sobre o mesmo dataset.

O outro é [analise-diagnostico-operacional-olist](https://github.com/Lucasfontez/analise-diagnostico-operacional-olist), construído com PostgreSQL e SQL, que olha para atraso de entrega, satisfação do comprador e peso do frete por região.

A divisão é intencional. O operacional pergunta "o que está dando errado na entrega". O B2B pergunta "quem são os vendedores por trás disso". Mesma base, duas lentes.

A conexão explícita entre os dois — cruzar os vendedores de Risco Crítico com as rotas de maior atraso — estava prevista e não foi executada. Está registrada como limitação e como próximo passo.
