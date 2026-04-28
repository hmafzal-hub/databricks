# Bronze Validations

## Q: What are the total number of records in Bronze tables?
Ans: Shows row count for every Bronze table after full/incremental loading.

Query:
```sql
SELECT 'bronze.ns_brands' AS table_name, COUNT(*) AS cnt FROM retail.bronze.ns_brands
UNION ALL
SELECT 'bronze.ns_customers', COUNT(*) FROM retail.bronze.ns_customers
UNION ALL
SELECT 'bronze.ns_items', COUNT(*) FROM retail.bronze.ns_items
UNION ALL
SELECT 'bronze.ns_sales_orders', COUNT(*) FROM retail.bronze.ns_sales_orders
UNION ALL
SELECT 'bronze.ns_sales_order_lines', COUNT(*) FROM retail.bronze.ns_sales_order_lines
UNION ALL
SELECT 'bronze.ns_shipments', COUNT(*) FROM retail.bronze.ns_shipments;
```

## Q: Were any source files processed more than once?
Ans: Should return 0 rows. If rows appear, file tracking/idempotency failed.

Query:
```sql
SELECT table_name, source_file_name, COUNT(*) AS cnt
FROM retail.audit.ingested_files
WHERE ingestion_status = 'SUCCESS'
GROUP BY table_name, source_file_name
HAVING COUNT(*) > 1;
```

## Q: Which files were ingested into Bronze?
Ans: Shows file-level ingestion history.

Query:
```sql
SELECT table_name, source_file_name, rows_read, rows_written, ingested_at
FROM retail.audit.ingested_files
ORDER BY ingested_at DESC;
```

## Q: Are all Bronze records populated with etl_run_id and record_hash?
Ans: total, run_id_count, and hash_count should match.

Query:
```sql
SELECT 
  'ns_brands' AS table_name,
  COUNT(*) AS total,
  COUNT(etl_run_id) AS run_id_count,
  COUNT(record_hash) AS hash_count
FROM retail.bronze.ns_brands
UNION ALL
SELECT 'ns_customers', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.bronze.ns_customers
UNION ALL
SELECT 'ns_items', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.bronze.ns_items
UNION ALL
SELECT 'ns_sales_orders', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.bronze.ns_sales_orders
UNION ALL
SELECT 'ns_sales_order_lines', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.bronze.ns_sales_order_lines
UNION ALL
SELECT 'ns_shipments', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.bronze.ns_shipments;
```

## Q: Are there duplicate business keys in Bronze?
Ans: Duplicates can exist in Bronze because Bronze is append-only, but this helps identify repeated/new versions.

Query:
```sql
SELECT internal_id, COUNT(*) AS cnt
FROM retail.bronze.ns_brands
GROUP BY internal_id
HAVING COUNT(*) > 1;
```

## Q: Are there duplicate identical records in Bronze?
Ans: Same record_hash repeated may mean exact duplicate records were loaded.

Query:
```sql
SELECT record_hash, COUNT(*) AS cnt
FROM retail.bronze.ns_brands
GROUP BY record_hash
HAVING COUNT(*) > 1;
```

## Q: Are there rejected Bronze rows?
Ans: Shows records rejected due to missing required fields.

Query:
```sql
SELECT table_name, reject_reason, reject_column, COUNT(*) AS rejected_rows
FROM retail.audit.etl_reject_log
WHERE reject_stage = 'BRONZE'
GROUP BY table_name, reject_reason, reject_column
ORDER BY table_name, rejected_rows DESC;
```

## Q: Are required columns missing in Bronze brands?
Ans: Should return 0 rows for valid Bronze records.

Query:
```sql
SELECT *
FROM retail.bronze.ns_brands
WHERE internal_id IS NULL OR TRIM(internal_id) = ''
   OR brand_code IS NULL OR TRIM(brand_code) = ''
   OR brand_name IS NULL OR TRIM(brand_name) = ''
   OR is_active IS NULL OR TRIM(is_active) = ''
   OR created_at IS NULL OR TRIM(created_at) = '';
```

## Q: Are required columns missing in Bronze customers?
Ans: Should return 0 rows for valid Bronze records.

Query:
```sql
SELECT *
FROM retail.bronze.ns_customers
WHERE internal_id IS NULL OR TRIM(internal_id) = ''
   OR entity_id IS NULL OR TRIM(entity_id) = ''
   OR company_name IS NULL OR TRIM(company_name) = ''
   OR customer_type IS NULL OR TRIM(customer_type) = ''
   OR subsidiary IS NULL OR TRIM(subsidiary) = ''
   OR currency_code IS NULL OR TRIM(currency_code) = ''
   OR terms_name IS NULL OR TRIM(terms_name) = ''
   OR is_active IS NULL OR TRIM(is_active) = ''
   OR created_at IS NULL OR TRIM(created_at) = '';
```

## Q: Are required columns missing in Bronze items?
Ans: Should return 0 rows for valid Bronze records.

