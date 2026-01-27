# Case 01: Amazon Seller Sales Analysis

## 📊 Business Context

You are a Data Analyst on the **Amazon Marketplace Analytics team** focused on optimizing fee structures for third-party sellers. Your goal is to analyze how transaction amounts and fee percentages impact seller performance, with an emphasis on identifying top sales, weekly trends, and cumulative transaction counts. The insights you uncover will guide strategic fee adjustments to incentivize high-performing sellers and enhance overall marketplace efficiency.

## 🗄️ Database Schema

### `fct_seller_sales`
| Column | Type | Description |
|--------|------|-------------|
| sale_id | INTEGER | Unique sale identifier |
| seller_id | INTEGER | Foreign key to dim_seller |
| sale_amount | DECIMAL | Transaction amount |
| fee_amount_percentage | DECIMAL | Fee percentage charged |
| sale_date | DATE | Date of sale |

### `dim_seller`
| Column | Type | Description |
|--------|------|-------------|
| seller_id | INTEGER | Unique seller identifier |
| seller_name | VARCHAR | Name of the seller |

---

## Question 1: Top Sale in April 2024

### 📋 Problem
For each seller, identify their **top sale transaction** in April 2024 based on sale amount. If there are multiple transactions with the same sale amount, select the one with the **most recent** sale_date.

### Expected Output
- `seller_name`
- `sale_id`
- `sale_amount`
- `sale_date`

---

### ✅ Solution 1 (Original)

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
- Uses `ROW_NUMBER()` to rank sales within each seller
- Orders by amount (DESC) then date (DESC) to handle ties
- Filters to April 2024 using date range
- Quotes around "rank" because it's a reserved keyword

**Pros:** Clear, readable, handles ties correctly, portable across SQL dialects  
**Cons:** Requires subquery/CTE

---

### 🔄 Alternative Solution 1A: DISTINCT ON (PostgreSQL-specific)

```sql
SELECT DISTINCT ON (s.seller_id)
    s.seller_name,
    fs.sale_id,
    fs.sale_amount,
    fs.sale_date
FROM fct_seller_sales fs
JOIN dim_seller s ON fs.seller_id = s.seller_id
WHERE fs.sale_date >= '2024-04-01' 
  AND fs.sale_date < '2024-05-01'
ORDER BY s.seller_id, 
         fs.sale_amount DESC, 
         fs.sale_date DESC;
```

**Pros:** More concise, PostgreSQL-optimized, no subquery needed  
**Cons:** PostgreSQL-specific syntax, less portable

---

### 🔄 Alternative Solution 1B: Using RANK()

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
           RANK() OVER(PARTITION BY fs.seller_id 
               ORDER BY fs.sale_amount DESC, fs.sale_date DESC) AS rank
    FROM fct_seller_sales fs
    JOIN dim_seller s ON fs.seller_id = s.seller_id
    WHERE fs.sale_date >= '2024-04-01' 
      AND fs.sale_date < '2024-05-01'
) t
WHERE t.rank = 1;
```

**Note:** `RANK()` would include all tied records if there were multiple transactions with the same amount AND date. `ROW_NUMBER()` is better here as it guarantees exactly one row per seller.

---

## Question 2: Weekly Sales Summary in May 2024

### 📋 Problem
Within May 2024, for each seller ID, generate a **weekly summary** that reports:
- Total number of sales transactions
- Fee amount from the **most recent sale** in that week

This analysis will help correlate fee changes with weekly seller performance trends.

### Expected Output
- `seller_id`
- `seller_name`
- `week_start` (start of the week)
- `total_sales` (count of transactions)
- `latest_fee_percentage` (fee from most recent sale in that week)

---

### ✅ Solution 2 (Original)

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

**Pros:** Very clear separation of concerns, easy to debug, readable  
**Cons:** Multiple CTEs may be slightly less efficient

---

### 🔄 Alternative Solution 2A: DISTINCT ON Approach

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
    SELECT DISTINCT ON (seller_id, DATE_TRUNC('week', sale_date))
        seller_id,
        DATE_TRUNC('week', sale_date) AS week_start,
        fee_amount_percentage
    FROM sales_in_may
    ORDER BY seller_id, DATE_TRUNC('week', sale_date), sale_date DESC
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
JOIN dim_seller s
    ON ws.seller_id = s.seller_id
ORDER BY ws.seller_id, ws.week_start;
```

**Pros:** More concise, potentially better performance  
**Cons:** PostgreSQL-specific

---

### 🔄 Alternative Solution 2B: Single CTE with Window Functions

