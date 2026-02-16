# Case 21: Meta Creator Growth Analysis

## Business Context

You are a Data Analyst on the Creator Growth team at Meta, focused on evaluating how different content types influence creator success. Your team aims to determine which content types most effectively drive engagement and follower growth for creators. The ultimate goal is to provide creators with actionable insights to optimize their content strategies for maximum audience expansion.

## Database Schema

### fct_creator_content
- content_id: INTEGER
- creator_id: INTEGER
- published_date: DATE
- content_type: VARCHAR
- impressions_count: INTEGER
- likes_count: INTEGER
- comments_count: INTEGER
- shares_count: INTEGER
- new_followers_count: INTEGER

### dim_creator
- creator_id: INTEGER
- creator_name: VARCHAR
- category: VARCHAR

---

## Question 1: Highest New Follower Growth per Content Type (May 2024)

### Problem
For content published in May 2024, which creator IDs show the highest new follower growth within each content type? If a creator published multiple of the same content type, we want to look at the total new follower growth from that content type.

### Expected Output
- creator_id
- content_type
- followers_growth

### Solution

```sql
WITH may_growth AS (
    SELECT creator_id,
           content_type,
           SUM(new_followers_count) AS followers_growth
    FROM fct_creator_content
    WHERE published_date >= '2024-05-01' 
      AND published_date < '2024-06-01'
    GROUP BY creator_id, content_type
),
ranking AS (
    SELECT creator_id,
           content_type,
           followers_growth,
           RANK() OVER (
               PARTITION BY content_type 
               ORDER BY followers_growth DESC
           ) AS rnk
    FROM may_growth
)
SELECT creator_id,
       content_type,
       followers_growth
FROM ranking
WHERE rnk = 1
ORDER BY creator_id;
```

**Explanation:**
- First CTE aggregates total new followers per creator per content type for May 2024
- Handles creators who published multiple pieces of same content type by summing
- Second CTE applies RANK() partitioned by content_type
- Orders by followers_growth DESC within each content type to rank creators
- RANK() allows ties (multiple creators can share rank 1)
- Main query filters to rnk = 1 to get top performer(s) per content type
- Shows which creators are most successful with each content type

---

## Question 2: Unpivoted Engagement Metrics by Content Type (April 8-21, 2024)

### Problem
Your Product Manager requests a report that shows impressions, likes, comments, and shares for each content type between April 8 and 21, 2024. She specifically requests that engagement metrics are unpivoted into a single 'metric type' column.

### Expected Output
- content_type
- metric_type
- total_metric_value

### Solution

```sql
SELECT
    content_type,
    v.metric_type,
    SUM(v.metric_value) AS total_metric_value
FROM fct_creator_content c
CROSS JOIN LATERAL (
    VALUES
        ('impressions', c.impressions_count),
        ('likes',       c.likes_count),
        ('comments',    c.comments_count),
        ('shares',      c.shares_count)
) AS v(metric_type, metric_value)
WHERE c.published_date >= DATE '2024-04-08'
  AND c.published_date <= DATE '2024-04-21'
GROUP BY content_type, v.metric_type
ORDER BY content_type, v.metric_type;
```

**Explanation:**
- Uses CROSS JOIN LATERAL with VALUES to unpivot columns into rows
- LATERAL allows the VALUES clause to reference columns from the main table (c)
- Each row in the original table generates 4 rows (one per metric type)
- VALUES clause creates inline table with metric_type and metric_value pairs
- Groups by content_type and metric_type to aggregate across all content
- Transforms wide format (multiple metric columns) to long format (single metric column)
- This is PostgreSQL-specific unpivoting technique

---

## Question 3: New Follower Percentage by Content Type per Creator (April-June 2024)

### Problem
For content published between April and June 2024, can you calculate for each creator, what percentage of their new followers came from each content type?

### Expected Output
- creator_name
- content_type
- percentage

### Solution

```sql
WITH base AS (
    SELECT
        f.creator_id,
        d.creator_name,
        content_type,
        new_followers_count
    FROM fct_creator_content f
    JOIN dim_creator d
        ON f.creator_id = d.creator_id
    WHERE published_date >= DATE '2024-04-01'
      AND published_date < DATE '2024-07-01'
),
apr_jun_cont AS (
    SELECT
        creator_id,
        creator_name,
        content_type,
        SUM(new_followers_count) AS new_foll_per_content
    FROM base
    GROUP BY creator_id, creator_name, content_type
),
total_new_foll AS (
    SELECT
        creator_id,
        SUM(new_followers_count) AS total_new_followers
    FROM base
    GROUP BY creator_id
)
SELECT
    c.creator_name,
    c.content_type,
    ROUND(
        c.new_foll_per_content::numeric
        / NULLIF(t.total_new_followers, 0) * 100,
        1
    ) AS percentage
FROM apr_jun_cont c
JOIN total_new_foll t
    ON c.creator_id = t.creator_id
ORDER BY c.creator_name, percentage DESC;
```

**Explanation:**
- First CTE joins fact table with dimension to get creator names and filters to Q2 2024
- Second CTE aggregates new followers per creator per content type
- Third CTE calculates total new followers per creator (all content types combined)
- Main query calculates percentage: (content_type_followers / total_followers) * 100
- Uses ::numeric cast for precise decimal division
- NULLIF prevents division by zero
- ROUND to 1 decimal place for clean output
- Orders by creator name and percentage descending to show top performing content types first
- Shows content type contribution to each creator's growth
