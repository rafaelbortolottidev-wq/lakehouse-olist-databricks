![Arquitetura Lakehouse Olist — Databricks e AWS](./Infográfico%20Lakehouse%20Olist%20no%20Databricks%20e%20AWS_1793639726589.png)

# Lakehouse Olist — Databricks + AWS

# Lakehouse Olist — Databricks + AWS

## 1. Objetivo

Construir um Lakehouse utilizando dados públicos do e-commerce brasileiro Olist, aplicando arquitetura Medallion, processamento com PySpark e armazenamento em Delta Lake.

**Plataforma utilizada:** Databricks Free Edition. A integração direta com serviços AWS ainda não foi implementada.

## 2. Tecnologias

- Databricks
- Apache Spark e PySpark
- Delta Lake
- Unity Catalog
- Lakeflow Jobs
- Git e GitHub

## 3. Arquitetura

O pipeline está organizado em três camadas:

- **Bronze:** ingestão dos nove arquivos CSV originais em tabelas Delta.
- **Silver:** conversão de tipos, padronização e validações essenciais.
- **Gold:** modelo dimensional para análises de vendas.

As tabelas estão organizadas nos schemas `workspace.bronze`, `workspace.silver` e `workspace.gold`.

## 4. Camada Gold

Modelo dimensional inicial composto por:

| Tabela | Finalidade |
|---|---|
| `fato_vendas` | Itens dos pedidos, valores e informações de vendas |
| `dim_cliente` | Identificação e localização dos clientes |
| `dim_produto` | Produtos, categorias e características |
| `dim_vendedor` | Identificação e localização dos vendedores |
| `dim_tempo` | Calendário analítico |

A `fato_vendas` utiliza granularidade de um registro por item do pedido. Os valores dos produtos e fretes permanecem separados.

## 5. Orquestração — Lakeflow Jobs

**Job:** `p04_lakehouse_olist_pipeline`

A execução foi configurada com cinco tarefas:

| Tarefa | Notebook | Dependência |
|---|---|---|
| `bronze_orders` | `01_Bronze_Orders_Pipeline` | Nenhuma |
| `bronze_demais_tabelas` | `02_bronze_demais_tabelas` | Nenhuma |
| `silver_orders` | `03_silver-orders` | `bronze_orders` |
| `silver_demais_tabelas` | `04_silver` | `bronze_demais_tabelas` |
| `gold_modelagem` | `05_gold_modelagem` | Ambas as tarefas Silver |

As tarefas Bronze podem executar independentemente. A Gold depende da conclusão bem-sucedida das duas tarefas Silver.

### Resultado da execução

- **Data:** 02/11/2026
- **Modo:** execução manual
- **Compute:** Serverless
- **Status final do Job:** `Succeeded`
- **Agendamento automático:** não configurado

### Limitações atuais

Os notebooks verificam se as tabelas já existem antes da gravação. Essa abordagem evita sobrescritas acidentais, mas não implementa atualização incremental.

Consequentemente, novas informações nas fontes não são incorporadas automaticamente às tabelas existentes.

A configuração do Lakeflow Job foi realizada pela interface do Databricks e ainda não está declarada como código versionado no repositório.

## 6. Versionamento

O desenvolvimento segue GitHub Flow:

1. Branch de desenvolvimento criada a partir da `main`.
2. Implementação e validação.
3. Commit e Push.
4. Pull Request e Merge.
5. Sincronização da `main`.

## 7. Próximas melhorias

- Atualização incremental e reprocessamento seguro.
- Testes automatizados e regras adicionais de qualidade.
- Monitoramento e tratamento de falhas.
- CI/CD e versionamento da configuração dos Jobs.
- Avaliação da integração com Amazon S3.

## 8. Status

**Em desenvolvimento.**

Camadas Bronze, Silver e Gold implementadas. Primeira execução completa do Lakeflow Job concluída com sucesso. Melhorias de produção e automação ainda pendentes.
