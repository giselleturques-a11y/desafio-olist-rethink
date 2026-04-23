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


### 🛠️  Refatoração e Ajustes Técnicos (Silver)

Durante o desenvolvimento da Camada Silver, foram aplicadas melhorias estruturais para garantir a resiliência do pipeline de orquestração:

* **Saneamento de Caminhos (Paths):** Após a limpeza da Camada Bronze, os caminhos de leitura foram normalizados para refletir a nova nomenclatura das entidades (ex: de `olist_products_dataset` para `products`). Isso eliminou erros de `ResourceNotFound` durante a execução automatizada.
* **Consistência de Esquema:** Implementação do uso de `overwriteSchema: true` nas gravações Delta da Silver. Essa prática permite que alterações no contrato de dados (como a renomeação de colunas de auditoria para evitar duplicidade em Joins) sejam aplicadas sem conflitos de metadados.
* **Rastreabilidade de Auditoria:** As colunas de timestamp foram renomeadas dinamicamente durante os Joins (ex: `ingestion_timestamp_items`, `ingestion_timestamp_cust`), preservando a linhagem de cada registro sem causar colisões de nomes no DataFrame final.


## ⚙️ Etapa 4: Orquestração e Entrega Final (04_pipeline_runner)

### 🤖 Automação do Pipeline (`pipeline_runner.py`)
Para garantir a integridade e a ordem de execução das camadas (Bronze → Silver → Gold), foi desenvolvido um orquestrador centralizado que simula o comportamento de ferramentas de mercado como o **Azure Data Factory** ou **Airflow**.

* **Resiliência:** O script utiliza blocos de tratamento de erro que capturam falhas em notebooks específicos, registram o log detalhado com *timestamp* e permitem que o pipeline prossiga, garantindo que o sistema não fique travado por erros isolados.
* **Monitoramento:** Logs de console indicam o tempo de início e término de cada etapa, facilitando a auditoria de performance.

### 📤 Simulação de Delta Sharing (`04_share_simulation.py`)
Como etapa final de entrega de valor, foi implementada uma rotina que prepara os dados para o consumo por sistemas externos e ferramentas de BI:

* **Consumo Interoperável:** As tabelas da Camada Gold são lidas em formato Delta e convertidas para **CSV**.
* **Otimização de Ficheiros:** Utilização da função `coalesce(1)` para garantir que cada relatório (Clientes, Produtos e Vendedores) seja entregue como um ficheiro único, facilitando a importação direta em dashboards.
* **Diretório de Output:** Os ficheiros finais são disponibilizados na pasta `/data/delta/output/`, prontos para distribuição.



## 📤 Etapa 5: Simulação de Delta Sharing (`05_Simulação_Delta_Sharing.py`)

Esta etapa final do pipeline simula a distribuição segura de dados para parceiros ou ferramentas de visualização, transformando as tabelas analíticas em formatos de fácil consumo.

### ⚙️ Lógica de Processamento
* **Interoperabilidade:** O script consome as tabelas Delta da Camada Gold e as converte para o formato **CSV**, garantindo que os dados possam ser lidos por qualquer ferramenta (Excel, Power BI, Python, etc.).
* **Consolidação:** Utilização da função `.coalesce(1)` para evitar a fragmentação do arquivo. Isso garante que cada tabela Gold resulte em um único arquivo CSV legível, em vez de múltiplas partições.
* **Automação:** O processo percorre automaticamente a lista de tabelas geradas na Gold, aplicando o cabeçalho (`header=true`) e sobrescrevendo versões antigas no diretório de saída.

### 📂 Estrutura de Saída (Output)
Os arquivos são disponibilizados no caminho `dbfs:/Volumes/workspace/default/data/delta/output/`:
- `gold_customer_summary_export.csv`
- `gold_product_summary_export.csv`
- `gold_seller_summary_export.csv`



### 📊 Resumo Executivo (Insights de Negócio)
Ao final da execução, o script de simulação gera um relatório executivo diretamente no console, fornecendo métricas críticas para a tomada de decisão:
- **Abrangência de Mercado:** Total de clientes únicos processados.
- **Saúde Financeira:** Receita total consolidada do ecossistema.
- **Mix de Produtos:** Ranking das 3 categorias mais rentáveis.
- **Geolocalização:** Identificação do estado líder em volumetria de pedidos.
- **Qualidade de Serviço:** Destaque para o vendedor com melhor reputação (mínimo de 10 pedidos para relevância estatística).



