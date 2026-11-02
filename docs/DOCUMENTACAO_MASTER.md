# DOCUMENTAÇÃO MASTER — P04 Lakehouse Olist Databricks + AWS

**Atualização:** 01/11/2026  
**Etapa atual:** 3 — Camada Silver em desenvolvimento  
**Status:** Bronze concluída; primeira tabela Silver validada e versionada.

## 1. Objetivo e escopo

Desenvolver um Lakehouse de e-commerce utilizando Databricks, PySpark, Delta Lake e Unity Catalog, seguindo a arquitetura Medallion (Bronze, Silver e Gold).

O projeto tem como finalidade demonstrar competências práticas de Engenharia de Dados, incluindo ingestão, transformação, qualidade, governança, versionamento e automação.

**Estado atual:**
- 9 arquivos CSV originais carregados no Unity Catalog Volume.
- 9 tabelas Bronze implementadas em Delta Lake.
- Primeira tabela Silver (`orders`) criada e validada.
- Código versionado no GitHub utilizando GitHub Flow.

A implementação utiliza Databricks Free Edition com compute Serverless. A integração operacional com serviços AWS permanece planejada.

## 2. Arquitetura implementada

```text
Olist — 9 arquivos CSV
         |
         v
Upload manual
         |
         v
Unity Catalog Volume
workspace.default.olist_raw
         |
         v
Databricks / PySpark
         |
         v
BRONZE — Delta Lake
workspace.bronze
         |
         +-- orders
         +-- customers
         +-- geolocation
         +-- order_items
         +-- order_payments
         +-- order_reviews
         +-- products
         +-- sellers
         +-- product_category_name_translation
         |
         v
SILVER — Delta Lake
workspace.silver
         |
         +-- orders (implementada)
         |
         v
GOLD — Pendente
```

Os arquivos Raw são mantidos em um Volume gerenciado pelo Unity Catalog.

As tabelas Bronze e Silver são gerenciadas pelo Databricks e armazenadas no formato Delta Lake.

Ainda não existe ingestão automática a partir do Amazon S3.

## 3. Dados de origem

**Dataset:** Olist Brazilian E-Commerce.

**Volume:** `workspace.default.olist_raw`

**Caminho:** `/Volumes/workspace/default/olist_raw/`

Foram carregados nove arquivos CSV originais, preservados para rastreabilidade e eventual reprocessamento.

## 4. Notebooks e processamento

**Repositório:** `lakehouse-olist-databricks-aws`

| Notebook | Finalidade |
|---|---|
| `01_Bronze_Orders_Pipeline.ipynb` | Ingestão inicial de pedidos |
| `02_bronze_demais_tabelas.ipynb` | Ingestão parametrizada das demais tabelas Bronze |
| `03_silver-orders.ipynb` | Profiling, tipagem, validação e persistência de pedidos na Silver |

### 4.1. Processamento Bronze

- Leitura dos arquivos CSV com PySpark.
- Preservação inicial dos tipos como `string`.
- Persistência em tabelas Delta Lake.
- Verificação de existência antes da criação.
- Validação de contagens e formato Delta.

Essa estratégia protege a carga inicial contra sobrescritas, mas ainda não implementa ingestão incremental ou sincronização automática com a fonte.

### 4.2. Processamento Silver — Orders

A Silver utiliza como fonte a tabela `workspace.bronze.orders`.

O notebook realiza:

1. Análise de duplicidades em `order_id`.
2. Identificação de valores ausentes.
3. Conversão das cinco colunas temporais para `timestamp`.
4. Verificação de falhas de conversão.
5. Gravação da tabela `workspace.silver.orders` em Delta Lake.
6. Validação da quantidade de registros e do formato.

A gravação verifica previamente se a tabela existe para evitar substituição acidental.

## 5. Resultados validados — Bronze

| Tabela | Registros | Formato |
|---|---:|---|
| customers | 99.441 | Delta |
| geolocation | 1.000.163 | Delta |
| order_items | 112.650 | Delta |
| order_payments | 103.886 | Delta |
| order_reviews | 99.224 | Delta |
| orders | 99.441 | Delta |
| product_category_name_translation | 71 | Delta |
| products | 32.951 | Delta |
| sellers | 3.095 | Delta |

**Resultado:** nove tabelas criadas e validadas quanto à existência, formato e quantidade de registros.

## 6. Incidente técnico — CSV order_reviews

Durante a ingestão Bronze, foi encontrada uma divergência na leitura do arquivo `olist_order_reviews_dataset.csv`.

| Configuração | Registros |
|---|---:|
| Leitura inicial | 104.162 |
| `multiLine=true` | 99.249 |
| `multiLine=true`, `quote` e `escape` | 99.224 |

