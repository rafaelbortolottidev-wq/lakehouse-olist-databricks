# Lakehouse Olist — Databricks

## 1. Objetivo

Construir um Lakehouse utilizando dados públicos do e-commerce brasileiro Olist, aplicando arquitetura Medallion, processamento com PySpark, armazenamento em Delta Lake, orquestração e CI/CD.

**Ambiente utilizado:** Databricks Free Edition, com compute Serverless e Unity Catalog.

A integração direta com serviços AWS ainda não foi implementada.

## 2. Tecnologias

- Databricks
- Apache Spark e PySpark
- Delta Lake
- Unity Catalog
- Lakeflow Jobs
- Databricks Declarative Automation Bundles
- Git e GitHub
- GitHub Actions
- OAuth M2M

## 3. Arquitetura

O pipeline utiliza três camadas:

- **Bronze:** ingestão dos nove arquivos CSV originais em tabelas Delta.
- **Silver:** conversão de tipos, padronização e validações essenciais.
- **Gold:** modelo dimensional para análises de vendas.

Os arquivos originais foram disponibilizados em um Volume do Unity Catalog.

As tabelas estão organizadas nos schemas:

- `workspace.bronze`
- `workspace.silver`
- `workspace.gold`

## 4. Camada Gold

Modelo dimensional implementado:

| Tabela | Finalidade |
|---|---|
| `fato_vendas` | Itens dos pedidos, valores e informações de vendas |
| `dim_cliente` | Identificação e localização dos clientes |
| `dim_produto` | Produtos, categorias e características |
| `dim_vendedor` | Identificação e localização dos vendedores |
| `dim_tempo` | Calendário analítico |

A `fato_vendas` utiliza granularidade de um registro por item do pedido, mantendo valores dos produtos e fretes separados.

**Validação:** 112.650 registros na tabela fato.

## 5. Orquestração — Lakeflow Jobs

**Job original:** `p04_lakehouse_olist_pipeline`

| Tarefa | Notebook | Dependência |
|---|---|---|
| `bronze_orders` | `01_Bronze_Orders_Pipeline` | Nenhuma |
| `bronze_demais_tabelas` | `02_bronze_demais_tabelas` | Nenhuma |
| `silver_orders` | `03_silver-orders` | `bronze_orders` |
| `silver_demais_tabelas` | `04_silver` | `bronze_demais_tabelas` |
| `gold_modelagem` | `05_gold_modelagem` | Ambas as tarefas Silver |
| `validar_qualidade` | `06_qualidade_dados` | `gold_modelagem` |

As tarefas Bronze são independentes. A Gold depende das duas tarefas Silver, e a validação de qualidade depende da Gold.

**Execução validada:** manual, utilizando Serverless, com status `Succeeded`.

**Agendamento:** não configurado.

## 6. Testes de qualidade

O notebook `06_qualidade_dados` consulta dados reais de `workspace.gold.fato_vendas`.

**Validações implementadas:**

1. Ausência de nulos em `order_id` e `order_item_id`.
2. Ausência de duplicidades na combinação dessas duas colunas.
3. Integridade referencial entre `fato_vendas.customer_id` e `dim_cliente.customer_id`.

As verificações utilizam PySpark e `assert`. Uma condição inválida provoca falha na tarefa de qualidade.

**Resultado:** testes executados com sucesso no Lakeflow Jobs.

## 7. Integração contínua — GitHub Actions

**Workflow:** `.github/workflows/ci.yml`

A integração contínua executa verificações em Pull Requests direcionados à `main`.

**Verificações:**

- Validade da estrutura JSON dos notebooks.
- Verificação da estrutura das células.
- Análise da sintaxe Python.

**Resultado:** workflow executado com sucesso, incluindo os Pull Requests de configuração do CI/CD.

O CI não executa o processamento Spark e não substitui os testes de qualidade realizados no Databricks.

## 8. Monitoramento — Lakeflow Jobs

O pipeline utiliza os recursos nativos de monitoramento do Databricks.

**Configurações implementadas:**

- Histórico de execuções na aba Runs.
- Acompanhamento do status individual das seis tarefas.
- Notificação por e-mail configurada para falhas do Job original.
- Execução manual, sem agendamento automático.

**Validações realizadas:**

- Execuções completas com status `Succeeded`.
- Tarefa `validar_qualidade` concluída após a Gold.
- Configuração de notificação por falha salva.

**Limitação:** o recebimento de uma notificação real de falha ainda não foi testado.

## 9. Implantação contínua — Databricks Bundle e GitHub Actions

### 9.1 Objetivo

Automatizar a publicação da configuração do Lakeflow Job e dos notebooks no Databricks, utilizando arquivos versionados no GitHub.

### 9.2 Estrutura do projeto

