# Case 02: Netflix Marketing Efficiency Analysis

## 📊 Business Context

As a **Data Analyst on the Netflix Marketing Data Team**, you are tasked with analyzing the efficiency of marketing spend in various emerging markets. Your analysis will focus on understanding the allocation of marketing budgets and the resulting subscriber acquisition. The end goal is to provide insights that will guide the team in optimizing marketing strategies and budget distribution across different countries.

## 🗄️ Database Schema

### `fact_marketing_spend`
| Column | Type | Description |
|--------|------|-------------|
| spend_id | INTEGER | Unique spend record identifier |
| country_id | INTEGER | Foreign key to dimension_country |
| campaign_date | DATE | Date of marketing campaign |
| amount_spent | DECIMAL | Amount spent on campaign |

### `fact_daily_subscriptions`
| Column | Type | Description |
|--------|------|-------------|
| subscription_id | INTEGER | Unique subscription record identifier |
| country_id | INTEGER | Foreign key to dimension_country |
| signup_date | DATE | Date of subscriber signup |
| num_new_subscribers | INTEGER | Number of new subscribers on that date |

### `dimension_country`
| Column | Type | Description |
|--------|------|-------------|
| country_id | INTEGER | Unique country identifier |
| country_name | VARCHAR | Name of the country |

---

## Question 1: Total Marketing Spend by Country (Q1 2024)

### 📋 Problem
Retrieve the **total marketing spend** in each country for **Q1 2024** to help inform budget distribution across regions.

### Expected Output
- `country_name`
- `total_marketing_spend`

---

### ✅ Solution 1 (Original)

```sql
SELECT 
    dc.country_name,
    SUM(ms.amount_spent) AS "total marketing spend"
FROM fact_marketing_spend ms
LEFT JOIN dimension_country dc 
    ON ms.country_id = dc.country_id
WHERE EXTRACT(YEAR FROM ms.campaign_date) = 2024
  AND EXTRACT(QUARTER FROM ms.campaign_date) = 1
GROUP BY dc.country_name
ORDER BY dc.country_name;
```

**Explanation:**
- Uses `EXTRACT(QUARTER FROM date)` to filter Q1 (Jan-Mar)
- LEFT JOIN ensures all spend records included even if country info is missing
- Groups by country and sums all spend
- Orders alphabetically by country name

**Pros:** Clear use of EXTRACT for quarter filtering  
**Cons:** EXTRACT can be less efficient than date range filtering

---

### 🔄 Alternative Solution 1A: Using Date Range (More Efficient)

```sql
SELECT 
    dc.country_name,
    SUM(ms.amount_spent) AS total_marketing_spend
FROM fact_marketing_spend ms
LEFT JOIN dimension_country dc 
    ON ms.country_id = dc.country_id
WHERE ms.campaign_date >= DATE '2024-01-01'
  AND ms.campaign_date < DATE '2024-04-01'
GROUP BY dc.country_name
ORDER BY dc.country_name;
```

**Pros:** More efficient, can use indexes better, clearer date boundaries  
**Cons:** Must manually calculate quarter start/end dates

---

### 🔄 Alternative Solution 1B: Using DATE_TRUNC

```sql
SELECT 
    dc.country_name,
    SUM(ms.amount_spent) AS total_marketing_spend
FROM fact_marketing_spend ms
LEFT JOIN dimension_country dc 
    ON ms.country_id = dc.country_id
WHERE DATE_TRUNC('quarter', ms.campaign_date) = DATE '2024-01-01'
GROUP BY dc.country_name
ORDER BY dc.country_name;
```

**Pros:** Clean quarter comparison  
**Cons:** May not use indexes as efficiently as date range

---

### 🔄 Alternative Solution 1C: With Ranking by Spend

```sql
SELECT 
    dc.country_name,
    SUM(ms.amount_spent) AS total_marketing_spend,
    RANK() OVER (ORDER BY SUM(ms.amount_spent) DESC) AS spend_rank
FROM fact_marketing_spend ms
LEFT JOIN dimension_country dc 
    ON ms.country_id = dc.country_id
WHERE ms.campaign_date >= DATE '2024-01-01'
  AND ms.campaign_date < DATE '2024-04-01'
GROUP BY dc.country_name
ORDER BY total_marketing_spend DESC;
```

**Additional Value:** Shows which countries received the most marketing investment

---

## Question 2: New Subscribers by Country (January 2024)

### 📋 Problem
List the **number of new subscribers acquired** in each country (with name) during **January 2024**, renaming the subscriber count column to `'new_subscribers'` for clearer reporting purposes.

### Expected Output
- `country_name`
- `new_subscribers`

---

### ✅ Solution 2 (Original)