**Diagnóstico:** campos contendo aspas e quebras de linha afetavam a interpretação dos registros pelo parser CSV.

A configuração de leitura foi ajustada para:

```python
.option("header", "true")
.option("multiLine", "true")
.option("quote", '"')
.option("escape", '"')
.option("inferSchema", "false")
```

Após validar o DataFrame corrigido, a tabela Bronze `order_reviews` foi substituída de maneira controlada.

**Resultado:** 99.224 registros, sete colunas e formato Delta.

## 7. Resultados validados — Silver

### 7.1. Tabela `workspace.silver.orders`

| Validação | Resultado |
|---|---|
| Registros | 99.441 |
| IDs únicos | 99.441 |
| IDs duplicados | 0 |
| Colunas temporais convertidas | 5 |
| Conversões inválidas | 0 |
| Formato | Delta Lake |

**Colunas convertidas para `timestamp`:**

- `order_purchase_timestamp`
- `order_approved_at`
- `order_delivered_carrier_date`
- `order_delivered_customer_date`
- `order_estimated_delivery_date`

As colunas `order_id`, `customer_id` e `order_status` permanecem como `string`.

### 7.2. Tratamento de valores nulos

O profiling identificou campos temporais com valores ausentes, incluindo:

- `order_approved_at`: 160 registros.
- `order_delivered_carrier_date`: 1.783 registros.

Valores nulos originalmente presentes foram preservados, sem imputação artificial.

Nenhuma das cinco colunas apresentou falhas de conversão de valores não vazios para `timestamp`.

O processo ainda não contempla todas as regras de qualidade de negócio necessárias para produção.

## 8. Versionamento — GitHub Flow

### Entrega 1 — Estrutura inicial e Bronze Orders

- Branch: `feature/estrutura-inicial`.
- PR #1 integrado à `main`.
- Commit de merge: `6410a5d`.
- Branch excluída após a integração.

**Status:** concluída.

### Entrega 2 — Bronze completa

- Branch: `feature/bronze-demais-tabelas`.
- Nove tabelas Bronze validadas.
- Notebook parametrizado desenvolvido.
- Commit principal: `24a5afd`.
- PR #2 integrado à `main`.
- Branch excluída após a integração.

**Status:** concluída.

### Entrega 3 — Silver Orders

- Branch: `feature/silver-tratamento-dados`.
- Notebook `03_silver-orders.ipynb` desenvolvido e validado.
- Commit principal: `391132e`.
- Commit adicional de metadados informado como realizado.
- Pull Request #3 ainda pendente.

**Status:** implementação inicial concluída; documentação e integração à `main` pendentes.

O projeto segue GitHub Flow, com alterações realizadas em branches de desenvolvimento e integradas à `main` por Pull Requests.

## 9. Custos e segurança

Ambiente utilizado: Databricks Free Edition, sujeito às cotas e limitações da modalidade.

Nenhuma infraestrutura AWS paga foi criada especificamente para o P04 até esta etapa.

Diretrizes adotadas:

- Não armazenar arquivos CSV ou tabelas Delta no GitHub.
- Versionar notebooks e documentação técnica.
- Manter arquivos Raw para eventual reprocessamento.
- Não publicar credenciais, tokens ou segredos.
- Evitar recursos computacionais desnecessários.
- Não sobrescrever tabelas existentes sem validação e autorização.

**Pendência de segurança:** confirmar a revogação do token GitHub anteriormente exposto.

## 10. Limitações e pendências

- Ingestão inicial executada manualmente.
- Silver implementada apenas para `orders`.
- Ausência de testes automatizados.
- Ausência de ingestão incremental.
- Regras de qualidade ainda limitadas.
- Ausência de orquestração com Lakeflow Jobs.
- Camada Gold não implementada.
- Integração automática com Amazon S3 não implementada.
- CI/CD e monitoramento operacional pendentes.

## 11. Próximas entregas

1. Finalizar a documentação e o PR #3 da Silver Orders.
2. Expandir a Silver para as demais entidades Olist.
3. Implementar validações de qualidade e integridade referencial.
4. Construir tabelas Gold com modelagem analítica.
5. Automatizar os pipelines utilizando Lakeflow Jobs.
6. Implementar testes, monitoramento e CI/CD.
7. Avaliar integração com AWS S3 mediante análise prévia de custos.

## 12. Conclusão técnica

O projeto possui a camada Bronze completa e a primeira tabela Silver implementada.

A Bronze preserva os dados de origem em Delta Lake, enquanto a Silver inicia a aplicação de tipagem e verificações de qualidade.

O pipeline ainda está em desenvolvimento e não deve ser apresentado como uma solução totalmente automatizada ou pronta para produção.