# 🚀 Olist Data Pipeline: Medallion Architecture

Este repositório contém a implementação de um pipeline de dados escalável utilizando a **Arquitetura Medalhão** (Bronze, Silver e Gold) no Databricks. O projeto transforma dados brutos do e-commerce Olist em indicadores estratégicos de negócio, garantindo governança, qualidade e resiliência.

---

## a) 🏗️ Diagrama da Arquitetura

O fluxo segue o padrão de camadas para garantir a separação de responsabilidades e a linhagem dos dados:

1.  **Bronze (Raw):** Ingestão dos arquivos CSV originais para o formato **Delta**. Mantemos a fidelidade total aos dados de origem, adicionando colunas de auditoria (`ingestion_timestamp` e `source_file`).
2.  **Silver (Trusted):** Limpeza, tipagem rigorosa e filtragem de status de pedidos (apenas `delivered` e `shipped`). Consolidação das entidades (Pedidos, Itens, Clientes, Produtos e Vendedores) via Joins.
3.  **Gold (Analytics):** Criação de tabelas agregadas (KPIs) modeladas por entidade para consumo de BI.
4.  **Output (Delivery):** Simulação de consumo via **Delta Sharing**, exportando os resultados finais em CSV para interoperabilidade.

---

## b) 🧠 Decisões de Design

Durante o desenvolvimento, tomei decisões críticas para garantir a integridade dos KPIs:

* **Filtro de Status de Pedidos:** Decidimos manter apenas pedidos com status `delivered` ou `shipped`. Pedidos cancelados ou pendentes foram descartados para evitar inflar artificialmente as métricas de faturamento e volume.
* **Tratamento de Dados Malformados:** Implementamos a função `try_cast` no processamento da camada Gold. Isso permite que o pipeline lide com falhas na origem (como datas inseridas em campos de preço) sem interromper o fluxo, gerando valores `NULL` para auditoria posterior.
* **Gestão de Metadados Delta:** Utilizamos a opção `overwriteSchema: true` para permitir a evolução ágil das tabelas agregadas, garantindo que mudanças nos KPIs fossem aplicadas sem conflitos de metadados.

---

## c) 🛠️ Como rodar o projeto

1.  **Ambiente:** Importe os notebooks para o seu Workspace Databricks.
2.  **Dados:** Certifique-se de que os arquivos CSV da Olist estão carregados no diretório configurado no notebook Bronze.
3.  **Execução:** Execute o notebook `pipeline_runner.py`.
    * Este orquestrador gerencia a execução sequencial: `01_bronze` ➡️ `02_silver` ➡️ `03_gold` ➡️ `05_Simulação_Delta_Sharing`.
4.  **Monitoramento:** Acompanhe os logs no console para verificar o status de cada etapa e visualizar o **Resumo Executivo** final.

---

## d) 🏭 O que mudaria em produção?

Em um ambiente Enterprise real, as seguintes melhorias seriam fundamentais:

* **Orquestração via ADF/Airflow:** Substituiria o script manual por ferramentas como **Azure Data Factory** para gerenciar retentativas (retries) e dependências complexas.
* **Unity Catalog:** Implementação de governança centralizada para controle de acesso a nível de linha e coluna e linhagem de dados automática.
* **Data Quality Framework:** Integração com bibliotecas como *Great Expectations* para bloquear o pipeline caso a qualidade dos dados (ex: receita negativa) seja comprometida.

---

## e) ⚠️ Limitações e Melhorias Futuras

Este projeto foi desenvolvido como um desafio técnico estruturado e possui as seguintes limitações que seriam abordadas em um cenário de longo prazo:

1.  **Processamento Full Load:** Atualmente, o pipeline reprocessa todos os dados em cada execução. O ideal seria implementar uma **Carga Incremental** para reduzir custos computacionais e tempo de processamento.
2.  **Ausência de Testes Unitários:** Não foram implementados testes automatizados para validar as funções de transformação individuais antes do deploy.
3.  **Sistema de Alerta:** O orquestrador captura erros, mas não envia notificações. Em produção, seria necessária a integração com **Slack** ou **PagerDuty** para alertar a equipe de plantão sobre falhas.
4.  **Hardcoded Paths:** Alguns caminhos de diretórios estão fixos no código. O uso de **Parâmetros de Notebook** ou **Key Vaults** seria a prática recomendada para maior segurança e flexibilidade.

---
**Desenvolvido por:** Giselle da Silva Turques
**Ferramentas Utilizadas:** PySpark, Delta Lake, Databricks.