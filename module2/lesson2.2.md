# CTEs: The Analytics Engineer's Best Friend

## What CTEs Are and Why They Exist

"At the end of Lesson 2.1, you wrote — or saw — a query with a nested subquery. Let's look at it again."

```sql
-- The problem query from Lesson 2.1 — nested subquery
select
    store_id, s.name as store_name,
    store_daily_rev.avg_daily_rev as store_avg_daily_rev,
    avg_daily_rev.avg_daily_rev as overall_avg_daily_rev,
    (avg_daily_rev.avg_daily_rev - store_daily_rev.avg_daily_rev)/(avg_daily_rev.avg_daily_rev)*100 as daily_rev_percent
from (
select store_id, avg(revenue) as avg_daily_rev
from (
    select store_id, strftime(cast(ordered_at as date), '%Y-%m-%d') as ordered_date, sum(subtotal) as revenue
    from raw.orders
    group by 1, 2
) as daily_rev_by_store
group by 1
) as store_daily_rev
join raw.stores as s
on store_daily_rev.store_id = s.id, (
    select avg(revenue) as avg_daily_rev
    from (
        select strftime(cast(ordered_at as date), '%Y-%m-%d') as ordered_date, sum(subtotal) as revenue
        from raw.orders
        group by 1,
    ) as daily_rev_by_store
) avg_daily_rev
```

"What are the problems above?"
...

"CTEs — Common Table Expressions — solve all of these. A CTE is a named, temporary result set that you define BEFORE your main query and reference by name. It's like giving a meaningful name to each transformation step."

"The syntax:"

```sql
-- CTE syntax structure
WITH stores AS (
  SELECT
    id AS store_id,
    name AS store_name
  FROM raw.stores
),

daily_tot_rev_per_store AS (
  SELECT
    store_id,
    strftime(cast(ordered_at AS date), '%Y-%m-%d') AS ordered_date,
    sum(subtotal) AS revenue
  FROM raw.orders
  GROUP BY 1, 2
),

avg_daily_rev_by_store AS (
  SELECT
    store_id,
    avg(revenue) AS avg_daily_rev
  FROM daily_tot_rev_per_store
  GROUP BY 1
),

daily_total_rev AS (

  SELECT
    strftime(cast(ordered_at AS date), '%Y-%m-%d') AS ordered_date,
    sum(subtotal) AS daily_revenue
  FROM raw.orders
  GROUP BY 1
),

overall_avg_rev AS (

  SELECT avg(revenue) AS avg_daily_rev
  FROM daily_total_rev
),

final AS (
  SELECT
    s.store_name,
    rs.avg_daily_rev AS store_avg_daily_rev,
    ar.avg_daily_rev AS overall_avg_daily_rev,
    (ar.avg_daily_rev - rs.avg_daily_rev) / (ar.avg_daily_rev) * 100 AS daily_rev_percent
  FROM avg_daily_rev_by_store AS rs
  INNER JOIN stores AS s
    ON rs.store_id = s.store_id
  INNER JOIN overall_avg_rev AS ar
    ON 1 = 1
)

SELECT *
FROM final
```

"Each CTE is separated by a comma. The last CTE has no comma after it. Then your final SELECT statement at the bottom — that's what actually returns rows."

"a CTE is a named pipeline step. Its name should tell you exactly what it does."

### The No-Nested-Subqueries Rule

"No nested subqueries in any code you submit. Ever."

"If your logic is complex enough to require nesting — it is complex enough to deserve a named CTE. Full stop."

"The only exception: scalar subqueries in a WHERE clause for a single value lookup. Everything else: CTE."

"This rule exists because:"

Nested subqueries          Named CTEs
───────────────────────    ────────────────────────────
Read inside-out            Read top to bottom
Unnamed steps              Every step has a meaningful name
Hard to debug              Can SELECT from any CTE during debug
Hard to review in a PR     Reviewer can follow each step
No reuse possible          CTEs can be referenced multiple times

"Professional analytics engineers don't write nested subqueries. Code reviewers on data teams will ask you to refactor them. We enforce this standard now so it becomes natural."

### Anti-Pattern Workshop

Example 1: Simple nesting refactor together

```sql
SELECT
    customer_id,
    total_revenue
FROM (
    SELECT customer_id, SUM(subtotal) as total_revenue
    FROM (
        SELECT o.id as order_id, customer as customer_id, subtotal
        FROM raw.orders o
        JOIN raw.stores s
        ON o.store_id = s.id
        WHERE s.tax_rate >= 0.05 and YEAR(ordered_at) >= 2020
    ) as orders_in_2020_rate_over_5p
    GROUP BY customer_id
) as tot_revenue_per_cust
```

