![Arquitetura Lakehouse Olist — Databricks e AWS](./src/Lakehouse%20Olist_%20Arquitetura%20e%20CI_CD%20Implementados.png)

# Lakehouse Olist — Databricks + AWS

## Objetivo

Construir um Lakehouse com dados públicos do e-commerce brasileiro Olist, utilizando arquitetura Medallion, PySpark, Delta Lake, modelagem dimensional e automação de CI/CD.

**Ambiente:** Databricks Free Edition com Serverless e Unity Catalog.

**Observação:** a integração direta com serviços AWS ainda não foi implementada.

## Tecnologias

- Databricks, Apache Spark e PySpark
- Delta Lake e Unity Catalog
- Lakeflow Jobs
- Databricks Declarative Automation Bundles
- Git, GitHub e GitHub Actions
- OAuth M2M

## Arquitetura

O pipeline utiliza três camadas:

- **Bronze:** ingestão de nove arquivos CSV em tabelas Delta.
- **Silver:** tratamento, padronização e validação dos dados.
- **Gold:** modelagem dimensional para análises de vendas.

As tabelas estão organizadas em `workspace.bronze`, `workspace.silver` e `workspace.gold`.

## Modelo dimensional — Gold

| Tabela | Finalidade |
|---|---|
| `fato_vendas` | Vendas por item do pedido |
| `dim_cliente` | Informações dos clientes |
| `dim_produto` | Produtos e categorias |
| `dim_vendedor` | Informações dos vendedores |
| `dim_tempo` | Calendário analítico |

A tabela `fato_vendas` contém **112.650 registros**.

## Orquestração e qualidade

O Lakeflow Job `p04_lakehouse_olist_pipeline` organiza seis tarefas, incluindo ingestão, transformação, modelagem e validação de qualidade.

Os testes verificam chaves nulas, duplicidades e integridade referencial.

**Resultado:** pipeline original executado com sucesso usando Serverless.

## CI/CD — GitHub Actions

O projeto utiliza dois workflows:

- **CI (`ci.yml`):** valida a estrutura e a sintaxe dos notebooks nos Pull Requests.
- **CD (`cd.yml`):** valida e implanta automaticamente o Databricks Bundle após alterações na `main`, mantendo execução manual opcional.

A implantação usa um Service Principal com autenticação OAuth M2M e credencial protegida no GitHub Secrets.

**Resultados validados:**

- CI aprovado.
- Validação do Bundle aprovada.
- Deploy manual aprovado.
- Deploy automático após merge aprovado.
- Job de desenvolvimento criado no Databricks, preservando o Job original.

O Job implantado pelo Bundle ainda não foi executado de ponta a ponta.

## Versionamento

O desenvolvimento segue GitHub Flow: branch por alteração, testes, Commit, Push, Pull Request, Merge e sincronização da `main`.

## Status

**Versão 1.0 — Concluída no escopo de portfólio.**

As camadas Bronze, Silver e Gold, os testes de qualidade, a orquestração e o CI/CD estão implementados.

**Evoluções futuras:** processamento incremental, testes adicionais, integração com Amazon S3 e validação do Job de desenvolvimento.

A documentação técnica completa está disponível em [DOCUMENTACAO_MASTER.md](docs/DOCUMENTACAO_MASTER.md).