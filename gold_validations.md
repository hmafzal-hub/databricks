# Gold Validations

## Q: What are the total number of records in Gold tables?
Ans: Shows row count for every Gold dimension and fact table.

Query:
```sql
SELECT 'gold.dim_brand' AS table_name, COUNT(*) AS cnt FROM retail.gold.dim_brand
UNION ALL
SELECT 'gold.dim_customer', COUNT(*) FROM retail.gold.dim_customer
UNION ALL
SELECT 'gold.dim_item', COUNT(*) FROM retail.gold.dim_item
UNION ALL
SELECT 'gold.dim_date', COUNT(*) FROM retail.gold.dim_date
UNION ALL
SELECT 'gold.fact_sales', COUNT(*) FROM retail.gold.fact_sales;
```

## Q: Are there multiple current records for the same brand?
Ans: Should return 0 rows. SCD2 requires only one current row per business key.

Query:
```sql
SELECT brand_internal_id, COUNT(*) AS current_count
FROM retail.gold.dim_brand
WHERE is_current = true
GROUP BY brand_internal_id
HAVING COUNT(*) > 1;
```

## Q: Are there multiple current records for the same customer?
Ans: Should return 0 rows.

Query:
```sql
SELECT customer_internal_id, COUNT(*) AS current_count
FROM retail.gold.dim_customer
WHERE is_current = true
GROUP BY customer_internal_id
HAVING COUNT(*) > 1;
```

## Q: Are there multiple current records for the same item?
Ans: Should return 0 rows.

Query:
```sql
SELECT item_internal_id, COUNT(*) AS current_count
FROM retail.gold.dim_item
WHERE is_current = true
GROUP BY item_internal_id
HAVING COUNT(*) > 1;
```

## Q: Do current SCD2 records have end dates?
Ans: Should return 0 rows. Current records must have effective_end_date as NULL.

Query:
```sql
SELECT *
FROM retail.gold.dim_customer
WHERE is_current = true
  AND effective_end_date IS NOT NULL;
```

## Q: Do expired SCD2 records have missing end dates?
Ans: Should return 0 rows. Expired records must have effective_end_date populated.

Query:
```sql
SELECT *
FROM retail.gold.dim_customer
WHERE is_current = false
  AND effective_end_date IS NULL;
```

## Q: Are effective start dates missing?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.gold.dim_customer
WHERE effective_start_date IS NULL;
```

## Q: Is any effective_end_date earlier than effective_start_date?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.gold.dim_customer
WHERE effective_end_date IS NOT NULL
  AND effective_end_date < effective_start_date;
```

## Q: Are there SCD2 versions for changed customers?
Ans: Returns customers with history. Expected > 0 only if changes were processed.

Query:
```sql
SELECT customer_internal_id, COUNT(*) AS version_count
FROM retail.gold.dim_customer
GROUP BY customer_internal_id
HAVING COUNT(*) > 1
ORDER BY version_count DESC;
```

## Q: Are there SCD2 versions for changed brands?
Ans: Returns brands with history. Expected > 0 only if changes were processed.

Query:
```sql
SELECT brand_internal_id, COUNT(*) AS version_count
FROM retail.gold.dim_brand
GROUP BY brand_internal_id
HAVING COUNT(*) > 1
ORDER BY version_count DESC;
```

## Q: Are there SCD2 versions for changed items?
Ans: Returns items with history. Expected > 0 only if changes were processed.

Query:
```sql
SELECT item_internal_id, COUNT(*) AS version_count
FROM retail.gold.dim_item
GROUP BY item_internal_id
HAVING COUNT(*) > 1
ORDER BY version_count DESC;
```

## Q: Do facts join to expired customer dimension rows?
Ans: Should return 0 rows. Fact incremental should join only current SCD2 dimensions.

Query:
```sql
SELECT COUNT(*) AS fact_rows_joining_expired_customer
FROM retail.gold.fact_sales f
JOIN retail.gold.dim_customer d
  ON f.customer_key = d.customer_key
WHERE d.is_current = false;
```

## Q: Do facts join to expired item dimension rows?
Ans: Should return 0 rows.

Query:
```sql
SELECT COUNT(*) AS fact_rows_joining_expired_item
FROM retail.gold.fact_sales f
JOIN retail.gold.dim_item d
  ON f.item_key = d.item_key
WHERE d.is_current = false;
```

## Q: Are fact rows missing customer dimension keys?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.gold.fact_sales
WHERE customer_key IS NULL;
```

## Q: Are fact rows missing item dimension keys?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.gold.fact_sales
WHERE item_key IS NULL;
```

