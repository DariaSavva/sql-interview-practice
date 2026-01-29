# Case 07: Google Ads Performance Analysis

## 📊 Business Context

You are a **Data Analyst on the Google Ads Performance team** working to optimize ad campaign strategies. The goal is to assess the diversity of ad formats, identify high-reach campaigns, and evaluate the return on investment across different campaign segments. Your team will use these insights to make strategic budget allocations and targeting adjustments for future campaigns.

## 🗄️ Database Schema

### `fct_ad_performance`
| Column | Type | Description |
|--------|------|-------------|
| campaign_id | INTEGER | Campaign identifier |
| ad_format | VARCHAR | Type of ad format (video, display, search, etc.) |
| impressions | INTEGER | Number of ad impressions |
| clicks | INTEGER | Number of ad clicks |
| cost | FLOAT | Advertising cost |
| revenue | FLOAT | Revenue generated |
| campaign_date | DATE | Date of campaign performance |

### `dim_campaign`
| Column | Type | Description |
|--------|------|-------------|
| campaign_id | INTEGER | Unique campaign identifier |
| segment | VARCHAR | Campaign segment/category |

---

## Question 1: Unique Ad Formats by Segment (July 2024)

### 📋 Problem
For each **ad campaign segment**, what are the **unique ad formats** used during **July 2024**? 

This will help us understand the diversity in our ad formats.

### Expected Output
- `segment`
- `ad_format`

Ordered by segment

---

### ✅ Solution 1 (Original)

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
- Uses `DISTINCT` to get unique combinations of segment and ad_format
- Joins fact table with dimension table to get segment information
- Filters to July 2024 using date range
- Orders results by segment for clear presentation

**Pros:** Simple and straightforward, returns all unique combinations  
**Cons:** DISTINCT on multiple columns can be expensive on large datasets

---

### 🔄 Alternative Solution: Using GROUP BY

```sql
SELECT d.segment,
       f.ad_format
FROM fct_ad_performance f
JOIN dim_campaign d
    ON f.campaign_id = d.campaign_id
WHERE f.campaign_date >= DATE '2024-07-01'
  AND f.campaign_date < DATE '2024-08-01'
GROUP BY d.segment, f.ad_format
ORDER BY d.segment, f.ad_format;
```

**Pros:** GROUP BY often more efficient than DISTINCT, clearer intent  
**Cons:** Slightly more verbose, functionally equivalent to DISTINCT

---

## Question 2: High-Reach Campaigns with Rolling 7-Day Windows (August 2024)

### 📋 Problem
How many **unique campaigns** had at least one **rolling 7-day period** in August 2024 where their **total impressions exceeded 1,000**? 

We want to identify campaigns that had a high reach in at least one 7-day window during this month.

### Expected Output
- `num_high_reach_campaigns` (count of campaigns)

---

### ✅ Solution 2 (Original)

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
- `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` creates 7-day window (6 previous days + current day)
- Window function partitioned by campaign_id and ordered by date
- Main query counts distinct campaigns that had any 7-day period exceeding 1,000 impressions
- Uses `COUNT(DISTINCT campaign_id)` since a campaign may exceed threshold on multiple days

**Pros:** Elegant use of window functions, handles gaps in data gracefully  
**Cons:** Window function can be expensive on large datasets

---

### 🔄 Alternative Solution: Using Self-Join for 7-Day Windows

```sql
WITH date_range_campaigns AS (
    SELECT DISTINCT 
        a1.campaign_id,
        a1.campaign_date AS window_start_date,
        SUM(a2.impressions) AS total_impressions_7day
    FROM fct_ad_performance a1
    JOIN fct_ad_performance a2
        ON a1.campaign_id = a2.campaign_id
        AND a2.campaign_date BETWEEN a1.campaign_date AND a1.campaign_date + INTERVAL '6 days'
    WHERE a1.campaign_date >= DATE '2024-08-01' 
      AND a1.campaign_date < DATE '2024-09-01'
    GROUP BY a1.campaign_id, a1.campaign_date
    HAVING SUM(a2.impressions) > 1000
)
SELECT COUNT(DISTINCT campaign_id) AS num_high_reach_campaigns
FROM date_range_campaigns;
```

**Pros:** More explicit about 7-day window logic, can be easier to understand  
**Cons:** Self-join can be slower than window functions, more complex

---

## Question 3: Segment ROI Analysis and Comparison (Q3 2024)

### 📋 Problem
What is the **total ROI** for each campaign segment in **Q3 2024**? And, how does it compare to the **average ROI** of all campaigns (return labels 'higher than average' or 'lower than average')? 

We will use this to identify which segments are outperforming the average.

**Note 1**: ROI is defined as `(revenue - cost) / cost`  
**Note 2**: For average ROI across segments, calculate the ROI per segment and then calculate the average ROI across segments.

### Expected Output
- `segment`
- `segment_roi`
- `campaign_performance` ('higher than average' or 'lower than average')

---

### ✅ Solution 3 (Original)

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
- First CTE calculates ROI per segment for Q3 2024 (July-September)
- ROI formula: (total_revenue - total_cost) / total_cost
- Second CTE adds average ROI across all segments using `AVG(roi) OVER ()`
- Calculates difference between segment ROI and average ROI
- Final query uses CASE to label performance as 'higher' or 'lower' than average
- Orders by segment for clear presentation

