# Case 07: Google Ads Performance Analysis

## Business Context

You are a Data Analyst on the Google Ads Performance team working to optimize ad campaign strategies. The goal is to assess the diversity of ad formats, identify high-reach campaigns, and evaluate the return on investment across different campaign segments.

## Database Schema

### fct_ad_performance
- campaign_id: INTEGER
- ad_format: VARCHAR
- impressions: INTEGER
- clicks: INTEGER
- cost: FLOAT
- revenue: FLOAT
- campaign_date: DATE

### dim_campaign
- campaign_id: INTEGER
- segment: VARCHAR

---

## Question 1: Unique Ad Formats by Segment (July 2024)

### Problem
For each ad campaign segment, what are the unique ad formats used during July 2024?

### Expected Output
- segment
- ad_format

### Solution

```sql
SELECT DISTINCT segment,
                ad_format
FROM fct_ad_performance f
JOIN dim_campaign d
    ON f.campaign_id = d.campaign_id
WHERE f.campaign_date >= '2024-07-01'
  AND f.campaign_date < '2024-08-01'
ORDER BY segment;
```

**Explanation:**
- Uses DISTINCT to get unique combinations of segment and ad_format
- Joins fact table with dimension table to get segment information
- Filters to July 2024 using date range

---

## Question 2: High-Reach Campaigns with Rolling 7-Day Windows (August 2024)

### Problem
How many unique campaigns had at least one rolling 7-day period in August 2024 where their total impressions exceeded 1,000?

### Expected Output
- num_high_reach_campaigns

### Solution

```sql
WITH aug_campaigns AS ( 
    SELECT campaign_id,
           campaign_date,
           SUM(impressions) OVER (
               PARTITION BY campaign_id 
               ORDER BY campaign_date
               ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
           ) AS "rolling_7day"
    FROM fct_ad_performance
    WHERE campaign_date >= '2024-08-01' 
      AND campaign_date < '2024-09-01'
)
SELECT COUNT(DISTINCT campaign_id) AS num_high_reach_campaigns
FROM aug_campaigns c
WHERE c."rolling_7day" > 1000;
```

**Explanation:**
- CTE creates rolling 7-day impression sums using window function
- ROWS BETWEEN 6 PRECEDING AND CURRENT ROW creates 7-day window
- Main query counts distinct campaigns that exceeded 1,000 impressions

---

## Question 3: Segment ROI Analysis and Comparison (Q3 2024)

### Problem
What is the total ROI for each campaign segment in Q3 2024? And, how does it compare to the average ROI of all campaigns?

Note 1: ROI is defined as (revenue - cost) / cost.
Note 2: For average ROI across segment, calculate the ROI per segment and then calculate the average ROI across segments.

### Expected Output
- segment
- segment_roi
- campaign_performance ('higher than average' or 'lower than average')

### Solution

```sql
WITH q3_campaigns AS (
    SELECT
        d.segment,
        SUM(f.revenue) AS total_revenue,
        SUM(f.cost) AS total_cost,
        (SUM(f.revenue) - SUM(f.cost)) / SUM(f.cost) AS roi
    FROM fct_ad_performance f
    JOIN dim_campaign d
        ON f.campaign_id = d.campaign_id
    WHERE f.campaign_date >= DATE '2024-07-01'
      AND f.campaign_date < DATE '2024-10-01'
    GROUP BY d.segment
),
avg_roi AS (
    SELECT
        segment,
        roi,
        AVG(roi) OVER () AS avg_roi,
        roi - AVG(roi) OVER () AS roi_vs_avg_roi
    FROM q3_campaigns
)
SELECT
    segment,
    roi AS segment_roi,
    CASE
        WHEN roi_vs_avg_roi > 0 THEN 'higher than average'
        ELSE 'lower than average'
    END AS campaign_performance
FROM avg_roi
ORDER BY segment;
```

**Explanation:**
- First CTE calculates ROI per segment for Q3 2024
- ROI formula: (total_revenue - total_cost) / total_cost
- Second CTE adds average ROI across all segments using AVG(roi) OVER ()
- Final query uses CASE to label performance as higher or lower than average
