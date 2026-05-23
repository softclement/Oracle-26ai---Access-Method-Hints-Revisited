# Oracle 26ai - Access Method Hints Revisited

## Overview

I revisited Oracle Access Method Hints in Oracle 26ai to understand:

* Which hints are still practically useful
* How the modern optimizer behaves
* Which hints are often ignored
* How adaptive optimization influences execution plans

This repository contains:

* High-volume test dataset
* DDL and DML scripts
* Relevant index creation
* Explain Plan demonstrations
* Real optimizer observations from Oracle 26ai

---

# Test Environment

* Oracle Database 26ai
* 1 Million Rows
* Mixed OLTP-style dataset
* B-tree and Bitmap indexes

---

# Table Creation

```sql
DROP TABLE sales PURGE;

CREATE TABLE sales
(
    sale_id         NUMBER,
    customer_id     NUMBER,
    product_id      NUMBER,
    sale_date       DATE,
    region          VARCHAR2(20),
    amount          NUMBER,
    status          VARCHAR2(10),
    remarks         VARCHAR2(100)
);
```

---

# Data Generation

```sql
BEGIN
    FOR i IN 1 .. 1000000 LOOP
        INSERT INTO sales
        VALUES
        (
            i,
            MOD(i,10000),
            MOD(i,5000),
            SYSDATE - MOD(i,365),

            CASE
                WHEN MOD(i,4)=0 THEN 'NORTH'
                WHEN MOD(i,4)=1 THEN 'SOUTH'
                WHEN MOD(i,4)=2 THEN 'EAST'
                ELSE 'WEST'
            END,

            ROUND(DBMS_RANDOM.VALUE(100,50000),2),

            CASE
                WHEN MOD(i,10)=0 THEN 'FAILED'
                ELSE 'SUCCESS'
            END,

            RPAD('COMMENTS',50,'X')
        );

        IF MOD(i,10000)=0 THEN
            COMMIT;
        END IF;
    END LOOP;

    COMMIT;
END;
/
```

---

# Index Creation

```sql
CREATE INDEX idx_sales_cust
ON sales(customer_id);

CREATE INDEX idx_sales_date
ON sales(sale_date);

CREATE INDEX idx_sales_amount
ON sales(amount);

CREATE INDEX idx_sales_cust_amount
ON sales(customer_id, amount);

CREATE INDEX idx_sales_region_amount2
ON sales(region, amount);

CREATE BITMAP INDEX bix_sales_region
ON sales(region);

CREATE BITMAP INDEX bix_sales_status
ON sales(status);
```

---

# Gather Statistics

```sql
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => USER,
        tabname => 'SALES',
        cascade => TRUE
    );
END;
/
```

---

# Relevant Access Method Hints in Oracle 26ai

## 1. INDEX

Still one of the most useful hints for troubleshooting and plan testing.

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX(s idx_sales_cust) */
       *
FROM sales s
WHERE customer_id = 100;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Plan

```text
INDEX RANGE SCAN IDX_SALES_CUST
TABLE ACCESS BY INDEX ROWID
```

### Observation

* Still highly relevant
* Commonly used in performance troubleshooting
* Useful for validating optimizer choices

---

# 2. INDEX_DESC

Useful for latest-record or Top-N queries.

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX_DESC(s idx_sales_date) */
       sale_date
FROM sales s
WHERE sale_date >= SYSDATE-30
ORDER BY sale_date DESC;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Plan

```text
INDEX RANGE SCAN DESCENDING
```

### Observation

Still useful in:

* ORDER BY DESC
* Recent transaction retrieval
* Latest data lookups

---

# 3. INDEX_FFS (Fast Full Scan)

One of the most important modern access hints.

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX_FFS(s idx_sales_cust) */
       customer_id
FROM sales s
WHERE customer_id IS NOT NULL;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Plan

```text
INDEX FAST FULL SCAN
```

### Observation