```sql
SELECT 
    dc.country_name,
    SUM(ds.num_new_subscribers) AS new_subscribers
FROM fact_daily_subscriptions ds
JOIN dimension_country dc 
    ON ds.country_id = dc.country_id
WHERE ds.signup_date >= '2024-01-01'
  AND ds.signup_date < '2024-02-01'
GROUP BY dc.country_name
ORDER BY new_subscribers DESC;
```

**Explanation:**
- Filters January 2024 using date range (`>= Jan 1` and `< Feb 1`)
- Joins to get country names
- Sums all daily subscriber counts per country
- Orders by subscriber count (descending) to see top performers first

**Pros:** Clean, efficient date filtering, good ordering choice  
**Cons:** None - this is a solid solution!

---

### 🔄 Alternative Solution 2A: With Month Validation

```sql
SELECT 
    dc.country_name,
    SUM(ds.num_new_subscribers) AS new_subscribers
FROM fact_daily_subscriptions ds
JOIN dimension_country dc 
    ON ds.country_id = dc.country_id
WHERE DATE_TRUNC('month', ds.signup_date) = DATE '2024-01-01'
GROUP BY dc.country_name
ORDER BY new_subscribers DESC;
```

**Pros:** Explicit month matching  
**Cons:** Slightly less efficient than date range

---

### 🔄 Alternative Solution 2B: With Growth Percentage Context

```sql
WITH jan_subs AS (
    SELECT 
        dc.country_name,
        SUM(ds.num_new_subscribers) AS new_subscribers
    FROM fact_daily_subscriptions ds
    JOIN dimension_country dc ON ds.country_id = dc.country_id
    WHERE ds.signup_date >= '2024-01-01'
      AND ds.signup_date < '2024-02-01'
    GROUP BY dc.country_name
),
total AS (
    SELECT SUM(new_subscribers) AS total_new_subs
    FROM jan_subs
)
SELECT 
    js.country_name,
    js.new_subscribers,
    ROUND(100.0 * js.new_subscribers / t.total_new_subs, 2) AS percent_of_total
FROM jan_subs js
CROSS JOIN total t
ORDER BY js.new_subscribers DESC;
```

**Additional Value:** Shows each country's contribution to total subscriber growth

---

## Question 3: Average Marketing Spend per Subscriber (Q1 2024)

### 📋 Problem
Determine the **average marketing spend per new subscriber** for each country in Q1 2024 by **rounding up to the nearest whole number** to evaluate campaign efficiency.

### Expected Output
- `country_name`
- `avg_spend_per_subscriber` (rounded up)

---

### ✅ Solution 3 (Original)

```sql
WITH spend AS (
    SELECT 
        country_id,
        SUM(amount_spent) AS total_spend
    FROM fact_marketing_spend
    WHERE campaign_date >= '2024-01-01' 
      AND campaign_date < '2024-04-01'
    GROUP BY country_id
),
subs AS (
    SELECT 
        country_id,
        SUM(num_new_subscribers) AS total_subscribers
    FROM fact_daily_subscriptions
    WHERE signup_date >= '2024-01-01' 
      AND signup_date < '2024-04-01'
    GROUP BY country_id
)
SELECT 
    dc.country_name,
    CEIL(s.total_spend::numeric / NULLIF(sb.total_subscribers, 0)) AS avg_spend_per_subscriber
FROM spend s
JOIN subs sb ON s.country_id = sb.country_id
JOIN dimension_country dc ON s.country_id = dc.country_id
ORDER BY avg_spend_per_subscriber DESC;
```

**Explanation:**
- First CTE aggregates total spend per country in Q1
- Second CTE aggregates total new subscribers per country in Q1
- Main query joins both CTEs and calculates spend per subscriber
- Uses `CEIL()` to round up to nearest whole number
- Uses `NULLIF(sb.total_subscribers, 0)` to avoid division by zero
- Casts to `numeric` for precise decimal division

**Pros:** Safe division with NULLIF, clear CTE separation, proper rounding  
**Cons:** Could be combined into fewer CTEs

---

### 🔄 Alternative Solution 3A: Single CTE Approach

```sql
WITH q1_data AS (
    SELECT 
        ms.country_id,
        SUM(ms.amount_spent) AS total_spend,
        SUM(ds.num_new_subscribers) AS total_subscribers
    FROM fact_marketing_spend ms
    FULL OUTER JOIN fact_daily_subscriptions ds
        ON ms.country_id = ds.country_id
        AND ms.campaign_date = ds.signup_date
    WHERE (ms.campaign_date >= '2024-01-01' AND ms.campaign_date < '2024-04-01')
       OR (ds.signup_date >= '2024-01-01' AND ds.signup_date < '2024-04-01')
    GROUP BY ms.country_id, ds.country_id
)
SELECT 
    dc.country_name,
    CEIL(COALESCE(qd.total_spend, 0)::numeric / 
         NULLIF(COALESCE(qd.total_subscribers, 0), 0)) AS avg_spend_per_subscriber
FROM q1_data qd
JOIN dimension_country dc ON COALESCE(qd.country_id, qd.country_id) = dc.country_id
ORDER BY avg_spend_per_subscriber DESC;
```