```text
lakehouse-olist-databricks
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── docs/
│   └── DOCUMENTACAO_MASTER.md
├── notebooks/
│   └── 6 notebooks do pipeline
├── resources/
│   └── p04_lakehouse_olist_pipeline.job.yml
├── databricks.yml
└── README.md
```

Os notebooks originais da pasta `notebooks/` são a fonte única do código do pipeline.

### 9.3 Databricks Bundle

O `databricks.yml` define o Bundle e o target de implantação `dev`.

O arquivo `resources/p04_lakehouse_olist_pipeline.job.yml` declara:

- As seis tarefas do Lakeflow Job.
- As dependências entre tarefas.
- Os caminhos dos notebooks.
- As notificações e configurações de execução.

O Bundle permite reproduzir e atualizar esses recursos por meio de implantação declarativa.

### 9.4 Autenticação

Foi criado o Service Principal:

`p04-github-actions-cd`

A autenticação do GitHub Actions utiliza OAuth Machine-to-Machine (M2M).

**Configuração:**

- `DATABRICKS_HOST`: variável do GitHub Actions.
- `DATABRICKS_CLIENT_ID`: variável do GitHub Actions.
- `DATABRICKS_CLIENT_SECRET`: segredo protegido no GitHub Actions.
- `DATABRICKS_AUTH_TYPE`: `oauth-m2m`.

A credencial OAuth foi configurada com validade de 30 dias. Sua renovação é necessária para manter as implantações funcionando após o vencimento.

Nenhum segredo deve ser armazenado no código ou versionado no GitHub.

### 9.5 Workflow de CD

**Arquivo:** `.github/workflows/cd.yml`

O workflow executa:

1. Checkout do repositório.
2. Instalação da Databricks CLI.
3. `databricks bundle validate --target dev`
4. `databricks bundle deploy --target dev`

**Gatilhos:**

- `push` na branch `main`, incluindo alterações integradas por merge.
- `workflow_dispatch`, para execução manual.

O deploy publica ou atualiza os recursos declarados no Bundle. Ele não executa automaticamente o processamento dos notebooks.

### 9.6 Validação do CD

**Resultados comprovados no GitHub Actions:**

- Validação manual do Bundle concluída com sucesso.
- Primeiro deploy manual concluído com sucesso.
- Execução automática após merge do PR #12 concluída com sucesso.

**Resultado no Databricks:**

Foi criado o Job de desenvolvimento gerenciado pelo Bundle, identificado pelo prefixo `[dev p04_github_actions_cd]`.

O Job original `p04_lakehouse_olist_pipeline` foi preservado.

A existência do Job de desenvolvimento foi confirmada na interface do Databricks.

**Limitação:** o Job de desenvolvimento foi implantado, mas ainda não foi executado de ponta a ponta. A execução completa validada corresponde ao Job original.

## 10. Versionamento — GitHub Flow

O desenvolvimento utiliza GitHub Flow:

1. Atualizar a branch `main`.
2. Criar branch específica para cada alteração.
3. Implementar e validar.
4. Executar Commit e Push.
5. Abrir Pull Request com base `main`.
6. Validar o CI e integrar a alteração.
7. Sincronizar a `main`.
8. Encerrar a branch quando apropriado.

O CD automático utiliza a versão integrada à `main`.

## 11. Custos e segurança

O projeto foi desenvolvido no Databricks Free Edition, com utilização de Serverless.

Não foram provisionados recursos AWS específicos para este Lakehouse.

**Cuidados adotados:**

- Uso de Service Principal para automação.
- Credencial OAuth armazenada em GitHub Secrets.
- Ausência de credenciais no repositório.
- Sem agendamento automático de processamento de dados.
- Separação entre Job original e Job de desenvolvimento.

A disponibilidade de recursos e as cotas do Databricks Free Edition devem ser observadas.

## 12. Limitações e próximas melhorias

- Implementar atualização incremental e reprocessamento seguro.
- Ampliar os testes de qualidade quando necessário.
- Validar o recebimento real de alertas de falha.
- Executar e validar o Job implantado pelo Bundle, se necessário.
- Avaliar integração com Amazon S3 e seus custos.
- Evoluir o pipeline para cenários mais próximos da produção.

## 13. Status do projeto

**Versão 1.0 — Implementada e validada no escopo de portfólio.**

**Atualização:** 03/11/2026.

As camadas Bronze, Silver e Gold foram implementadas e validadas.

O Lakeflow Job original possui seis tarefas e foi executado com sucesso, incluindo os testes de qualidade.

O CI e o CD estão configurados no GitHub Actions. O deploy do Databricks Bundle foi validado manualmente e também executado automaticamente após merge na `main`.

A integração AWS, o processamento incremental e a execução de ponta a ponta do Job de desenvolvimento permanecem fora do escopo validado da v1.0.