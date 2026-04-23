# Desafio Engenharia de Dados - Rethink
Projeto de processamento de dados Olist usando arquitetura Medallion.


## 🛠️ Decisões de Arquitetura e Processamento

### 1. Camada Bronze (Ingestão)
- **Origem:** Arquivos CSV brutos carregados na pasta `/raw`.
- **Destino:** Tabelas no formato **Delta** em `/delta/bronze`.
- **Auditoria:** Adicionada a coluna `ingestion_timestamp` em todas as tabelas para garantir a rastreabilidade de quando o dado entrou no Lake.
- **Saneamento de Dados:** - Os arquivos de `geolocation` e `product_category_name_translation` foram processados inicialmente, mas posteriormente **removidos das camadas Raw e Bronze**.
  - **Motivo:** Após análise exploratória, identificou-se que esses dados não impactariam os KPIs solicitados (Faturamento e Volumetria), otimizando assim o armazenamento e a performance do pipeline (FinOps).

### 2. Estrutura de Pastas
Para manter a organização, os nomes das tabelas foram padronizados removendo prefixos (`olist_`) e sufixos (`_dataset`), resultando em nomes simplificados como `orders`, `customers` e `payments`.


### 🧹 Padronização e Saneamento (Camada Bronze)

Para garantir a integridade e a organização do Data Lakehouse, foram aplicadas as seguintes ações de limpeza:

* **Refatoração de Nomenclatura:** Os diretórios da Camada Bronze foram renomeados para remover prefixos (`olist_`) e sufixos (`_dataset`), estabelecendo nomes amigáveis e padronizados para as tabelas (ex: `orders`, `customers`, `payments`).
* **Remoção de Redundâncias:** Arquivos temporários e pastas com nomenclaturas inconsistentes geradas durante as iterações de desenvolvimento foram removidos via script utilizando `dbutils.fs.rm`.
* **Otimização de Escopo:** Os dados de `geolocation` e `product_category_name_translation` foram excluídos tanto da Camada Raw quanto da Bronze. Esta decisão foi tomada para otimizar o custo de armazenamento (FinOps), visto que estas entidades não são necessárias para o cálculo dos KPIs de faturamento e volumetria solicitados no desafio.


## 🥈 Camada Silver (Transformação e Consolidação)

Nesta etapa, os dados provenientes da Camada Bronze foram refinados, padronizados e consolidados em tabelas prontas para análise, aplicando regras de negócio críticas e garantindo a qualidade da informação.

### 🎯 Regras de Negócio e Filtros
Conforme os requisitos do desafio, foi aplicada uma filtragem estratégica no status dos pedidos:

* **Status Preservados:** Apenas pedidos com status `delivered` (entregue) ou `shipped` (enviado).
* **Justificativa:** Registos com status como "canceled", "unavailable" ou "processing" foram descartados nesta fase, pois não representam faturamento real concluído. Esta ação evita a distorção de KPIs de venda e garante que as análises futuras reflitam apenas transações comerciais efetivadas.

### 🛠️ Transformações e Qualidade de Dados
Para elevar a maturidade dos dados, foram executadas as seguintes operações:

* **Tratamento de Nulos:** Filtragem de registos onde campos obrigatórios (como `order_id` e `customer_id`) estavam ausentes.
* **Deduplicação:** Aplicação de `dropDuplicates` para garantir a unicidade de cada pedido e evitar inflação de valores.
* **Conversão de Tipos:** Transformação de colunas de data (originalmente Strings no CSV) para o tipo `TimestampType`, permitindo cálculos temporais precisos.

### 🔗 Linhagem e Consolidação (Joins)
Foi criada a tabela mestra **`orders_consolidated`**, que unifica as informações de:
- Pedidos (`orders`)
- Itens de Pedido (`order_items`)
- Clientes (`customers`)
- Produtos (`products`)
- Vendedores (`sellers`)

**Nota Técnica sobre Auditoria:** Para evitar conflitos de nomes (`DELTA_DUPLICATE_COLUMNS_FOUND`) ao unir as tabelas, as colunas de auditoria foram preservadas através de renomeação dinâmica (ex: `ingestion_timestamp_orders`, `ingestion_timestamp_items`). Isso garante que a linhagem de ingestão de cada entidade seja mantida sem comprometer o esquema da tabela Delta.