## Q: Are fact rows missing order date keys?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.gold.fact_sales
WHERE order_date_key IS NULL;
```

## Q: Are fact rows failing customer dimension lookup?
Ans: Should return 0 rows.

Query:
```sql
SELECT f.sales_order_line_internal_id, f.customer_key
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_customer d
  ON f.customer_key = d.customer_key
WHERE d.customer_key IS NULL;
```

## Q: Are fact rows failing item dimension lookup?
Ans: Should return 0 rows.

Query:
```sql
SELECT f.sales_order_line_internal_id, f.item_key
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_item d
  ON f.item_key = d.item_key
WHERE d.item_key IS NULL;
```

## Q: Are fact rows failing date dimension lookup?
Ans: Should return 0 rows.

Query:
```sql
SELECT f.sales_order_line_internal_id, f.order_date_key
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_date d
  ON f.order_date_key = d.date_key
WHERE d.date_key IS NULL;
```

## Q: Are there duplicate fact business keys?
Ans: Should return 0 rows.

Query:
```sql
SELECT sales_order_line_internal_id, COUNT(*) AS cnt
FROM retail.gold.fact_sales
GROUP BY sales_order_line_internal_id
HAVING COUNT(*) > 1;
```

## Q: Are there duplicate dimension surrogate keys?
Ans: Should return 0 rows.

Query:
```sql
SELECT customer_key, COUNT(*) AS cnt
FROM retail.gold.dim_customer
GROUP BY customer_key
HAVING COUNT(*) > 1;
```

## Q: Are there null surrogate keys in dimensions?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.gold.dim_customer
WHERE customer_key IS NULL;
```

## Q: Is the date dimension continuous?
Ans: Missing_dates should be 0 if dim_date covers a continuous date range.

Query:
```sql
WITH bounds AS (
  SELECT MIN(full_date) AS min_date, MAX(full_date) AS max_date
  FROM retail.gold.dim_date
),
calendar AS (
  SELECT EXPLODE(SEQUENCE(min_date, max_date)) AS full_date
  FROM bounds
)
SELECT COUNT(*) AS missing_dates
FROM calendar c
LEFT JOIN retail.gold.dim_date d
  ON c.full_date = d.full_date
WHERE d.full_date IS NULL;
```

## Q: Are Gold fact financial amounts valid?
Ans: Should return 0 rows unless negative facts are intentionally allowed.

Query:
```sql
SELECT *
FROM retail.gold.fact_sales
WHERE quantity < 0
   OR unit_rate < 0
   OR line_amount < 0
   OR order_total < 0;
```

## Q: Are fact line amounts consistent?
Ans: line_amount should approximately equal quantity * unit_rate.

Query:
```sql
SELECT sales_order_line_internal_id, quantity, unit_rate, line_amount, quantity * unit_rate AS expected_amount
FROM retail.gold.fact_sales
WHERE ABS(line_amount - (quantity * unit_rate)) > 0.05;
```

## Q: Are Gold lookup rejects present?
Ans: Shows fact rows rejected because dimension lookups failed.

Query:
```sql
SELECT source_record_id, reject_reason, reject_column, error_message, rejected_at
FROM retail.audit.etl_reject_log
WHERE table_name = 'fact_sales'
  AND reject_stage = 'GOLD'
ORDER BY rejected_at DESC;
```

## Q: Were Silver runs processed into Gold successfully?
Ans: Shows Silver-to-Gold run tracking.

Query:
```sql
SELECT table_name, source_run_id, target_run_id, status, rows_read, rows_written, rows_rejected, created_at
FROM retail.audit.layer_processed_runs
WHERE source_layer = 'silver'
  AND target_layer = 'gold'
ORDER BY created_at DESC;
```

## Q: Are there unprocessed Silver runs for Gold dimensions?
Ans: Should return 0 rows after Gold dimension incremental finishes.

Query:
```sql
SELECT DISTINCT s.etl_run_id
FROM retail.silver.ns_customers s
LEFT ANTI JOIN retail.audit.layer_processed_runs p
  ON s.etl_run_id = p.source_run_id
 AND p.table_name = 'dim_customer'
 AND p.source_layer = 'silver'
 AND p.target_layer = 'gold'
 AND p.status = 'SUCCESS'
WHERE s.etl_run_id IS NOT NULL;
```

## Q: Are there unprocessed Silver runs for Gold fact?
Ans: Should return 0 rows after Gold fact incremental finishes.

Query:
```sql
SELECT DISTINCT s.etl_run_id
FROM retail.silver.ns_sales_order_lines s
LEFT ANTI JOIN retail.audit.layer_processed_runs p
  ON s.etl_run_id = p.source_run_id
 AND p.table_name = 'fact_sales'
 AND p.source_layer = 'silver'
 AND p.target_layer = 'gold'
 AND p.status = 'SUCCESS'
WHERE s.etl_run_id IS NOT NULL;
```
