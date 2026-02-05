# Case 02: Netflix Marketing Efficiency Analysis

## Business Context

As a Data Analyst on the Netflix Marketing Data Team, you are tasked with analyzing the efficiency of marketing spend in various emerging markets. Your analysis will focus on understanding the allocation of marketing budgets and the resulting subscriber acquisition. The end goal is to provide insights that will guide the team in optimizing marketing strategies and budget distribution across different countries.

## Database Schema

### fact_marketing_spend
- spend_id: INTEGER
- country_id: INTEGER
- campaign_date: DATE
- amount_spent: DECIMAL

### fact_daily_subscriptions
- subscription_id: INTEGER
- country_id: INTEGER
- signup_date: DATE
- num_new_subscribers: INTEGER

### dimension_country
- country_id: INTEGER
- country_name: VARCHAR

---

## Question 1: Total Marketing Spend by Country (Q1 2024)

### Problem
Retrieve the total marketing spend in each country for Q1 2024 to help inform budget distribution across regions.

### Expected Output
- country_name
- total_marketing_spend

### Solution

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
- Uses EXTRACT(QUARTER FROM date) to filter Q1 (Jan-Mar)
- LEFT JOIN ensures all spend records included even if country info is missing
- Groups by country and sums all spend
- Orders alphabetically by country name

---

## Question 2: New Subscribers by Country (January 2024)

### Problem
List the number of new subscribers acquired in each country (with name) during January 2024, renaming the subscriber count column to 'new_subscribers' for clearer reporting purposes.

### Expected Output
- country_name
- new_subscribers

### Solution

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
- Filters January 2024 using date range
- Joins to get country names
- Sums all daily subscriber counts per country
- Orders by subscriber count descending to see top performers first

---

## Question 3: Average Marketing Spend per Subscriber (Q1 2024)

### Problem
Determine the average marketing spend per new subscriber for each country in Q1 2024 by rounding up to the nearest whole number to evaluate campaign efficiency.

Note: ROI is defined as (revenue - cost) / cost.

### Expected Output
- country_name
- avg_spend_per_subscriber

### Solution

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
- Uses CEIL() to round up to nearest whole number
- Uses NULLIF(sb.total_subscribers, 0) to avoid division by zero
- Casts to numeric for precise decimal division