Query:
```sql
SELECT *
FROM retail.bronze.ns_items
WHERE internal_id IS NULL OR TRIM(internal_id) = ''
   OR item_id IS NULL OR TRIM(item_id) = ''
   OR item_name IS NULL OR TRIM(item_name) = ''
   OR item_type IS NULL OR TRIM(item_type) = ''
   OR category IS NULL OR TRIM(category) = ''
   OR subcategory IS NULL OR TRIM(subcategory) = ''
   OR base_price IS NULL OR TRIM(base_price) = ''
   OR cost_price IS NULL OR TRIM(cost_price) = ''
   OR is_active IS NULL OR TRIM(is_active) = ''
   OR created_at IS NULL OR TRIM(created_at) = '';
```

## Q: Are required columns missing in Bronze sales orders?
Ans: Should return 0 rows for valid Bronze records.

Query:
```sql
SELECT *
FROM retail.bronze.ns_sales_orders
WHERE internal_id IS NULL OR TRIM(internal_id) = ''
   OR tran_id IS NULL OR TRIM(tran_id) = ''
   OR customer_internal_id IS NULL OR TRIM(customer_internal_id) = ''
   OR order_date IS NULL OR TRIM(order_date) = ''
   OR order_status IS NULL OR TRIM(order_status) = ''
   OR sales_channel IS NULL OR TRIM(sales_channel) = ''
   OR payment_status IS NULL OR TRIM(payment_status) = ''
   OR currency_code IS NULL OR TRIM(currency_code) = ''
   OR integration_source IS NULL OR TRIM(integration_source) = ''
   OR order_total IS NULL OR TRIM(order_total) = ''
   OR created_at IS NULL OR TRIM(created_at) = '';
```

## Q: Are required columns missing in Bronze sales order lines?
Ans: Should return 0 rows for valid Bronze records.

Query:
```sql
SELECT *
FROM retail.bronze.ns_sales_order_lines
WHERE internal_id IS NULL OR TRIM(internal_id) = ''
   OR sales_order_internal_id IS NULL OR TRIM(sales_order_internal_id) = ''
   OR line_num IS NULL OR TRIM(line_num) = ''
   OR item_internal_id IS NULL OR TRIM(item_internal_id) = ''
   OR quantity IS NULL OR TRIM(quantity) = ''
   OR unit_rate IS NULL OR TRIM(unit_rate) = ''
   OR line_amount IS NULL OR TRIM(line_amount) = ''
   OR created_at IS NULL OR TRIM(created_at) = '';
```

## Q: Are required columns missing in Bronze shipments?
Ans: Should return 0 rows for valid Bronze records.

Query:
```sql
SELECT *
FROM retail.bronze.ns_shipments
WHERE internal_id IS NULL OR TRIM(internal_id) = ''
   OR shipment_number IS NULL OR TRIM(shipment_number) = ''
   OR sales_order_internal_id IS NULL OR TRIM(sales_order_internal_id) = ''
   OR warehouse_name IS NULL OR TRIM(warehouse_name) = ''
   OR shipment_date IS NULL OR TRIM(shipment_date) = ''
   OR shipment_status IS NULL OR TRIM(shipment_status) = ''
   OR created_at IS NULL OR TRIM(created_at) = '';
```

## Q: Are numeric-looking fields invalid in Bronze?
Ans: Should return 0 rows if Bronze data can be cast safely in Silver.

Query:
```sql
SELECT internal_id, base_price, cost_price
FROM retail.bronze.ns_items
WHERE TRY_CAST(internal_id AS INT) IS NULL
   OR TRY_CAST(base_price AS DECIMAL(18,2)) IS NULL
   OR TRY_CAST(cost_price AS DECIMAL(18,2)) IS NULL;
```

## Q: Are date/timestamp fields invalid in Bronze?
Ans: Should return 0 rows if dates are valid for Silver casting.

Query:
```sql
SELECT internal_id, created_at
FROM retail.bronze.ns_brands
WHERE TRY_CAST(created_at AS TIMESTAMP) IS NULL;
```

## Q: Are boolean flags invalid in Bronze?
Ans: Should return 0 rows if boolean-like values are valid.

Query:
```sql
SELECT internal_id, is_active
FROM retail.bronze.ns_brands
WHERE LOWER(TRIM(is_active)) NOT IN ('true','false','1','0','yes','no','y','n');
```

## Q: Did each Bronze run write the expected number of records?
Ans: rows_written should match the count in Bronze for that run_id.

Query:
```sql
SELECT 
  l.table_name,
  l.run_id,
  l.rows_written AS audit_rows_written,
  COUNT(b.etl_run_id) AS actual_rows_in_bronze
FROM retail.audit.etl_run_log l
LEFT JOIN retail.bronze.ns_brands b
  ON l.run_id = b.etl_run_id
WHERE l.layer_name = 'bronze'
  AND l.table_name = 'ns_brands'
GROUP BY l.table_name, l.run_id, l.rows_written
ORDER BY l.run_id;
```
