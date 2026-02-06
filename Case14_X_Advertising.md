# Case 14: X (Twitter) Advertising Campaigns Analysis

## Business Context

You are a Data Analyst working with the X for Advertisers team to evaluate the effectiveness of advertising campaigns across various audience segments. Your team aims to understand how different audience groups are engaging with the campaigns to optimize targeting strategies. By analyzing engagement metrics, you will identify trends and growth rates to enhance campaign performance.

## Database Schema

### fct_campaign_engagement
- engagement_id: INTEGER
- audience_segment: VARCHAR
- engagement_date: DATE
- engagement_type: VARCHAR
- engagement_count: INTEGER

Note: The audience_segment column contains inconsistent capitalization that needs cleaning.

---

## Question 1: Monthly Engagement Count by Audience Segment (April-May 2024)

### Problem
What is the monthly total engagement count for each audience segment in April 2024 and May 2024?

Hint: There is some messy data in the audience segment column, so you might want to clean that up first.

### Expected Output
- aud_segment
- apr_count
- may_count

### Solution

```sql
SELECT UPPER(audience_segment) AS aud_segment,
       SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-04-01'
            THEN engagement_count END) AS apr_count,
       SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-05-01'
            THEN engagement_count END) AS may_count
FROM fct_campaign_engagement
WHERE engagement_date >= '2024-04-01' 
  AND engagement_date < '2024-06-01'
GROUP BY UPPER(audience_segment);
```

**Explanation:**
- Uses UPPER() to standardize audience_segment capitalization
- DATE_TRUNC('month', engagement_date) groups dates to first day of month
- Conditional aggregation with CASE to pivot months into columns
- SUM with CASE WHEN creates separate columns for April and May
- Filters to April-May 2024 date range
- Groups by cleaned audience segment

---

## Question 2: Monthly Engagement Growth Rate (April to May 2024)

### Problem
What is the monthly engagement growth rate from April 2024 to May 2024? This calculation will help the X for Advertisers team understand basic engagement performance trends across audience groups for campaign optimization.

### Expected Output
- aud_segment
- growth_rate_pct

### Solution

```sql
WITH apr_may_cnt AS (
    SELECT UPPER(audience_segment) AS aud_segment,
           SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-04-01'
                THEN engagement_count END) AS apr_count,
           SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-05-01'
                THEN engagement_count END) AS may_count
    FROM fct_campaign_engagement
    WHERE engagement_date >= '2024-04-01' 
      AND engagement_date < '2024-06-01'
    GROUP BY UPPER(audience_segment) 
)
SELECT aud_segment,
       ((may_count - apr_count)::NUMERIC / NULLIF(apr_count, 0)) * 100 AS growth_rate_pct
FROM apr_may_cnt;
```

**Explanation:**
- CTE calculates April and May engagement counts per segment
- Main query calculates growth rate: ((May - April) / April) * 100
- Uses ::NUMERIC to cast to numeric type for precise decimal division
- NULLIF(apr_count, 0) prevents division by zero
- Returns percentage growth for each audience segment

---

## Question 3: Highest and Lowest Growth Rate Segments (April to May 2024)

### Problem
Which audience segments had the highest engagement growth rate from April 2024 to May 2024, and which had the lowest? Don't forget to clean up the inconsistent capitalization in the audience segment column.

Identifying these trends will help the X for Advertisers team optimize targeting strategies by focusing on the most responsive audience segments.

### Expected Output
- aud_segment
- growth_rate_pct
- growth_category ('highest growth' or 'lowest growth')

### Solution

```sql
WITH apr_may_growth AS (
    SELECT
        UPPER(audience_segment) AS aud_segment,
        (
            (SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-05-01'
                      THEN engagement_count ELSE 0 END)
           -
             SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-04-01'
                      THEN engagement_count ELSE 0 END)
            )::NUMERIC
            /
            NULLIF(
                SUM(CASE WHEN DATE_TRUNC('month', engagement_date) = DATE '2024-04-01'
                         THEN engagement_count ELSE 0 END),
                0
            )
        ) * 100 AS growth_rate_pct
    FROM fct_campaign_engagement
    WHERE engagement_date >= DATE '2024-04-01'
      AND engagement_date < DATE '2024-06-01'
    GROUP BY UPPER(audience_segment)
)
SELECT
    aud_segment,
    growth_rate_pct,
    CASE
        WHEN rn_desc = 1 THEN 'highest growth'
        WHEN rn_asc  = 1 THEN 'lowest growth'
    END AS growth_category
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY growth_rate_pct DESC) AS rn_desc,
           ROW_NUMBER() OVER (ORDER BY growth_rate_pct ASC)  AS rn_asc
    FROM apr_may_growth
) t
WHERE rn_desc = 1
   OR rn_asc = 1;
```

**Explanation:**
- First CTE calculates growth rate for each audience segment
- Uses UPPER() to clean inconsistent capitalization
- Calculates May count minus April count, divided by April count, times 100
- Subquery applies two ROW_NUMBER() window functions:
  - rn_desc: ranks by growth_rate_pct DESC (highest first)
  - rn_asc: ranks by growth_rate_pct ASC (lowest first)
- Main query filters to rows where rn_desc = 1 (highest) OR rn_asc = 1 (lowest)
- CASE statement labels each as 'highest growth' or 'lowest growth'
- Returns both extremes in a single result set