Oracle scans entire index segment using:

* Multiblock I/O
* Parallel-capable reads
* No table access

Very useful when:

* Query uses only indexed columns
* Index is smaller than table

---

# 4. INDEX_SS (Skip Scan)

Skip scan is still actively used in Oracle 26ai.

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX_SS(s idx_sales_region_amount2) */
       *
FROM sales s
WHERE amount > 40000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Plan

```text
INDEX SKIP SCAN
```

### Observation

Useful when:

* Query misses leading column
* Leading column has low cardinality

Example:

```text
(region, amount)
```

Query filters only:

```sql
amount > 40000
```

Oracle internally scans logical subindexes.

---

# 5. INDEX_COMBINE

Still relevant mainly in Data Warehouse systems.

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX_COMBINE(s bix_sales_region bix_sales_status) */
       COUNT(*)
FROM sales s
WHERE region='NORTH'
AND status='FAILED';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Plan

```text
BITMAP AND
BITMAP INDEX SINGLE VALUE
```

### Observation

Mostly useful for:

* Bitmap indexes
* Analytics workloads
* Reporting systems
* Data warehouse environments

Less common in OLTP systems.

---

# 6. INDEX_JOIN

Rare but interesting optimizer behavior.

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX_JOIN(s idx_sales_cust idx_sales_amount) */
       customer_id,
       amount
FROM sales s
WHERE customer_id = 100
AND amount > 49000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Plan

```text
VIEW index$_join$_001
HASH JOIN
INDEX RANGE SCAN
INDEX RANGE SCAN
```

### Observation

Oracle dynamically joins multiple indexes using ROWIDs.

Works best when:

* Predicates are highly selective
* Query uses only indexed columns
* Table access is expensive

Rarely seen in OLTP systems.

---

# 7. NO_INDEX

Very useful for benchmarking and optimizer comparison.

```sql
EXPLAIN PLAN FOR
SELECT /*+ FULL(s) */
       *
FROM sales s
WHERE customer_id = 100;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Observation

Useful for:

* Comparing full scan vs index scan
* Testing index usefulness
* Performance analysis
* Troubleshooting

---

# Access Method Hints Less Relevant in Oracle 26ai

The following hints are now mostly optimizer-controlled automatically:

| Hint          | Reason                          |
| ------------- | ------------------------------- |
| INDEX_ASC     | Default behavior already        |
| INDEX_RS      | Rarely needed                   |
| INDEX_RS_ASC  | Optimizer handles automatically |
| INDEX_RS_DESC | Optimizer handles automatically |
| INDEX_SS_ASC  | Limited practical use           |
| INDEX_SS_DESC | Limited practical use           |

---

# Key Findings

## Modern Oracle Optimizer Is Highly Intelligent

Oracle 26ai optimizer evaluates:

* Statistics
* Object size
* Selectivity
* Runtime cost
* Adaptive optimization
* Automatic indexing
* Dynamic statistics

before blindly accepting hints.

---

# Important Reality

Hints are no longer strict directives.

Modern Oracle treats hints more like:

```text
Strong optimizer recommendations
```

Some hints may be:

* Accepted
* Partially obeyed
* Completely ignored

depending on optimizer cost calculations.

---

# Practical Recommendation

Use hints:

* For troubleshooting
* During RCA
* For testing
* For plan stability
* For temporary tuning

Avoid excessive hardcoded hints in application SQL.

---

# Final Takeaway

In Oracle 26ai:

* Optimizer intelligence matters more than manual hinting
* Good indexing and statistics are more important
* Hints should support tuning, not replace design

Still highly relevant:

* INDEX
* INDEX_DESC
* INDEX_FFS
* INDEX_SS
* INDEX_COMBINE
* NO_INDEX

Specialized / niche:

* INDEX_JOIN

Mostly unnecessary today:

* INDEX_ASC
* INDEX_RS variants
* INDEX_SS_ASC / DESC

```
```
