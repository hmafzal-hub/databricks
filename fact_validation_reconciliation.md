# Fact Table Validation + Reconciliation

Project: Retail Lakehouse Data Engineering Project  
Layer: Gold  
Main Table: `retail.gold.fact_sales`  
Purpose: Validate fact table quality, dimension integrity, duplicate handling, SCD2 joins, and revenue reconciliation.

---

## 1. Fact Table Row Count

**Q: What is the total number of records in the Gold fact table?**

**Ans:** Shows total rows loaded into `fact_sales`.

**Query:**

```sql
SELECT COUNT(*) AS fact_sales_count
FROM retail.gold.fact_sales;
```

---

## 2. Null Surrogate Key Check

**Q: Are there any fact rows with missing surrogate keys?**

**Ans:** Expected result should be `0` for all null key counts.

**Query:**

```sql
SELECT
  SUM(CASE WHEN sales_key IS NULL THEN 1 ELSE 0 END) AS null_sales_key,
  SUM(CASE WHEN customer_key IS NULL THEN 1 ELSE 0 END) AS null_customer_key,
  SUM(CASE WHEN item_key IS NULL THEN 1 ELSE 0 END) AS null_item_key,
  SUM(CASE WHEN order_date_key IS NULL THEN 1 ELSE 0 END) AS null_order_date_key
FROM retail.gold.fact_sales;
```

---

## 3. Duplicate Fact Business Key Check

**Q: Are there duplicate fact rows for the same sales order line?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  sales_order_line_internal_id,
  COUNT(*) AS duplicate_count
FROM retail.gold.fact_sales
GROUP BY sales_order_line_internal_id
HAVING COUNT(*) > 1;
```

---

## 4. Fact to Customer Dimension Integrity

**Q: Does every fact row have a valid customer dimension record?**

**Ans:** Expected missing customer count should be `0`.

**Query:**

```sql
SELECT COUNT(*) AS missing_customer_dim_rows
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_customer c
  ON f.customer_key = c.customer_key
WHERE c.customer_key IS NULL;
```

---

## 5. Fact to Item Dimension Integrity

**Q: Does every fact row have a valid item dimension record?**

**Ans:** Expected missing item count should be `0`.

**Query:**

```sql
SELECT COUNT(*) AS missing_item_dim_rows
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_item i
  ON f.item_key = i.item_key
WHERE i.item_key IS NULL;
```

---

## 6. Fact to Date Dimension Integrity

**Q: Does every fact row have a valid order date dimension record?**

**Ans:** Expected missing date count should be `0`.

**Query:**

```sql
SELECT COUNT(*) AS missing_date_dim_rows
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_date d
  ON f.order_date_key = d.date_key
WHERE d.date_key IS NULL;
```

---

## 7. Current Dimension Join Check

**Q: How many fact rows join to current dimension versions?**

**Ans:** Useful for checking active dimension linkage. For SCD2 historical facts, some rows may intentionally link to older non-current dimension versions.

**Query:**

```sql
SELECT
  SUM(CASE WHEN c.is_current = true THEN 1 ELSE 0 END) AS rows_with_current_customer,
  SUM(CASE WHEN i.is_current = true THEN 1 ELSE 0 END) AS rows_with_current_item,
  COUNT(*) AS total_fact_rows
FROM retail.gold.fact_sales f
LEFT JOIN retail.gold.dim_customer c
  ON f.customer_key = c.customer_key
LEFT JOIN retail.gold.dim_item i
  ON f.item_key = i.item_key;
```

---

## 8. SCD2 Effective Date Validation - Customer

**Q: Does each fact row point to the customer version valid on the order date?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  f.sales_order_line_internal_id,
  f.order_date_key,
  d.full_date AS order_date,
  f.customer_key,
  c.customer_internal_id,
  c.effective_start_date,
  c.effective_end_date
FROM retail.gold.fact_sales f
JOIN retail.gold.dim_date d
  ON f.order_date_key = d.date_key
JOIN retail.gold.dim_customer c
  ON f.customer_key = c.customer_key
WHERE d.full_date < DATE(c.effective_start_date)
   OR d.full_date >= DATE(COALESCE(c.effective_end_date, TIMESTAMP('9999-12-31')));
```

---

## 9. SCD2 Effective Date Validation - Item

**Q: Does each fact row point to the item version valid on the order date?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  f.sales_order_line_internal_id,
  f.order_date_key,
  d.full_date AS order_date,
  f.item_key,
  i.item_internal_id,
  i.effective_start_date,
  i.effective_end_date
FROM retail.gold.fact_sales f
JOIN retail.gold.dim_date d
  ON f.order_date_key = d.date_key
JOIN retail.gold.dim_item i
  ON f.item_key = i.item_key
