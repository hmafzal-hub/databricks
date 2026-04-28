# Gold Reconciliation

## Q: Do Gold dimension current counts reconcile with Silver source counts?
Ans: Current dimension count should match Silver count for each dimension business entity.

Query:
```sql
SELECT 'dim_brand' AS table_name,
       (SELECT COUNT(*) FROM retail.silver.ns_brands) AS silver_count,
       (SELECT COUNT(*) FROM retail.gold.dim_brand WHERE is_current = true) AS gold_current_count
UNION ALL
SELECT 'dim_customer',
       (SELECT COUNT(*) FROM retail.silver.ns_customers),
       (SELECT COUNT(*) FROM retail.gold.dim_customer WHERE is_current = true)
UNION ALL
SELECT 'dim_item',
       (SELECT COUNT(*) FROM retail.silver.ns_items),
       (SELECT COUNT(*) FROM retail.gold.dim_item WHERE is_current = true);
```

## Q: Do Gold dimension run counts reconcile with Silver-to-Gold processed runs?
Ans: Shows processed Silver runs for Gold dimensions.

Query:
```sql
SELECT table_name, source_run_id, target_run_id, rows_read, rows_written, rows_rejected, status
FROM retail.audit.layer_processed_runs
WHERE source_layer = 'silver'
  AND target_layer = 'gold'
  AND table_name IN ('dim_brand','dim_customer','dim_item','dim_date')
ORDER BY created_at DESC;
```

## Q: Are there successful Silver runs not processed by Gold dimensions?
Ans: Should return 0 rows after Gold dimension incremental finishes.

Query:
```sql
SELECT DISTINCT s.etl_run_id
FROM retail.silver.ns_brands s
LEFT ANTI JOIN retail.audit.layer_processed_runs p
  ON s.etl_run_id = p.source_run_id
 AND p.table_name = 'dim_brand'
 AND p.source_layer = 'silver'
 AND p.target_layer = 'gold'
 AND p.status = 'SUCCESS'
WHERE s.etl_run_id IS NOT NULL;
```

## Q: Are there successful Silver fact source runs not processed by Gold fact?
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

## Q: Does Gold fact count reconcile with Silver sales order lines minus Gold rejects?
Ans: Gold fact count should be close to valid Silver order lines after failed dimension lookups are excluded.

Query:
```sql
SELECT
  (SELECT COUNT(*) FROM retail.silver.ns_sales_order_lines) AS silver_order_lines,
  (SELECT COUNT(*) FROM retail.gold.fact_sales) AS gold_fact_rows,
  (SELECT COUNT(*) FROM retail.audit.etl_reject_log WHERE table_name = 'fact_sales' AND reject_stage = 'GOLD') AS gold_reject_rows;
```

## Q: Do Gold fact rows reconcile with current dimension keys?
Ans: Should return 0 rows.

Query:
```sql
SELECT f.sales_order_line_internal_id
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_customer c
  ON f.customer_key = c.customer_key
 AND c.is_current = true
LEFT JOIN retail.gold.dim_item i
  ON f.item_key = i.item_key
 AND i.is_current = true
LEFT JOIN retail.gold.dim_date d
  ON f.order_date_key = d.date_key
WHERE c.customer_key IS NULL
   OR i.item_key IS NULL
   OR d.date_key IS NULL;
```

## Q: Do Gold fact financial totals reconcile with Silver source line amounts?
Ans: Compare total line_amount from Gold fact and Silver valid lines.

Query:
```sql
SELECT
  (SELECT ROUND(SUM(line_amount), 2) FROM retail.silver.ns_sales_order_lines) AS silver_line_amount,
  (SELECT ROUND(SUM(line_amount), 2) FROM retail.gold.fact_sales) AS gold_line_amount;
```

## Q: Do Gold fact quantities reconcile with Silver source quantities?
Ans: Compare total quantity from Gold fact and Silver lines.

Query:
```sql
SELECT
  (SELECT SUM(quantity) FROM retail.silver.ns_sales_order_lines) AS silver_quantity,
  (SELECT SUM(quantity) FROM retail.gold.fact_sales) AS gold_quantity;
```

## Q: Are rejected Gold fact rows explainable?
Ans: Shows lookup-failed records with raw payload for investigation.

Query:
```sql
SELECT source_record_id, reject_column, raw_payload, rejected_at
FROM retail.audit.etl_reject_log
WHERE table_name = 'fact_sales'
  AND reject_stage = 'GOLD'
ORDER BY rejected_at DESC;
```

## Q: Does SCD2 have only one current version per business key?
Ans: Should return 0 rows.

Query:
```sql
SELECT 'customer' AS dim_name, customer_internal_id AS business_key, COUNT(*) AS current_count
FROM retail.gold.dim_customer
WHERE is_current = true
GROUP BY customer_internal_id
HAVING COUNT(*) > 1
UNION ALL
SELECT 'item', item_internal_id, COUNT(*)
FROM retail.gold.dim_item
WHERE is_current = true
GROUP BY item_internal_id
HAVING COUNT(*) > 1
UNION ALL
SELECT 'brand', brand_internal_id, COUNT(*)
FROM retail.gold.dim_brand
WHERE is_current = true
GROUP BY brand_internal_id
HAVING COUNT(*) > 1;
```

## Q: Are SCD2 expired rows properly closed?
Ans: Should return 0 rows.

Query:
```sql
SELECT 'customer' AS dim_name, customer_internal_id AS business_key
FROM retail.gold.dim_customer
WHERE is_current = false
  AND effective_end_date IS NULL
UNION ALL
SELECT 'item', item_internal_id
FROM retail.gold.dim_item
WHERE is_current = false
  AND effective_end_date IS NULL
UNION ALL
SELECT 'brand', brand_internal_id
FROM retail.gold.dim_brand
WHERE is_current = false
  AND effective_end_date IS NULL;
```

## Q: Are current SCD2 rows incorrectly closed?
Ans: Should return 0 rows.

Query:
```sql
SELECT 'customer' AS dim_name, customer_internal_id AS business_key
FROM retail.gold.dim_customer
WHERE is_current = true
  AND effective_end_date IS NOT NULL
UNION ALL
SELECT 'item', item_internal_id
FROM retail.gold.dim_item
WHERE is_current = true
  AND effective_end_date IS NOT NULL
UNION ALL
SELECT 'brand', brand_internal_id
FROM retail.gold.dim_brand
WHERE is_current = true
  AND effective_end_date IS NOT NULL;
```

## Q: Does Gold audit records_after make sense for SCD2?
Ans: For SCD2, records_after may increase when changed records create new versions.

Query:
```sql
SELECT table_name, run_id, records_before, rows_written, records_after
FROM retail.audit.etl_run_log
WHERE layer_name = 'gold'
  AND table_name IN ('dim_brand','dim_customer','dim_item')
ORDER BY start_time DESC;
```

## Q: Are failed Gold runs present?
Ans: Shows failed Gold runs that need investigation.

Query:
```sql
SELECT table_name, run_id, status, error_message, start_time, end_time
FROM retail.audit.etl_run_log
WHERE layer_name = 'gold'
  AND status = 'FAILED'
ORDER BY start_time DESC;
```

## Q: Did Gold process each Silver source_run_id only once?
Ans: Should return 0 rows.

Query:
```sql
SELECT table_name, source_run_id, COUNT(*) AS cnt
FROM retail.audit.layer_processed_runs
WHERE source_layer = 'silver'
  AND target_layer = 'gold'
  AND status = 'SUCCESS'
GROUP BY table_name, source_run_id
HAVING COUNT(*) > 1;
```
