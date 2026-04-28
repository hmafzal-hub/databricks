# Silver Reconciliation

## Q: Do Silver run counts reconcile with Bronze processed run counts?
Ans: For each Bronze-to-Silver run, rows_read should match the Bronze rows for source_run_id.

Query:
```sql
SELECT 
  p.table_name,
  p.source_run_id,
  p.rows_read AS processed_rows_read,
  COUNT(b.etl_run_id) AS bronze_rows
FROM retail.audit.layer_processed_runs p
LEFT JOIN retail.bronze.ns_brands b
  ON p.source_run_id = b.etl_run_id
WHERE p.source_layer = 'bronze'
  AND p.target_layer = 'silver'
  AND p.table_name = 'ns_brands'
GROUP BY p.table_name, p.source_run_id, p.rows_read
HAVING COALESCE(p.rows_read,0) <> COUNT(b.etl_run_id);
```

## Q: Do Silver rows_read, rows_written, and rows_rejected reconcile?
Ans: For each processed run, rows_read should be >= rows_rejected and rows_written should represent valid prepared rows.

Query:
```sql
SELECT table_name, source_run_id, rows_read, rows_written, rows_rejected
FROM retail.audit.layer_processed_runs
WHERE source_layer = 'bronze'
  AND target_layer = 'silver'
  AND status = 'SUCCESS'
  AND (
    rows_read < rows_rejected
    OR rows_written < 0
    OR rows_rejected < 0
  );
```

## Q: Do Silver rejects reconcile with processed run rows_rejected?
Ans: Reject log count should match rows_rejected by target_run_id.

Query:
```sql
SELECT 
  p.table_name,
  p.target_run_id,
  SUM(p.rows_rejected) AS processed_rejected,
  COUNT(r.run_id) AS reject_log_rows
FROM retail.audit.layer_processed_runs p
LEFT JOIN retail.audit.etl_reject_log r
  ON p.target_run_id = r.run_id
 AND r.reject_stage = 'SILVER'
WHERE p.source_layer = 'bronze'
  AND p.target_layer = 'silver'
GROUP BY p.table_name, p.target_run_id
HAVING COALESCE(SUM(p.rows_rejected),0) <> COUNT(r.run_id);
```

## Q: Are there successful Bronze runs not processed by Silver?
Ans: Should return 0 rows after Silver incremental completes.

Query:
```sql
SELECT f.table_name, f.run_id
FROM retail.audit.ingested_files f
LEFT ANTI JOIN retail.audit.layer_processed_runs p
  ON f.run_id = p.source_run_id
 AND f.table_name = p.table_name
 AND p.source_layer = 'bronze'
 AND p.target_layer = 'silver'
 AND p.status = 'SUCCESS'
WHERE f.ingestion_status = 'SUCCESS';
```

## Q: Are there duplicate current business keys in Silver after MERGE?
Ans: Should return 0 rows for each Silver table.

Query:
```sql
SELECT internal_id, COUNT(*) AS cnt
FROM retail.silver.ns_sales_orders
GROUP BY internal_id
HAVING COUNT(*) > 1;
```

## Q: Does Silver count stay less than or equal to Bronze for merge tables?
Ans: Silver should be <= Bronze because Silver deduplicates/merges.

Query:
```sql
SELECT 'ns_brands' AS table_name,
       (SELECT COUNT(*) FROM retail.bronze.ns_brands) AS bronze_count,
       (SELECT COUNT(*) FROM retail.silver.ns_brands) AS silver_count
UNION ALL
SELECT 'ns_customers',
       (SELECT COUNT(*) FROM retail.bronze.ns_customers),
       (SELECT COUNT(*) FROM retail.silver.ns_customers)
UNION ALL
SELECT 'ns_items',
       (SELECT COUNT(*) FROM retail.bronze.ns_items),
       (SELECT COUNT(*) FROM retail.silver.ns_items)
UNION ALL
SELECT 'ns_sales_orders',
       (SELECT COUNT(*) FROM retail.bronze.ns_sales_orders),
       (SELECT COUNT(*) FROM retail.silver.ns_sales_orders)
UNION ALL
SELECT 'ns_sales_order_lines',
       (SELECT COUNT(*) FROM retail.bronze.ns_sales_order_lines),
       (SELECT COUNT(*) FROM retail.silver.ns_sales_order_lines)
UNION ALL
SELECT 'ns_shipments',
       (SELECT COUNT(*) FROM retail.bronze.ns_shipments),
       (SELECT COUNT(*) FROM retail.silver.ns_shipments);
```

## Q: Are Silver records missing corresponding Bronze record_hash?
Ans: Should return 0 rows if every Silver row came from Bronze.

Query:
```sql
SELECT s.internal_id, s.record_hash
FROM retail.silver.ns_brands s
LEFT JOIN retail.bronze.ns_brands b
  ON s.record_hash = b.record_hash
WHERE b.record_hash IS NULL;
```

## Q: Do Silver audit records have valid duration?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.audit.etl_run_log
WHERE layer_name = 'silver'
  AND status = 'SUCCESS'
  AND (duration_seconds IS NULL OR duration_seconds < 0);
```

## Q: Are failed Silver runs present?
Ans: Shows failed Silver runs that need investigation.

Query:
```sql
SELECT table_name, run_id, status, error_message, start_time, end_time
FROM retail.audit.etl_run_log
WHERE layer_name = 'silver'
  AND status = 'FAILED'
ORDER BY start_time DESC;
```

## Q: Did Silver process each Bronze source_run_id only once?
Ans: Should return 0 rows.

Query:
```sql
SELECT table_name, source_run_id, COUNT(*) AS cnt
FROM retail.audit.layer_processed_runs
WHERE source_layer = 'bronze'
  AND target_layer = 'silver'
  AND status = 'SUCCESS'
GROUP BY table_name, source_run_id
HAVING COUNT(*) > 1;
```
