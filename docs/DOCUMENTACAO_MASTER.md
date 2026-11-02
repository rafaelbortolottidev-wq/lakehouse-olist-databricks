# Lakehouse Olist — Databricks + AWS

## 1. Objetivo

Construir um Lakehouse utilizando dados públicos do e-commerce brasileiro Olist, aplicando arquitetura Medallion, processamento com PySpark e armazenamento em Delta Lake.

**Ambiente utilizado:** Databricks Free Edition. A integração direta com serviços AWS ainda não foi implementada.

## 2. Tecnologias

- Databricks
- Apache Spark e PySpark
- Delta Lake
- Unity Catalog
- Lakeflow Jobs
- Git e GitHub
- GitHub Actions

## 3. Arquitetura

O pipeline utiliza três camadas:

- **Bronze:** ingestão dos nove arquivos CSV originais em tabelas Delta.
- **Silver:** conversão de tipos, padronização e validações essenciais.
- **Gold:** modelo dimensional para análises de vendas.

As tabelas estão organizadas nos schemas `workspace.bronze`, `workspace.silver` e `workspace.gold`.

## 4. Camada Gold

Modelo dimensional inicial:

| Tabela | Finalidade |
|---|---|
| `fato_vendas` | Itens dos pedidos, valores e informações de vendas |
| `dim_cliente` | Identificação e localização dos clientes |
| `dim_produto` | Produtos, categorias e características |
| `dim_vendedor` | Identificação e localização dos vendedores |
| `dim_tempo` | Calendário analítico |

A `fato_vendas` utiliza granularidade de um registro por item do pedido, mantendo valores dos produtos e fretes separados.

## 5. Orquestração — Lakeflow Jobs

**Job:** `p04_lakehouse_olist_pipeline`

| Tarefa | Notebook | Dependência |
|---|---|---|
| `bronze_orders` | `01_Bronze_Orders_Pipeline` | Nenhuma |
| `bronze_demais_tabelas` | `02_bronze_demais_tabelas` | Nenhuma |
| `silver_orders` | `03_silver-orders` | `bronze_orders` |
| `silver_demais_tabelas` | `04_silver` | `bronze_demais_tabelas` |
| `gold_modelagem` | `05_gold_modelagem` | Ambas as tarefas Silver |
| `validar_qualidade` | `06_qualidade_dados` | `gold_modelagem` |

As tarefas Bronze são independentes. A tarefa de qualidade é executada após a conclusão bem-sucedida da Gold.

**Execução:** manual, com compute Serverless e sem agendamento automático.

**Resultado:** execuções completas com status `Succeeded`.

## 6. Testes de qualidade

O notebook `06_qualidade_dados` consulta dados reais da tabela `workspace.gold.fato_vendas`.

**Validações implementadas:**

1. Ausência de valores nulos em `order_id` e `order_item_id`.
2. Ausência de duplicidades na combinação dessas duas colunas.
3. Integridade referencial entre `fato_vendas.customer_id` e `dim_cliente.customer_id`.

As verificações utilizam PySpark e `assert`. Uma condição inválida interrompe a execução da tarefa de qualidade.

As regras foram executadas com sucesso no Lakeflow Jobs.

## 7. Integração contínua — GitHub Actions

Workflow: `.github/workflows/ci.yml`

A integração contínua executa verificações em Pull Requests direcionados à `main`.

**Verificações:**
- Leitura da estrutura JSON dos notebooks.
- Existência de células nos notebooks.
- Análise da sintaxe Python.

O workflow foi executado com sucesso no GitHub Actions.

Essas verificações não executam o processamento Spark nem substituem os testes de qualidade realizados no Databricks.

## 8. Monitoramento — Lakeflow Jobs

O pipeline utiliza os recursos nativos de monitoramento do Databricks.

**Configurações implementadas:**
- Histórico de execuções disponível na aba Runs.
- Acompanhamento do status individual das seis tarefas.
- Notificação por e-mail configurada para falhas do Job.
- Execução manual, sem gatilhos ou agendamentos automáticos.

**Validações realizadas:**
- Execuções completas com status `Succeeded`.
- Tarefa `validar_qualidade` executada com sucesso após a Gold.
- Configuração de notificação por falha salva.

**Limitação:** o recebimento de uma notificação real de falha ainda não foi testado.

## 9. Versionamento

O desenvolvimento segue GitHub Flow:

1. Atualização da `main`.
2. Criação de branch específica.
3. Implementação e validação.
4. Commit e Push.
5. Pull Request e Merge.
6. Sincronização da `main` e encerramento da branch.

## 10. Limitações e próximas melhorias

- Implementar atualização incremental e reprocessamento seguro.
- Ampliar as regras de qualidade quando necessário.
- Validar o recebimento dos alertas de falha.
- Implementar CD e versionamento da definição do Lakeflow Job.
- Avaliar integração com Amazon S3 e respectivos custos.

## 11. Status do projeto

**Em desenvolvimento — atualização em 02/11/2026.**

As camadas Bronze, Silver e Gold estão implementadas. O Lakeflow Job possui seis tarefas e foi executado com sucesso, incluindo validações de qualidade dos dados reais.

O GitHub Actions está configurado e validado. O monitoramento possui histórico de execuções e notificação por falha configurada, ainda sem teste de entrega do alerta.