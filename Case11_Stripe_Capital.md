# Case 11: Stripe Capital Lending Analysis

## Business Context

You are a Data Analyst on the Stripe Capital team, focused on assessing lending patterns for small businesses. Your team is investigating how transaction volumes and growth trends can inform financing eligibility decisions.

## Database Schema

### fct_transactions
- transaction_id: INTEGER
- business_id: INTEGER
- transaction_amount: DECIMAL
- transaction_date: DATE

### dim_businesses
- business_id: INTEGER
- business_name: VARCHAR
- industry: VARCHAR

---

## Question 1: Transaction Volume and Ranking (January 2024)

### Problem
For each business, what is the total transaction amount in January 2024, and how does its ranking compare to other businesses?

### Expected Output
- business_name
- total_transaction_amount
- rn (rank number)

### Solution

```sql
WITH jan_transactions AS (
    SELECT d.business_name,
           SUM(transaction_amount) AS total_transaction_amount
    FROM fct_transactions f
    JOIN dim_businesses d
        ON f.business_id = d.business_id
    WHERE transaction_date >= '2024-01-01' 
      AND transaction_date < '2024-02-01'
    GROUP BY d.business_name
)
SELECT j.business_name,
       j.total_transaction_amount,
       ROW_NUMBER() OVER (ORDER BY j.total_transaction_amount DESC) AS rn
FROM jan_transactions j;
```

**Explanation:**
- CTE aggregates total transaction amounts per business for January 2024
- Main query applies ROW_NUMBER() to rank businesses by transaction volume
- Orders by total_transaction_amount DESC so rank 1 = highest volume

---

## Question 2: Percentage Change from January to February 2024

### Problem
For each business, calculate the percentage change in total transaction amount from January 2024 to February 2024.

### Expected Output
- business_name
- pct_change_jan_to_feb

### Solution

```sql
WITH jan_feb_transactions AS (
    SELECT d.business_name,
           DATE_TRUNC('month', f.transaction_date) AS txn_month,
           SUM(transaction_amount) AS total_transaction_amount
    FROM fct_transactions f
    JOIN dim_businesses d
        ON f.business_id = d.business_id
    WHERE transaction_date >= '2024-01-01' 
      AND transaction_date < '2024-03-01'
    GROUP BY d.business_name, DATE_TRUNC('month', f.transaction_date)
)
SELECT jf.business_name,
    CASE 
        WHEN SUM(CASE WHEN txn_month = DATE '2024-01-01' THEN total_transaction_amount END) = 0 
        THEN NULL
        ELSE
            (SUM(CASE WHEN txn_month = DATE '2024-02-01' THEN total_transaction_amount END)
             -
             SUM(CASE WHEN txn_month = DATE '2024-01-01' THEN total_transaction_amount END))
            / SUM(CASE WHEN txn_month = DATE '2024-01-01' THEN total_transaction_amount END) * 100
    END AS pct_change_jan_to_feb
FROM jan_feb_transactions jf
GROUP BY jf.business_name;
```

**Explanation:**
- First CTE aggregates monthly totals using DATE_TRUNC('month', ...)
- Main query pivots months into columns using conditional aggregation
- Calculates percentage change: ((Feb - Jan) / Jan) x 100
- Handles division by zero with CASE check

---

## Question 3: Average Month-over-Month Growth Ranking (Q1 2024)

### Problem
For each business, compute the month-over-month growth in total transaction amounts from January 2024 through March 2024. Rank each business based on this average growth (from highest to lowest).

### Expected Output
- growth_rank
- business_name
- avg_monthly_growth

### Solution

```sql
WITH monthly_totals AS (
    SELECT
        d.business_name,
        DATE_TRUNC('month', f.transaction_date) AS txn_month,
        SUM(f.transaction_amount) AS total_amount
    FROM fct_transactions f
    JOIN dim_businesses d
        ON f.business_id = d.business_id
    WHERE f.transaction_date >= DATE '2024-01-01'
      AND f.transaction_date < DATE '2024-04-01'
    GROUP BY d.business_name, DATE_TRUNC('month', f.transaction_date)
),
mom_growth AS (
    SELECT
        business_name,
        (total_amount - LAG(total_amount) OVER (
            PARTITION BY business_name ORDER BY txn_month
        ))
        /
        LAG(total_amount) OVER (
            PARTITION BY business_name ORDER BY txn_month
        ) * 100 AS mom_growth_pct
    FROM monthly_totals
)
SELECT
    RANK() OVER (ORDER BY AVG(mom_growth_pct) DESC) AS growth_rank,
    business_name,
    AVG(mom_growth_pct) AS avg_monthly_growth
FROM mom_growth
WHERE mom_growth_pct IS NOT NULL
GROUP BY business_name
ORDER BY growth_rank;
```

**Explanation:**
- First CTE aggregates monthly transaction totals for Q1 2024
- Second CTE calculates month-over-month growth percentage using LAG
- Main query averages the growth percentages per business
- RANK orders businesses by average growth (highest first)