```sql
-- Refactoring above to CTE
WITH
    orders_from_2020_less_than_5p AS (
        SELECT
            o.id as order_id,
            customer as customer_id,
            subtotal
        FROM raw.orders o
        JOIN raw.stores s
        ON o.store_id = s.id
        WHERE s.tax_rate >= 0.05 and YEAR(ordered_at) >= 2020
    ),
    customer_rev AS (
        SELECT
            customer_id,
            SUM(subtotal) as total_revenue
        FROM orders_from_2020_less_than_5p
        GROUP BY 1
    ),
    final AS (
        SELECT
            customer_id,
            total_revenue
        FROM customer_rev
    )
    SELECT *
    FROM final
```

Example 2: Pair refactor

```sql
SELECT c.customer_id, c.customer_name, customer_metrics.total_ltv, customer_metrics.orders_count
FROM (
    SELECT id as customer_id, name as customer_name
    FROM raw.customers
    WHERE id IN (
        SELECT customer
        FROM raw.orders
        GROUP BY customer
        HAVING MIN(ordered_at) >= '2018-01-01' AND MIN(ordered_at) <= '2018-12-31'
    )
) c
JOIN (
    SELECT o.customer as customer_id, SUM(subtotal) as total_ltv, COUNT(id) as orders_count
    FROM raw.orders as o
    GROUP by customer
) as customer_metrics
ON c.customer_id = customer_metrics.customer_id
WHERE customer_metrics.total_ltv > (
    SELECT AVG(all_customers.total_spent)
    FROM(
        SELECT o2.customer, SUM(subtotal) as total_spent
        FROM raw.orders as o2
        GROUP BY o2.customer
    ) as all_customers
)
ORDER BY customer_metrics.total_ltv DESC;
```

### Guided Build: The Full Pipeline as CTEs

"Now let's revisit the question from Lesson 2.1 — the one that ended in a messy nested subquery. Let's build it properly as a CTE chain."

"I'll build this live. Talk through the steps as I go."

```sql
-- ──────────────────────────────────────────────────────────────────────────────────
-- FULL CTE CHAIN: Revenue by product
-- Question: revenue generated by each product sold more than once on weekends in 2020
-- ──────────────────────────────────────────────────────────────────────────────────
--1. Filter out orders sold in 2020, and on weekends (enrich to get out a weekday)
--2. Aggregated revenue by product, filtering aggregates using order count > 1
WITH orders_on_2020_weekends AS (
    SELECT
        id as order_id,
        DAYOFWEEK(ordered_at) AS day_of_week
    FROM raw.orders
    WHERE YEAR(ordered_at) >= 2020 AND DAYOFWEEK(ordered_at) IN (0, 6)
)
```

### CTE Naming Conventions

"Names matter. Here are the conventions we use in this program — these come directly from the dbt style guide."

CTE Name           When to use it
──────────────     ──────────────────────────────────────────────────────
source             Always the first CTE — pulls from the raw table.
                   In dbt: SELECT * FROM {{ source(...) }}

renamed            Renames columns to clear snake_case names.
                   Cast to correct types. No logic.

filtered           WHERE clauses to remove invalid or out-of-scope rows.
                   Never in staging models — only in intermediate/mart.

enriched           Adds derived columns. No aggregation, no filtering.
                   One step: transform values, add metrics.

joined             JOINs to another model or table.
                   No aggregation until this is done.

aggregated         GROUP BY. Rolls rows up to a coarser grain.

final              The last CTE before SELECT *FROM final.
                   Use this pattern: the SELECT at the bottom is just:
                   SELECT* FROM final;

```sql
-- ✅ The canonical dbt CTE pattern
WITH

source AS (
    SELECT * FROM raw_table       -- or {{ source(...) }} in dbt
),

renamed AS (
    SELECT
        col_a  AS clean_name_a,
        col_b  AS clean_name_b
    FROM source
),

filtered AS (
    SELECT *
    FROM renamed
    WHERE condition = true
),

final AS (
    SELECT
        clean_name_a,
        clean_name_b
    FROM filtered
)

SELECT * FROM final;
-- Notice: the last line is always SELECT * FROM final
-- This makes it easy to add a new step — just insert before final
-- and update final to reference the new CTE
```

### Exercise Brief

"For homework before Lesson 2.3 — or start now if you have time:"

Using the taxi dataset, write a CTE chain with at least 4 named CTEs that answers:

"Which 10 customers has the highest revenue for: (a) customers that shop at least two stores in a month, (b) orders in 2020 (c) customers that shop at least once in at least 6 months out of 12?"

Submission: Push to your GitHub repo in a file called exercises/lesson_2.2_ctes.sql. Post the link in #standups.