WHERE d.full_date < DATE(i.effective_start_date)
   OR d.full_date >= DATE(COALESCE(i.effective_end_date, TIMESTAMP('9999-12-31')));
```

---

## 10. Fact Count vs Silver Sales Order Lines

**Q: Does Gold fact row count match Silver sales order lines count?**

**Ans:** Counts should match unless some Silver rows were rejected due to failed dimension lookups.

**Query:**

```sql
SELECT 'silver_sales_order_lines' AS layer_table, COUNT(*) AS row_count
FROM retail.silver.ns_sales_order_lines

UNION ALL

SELECT 'gold_fact_sales' AS layer_table, COUNT(*) AS row_count
FROM retail.gold.fact_sales;
```

---

## 11. Missing Fact Rows Compared to Silver Lines

**Q: Which Silver sales order lines are missing from Gold fact table?**

**Ans:** Expected result should return `0 rows`, unless rejected due to failed dimension lookup.

**Query:**

```sql
SELECT
  sol.internal_id AS missing_sales_order_line_internal_id,
  sol.sales_order_internal_id,
  sol.item_internal_id,
  sol.quantity,
  sol.line_amount
FROM retail.silver.ns_sales_order_lines sol
LEFT JOIN retail.gold.fact_sales f
  ON sol.internal_id = f.sales_order_line_internal_id
WHERE f.sales_order_line_internal_id IS NULL;
```

---

## 12. Extra Fact Rows Not Found in Silver

**Q: Are there fact rows that no longer exist in Silver sales order lines?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  f.sales_order_line_internal_id,
  f.sales_order_internal_id,
  f.item_key,
  f.quantity,
  f.line_amount
FROM retail.gold.fact_sales f
LEFT JOIN retail.silver.ns_sales_order_lines sol
  ON f.sales_order_line_internal_id = sol.internal_id
WHERE sol.internal_id IS NULL;
```

---

## 13. Revenue Reconciliation - Silver Lines vs Gold Fact

**Q: Does total revenue match between Silver sales order lines and Gold fact table?**

**Ans:** Total `line_amount` should match unless rejected rows exist.

**Query:**

```sql
SELECT 'silver_sales_order_lines' AS layer_table, SUM(line_amount) AS total_revenue
FROM retail.silver.ns_sales_order_lines

UNION ALL

SELECT 'gold_fact_sales' AS layer_table, SUM(line_amount) AS total_revenue
FROM retail.gold.fact_sales;
```

---

## 14. Quantity Reconciliation - Silver Lines vs Gold Fact

**Q: Does total quantity match between Silver sales order lines and Gold fact table?**

**Ans:** Total quantity should match unless rejected rows exist.

**Query:**

```sql
SELECT 'silver_sales_order_lines' AS layer_table, SUM(quantity) AS total_quantity
FROM retail.silver.ns_sales_order_lines

UNION ALL

SELECT 'gold_fact_sales' AS layer_table, SUM(quantity) AS total_quantity
FROM retail.gold.fact_sales;
```

---

## 15. Revenue Difference Check

**Q: What is the exact revenue difference between Silver and Gold?**

**Ans:** Expected difference should be `0.00`.

**Query:**

```sql
WITH silver_revenue AS (
  SELECT SUM(line_amount) AS total_revenue
  FROM retail.silver.ns_sales_order_lines
),
gold_revenue AS (
  SELECT SUM(line_amount) AS total_revenue
  FROM retail.gold.fact_sales
)
SELECT
  s.total_revenue AS silver_total_revenue,
  g.total_revenue AS gold_total_revenue,
  s.total_revenue - g.total_revenue AS revenue_difference
FROM silver_revenue s
CROSS JOIN gold_revenue g;
```

---

## 16. Order Total Reconciliation

**Q: Does fact line revenue reconcile with order total at sales order level?**

**Ans:** Differences should be checked. Some difference may exist if order total includes tax, discount, or shipping.

**Query:**

```sql
SELECT
  sales_order_internal_id,
  SUM(line_amount) AS fact_line_total,
  MAX(order_total) AS order_total,
  SUM(line_amount) - MAX(order_total) AS difference
FROM retail.gold.fact_sales
GROUP BY sales_order_internal_id
HAVING ROUND(SUM(line_amount) - MAX(order_total), 2) <> 0;
```

---

## 17. Negative or Invalid Amount Check

**Q: Are there invalid negative quantities, rates, or amounts in the fact table?**

**Ans:** Expected result should return `0 rows`, unless returns/refunds are intentionally modeled.

**Query:**

```sql
SELECT *
FROM retail.gold.fact_sales
WHERE quantity < 0
   OR unit_rate < 0
   OR line_amount < 0
   OR order_total < 0;
```

---

## 18. Line Amount Calculation Validation

