# Case 09: PayPal Refund Disputes Analysis

## Business Context

You are a Product Analyst at PayPal investigating customer refund dispute characteristics across transaction types. Your team wants to optimize the buyer protection process for more efficient resolutions.

## Database Schema

### fct_disputed_transactions
- transaction_id: INTEGER
- dispute_initiated_date: DATE
- refund_amount: DECIMAL
- resolution_time_days: INTEGER
- product_type: VARCHAR
- transaction_category: VARCHAR

---

## Question 1: Average Refund Amount for Digital Goods (Oct 1-7, 2024)

### Problem
For disputes involving digital goods (where the product_type begins with 'DIG'), what is the average refund amount for disputes initiated from October 1st to October 7th, 2024?

### Expected Output
- avg_refund_amount

### Solution

```sql
SELECT AVG(refund_amount) AS avg_refund_amount
FROM fct_disputed_transactions
WHERE dispute_initiated_date >= '2024-10-01' 
  AND dispute_initiated_date < '2024-10-08' 
  AND product_type LIKE 'DIG%';
```

**Explanation:**
- Uses AVG() to calculate mean refund amount
- Filters to specific week using date range (Oct 1-7)
- Uses LIKE 'DIG%' pattern to match digital goods

---

## Question 2: Highest Average Resolution Time Among Top 5 Categories (October 2024)

### Problem
For disputes in October 2024, first find the top 5 transaction categories by the number of disputes. Among these top 5 categories, identify the category with the highest average dispute resolution time.

### Expected Output
- transaction_category
- avg_resol_time

### Solution

```sql
WITH top_5 AS (
    SELECT transaction_category,
           COUNT(*) AS dispute_number,
           AVG(resolution_time_days) AS avg_resol_time
    FROM fct_disputed_transactions
    WHERE dispute_initiated_date >= '2024-10-01' 
      AND dispute_initiated_date < '2024-11-01' 
    GROUP BY transaction_category
    ORDER BY COUNT(*) DESC, AVG(resolution_time_days) DESC
    LIMIT 5
),
ranking AS (
    SELECT transaction_category,
           avg_resol_time,
           ROW_NUMBER() OVER (ORDER BY avg_resol_time DESC) AS rn
    FROM top_5
)
SELECT transaction_category,
       avg_resol_time
FROM ranking
WHERE rn = 1;
```

**Explanation:**
- First CTE gets top 5 categories by dispute count
- Orders by COUNT(*) DESC to get highest volume categories
- LIMIT 5 restricts to top 5 categories
- Second CTE ranks these 5 categories by average resolution time

---

## Question 3: Physical Goods Percentage in Highest Resolution Time Quartile (October 2024)

### Problem
Segment all disputes from October 2024 into quartiles based on the resolution time. What percentage of disputes in the highest resolution time quartile involve physical goods (i.e. product_type values not starting with 'DIG')?

### Expected Output
- percent_physical_disputes

### Solution

```sql
WITH oct_disputes AS (
    SELECT product_type,
           resolution_time_days,
           NTILE(4) OVER (ORDER BY resolution_time_days) AS ntiles
    FROM fct_disputed_transactions
    WHERE dispute_initiated_date >= '2024-10-01' 
      AND dispute_initiated_date < '2024-11-01' 
)
SELECT
    100.0 * SUM(CASE WHEN product_type NOT LIKE 'DIG%' THEN 1 ELSE 0 END)
          / COUNT(*) AS percent_physical_disputes
FROM oct_disputes
WHERE ntiles = 4;
```

**Explanation:**
- CTE uses NTILE(4) to divide disputes into 4 equal-sized groups (quartiles)
- Filters to quartile 4 (ntiles = 4) which is highest resolution times
- Uses conditional aggregation to count physical goods (NOT LIKE 'DIG%')
- Calculates percentage: (physical goods count / total count) x 100