**Pros:** Clear multi-step logic, easy to understand and debug  
**Cons:** Multiple CTEs add some overhead, but acceptable for clarity

---

### 🔄 Alternative Solution: Single CTE with Subquery

```sql
WITH q3_campaigns AS (
    SELECT
        d.segment,
        (SUM(f.revenue) - SUM(f.cost)) / NULLIF(SUM(f.cost), 0) AS roi
    FROM fct_ad_performance f
    JOIN dim_campaign d
        ON f.campaign_id = d.campaign_id
    WHERE f.campaign_date >= DATE '2024-07-01'
      AND f.campaign_date < DATE '2024-10-01'
    GROUP BY d.segment
)
SELECT
    segment,
    roi AS segment_roi,
    CASE
        WHEN roi > (SELECT AVG(roi) FROM q3_campaigns) THEN 'higher than average'
        ELSE 'lower than average'
    END AS campaign_performance
FROM q3_campaigns
ORDER BY segment;
```

**Pros:** Fewer CTEs, more concise, adds NULLIF for division safety  
**Cons:** Subquery in CASE may execute multiple times (though optimized by most databases)

---

## 📊 Key Takeaways

### Rolling Window Functions with ROWS BETWEEN
```sql
SUM(column) OVER (
    PARTITION BY group_column
    ORDER BY date_column
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)
```
- **ROWS BETWEEN**: Defines physical window of rows
- **6 PRECEDING AND CURRENT ROW**: Creates 7-row window (6 before + current)
- **Use Cases**: Moving averages, rolling sums, streak analysis
- **Alternative**: `RANGE BETWEEN` uses value-based windows instead of row count

### ROI Calculation Best Practices
- **Formula**: `(Revenue - Cost) / Cost`
- **Interpretation**: 1.0 = 100% return (revenue = 2x cost), 0.5 = 50% return
- **Safety**: Use `NULLIF(cost, 0)` to avoid division by zero
- **Aggregation**: Sum before calculating, not average of individual ROIs

### Window Functions vs Aggregations
- **`AVG(roi) OVER ()`**: Computes average across all rows without grouping
- **`AVG(roi)`**: Collapses to single value
- **Pattern**: Use window functions when you need both detail and summary

### DISTINCT vs GROUP BY
- **DISTINCT**: Removes duplicate rows from result set
- **GROUP BY**: Groups rows for aggregation
- **Functionally equivalent** when no aggregation needed
- **Performance**: Usually similar, GROUP BY often slightly better

### Date Range Filtering for Quarters
- **Q1**: Jan-Mar → `>= '2024-01-01' AND < '2024-04-01'`
- **Q2**: Apr-Jun → `>= '2024-04-01' AND < '2024-07-01'`
- **Q3**: Jul-Sep → `>= '2024-07-01' AND < '2024-10-01'`
- **Q4**: Oct-Dec → `>= '2024-10-01' AND < '2025-01-01'`

### Comparative Analysis Pattern
1. Calculate metric per group (segment ROI)
2. Calculate overall average (average ROI)
3. Compare each group to average
4. Label performance (higher/lower than average)

### Performance Considerations
- **Question 1**: DISTINCT or GROUP BY both efficient for small-medium datasets
- **Question 2**: Window functions generally more efficient than self-joins
- **Question 3**: Multiple CTEs add readability with minimal performance cost

### Business Insights
- **Q1**: Identifies which ad formats are used in each segment for diversification analysis
- **Q2**: Finds campaigns with sustained high reach for replication strategies
- **Q3**: Segments performing above/below average ROI for budget reallocation

### Common Pitfalls to Avoid
1. **Division by Zero**: Always use `NULLIF(cost, 0)` in ROI calculations
2. **Rolling Windows**: Remember `ROWS BETWEEN 6 PRECEDING` = 7 days (includes current)
3. **ROI Averaging**: Don't average individual transaction ROIs; sum revenue and cost first
4. **Date Ranges**: Use `< next_month` instead of `<= last_day` to avoid time issues

### SQL Techniques Covered
- DISTINCT for unique combinations
- Window functions with ROWS BETWEEN
- Rolling window calculations
- PARTITION BY for per-group windows
- Multi-CTE queries for complex logic
- ROI and financial metric calculations
- Comparative analysis with CASE statements
- AVG() OVER () for overall averages
- Date range filtering for quarters
- Self-joins as alternative to window functions

### Practical Applications
- **Marketing Analytics**: Measure campaign effectiveness and ROI
- **Budget Allocation**: Identify high-performing segments for investment
- **Performance Monitoring**: Track campaigns with sustained high reach
- **Format Strategy**: Diversify ad formats based on segment usage
- **Competitive Analysis**: Compare segment performance to benchmarks
- **Time-Series Analysis**: Rolling windows for trend detection

### Advanced Concepts
- **Window Frames**: Understanding the difference between ROWS and RANGE
  - `ROWS BETWEEN`: Physical row count
  - `RANGE BETWEEN`: Logical value-based window
- **Multi-Level Aggregation**: Aggregate → Calculate → Compare pattern
- **Performance Labeling**: Transform numeric comparisons into business-friendly labels
