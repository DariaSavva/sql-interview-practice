# Case 01: Amazon Seller Sales Analysis

## Business Context

You are a Data Analyst on the Amazon Marketplace Analytics team focused on optimizing fee structures for third-party sellers. Your goal is to analyze how transaction amounts and fee percentages impact seller performance, with an emphasis on identifying top sales, weekly trends, and cumulative transaction counts. The insights you uncover will guide strategic fee adjustments to incentivize high-performing sellers and enhance overall marketplace efficiency.

## Database Schema

### fct_seller_sales
- sale_id: INTEGER
- seller_id: INTEGER
- sale_amount: DECIMAL
- fee_amount_percentage: DECIMAL
- sale_date: DATE

### dim_seller
- seller_id: INTEGER
- seller_name: VARCHAR

---

## Question 1: Top Sale in April 2024

### Problem
For each seller, identify their top sale transaction in April 2024 based on sale amount. If there are multiple transactions with the same sale amount, select the one with the most recent sale_date.

### Expected Output
- seller_name
- sale_id
- sale_amount
- sale_date

### Solution

```sql
SELECT t.seller_name,
       t.sale_id,
       t.sale_amount,
       t.sale_date
FROM (  
    SELECT s.seller_name,
           fs.sale_id,
           fs.sale_date,
           fs.sale_amount,
           ROW_NUMBER() OVER(PARTITION BY fs.seller_id 
               ORDER BY fs.sale_amount DESC, fs.sale_date DESC) AS "rank"
    FROM fct_seller_sales fs
    JOIN dim_seller s ON fs.seller_id = s.seller_id
    WHERE fs.sale_date >= '2024-04-01' 
      AND fs.sale_date < '2024-05-01'
) t
WHERE t."rank" = 1;
```

**Explanation:**
- Uses ROW_NUMBER() to rank sales within each seller
- Orders by amount DESC then date DESC to handle ties
- Filters to April 2024 using date range
- Quotes around "rank" because it's a reserved keyword

---

## Question 2: Weekly Sales Summary in May 2024

### Problem
Within May 2024, for each seller ID, generate a weekly summary that reports the total number of sales transactions and shows the fee amount from the most recent sale in that week.

### Expected Output
- seller_id
- seller_name
- week_start
- total_sales
- latest_fee_percentage

### Solution

```sql
WITH sales_in_may AS (
    SELECT *
    FROM fct_seller_sales
    WHERE sale_date >= DATE '2024-05-01'
      AND sale_date < DATE '2024-06-01'
),
weekly_sales AS (
    SELECT 
        seller_id,
        DATE_TRUNC('week', sale_date) AS week_start,
        COUNT(*) AS total_sales
    FROM sales_in_may
    GROUP BY seller_id, DATE_TRUNC('week', sale_date)
),
latest_sale_fee AS (
    SELECT 
        seller_id,
        DATE_TRUNC('week', sale_date) AS week_start,
        fee_amount_percentage,
        ROW_NUMBER() OVER (
            PARTITION BY seller_id, DATE_TRUNC('week', sale_date)
            ORDER BY sale_date DESC
        ) AS rn
    FROM sales_in_may
)
SELECT 
    ws.seller_id,
    s.seller_name,
    ws.week_start,
    ws.total_sales,
    lsf.fee_amount_percentage AS latest_fee_percentage
FROM weekly_sales ws
JOIN latest_sale_fee lsf 
    ON ws.seller_id = lsf.seller_id
    AND ws.week_start = lsf.week_start
    AND lsf.rn = 1
JOIN dim_seller s
    ON ws.seller_id = s.seller_id
ORDER BY ws.seller_id, ws.week_start;
```

**Explanation:**
- First CTE filters May 2024 sales
- Second CTE groups by seller and week, counting transactions
- Third CTE finds the most recent fee per seller per week using ROW_NUMBER()
- Final SELECT joins everything together

---

## Question 3: Daily Cumulative Transactions in June 2024

### Problem
Using June 2024, for each seller, create a daily report that computes a cumulative count of transactions up to that day. The report should include ALL days in June 2024, even if a seller had zero transactions on certain days.

### Expected Output
- seller_id
- seller_name
- sale_date
- daily_transaction_count
- cumulative_transaction_count

### Solution

```sql
WITH calendar AS (
    SELECT DATE '2024-06-01' + i AS sale_date
    FROM generate_series(0, 29) AS t(i)
),
sellers AS (
    SELECT seller_id, seller_name
    FROM dim_seller
),
seller_daily_grid AS (
    SELECT s.seller_id, s.seller_name, c.sale_date
    FROM sellers s
    CROSS JOIN calendar c
),
daily_sales AS (
    SELECT 
        seller_id,
        sale_date,
        COUNT(*) AS daily_transaction_count
    FROM fct_seller_sales
    WHERE sale_date >= DATE '2024-06-01' 
      AND sale_date < DATE '2024-07-01'
    GROUP BY seller_id, sale_date
),
joined AS (
    SELECT 
        g.seller_id,
        g.seller_name,
        g.sale_date,
        COALESCE(ds.daily_transaction_count, 0) AS daily_transaction_count
    FROM seller_daily_grid g
    LEFT JOIN daily_sales ds 
        ON g.seller_id = ds.seller_id 
        AND g.sale_date = ds.sale_date
)
SELECT 
    seller_id,
    seller_name,
    sale_date,
    daily_transaction_count,
    SUM(daily_transaction_count) OVER (
        PARTITION BY seller_id 
        ORDER BY sale_date
    ) AS cumulative_transaction_count
FROM joined
ORDER BY seller_id, sale_date;
```

**Explanation:**
- Generates all 30 days of June using generate_series
- Creates a grid of all sellers x all dates using CROSS JOIN
- Counts actual transactions per day
- Uses LEFT JOIN + COALESCE to fill in zeros for days without sales
- Applies window function for cumulative sum