**Note:** This is more complex and the original approach is actually better for this use case.

---

### 🔄 Alternative Solution 3B: Improved Version with Better Error Handling

```sql
WITH spend AS (
    SELECT 
        country_id,
        SUM(amount_spent) AS total_spend
    FROM fact_marketing_spend
    WHERE campaign_date >= DATE '2024-01-01' 
      AND campaign_date < DATE '2024-04-01'
    GROUP BY country_id
),
subs AS (
    SELECT 
        country_id,
        SUM(num_new_subscribers) AS total_subscribers
    FROM fact_daily_subscriptions
    WHERE signup_date >= DATE '2024-01-01' 
      AND signup_date < DATE '2024-04-01'
    GROUP BY country_id
)
SELECT 
    dc.country_name,
    s.total_spend,
    sb.total_subscribers,
    CASE 
        WHEN sb.total_subscribers > 0 
        THEN CEIL(s.total_spend / sb.total_subscribers)
        ELSE NULL 
    END AS avg_spend_per_subscriber
FROM spend s
JOIN subs sb ON s.country_id = sb.country_id
JOIN dimension_country dc ON s.country_id = dc.country_id
ORDER BY avg_spend_per_subscriber DESC NULLS LAST;
```

**Pros:** Shows intermediate values for debugging, explicit NULL handling  
**Cons:** More verbose

---

### 🔄 Alternative Solution 3C: With Efficiency Rating

```sql
WITH spend AS (
    SELECT 
        country_id,
        SUM(amount_spent) AS total_spend
    FROM fact_marketing_spend
    WHERE campaign_date >= DATE '2024-01-01' 
      AND campaign_date < DATE '2024-04-01'
    GROUP BY country_id
),
subs AS (
    SELECT 
        country_id,
        SUM(num_new_subscribers) AS total_subscribers
    FROM fact_daily_subscriptions
    WHERE signup_date >= DATE '2024-01-01' 
      AND signup_date < DATE '2024-04-01'
    GROUP BY country_id
),
cost_per_sub AS (
    SELECT 
        dc.country_name,
        s.total_spend,
        sb.total_subscribers,
        CEIL(s.total_spend::numeric / NULLIF(sb.total_subscribers, 0)) AS avg_spend_per_subscriber
    FROM spend s
    JOIN subs sb ON s.country_id = sb.country_id
    JOIN dimension_country dc ON s.country_id = dc.country_id
)
SELECT 
    country_name,
    total_spend,
    total_subscribers,
    avg_spend_per_subscriber,
    CASE 
        WHEN avg_spend_per_subscriber <= 50 THEN 'Highly Efficient'
        WHEN avg_spend_per_subscriber <= 100 THEN 'Efficient'
        WHEN avg_spend_per_subscriber <= 150 THEN 'Moderate'
        ELSE 'Needs Improvement'
    END AS efficiency_rating
FROM cost_per_sub
ORDER BY avg_spend_per_subscriber;
```

**Additional Value:** Provides actionable efficiency ratings for strategic decisions

---

## 📊 Key Takeaways

### Date Filtering
- **Date ranges** (`>= '2024-01-01' AND < '2024-02-01'`) are more efficient than EXTRACT
- Can use indexes effectively
- Clearer boundaries (avoids timezone issues)

### Aggregation Best Practices
- Use `SUM()` for totals across multiple records
- `COUNT()` vs `SUM()`: COUNT counts records, SUM adds values
- Always consider what happens with NULL values

### Division Safety
- **Always use `NULLIF()`** to prevent division by zero
- Cast to `numeric` or `decimal` for precise calculations
- `CEIL()` rounds up, `FLOOR()` rounds down, `ROUND()` goes to nearest

### Join Strategy
- Use **INNER JOIN** when you need matching records from both tables
- Use **LEFT JOIN** when you want all records from left table even without matches
- In Q3, INNER JOIN is correct since we only want countries with both spend AND subscribers

### Performance Tips
1. **Question 1**: Date range filtering > EXTRACT for performance
2. **Question 2**: Straightforward aggregation - already optimal
3. **Question 3**: Original solution is excellent - clear and efficient

### Business Insights
- **Q1**: Identifies budget allocation across markets
- **Q2**: Shows subscriber growth by region
- **Q3**: Reveals marketing efficiency - lower cost per subscriber = better ROI

### Code Quality
- Use meaningful aliases (`total_spend` not just `ts`)
- Add comments for complex calculations
- Consider NULL handling in every calculation
- Order results meaningfully (DESC for rankings, ASC for names)
