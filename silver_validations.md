# Silver Validations

## Q: What are the total number of records in Silver tables?
Ans: Shows current row count for every cleaned Silver table.

Query:
```sql
SELECT 'silver.ns_brands' AS table_name, COUNT(*) AS cnt FROM retail.silver.ns_brands
UNION ALL
SELECT 'silver.ns_customers', COUNT(*) FROM retail.silver.ns_customers
UNION ALL
SELECT 'silver.ns_items', COUNT(*) FROM retail.silver.ns_items
UNION ALL
SELECT 'silver.ns_sales_orders', COUNT(*) FROM retail.silver.ns_sales_orders
UNION ALL
SELECT 'silver.ns_sales_order_lines', COUNT(*) FROM retail.silver.ns_sales_order_lines
UNION ALL
SELECT 'silver.ns_shipments', COUNT(*) FROM retail.silver.ns_shipments;
```

## Q: Are there duplicate business keys in Silver?
Ans: Should return 0 rows. Silver should be deduplicated/merged by business key.

Query:
```sql
SELECT internal_id, COUNT(*) AS cnt
FROM retail.silver.ns_brands
GROUP BY internal_id
HAVING COUNT(*) > 1;
```

## Q: Are all Silver records populated with etl_run_id and record_hash?
Ans: total, run_id_count, and hash_count should match.

Query:
```sql
SELECT 
  'ns_brands' AS table_name,
  COUNT(*) AS total,
  COUNT(etl_run_id) AS run_id_count,
  COUNT(record_hash) AS hash_count
FROM retail.silver.ns_brands
UNION ALL
SELECT 'ns_customers', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.silver.ns_customers
UNION ALL
SELECT 'ns_items', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.silver.ns_items
UNION ALL
SELECT 'ns_sales_orders', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.silver.ns_sales_orders
UNION ALL
SELECT 'ns_sales_order_lines', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.silver.ns_sales_order_lines
UNION ALL
SELECT 'ns_shipments', COUNT(*), COUNT(etl_run_id), COUNT(record_hash)
FROM retail.silver.ns_shipments;
```

## Q: Were Bronze runs processed into Silver successfully?
Ans: Shows Bronze-to-Silver run tracking.

Query:
```sql
SELECT table_name, source_run_id, target_run_id, status, rows_read, rows_written, rows_rejected, created_at
FROM retail.audit.layer_processed_runs
WHERE source_layer = 'bronze'
  AND target_layer = 'silver'
ORDER BY created_at DESC;
```

## Q: Are there unprocessed successful Bronze runs?
Ans: Should return 0 rows after Silver incremental finishes.

Query:
```sql
SELECT f.table_name, f.run_id, f.source_file_name
FROM retail.audit.ingested_files f
LEFT ANTI JOIN retail.audit.layer_processed_runs p
  ON f.run_id = p.source_run_id
 AND f.table_name = p.table_name
 AND p.source_layer = 'bronze'
 AND p.target_layer = 'silver'
 AND p.status = 'SUCCESS'
WHERE f.ingestion_status = 'SUCCESS';
```

## Q: Are there Silver rejects?
Ans: Shows rows rejected during Silver validation/type casting.

Query:
```sql
SELECT table_name, reject_reason, reject_column, COUNT(*) AS rejected_rows
FROM retail.audit.etl_reject_log
WHERE reject_stage = 'SILVER'
GROUP BY table_name, reject_reason, reject_column
ORDER BY table_name, rejected_rows DESC;
```

## Q: Are Silver data types correct for brands?
Ans: Confirms schema for Silver brands.

Query:
```sql
DESCRIBE TABLE retail.silver.ns_brands;
```

## Q: Are required fields missing in Silver brands?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.silver.ns_brands
WHERE internal_id IS NULL
   OR brand_code IS NULL OR TRIM(brand_code) = ''
   OR brand_name IS NULL OR TRIM(brand_name) = ''
   OR created_at IS NULL;
```

## Q: Are required fields missing in Silver customers?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.silver.ns_customers
WHERE internal_id IS NULL
   OR entity_id IS NULL OR TRIM(entity_id) = ''
   OR company_name IS NULL OR TRIM(company_name) = ''
   OR customer_type IS NULL OR TRIM(customer_type) = ''
   OR subsidiary IS NULL OR TRIM(subsidiary) = ''
   OR currency_code IS NULL OR TRIM(currency_code) = ''
   OR terms_name IS NULL OR TRIM(terms_name) = ''
   OR created_at IS NULL;
```

## Q: Are required fields missing in Silver items?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.silver.ns_items
WHERE internal_id IS NULL
   OR item_id IS NULL OR TRIM(item_id) = ''
   OR item_name IS NULL OR TRIM(item_name) = ''
   OR item_type IS NULL OR TRIM(item_type) = ''
   OR category IS NULL OR TRIM(category) = ''
   OR subcategory IS NULL OR TRIM(subcategory) = ''
   OR base_price IS NULL
   OR cost_price IS NULL
   OR created_at IS NULL;
