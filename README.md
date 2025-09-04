

# My First dbt Project (with Snowflake)

This project is a **step-by-step dbt learning journey**.  
It covers: seeding data, building staging models, dimensions, facts, and adding tests.

---

## 🚀 Setup

### 1. Create Snowflake objects
Run in Snowsight (SQL Worksheet):

```sql
USE ROLE ACCOUNTADMIN;

CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH
  WAREHOUSE_SIZE = XSMALL
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

CREATE DATABASE IF NOT EXISTS DBT_TRAINING;
CREATE SCHEMA IF NOT EXISTS DBT_TRAINING.DBT_PACIFIQUE;

USE WAREHOUSE COMPUTE_WH;
USE DATABASE DBT_TRAINING;
USE SCHEMA DBT_PACIFIQUE;


### 2. Configure dbt profile (`~/.dbt/profiles.yml`)

```yaml
my_first_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: se69678.eu-central-1
      user: PACIFIQUE
      password: <YOUR_PASSWORD>
      role: ACCOUNTADMIN
      database: DBT_TRAINING
      warehouse: COMPUTE_WH
      schema: dbt_pacifique
      threads: 4
      client_session_keep_alive: false
```

Verify:

```bash
dbt debug
```

---

## 📦 Seeds (Raw Data)

CSV files in `seeds/`:

* `customers.csv`
* `orders.csv`
* `payments.csv`

Load them:

```bash
dbt seed
```

---

## 🗂 Sources

`models/sources.yml`

```yaml
version: 2

sources:
  - name: raw
    database: DBT_TRAINING
    schema: dbt_pacifique
    tables:
      - name: customers
      - name: orders
      - name: payments
```

---

## 🏗 Staging Models

Files in `models/staging/`:

* `stg_customers.sql`
* `stg_orders.sql`
* `stg_payments.sql`

Run:

```bash
dbt run --select staging
```

### Tests (`models/staging/schema.yml`)

* `customer_id`, `order_id`, `payment_id` → `unique`, `not_null`
* `email` → `not_null`

Run:

```bash
dbt test --select staging
```

---

## 📊 Dimension

`models/dim_customers.sql`

```sql
{{ config(materialized='table') }}

select
  c.customer_id,
  c.first_name,
  c.last_name,
  c.email,
  c.created_at
from {{ ref('stg_customers') }} as c
```

### Tests (`models/dim_customers.yml`)

* `customer_id` → `unique`, `not_null`
* `email` → `not_null`

Run:

```bash
dbt run --select dim_customers
dbt test --select dim_customers
```

---

## 📈 Fact Table

`models/fct_orders.sql`

```sql
{{ config(materialized='table') }}

with orders as (
  select * from {{ ref('stg_orders') }}
),
payments as (
  select
    order_id,
    sum(amount) as total_paid_amount,
    max(paid_at) as latest_paid_at,
    any_value(payment_method) as any_payment_method
  from {{ ref('stg_payments') }}
  group by 1
)

select
  o.order_id,
  o.customer_id,
  o.order_date,
  o.status,
  o.order_total,
  coalesce(p.total_paid_amount, 0) as total_paid_amount,
  p.latest_paid_at,
  p.any_payment_method
from orders o
left join payments p using (order_id)
```

### Tests (`models/fct_orders.yml`)

* `order_id` → `unique`, `not_null`
* `customer_id` → `not_null`
* `order_total`, `total_paid_amount` → `not_null`

Run:

```bash
dbt run --select fct_orders
dbt test --select fct_orders
```

---

## 🔍 Verification in Snowsight

```sql
USE DBT_TRAINING.DBT_PACIFIQUE;

SELECT * FROM dim_customers;
SELECT * FROM fct_orders;
```

---

## 📚 Docs

Generate interactive lineage and model docs:

```bash
dbt docs generate
dbt docs serve
```

---

## ✅ Summary

* Seeds → staging → dim/fact flow implemented
* Tests for data quality
* Queries verified in Snowflake
* Docs generated for lineage