### 📊 Tabelas Auxiliares
- **`payments_summary`:** Criada para consolidar os dados financeiros por pedido, agregando o valor total pago, o número máximo de parcelas e a lista de métodos de pagamento utilizados (`collect_set`).


### 📊 Tabela Auxiliar de Pagamentos (`payments_summary`)

Além da consolidação principal, foi desenvolvida uma tabela auxiliar focada na dimensão financeira dos pedidos, atendendo aos requisitos de agregação de dados:

* **Objetivo:** Consolidar todas as transações financeiras de um único pedido em uma única linha de resumo.
* **Agregações Realizadas:**
    - `total_pago`: Soma de todos os pagamentos realizados para o pedido (`sum`).
    - `quantidade_parcelas`: Identificação do número máximo de parcelas escolhido pelo cliente (`max`).
    - `formas_pagamento_utilizadas`: Geração de uma lista única com os métodos de pagamento aplicados (ex: [credit_card, voucher]), utilizando a função `collect_set`.
* **Benefício Analítico:** Esta estrutura simplifica o cruzamento de dados na Camada Gold, permitindo análises de faturamento por método de pagamento e comportamento de parcelamento sem a necessidade de joins complexos em tempo de execução.

# Conforme solicitado, segue abaixo a Justificatia solicitada no teste
* **Justificativa:** Registos com status como "canceled", "unavailable" ou "processing" foram descartados nesta fase, pois não representam faturamento real concluído. Esta ação evita a distorção de KPIs de venda e garante que as análises futuras reflitam apenas transações comerciais efetivadas.
* ** A Camada Bronze nao pode sofrer alterações nos seus dados, essa camada é responsável por armazenar as informações com a mesma estrutura de onde se originaram os dados. A camada Silver tem a finalidade de fazer as transformações necessárias.



## 🥇 Camada Gold (Business & Analytics)

A Camada Gold é o destino final do pipeline, onde os dados são transformados em **conhecimento estratégico**. Aqui, aplicamos agregações complexas para gerar tabelas prontas para consumo por ferramentas de BI (como Power BI ou Tableau) e analistas de negócio.

### 📋 Tabelas de Performance Geradas

Nesta etapa, consolidamos três visões principais para monitorar o ecossistema da Olist:

#### 1. 👥 `customer_summary` (Visão do Cliente)
Consolida o comportamento de compra e satisfação de cada cliente.
* **KPIs:** - `total_orders`: Volume de pedidos por cliente.
    - `total_revenue`: Faturamento total (Preço + Frete).
    - `avg_order_value`: Ticket médio por pedido.
    - `avg_review_score`: NPS médio (satisfação) do cliente.

#### 2. 📦 `product_summary` (Visão de Produto)
Analisa a performance de vendas por categoria de produto.
* **KPIs:**
    - `total_units_sold`: Volume total de itens vendidos.
    - `total_revenue`: Receita bruta acumulada por categoria.
    - `total_orders`: Quantidade de pedidos distintos que contêm a categoria.
    - `avg_freight_value`: Custo médio de frete (útil para estratégia logística).

#### 3. 🏪 `seller_summary` (Visão do Vendedor)
Mapeia a eficiência e a localização dos parceiros de venda.
* **KPIs:**
    - `total_orders`: Volume de pedidos processados pelo vendedor.
    - `total_revenue`: Faturamento total gerado pelo seller.
    - `avg_review_score`: Nota média de avaliação do vendedor (Qualidade do serviço).

---

### 🛠️ Resiliência e Qualidade de Dados (Tratamento de Erros)

Durante o desenvolvimento da Camada Gold, implementamos técnicas avançadas de Engenharia de Dados para garantir a estabilidade do pipeline:

* **Tratamento de Dados Malformados:** Utilizamos a função `try_cast` nas colunas de preço e avaliação. Isso evita que falhas na origem (como datas inseridas em campos numéricos) interrompam a execução, convertendo valores inválidos em `NULL` e preservando a integridade dos cálculos.
* **Deduplicação de Joins:** As avaliações (*reviews*) foram pré-agrupadas por pedido antes do join final, garantindo que a volumetria de vendas e faturamento não fosse inflada artificialmente.
* **Evolução de Esquema:** Foi aplicada a opção `overwriteSchema: true` nas gravações Delta, permitindo a atualização ágil dos KPIs sem conflitos de metadados.