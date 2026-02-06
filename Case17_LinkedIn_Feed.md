# Case 17: LinkedIn Feed Engagement Analysis

## Business Context

As a Product Analyst on the LinkedIn Feed team, you are tasked with analyzing how different content types impact user engagement on the platform. Your team is focused on understanding which types of content consistently drive higher engagement scores. The ultimate goal is to leverage these insights to enhance content recommendation strategies and improve user experience on the feed.

## Database Schema

### fct_user_engagement
- engagement_id: INTEGER
- content_id: INTEGER
- user_id: INTEGER
- engagement_score: FLOAT
- engagement_date: DATE

### dim_content
- content_id: INTEGER
- content_type: VARCHAR
- publish_date: DATE

---

## Question 1: Best Performing Content Formats (July 2024)

### Problem
Identify which content formats performed best in July 2024 by reporting their average engagement scores, but only list the formats that achieved an average score of 50 or higher.

### Expected Output
- content_type
- avg (average engagement score)

### Solution

```sql
SELECT content_type,
       AVG(engagement_score)
FROM fct_user_engagement f
JOIN dim_content d
    ON f.content_id = d.content_id
WHERE engagement_date >= '2024-07-01' 
  AND engagement_date < '2024-08-01'
GROUP BY content_type
HAVING AVG(engagement_score) >= 50
ORDER BY AVG(engagement_score) DESC;
```

**Explanation:**
- Joins engagement fact table with content dimension to get content types
- Filters to July 2024 using date range
- Groups by content_type to calculate averages per format
- HAVING clause filters groups to only those with average score >= 50
- HAVING is used instead of WHERE because it filters after aggregation
- Orders by average engagement score descending to show best performers first

---

## Question 2: Top Content Type Analysis (August 2024)

### Problem
For the first week of August 2024, identify the content type of the content that had the highest engagement score. For that content type, calculate the average engagement score for the entire month of August. If there is a tie for highest engagement score, select the content type with the earliest publish date.

### Expected Output
- content_type
- avg_august_engagement_score

### Solution

```sql
WITH first_week_aug AS (
    SELECT
        d.content_type,
        d.publish_date,
        f.engagement_score,
        RANK() OVER (
            ORDER BY f.engagement_score DESC, d.publish_date ASC
        ) AS rn
    FROM fct_user_engagement f
    JOIN dim_content d
        ON f.content_id = d.content_id
    WHERE f.engagement_date >= DATE '2024-08-01'
      AND f.engagement_date < DATE '2024-08-08'
), 
top_content_type AS (
    SELECT content_type
    FROM first_week_aug
    WHERE rn = 1
)
SELECT
    d.content_type,
    AVG(f.engagement_score) AS avg_august_engagement_score
FROM fct_user_engagement f
JOIN dim_content d
    ON f.content_id = d.content_id
JOIN top_content_type t
    ON d.content_type = t.content_type
WHERE f.engagement_date >= DATE '2024-08-01'
  AND f.engagement_date < DATE '2024-09-01'
GROUP BY d.content_type;
```

**Explanation:**
- First CTE gets engagement data for first week of August (Aug 1-7)
- Uses RANK() with multi-criteria ordering: engagement_score DESC, publish_date ASC
- This handles ties by selecting content with earliest publish date
- Second CTE extracts the winning content_type (rank = 1)
- Main query calculates average engagement for that content type across all of August
- Joins with top_content_type to filter to only the winning content type

---

## Question 3: Average Weekly Peak Engagement (Q3 2024)

### Problem
During Q3 2024, for each content type, calculate their highest engagement score each week. What was the average of those weekly highs over the quarter? This will help us identify content types that consistently generate peak engagement.

### Expected Output
- content_type
- avg_weekly_peak_engagement

### Solution

```sql
WITH weekly_highs AS (
    SELECT
        d.content_type,
        DATE_TRUNC('week', f.engagement_date) AS week_start,
        MAX(f.engagement_score) AS weekly_max_engagement
    FROM fct_user_engagement f
    JOIN dim_content d
        ON f.content_id = d.content_id
    WHERE f.engagement_date >= DATE '2024-07-01'
      AND f.engagement_date < DATE '2024-10-01'
    GROUP BY
        d.content_type,
        DATE_TRUNC('week', f.engagement_date)
)
SELECT
    content_type,
    AVG(weekly_max_engagement) AS avg_weekly_peak_engagement
FROM weekly_highs
GROUP BY content_type
ORDER BY avg_weekly_peak_engagement DESC;
```

**Explanation:**
- CTE groups engagement data by content_type and week
- Uses DATE_TRUNC('week', engagement_date) to group dates into weeks
- Filters to Q3 2024 (July, August, September)
- MAX(engagement_score) finds the peak engagement for each content type each week
- Main query averages those weekly peaks per content type
- This shows which content types consistently achieve high peaks
- Orders by average weekly peak descending to see top performers
