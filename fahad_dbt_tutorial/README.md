# practice-dbt-project

A practice dbt project built on Databricks, following a bronze / silver / gold (medallion) layout. It covers the core dbt building blocks: sources, layered models, seeds, snapshots (SCD Type 2), custom macros, and generic and singular tests.

## Tech stack

- Python 3.13+
- [dbt-core](https://docs.getdbt.com/) >= 1.12.0
- [dbt-databricks](https://docs.getdbt.com/docs/core/connect-data-platform/databricks-setup) >= 1.12.3
- [uv](https://docs.astral.sh/uv/) for dependency management

## Project structure

```
practice-dbt-project/
├── pyproject.toml
├── requirements.txt
├── uv.lock
└── fahad_dbt_tutorial/          # the dbt project
    ├── dbt_project.yml
    ├── models/
    │   ├── bronze/              # raw data pulled from sources
    │   ├── silver/              # joined and aggregated data
    │   └── gold/                # deduplicated, business-ready data
    ├── seeds/
    │   └── lookup.csv           # small static lookup table
    ├── snapshots/
    │   └── gold_items.yml       # SCD Type 2 history of items
    ├── macros/
    │   └── multiply.sql         # reusable multiplication macro
    └── tests/
        ├── generic/
        │   └── generic_non_negative.sql
        └── non_negative_test.sql
```

## Data layers

| Layer | Schema | Materialization | Purpose |
|-------|--------|-----------------|---------|
| Bronze | `bronze` | table (`bronze_sales` is a view) | Thin models selecting from the raw source tables such as `fact_sales` and `dim_product`. Seeds also load into this schema. |
| Silver | `silver` | table | `silver_salesinfo` joins sales with product and customer data and aggregates total sales by product category and customer gender. |
| Gold | `gold` | table | `source_gold_items` deduplicates the `items` source, keeping the latest row per `id` by `updateDate`. |

Layer-wide materializations and schemas are set in `dbt_project.yml`.

## Key features

- **Sources:** raw tables are referenced through `source()` rather than hard-coded table names.
- **Macro:** `multiply(col1, col2)` is a small reusable macro used to calculate the gross amount (`quantity * unit_price`).
- **Deduplication:** the gold model uses `row_number() OVER (PARTITION BY id ORDER BY updateDate DESC)` to keep only the latest version of each item.
- **Snapshot:** `gold_items` tracks changes to items over time with the `timestamp` strategy on `updateDate`, keyed on `id`. Currently valid rows get a `dbt_valid_to` of `9999-12-31`.
- **Tests:**
  - `generic_non_negative` is a reusable generic test that fails if a column has values below zero.
  - `non_negative_test` is a singular test that checks `gross_amount` and `net_amount` in `bronze_sales`.
- **Seed:** `lookup.csv` is a small customer lookup loaded with `dbt seed`.

## Getting started

### 1. Clone and install

```bash
git clone https://github.com/FahadMalik36/practice-dbt-project.git
cd practice-dbt-project
uv sync
```

### 2. Configure your Databricks connection

The dbt profile is named `fahad_dbt_tutorial`. Create a `profiles.yml` in `~/.dbt/` (outside the repo) so credentials are never committed. Read the token from an environment variable:

```yaml
fahad_dbt_tutorial:
  target: dev
  outputs:
    dev:
      type: databricks
      host: <your-workspace>.cloud.databricks.com
      http_path: <your-sql-warehouse-http-path>
      catalog: <your-catalog>
      schema: <your-default-schema>
      token: "{{ env_var('DATABRICKS_TOKEN') }}"
      threads: 4
```

Then export the token in your shell:

```bash
export DATABRICKS_TOKEN=<your-personal-access-token>
```

> **Never commit tokens or a `profiles.yml` containing credentials.** The snapshot also relies on `target.catalog`, so `catalog` must be set in your target.

### 3. Run the project

```bash
cd fahad_dbt_tutorial
dbt debug        # verify the connection
dbt seed         # load lookup.csv
dbt run          # build bronze, silver and gold models
dbt snapshot     # capture SCD Type 2 history for gold_items
dbt test         # run generic and singular tests
```

Or build everything in dependency order with `dbt build`.

## Notes

This is a learning project, not production code. It exists to practice dbt concepts on Databricks.