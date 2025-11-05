# 🌐 Projeto CKAN API – Continuidade e Compensação

## ⚙️ Contexto

A **ANEEL (Agência Nacional de Energia Elétrica)** disponibiliza mensalmente, em seu portal de dados abertos ([dados.aneel.gov.br](https://dados.aneel.gov.br/)), informações sobre a **qualidade do fornecimento de energia elétrica**, enviadas por todas as **distribuidoras do país**.

Os principais conjuntos de dados tratados neste projeto são:

| Indicador | Nome | Descrição | Unidade |
|------------|------|------------|----------|
| **DEC** | Duração Equivalente de Interrupção por Unidade Consumidora | Mede o tempo médio (em horas) que os consumidores ficaram sem energia em determinado período. | horas |
| **FEC** | Frequência Equivalente de Interrupção por Unidade Consumidora | Mede o número médio de interrupções no fornecimento de energia por unidade consumidora. | vezes |
| **Compensação** | Compensação Financeira Automática | Representa os valores (em R$) creditados aos consumidores quando os limites de continuidade (DEC/FEC) são ultrapassados. | reais |

Os indicadores **DEC** e **FEC** compõem o conjunto de **indicadores de continuidade do fornecimento**, enquanto o dado de **compensação** reflete o **impacto financeiro regulatório** dessas violações, conforme definido nos **Procedimentos de Distribuição (PRODIST) – Módulo 8** da ANEEL.

---

## ⚙️ Arquitetura e Tecnologias

A pipeline segue o modelo de arquitetura **Medallion (Bronze → Silver → Gold)** dentro do Databricks, com **Jobs** controlando o fluxo de execução.

| Camada | Descrição | Tecnologias |
|---------|------------|-------------|
| **Bronze** | Ingestão bruta dos dados extraídos da API CKAN. | `Python`, `Requests`, `Databricks Jobs` |
| **Silver** | Padronização, limpeza, enriquecimento e reconciliação de inconsistências. | `PySpark`, `Delta Lake` |
| **Gold** | Modelagem analítica final (tabelas fato e dimensão, métricas de continuidade e compensação). | `SQL`, `Power BI`, `Unity Catalog` |

---

## 🧠 Orquestração no Databricks Workflows

A execução do pipeline é feita via **Databricks Workflows (Jobs)** — uma ferramenta nativa de orquestração, agendamento e monitoramento.

### 🔁 Estrutura do Job

**Job: `ckan_continuidade_compensacao`**

| Task | Descrição | Tipo | Dependência |
|------|------------|------|--------------|
| **1. Extração CKAN** | Conecta à API CKAN, baixa os datasets e salva na camada Bronze. | Notebook Python | — |
| **2. Transformação / Compensação** | Aplica regras de continuidade e compensação (PySpark). | Notebook PySpark | Task 1 |
| **3. Publicação Final** | Atualiza tabelas Gold e expõe métricas analíticas. | Notebook SQL | Task 2 |

Cada task roda em **clusters otimizados**, com controle de versionamento e alertas configurados para falhas ou execuções parciais.

### 📅 Agendamentos e Alertas

- **Agendamento**: diário às 02h00 (ajustável conforme atualização da API CKAN)  
- **Retries automáticos** em caso de erro de rede na ingestão  
- **Notificação via e-mail ou webhook** quando o job falhar ou concluir com warnings  

---

```text
[runjobs.py] ──chama──>  [Wrappers]
                             │
                             ├── load_continuidades() ──► baixar_e_carregar(READ_CONT, "stg_continuidades_2020_2025", filtros)
                             ├── load_compensacoes() ───► baixar_e_carregar(READ_COMP, "stg_compensacoes_2020_2025", filtros)
                             └── load_limites() ────────► baixar_e_carregar(READ_LIMIT, "stg_limites")

                                   │
                                   ▼
                           [Função CORE]
                        baixar_e_carregar(...)
      ┌────────────────────────────────────────────────────────────────┐
      │  1) Monta request CKAN (resource_id, limit, offset, filters)  │
      │  2) Faz paginação (while offset += batch)                     │
      │  3) Converte p/ DataFrame + limpeza básica (trim, tipos)      │
      │  4) Grava em Postgres (to_sql append, chunks)                 │
      └────────────────────────────────────────────────────────────────┘
                                   │
           ┌───────────────────────┴────────────────────────┐
           ▼                                                ▼
    [API CKAN / dados abertos]                       [PostgreSQL / Staging]
   (datastore_search / _sql)                        stg_continuidades_2020_2025
                                                    stg_compensacoes_2020_2025
                                                          stg_limites
```

## Passos
1. **Banco**: criar DB `case_equatorial` e schemas `raw`, `stg`, `core`.
2. **Ingestão**: rodar scripts em `/src/ingestion/` (CKAN → `stg_*`).
3. **Transform**: `dbt init`, configurar profile Postgres, `dbt deps`, `dbt run`, `dbt test`.
4. **Observabilidade**: `edr report` (Elementary) para gerar relatório HTML de saúde.
5. **(Opcional)**: Painel Streamlit para métricas de qualidade (freshness, volumes, falhas).

## Qualidade & Observabilidade (o que é checado)
- **Conformidade**: tipos/valores válidos (`indicador ∈ {DEC,FEC}`, `mes ∈ 1..12`, `ano ∈ 2020..2025`)
- **Completude**: % nulos em campos críticos; meses faltantes por distribuidora
- **Consistência**: chaves únicas `(ide_conjunto, ano, mes, indicador)`; FK para `dim_conjunto`
- **Acurácia (pragmática)**: faixas plausíveis (FEC ≤ 50; DEC ≥ 0)
- **Pontualidade (Freshness)**: `MAX(dat_geracao)` dentro do SLA mensal
- **Volume**: linhas por mês comparado ao histórico

## Comandos úteis
```bash
# instalar pacotes
pip install -U pandas requests sqlalchemy psycopg2-binary python-dotenv dbt-postgres elementary-data

# rodar dbt
dbt deps
dbt run
dbt test

# relatório elementary
edr report

```

## Estrutura do Repositório

```text
/docs/            # visão, diagramas, decisões de arquitetura
/src/
  ingestion/      # scripts de ingestão (CKAN -> staging no Postgres)
  quality/        # validações de data quality (ex: Pandera / Great Expectations)
  transforms/     # SQL: dimensões, fatos, views (camada core)
  analytics/      # notebooks e análises exploratórias
/app/             # app (ex: Streamlit) e guias de visualização (Power BI)
/infra/           # infraestrutura (docker-compose, configs, .env.example)
README.md         # visão geral do projeto
LICENSE           # licença do repositório