```sql
WITH weekly_data AS (
    SELECT 
        fs.seller_id,
        s.seller_name,
        DATE_TRUNC('week', fs.sale_date) AS week_start,
        fs.sale_date,
        fs.fee_amount_percentage,
        COUNT(*) OVER (
            PARTITION BY fs.seller_id, DATE_TRUNC('week', fs.sale_date)
        ) AS total_sales,
        ROW_NUMBER() OVER (
            PARTITION BY fs.seller_id, DATE_TRUNC('week', fs.sale_date)
            ORDER BY fs.sale_date DESC
        ) AS rn
    FROM fct_seller_sales fs
    JOIN dim_seller s ON fs.seller_id = s.seller_id
    WHERE fs.sale_date >= DATE '2024-05-01'
      AND fs.sale_date < DATE '2024-06-01'
)
SELECT 
    seller_id,
    seller_name,
    week_start,
    total_sales,
    fee_amount_percentage AS latest_fee_percentage
FROM weekly_data
WHERE rn = 1
ORDER BY seller_id, week_start;
```

**Pros:** Single pass through data, potentially fastest  
**Cons:** Slightly more complex logic, window functions computed for all rows

---

## Question 3: Daily Cumulative Transactions in June 2024

### 📋 Problem
Using June 2024, for each seller, create a **daily report** that computes a **cumulative count** of transactions up to that day.

**Important:** The report should include **ALL days in June 2024**, even if a seller had zero transactions on certain days.

### Expected Output
- `seller_id`
- `seller_name`
- `sale_date`
- `daily_transaction_count`
- `cumulative_transaction_count`

---

### ✅ Solution 3 (Original)

```sql
-- Step 1: Generate a calendar for June 2024
WITH calendar AS (
    SELECT DATE '2024-06-01' + i AS sale_date
    FROM generate_series(0, 29) AS t(i)
),
-- Step 2: Get all sellers
sellers AS (
    SELECT seller_id, seller_name
    FROM dim_seller
),
-- Step 3: Create all combinations of sellers and dates
seller_daily_grid AS (
    SELECT s.seller_id, s.seller_name, c.sale_date
    FROM sellers s
    CROSS JOIN calendar c
),
-- Step 4: Count transactions per seller per day
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
-- Step 5: Combine all days per seller with actual sales (if any)
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
-- Step 6: Compute cumulative sum
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
- Generates all 30 days of June using `generate_series`
- Creates a grid of all sellers × all dates using CROSS JOIN
- Counts actual transactions per day
- Uses LEFT JOIN + COALESCE to fill in zeros for days without sales
- Applies window function for cumulative sum

**Pros:** Very clear, handles missing dates explicitly, easy to understand, well-documented  
**Cons:** Multiple CTEs, more verbose

---

### 🔄 Alternative Solution 3A: Condensed CTE Version

```sql
WITH calendar AS (
    SELECT generate_series(
        DATE '2024-06-01', 
        DATE '2024-06-30', 
        '1 day'::interval
    )::date AS sale_date
),
seller_daily_grid AS (
    SELECT 
        s.seller_id,
        s.seller_name,
        c.sale_date,
        COUNT(fs.sale_id) AS daily_transaction_count
    FROM dim_seller s
    CROSS JOIN calendar c
    LEFT JOIN fct_seller_sales fs 
        ON s.seller_id = fs.seller_id 
        AND fs.sale_date = c.sale_date
    GROUP BY s.seller_id, s.seller_name, c.sale_date
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
FROM seller_daily_grid
ORDER BY seller_id, sale_date;
```

**Pros:** More concise, fewer CTEs, better performance  
**Cons:** Slightly less readable

---

### 🔄 Alternative Solution 3B: Most Concise (Single Query)

```sql
SELECT 
    s.seller_id,
    s.seller_name,
    d.sale_date,
    COUNT(fs.sale_id) AS daily_transaction_count,
    SUM(COUNT(fs.sale_id)) OVER (
        PARTITION BY s.seller_id 
        ORDER BY d.sale_date
    ) AS cumulative_transaction_count
FROM dim_seller s
CROSS JOIN (
    SELECT generate_series(
        DATE '2024-06-01', 
        DATE '2024-06-30', 
        '1 day'::interval
    )::date AS sale_date
) d
LEFT JOIN fct_seller_sales fs 
    ON s.seller_id = fs.seller_id 
    AND fs.sale_date = d.sale_date
GROUP BY s.seller_id, s.seller_name, d.sale_date
ORDER BY s.seller_id, d.sale_date;
```

**Pros:** Most concise, single pass, best performance  
**Cons:** Inline subquery may be less readable for beginners

---

## 📊 Key Takeaways

### Window Functions
- `ROW_NUMBER()` vs `RANK()`: Use ROW_NUMBER when you need exactly one result per partition
- `SUM() OVER()`: Great for cumulative calculations

### Date Handling
- `DATE_TRUNC('week', date)`: Groups dates by week
- `generate_series()`: Powerful for creating date ranges
- Always use `>=` and `<` for date ranges to avoid timezone issues

### Performance
- **Question 1**: DISTINCT ON is fastest in PostgreSQL
- **Question 2**: Single CTE approach (2B) offers best balance
- **Question 3**: Condensed version (3B) is most efficient

### Code Style
- Use CTEs for complex logic to improve readability
- Add comments for multi-step processes
- Balance performance with maintainability
