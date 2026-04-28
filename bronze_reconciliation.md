# Bronze Reconciliation

## Q: Do ingested file audit counts reconcile with Bronze rows by run_id?
Ans: For each file/run, rows_written should match Bronze table rows for that etl_run_id.

Query:
```sql
SELECT 
  f.table_name,
  f.source_file_name,
  f.run_id,
  f.rows_written AS audit_rows_written,
  COUNT(b.etl_run_id) AS bronze_rows
FROM retail.audit.ingested_files f
LEFT JOIN retail.bronze.ns_brands b
  ON f.run_id = b.etl_run_id
WHERE f.table_name = 'ns_brands'
GROUP BY f.table_name, f.source_file_name, f.run_id, f.rows_written
ORDER BY f.source_file_name;
```

## Q: Do rows_read, rows_written, and rows_rejected reconcile in Bronze audit?
Ans: rows_read should equal rows_written + rows_rejected.

Query:
```sql
SELECT table_name, run_id, rows_read, rows_written, rows_rejected
FROM retail.audit.etl_run_log
WHERE layer_name = 'bronze'
  AND status = 'SUCCESS'
  AND COALESCE(rows_read,0) <> COALESCE(rows_written,0) + COALESCE(rows_rejected,0);
```

## Q: Does audit records_after equal records_before + rows_written for Bronze append?
Ans: Should return 0 rows for append-only successful Bronze runs.

Query:
```sql
SELECT table_name, run_id, records_before, rows_written, records_after
FROM retail.audit.etl_run_log
WHERE layer_name = 'bronze'
  AND status = 'SUCCESS'
  AND COALESCE(records_after,0) <> COALESCE(records_before,0) + COALESCE(rows_written,0);
```

## Q: Are all successful ingested files linked to successful Bronze run logs?
Ans: Should return 0 rows.

Query:
```sql
SELECT f.*
FROM retail.audit.ingested_files f
LEFT JOIN retail.audit.etl_run_log l
  ON f.run_id = l.run_id
WHERE f.ingestion_status = 'SUCCESS'
  AND (l.run_id IS NULL OR l.status <> 'SUCCESS' OR l.layer_name <> 'bronze');
```

## Q: Are there Bronze run logs without ingested file records?
Ans: Should return 0 rows except no-new-file runs with rows_read = 0.

Query:
```sql
SELECT l.*
FROM retail.audit.etl_run_log l
LEFT JOIN retail.audit.ingested_files f
  ON l.run_id = f.run_id
WHERE l.layer_name = 'bronze'
  AND l.status = 'SUCCESS'
  AND COALESCE(l.rows_read,0) > 0
  AND f.run_id IS NULL;
```

## Q: Do Bronze rejects reconcile with audit rows_rejected?
Ans: Reject log count should match audit rows_rejected.

Query:
```sql
SELECT 
  l.table_name,
  l.run_id,
  l.rows_rejected AS audit_rejected,
  COUNT(r.run_id) AS reject_log_rows
FROM retail.audit.etl_run_log l
LEFT JOIN retail.audit.etl_reject_log r
  ON l.run_id = r.run_id
WHERE l.layer_name = 'bronze'
GROUP BY l.table_name, l.run_id, l.rows_rejected
HAVING COALESCE(l.rows_rejected,0) <> COUNT(r.run_id);
```

## Q: Reconcile Bronze table counts against ingested_files totals.
Ans: Bronze count should equal sum rows_written from ingested_files if table was built only through this pipeline.

Query:
```sql
SELECT 
  'ns_brands' AS table_name,
  (SELECT COUNT(*) FROM retail.bronze.ns_brands) AS bronze_count,
  (SELECT SUM(rows_written) FROM retail.audit.ingested_files WHERE table_name = 'ns_brands' AND ingestion_status = 'SUCCESS') AS audit_written;
```

## Q: Are failed Bronze runs present?
Ans: Shows failed Bronze runs that need investigation.

Query:
```sql
SELECT table_name, run_id, status, error_message, start_time, end_time
FROM retail.audit.etl_run_log
WHERE layer_name = 'bronze'
  AND status = 'FAILED'
ORDER BY start_time DESC;
```

## Q: Are there partial or inconsistent Bronze runs?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.audit.etl_run_log
WHERE layer_name = 'bronze'
  AND status = 'SUCCESS'
  AND (
    rows_read IS NULL
    OR rows_written IS NULL
    OR rows_rejected IS NULL
    OR records_before IS NULL
    OR records_after IS NULL
    OR end_time IS NULL
  );
```

## Q: Are source_file_name and source_file_path consistent?
Ans: source_file_name should be the last part of source_file_path.

Query:
```sql
SELECT *
FROM retail.audit.ingested_files
WHERE source_file_name <> element_at(split(source_file_path, '/'), -1);
```