**Q: Does `quantity * unit_rate` match `line_amount`?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  sales_order_line_internal_id,
  quantity,
  unit_rate,
  line_amount,
  ROUND(quantity * unit_rate, 2) AS calculated_line_amount,
  ROUND(line_amount - (quantity * unit_rate), 2) AS difference
FROM retail.gold.fact_sales
WHERE ROUND(line_amount - (quantity * unit_rate), 2) <> 0;
```

---

## 19. Shipment Date Validation

**Q: Are there shipments where delivery date is before shipment date?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  sales_order_line_internal_id,
  shipment_date,
  delivery_date,
  shipment_status,
  tracking_number
FROM retail.gold.fact_sales
WHERE delivery_date IS NOT NULL
  AND shipment_date IS NOT NULL
  AND delivery_date < shipment_date;
```

---

## 20. Order Date vs Shipment Date Validation

**Q: Are there shipments before the order date?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  f.sales_order_line_internal_id,
  d.full_date AS order_date,
  f.shipment_date,
  f.delivery_date,
  f.shipment_status
FROM retail.gold.fact_sales f
JOIN retail.gold.dim_date d
  ON f.order_date_key = d.date_key
WHERE f.shipment_date IS NOT NULL
  AND f.shipment_date < d.full_date;
```

---

## 21. Fact Hash Duplicate Check

**Q: Are there duplicate fact records with the same business key and hash?**

**Ans:** Expected result should return `0 rows`.

**Query:**

```sql
SELECT
  sales_order_line_internal_id,
  record_hash,
  COUNT(*) AS duplicate_count
FROM retail.gold.fact_sales
GROUP BY sales_order_line_internal_id, record_hash
HAVING COUNT(*) > 1;
```

---

## 22. Fact Audit Run Check

**Q: How many fact rows were loaded by each ETL run?**

**Ans:** Helps verify incremental loads and audit traceability.

**Query:**

```sql
SELECT
  etl_run_id,
  COUNT(*) AS fact_rows,
  SUM(line_amount) AS total_revenue,
  MIN(etl_loaded_at) AS min_loaded_at,
  MAX(etl_loaded_at) AS max_loaded_at
FROM retail.gold.fact_sales
GROUP BY etl_run_id
ORDER BY max_loaded_at DESC;
```

---

## 23. Reject Log Check for Gold Fact

**Q: Were any fact rows rejected during Gold fact loading?**

**Ans:** Expected result should be `0` if all dimension lookups passed.

**Query:**

```sql
SELECT
  table_name,
  reject_stage,
  reject_reason,
  error_code,
  COUNT(*) AS rejected_rows
FROM retail.audit.etl_reject_log
WHERE table_name = 'fact_sales'
GROUP BY table_name, reject_stage, reject_reason, error_code;
```

---

## 24. Fact Load Audit Check

**Q: What is the audit status of Gold fact loads?**

**Ans:** All recent fact loads should show `SUCCESS`.

**Query:**

```sql
SELECT
  run_id,
  batch_id,
  table_name,
  layer_name,
  load_type,
  status,
  rows_read,
  rows_written,
  rows_rejected,
  records_before,
  records_after,
  start_time,
  end_time,
  duration_seconds,
  error_message
FROM retail.audit.etl_run_log
WHERE table_name = 'fact_sales'
ORDER BY start_time DESC;
```

---

## 25. Final Fact Health Summary

**Q: What is the overall health summary of the Gold fact table?**

**Ans:** Gives one quick quality summary for the fact table.

**Query:**

```sql
SELECT
  COUNT(*) AS total_fact_rows,
  COUNT(DISTINCT sales_order_line_internal_id) AS distinct_sales_order_lines,
  SUM(CASE WHEN customer_key IS NULL THEN 1 ELSE 0 END) AS null_customer_keys,
  SUM(CASE WHEN item_key IS NULL THEN 1 ELSE 0 END) AS null_item_keys,
  SUM(CASE WHEN order_date_key IS NULL THEN 1 ELSE 0 END) AS null_order_date_keys,
  SUM(CASE WHEN quantity < 0 THEN 1 ELSE 0 END) AS negative_quantity_rows,
  SUM(CASE WHEN line_amount < 0 THEN 1 ELSE 0 END) AS negative_amount_rows,
  SUM(line_amount) AS total_revenue,
  SUM(quantity) AS total_quantity
FROM retail.gold.fact_sales;
```

---

# Recommended Execution Order

1. Run checks 1 to 6 first for basic integrity.
2. Run checks 8 and 9 for SCD2 date-effective validation.
3. Run checks 10 to 15 for Silver-to-Gold reconciliation.
4. Run checks 16 to 20 for business-rule validation.
5. Run checks 23 to 25 for audit and final health summary.