```

## Q: Are required fields missing in Silver sales orders?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.silver.ns_sales_orders
WHERE internal_id IS NULL
   OR tran_id IS NULL OR TRIM(tran_id) = ''
   OR customer_internal_id IS NULL
   OR order_date IS NULL
   OR order_status IS NULL OR TRIM(order_status) = ''
   OR sales_channel IS NULL OR TRIM(sales_channel) = ''
   OR payment_status IS NULL OR TRIM(payment_status) = ''
   OR currency_code IS NULL OR TRIM(currency_code) = ''
   OR integration_source IS NULL OR TRIM(integration_source) = ''
   OR order_total IS NULL
   OR created_at IS NULL;
```

## Q: Are required fields missing in Silver sales order lines?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.silver.ns_sales_order_lines
WHERE internal_id IS NULL
   OR sales_order_internal_id IS NULL
   OR line_num IS NULL
   OR item_internal_id IS NULL
   OR quantity IS NULL
   OR unit_rate IS NULL
   OR line_amount IS NULL
   OR created_at IS NULL;
```

## Q: Are required fields missing in Silver shipments?
Ans: Should return 0 rows.

Query:
```sql
SELECT *
FROM retail.silver.ns_shipments
WHERE internal_id IS NULL
   OR shipment_number IS NULL OR TRIM(shipment_number) = ''
   OR sales_order_internal_id IS NULL
   OR warehouse_name IS NULL OR TRIM(warehouse_name) = ''
   OR shipment_date IS NULL
   OR shipment_status IS NULL OR TRIM(shipment_status) = ''
   OR created_at IS NULL;
```

## Q: Are item prices negative?
Ans: Should return 0 rows unless negative prices are intentionally allowed.

Query:
```sql
SELECT internal_id, base_price, cost_price
FROM retail.silver.ns_items
WHERE base_price < 0 OR cost_price < 0;
```

## Q: Are sales order totals negative?
Ans: Should return 0 rows unless refunds are modeled as negative orders.

Query:
```sql
SELECT internal_id, order_total
FROM retail.silver.ns_sales_orders
WHERE order_total < 0;
```

## Q: Are sales order line amounts inconsistent?
Ans: line_amount should approximately equal quantity * unit_rate.

Query:
```sql
SELECT internal_id, quantity, unit_rate, line_amount, quantity * unit_rate AS expected_amount
FROM retail.silver.ns_sales_order_lines
WHERE ABS(line_amount - (quantity * unit_rate)) > 0.05;
```

## Q: Are order lines referencing missing sales orders?
Ans: Should return 0 rows if all line orders exist.

Query:
```sql
SELECT l.internal_id AS line_id, l.sales_order_internal_id
FROM retail.silver.ns_sales_order_lines l
LEFT JOIN retail.silver.ns_sales_orders o
  ON l.sales_order_internal_id = o.internal_id
WHERE o.internal_id IS NULL;
```

## Q: Are sales orders referencing missing customers?
Ans: Should return 0 rows unless customer test data is intentionally bad.

Query:
```sql
SELECT o.internal_id AS order_id, o.customer_internal_id
FROM retail.silver.ns_sales_orders o
LEFT JOIN retail.silver.ns_customers c
  ON o.customer_internal_id = c.internal_id
WHERE c.internal_id IS NULL;
```

## Q: Are order lines referencing missing items?
Ans: Should return 0 rows unless item test data is intentionally bad.

Query:
```sql
SELECT l.internal_id AS line_id, l.item_internal_id
FROM retail.silver.ns_sales_order_lines l
LEFT JOIN retail.silver.ns_items i
  ON l.item_internal_id = i.internal_id
WHERE i.internal_id IS NULL;
```

## Q: Are shipments referencing missing sales orders?
Ans: Should return 0 rows.

Query:
```sql
SELECT s.internal_id AS shipment_id, s.sales_order_internal_id
FROM retail.silver.ns_shipments s
LEFT JOIN retail.silver.ns_sales_orders o
  ON s.sales_order_internal_id = o.internal_id
WHERE o.internal_id IS NULL;
```

## Q: Are delivery dates earlier than shipment dates?
Ans: Should return 0 rows unless partial bad test data exists.

Query:
```sql
SELECT internal_id, shipment_date, delivery_date
FROM retail.silver.ns_shipments
WHERE delivery_date IS NOT NULL
  AND delivery_date < shipment_date;
```

## Q: Are there invalid active flags?
Ans: Since Silver casts to boolean, this should return 0 rows.

Query:
```sql
SELECT internal_id, is_active
FROM retail.silver.ns_brands
WHERE is_active IS NULL;
```

## Q: Do Silver incremental runs have null audit counts?
Ans: Should return 0 rows after scripts are fixed.

Query:
```sql
SELECT *
FROM retail.audit.layer_processed_runs
WHERE target_layer = 'silver'
  AND status = 'SUCCESS'
  AND (rows_read IS NULL OR rows_written IS NULL OR rows_rejected IS NULL);
```
