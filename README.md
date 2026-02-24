# README: Retail Cohort Analysis Using MySQL

## **1. Overview**

This project performs **Cohort Analysis** on the `RETAIL` table in the `SALES` database.  

**Objectives:**
1. Explore and clean the retail sales data.
2. Perform **customer retention analysis** using cohorts.
3. Calculate **Customer Lifetime Value (CLV)** per cohort.
4. Generate monthly retention and sales patterns for strategic insights.

---

## **2. Data Exploration & Cleaning**

### Basic Data Checks:

```sql
-- Preview first 10 records
SELECT * FROM RETAIL LIMIT 10;

-- Total records
SELECT COUNT(*) FROM RETAIL; -- 521,773

-- Unique countries
SELECT DISTINCT COUNTRY FROM RETAIL;

-- Minimum quantity
SELECT MIN(Quantity) FROM RETAIL; -- -74,215

-- Suspicious records (quantity <= 0)
SELECT COUNT(*) FROM RETAIL WHERE QUANTITY <= 0; -- 10,363

-- Cancelled orders (Invoice starts with 'C')
SELECT COUNT(*) FROM RETAIL WHERE INVOICENO LIKE 'C%'; -- 9,054
```

### Filter Invalid Records:

```sql
-- Cancelled orders with negative quantity
SELECT * FROM RETAIL
WHERE INVOICENO NOT LIKE 'C%' AND QUANTITY <= 0;

-- Missing customer IDs
SELECT COUNT(*) FROM RETAIL WHERE CUSTOMERID = ''; -- 128,419
```

### Convert Date Strings:

```sql
SELECT
    INVOICEDATE,
    STR_TO_DATE(INVOICEDATE, '%m/%d/%Y %H:%i') AS DATETIME,
    DATE(STR_TO_DATE(INVOICEDATE, '%m/%d/%Y %H:%i')) AS DATE
FROM RETAIL
LIMIT 10;
```

---

## **3. Cohort Analysis: Customer Retention**

### Step 1: Filter Valid Records

```sql
WITH CTE1 AS (
    SELECT
        CUSTOMERID,
        STR_TO_DATE(INVOICEDATE, '%m/%d/%Y %H:%i') AS FORMATTED_DATE,
        ROUND(QUANTITY*UNITPRICE, 2) AS SALE_VALUE
    FROM RETAIL
    WHERE CUSTOMERID IS NOT NULL
      AND CUSTOMERID != ''
      AND INVOICENO NOT LIKE 'C%'
      AND QUANTITY > 0
      AND UNITPRICE > 0
)
```

### Step 2: Determine First Transaction Date

```sql
, CTE2 AS (
    SELECT
        CUSTOMERID,
        FORMATTED_DATE AS PURCHASE_DATE,
        MIN(FORMATTED_DATE) OVER (PARTITION BY CUSTOMERID) AS FIRST_TRANSACTION_DATE
    FROM CTE1
)
```

### Step 3: Assign Cohort Month

```sql
, CTE3 AS (
    SELECT 
        CUSTOMERID,
        FIRST_TRANSACTION_DATE,
        PURCHASE_DATE,
        CONCAT('Month_', ROUND(DATEDIFF(PURCHASE_DATE, FIRST_TRANSACTION_DATE)/30, 0)) AS COHORT_MONTH,
        DATE_FORMAT(PURCHASE_DATE, '%Y-%m-01') AS PURCHASE_MONTH,
        DATE_FORMAT(FIRST_TRANSACTION_DATE, '%Y-%m-01') AS FIRST_TRANSACTION_MONTH
    FROM CTE2
)
```

### Step 4: Aggregate Retention Counts

```sql
SELECT
    FIRST_TRANSACTION_MONTH AS COHORT,
    COUNT(DISTINCT CASE WHEN COHORT_MONTH = 'Month_0' THEN CUSTOMERID END) AS "MONTH_0",
    COUNT(DISTINCT CASE WHEN COHORT_MONTH = 'Month_1' THEN CUSTOMERID END) AS "MONTH_1",
    COUNT(DISTINCT CASE WHEN COHORT_MONTH = 'Month_2' THEN CUSTOMERID END) AS "MONTH_2",
    ...
    COUNT(DISTINCT CASE WHEN COHORT_MONTH = 'Month_12' THEN CUSTOMERID END) AS "MONTH_12"
FROM CTE3
GROUP BY FIRST_TRANSACTION_MONTH
ORDER BY FIRST_TRANSACTION_MONTH;
```

> This table shows **how many customers from each cohort return in subsequent months**.

---

## **4. Cohort Analysis: Customer Lifetime Value (CLV)**

### Step 1: Use Same Filtered Records as Retention

```sql
WITH CTE1 AS (...),
     CTE2 AS (...),
     CTE3 AS (...)
```

### Step 2: Aggregate Monthly Sales by Cohort

```sql
SELECT
    FIRST_TRANSACTION_MONTH AS COHORT,
    ROUND(SUM(CASE WHEN COHORT_MONTH = 'Month_0' THEN SALE_VALUE ELSE 0 END),0) AS "MONTH_0",
    ROUND(SUM(CASE WHEN COHORT_MONTH = 'Month_1' THEN SALE_VALUE ELSE 0 END),0) AS "MONTH_1",
    ...
    ROUND(SUM(CASE WHEN COHORT_MONTH = 'Month_12' THEN SALE_VALUE ELSE 0 END),0) AS "MONTH_12"
FROM CTE3
GROUP BY FIRST_TRANSACTION_MONTH
ORDER BY FIRST_TRANSACTION_MONTH;
```

> This table provides the **monetary contribution of each cohort over time**, helping estimate Customer Lifetime Value (CLV).

---

## **5. Notes & Insights**

1. Records with `QUANTITY <= 0` or `INVOICENO LIKE 'C%'` are **excluded** from cohort and CLV analysis.
2. Dates are **converted to proper datetime format** for calculations.
3. Cohort months are calculated as the number of months since the **first transaction**.
4. Both **retention and CLV analysis** use monthly cohorts to track engagement and revenue trends.
5. Useful for:
   - Targeted marketing
   - Customer loyalty analysis
   - Forecasting revenue per cohort

---

## **6. References**

- MySQL Functions:
  - `STR_TO_DATE()`, `DATEDIFF()`, `ROUND()`, `DATE_FORMAT()`
  - `COUNT(DISTINCT ...)` for unique customer counts
  - `CASE WHEN` for conditional aggregation
- Cohort Analysis concepts: *Customer Retention and CLV*
