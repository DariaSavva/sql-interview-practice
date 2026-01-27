# Case 04: Netflix Mobile Experience Analysis

## 📊 Business Context

As a **Data Analyst on the Netflix Mobile Experience team**, you are investigating how users resume watching shows on mobile devices to enhance the seamless viewing experience. Your goal is to analyze user engagement through resume events, understand the distribution of these events per user, and identify the most popular shows for resumption on different mobile platforms. This analysis will help product managers optimize the app's resume functionality and reduce friction in content consumption.

## 🗄️ Database Schema

### `fct_user_resumptions`
| Column | Type | Description |
|--------|------|-------------|
| user_id | INTEGER | Unique user identifier |
| device_type | VARCHAR | Type of device (iOS, Android, Desktop) |
| show_title | VARCHAR | Title of the show being resumed |
| resumption_timestamp | TIMESTAMP | Date and time of resume event |

**Device Types**: `iOS`, `Android`, `Desktop`

---

## Question 1: Mobile Resume Events (Week of Oct 1-7, 2024)

### 📋 Problem
For the week from **October 1st to October 7th, 2024**, how many total resume events occurred on **mobile devices**? 

This analysis will help the Netflix Mobile Experience team evaluate overall user engagement during content resumption.

### Expected Output
- `number_events` (total count of mobile resume events)

---

### ✅ Solution 1 (Original)

```sql
SELECT COUNT(*) AS number_events
FROM fct_user_resumptions
WHERE DATE(resumption_timestamp) BETWEEN '2024-10-01' AND '2024-10-07'
  AND device_type IN ('iOS', 'Android');
```

**Explanation:**
- Uses `DATE()` to extract date from timestamp
- `BETWEEN` is inclusive on both ends (Oct 1-7)
- Filters to mobile devices only using `IN ('iOS', 'Android')`
- Counts all matching rows

**Pros:** Clear and readable, straightforward logic  
**Cons:** `DATE()` function may not use timestamp indexes efficiently

---

### 🔄 Alternative Solution 1A: Using Timestamp Range (More Efficient)

```sql
SELECT COUNT(*) AS number_events
FROM fct_user_resumptions
WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
  AND resumption_timestamp < TIMESTAMP '2024-10-08 00:00:00'
  AND device_type IN ('iOS', 'Android');
```

**Pros:** Better for index usage, precise timestamp boundaries  
**Cons:** Slightly more verbose

---

### 🔄 Alternative Solution 1B: Using DATE_TRUNC

```sql
SELECT COUNT(*) AS number_events
FROM fct_user_resumptions
WHERE DATE_TRUNC('day', resumption_timestamp) BETWEEN DATE '2024-10-01' AND DATE '2024-10-07'
  AND device_type IN ('iOS', 'Android');
```

**Pros:** Clean date comparison  
**Cons:** May not use indexes as efficiently

---

### 🔄 Alternative Solution 1C: With Breakdown by Device

```sql
SELECT 
    device_type,
    COUNT(*) AS number_events
FROM fct_user_resumptions
WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
  AND resumption_timestamp < TIMESTAMP '2024-10-08 00:00:00'
  AND device_type IN ('iOS', 'Android')
GROUP BY device_type
ORDER BY number_events DESC;
```

**Additional Value:** Shows iOS vs Android engagement separately

---

## Question 2: Resume Event Distribution per User (October 2024)

### 📋 Problem
For **October 2024**, analyze the distribution of resume events per user. Calculate:
- **Median** number of resume events
- **90th percentile** number of resume events
- **Maximum** number of resume events

Include only users who had **at least 1 resume event**.

### Expected Output
- `median_events`
- `90th_percentile`
- `max_events_count`

---

### ✅ Solution 2 (Original)

```sql
WITH october_events AS (
    SELECT user_id,
           COUNT(*) AS num_events
    FROM fct_user_resumptions
    WHERE DATE(resumption_timestamp) BETWEEN '2024-10-01' AND '2024-10-31'
    GROUP BY user_id
)
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY num_events) AS median_events,
       PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY num_events) AS "90th_percentile",
       MAX(num_events) AS max_events_count
FROM october_events oe
WHERE num_events >= 1;
```

**Explanation:**
- CTE aggregates resume events per user for October
- Uses `PERCENTILE_CONT()` for continuous percentile calculation
- 0.5 percentile = median (50th percentile)
- 0.9 percentile = 90th percentile
- Filters to users with at least 1 event (though all users in CTE already have ≥1)

**Pros:** Clear percentile calculation, standard SQL approach  
**Cons:** `WHERE num_events >= 1` is redundant (CTE already ensures this)

---

### 🔄 Alternative Solution 2A: Using Timestamp Range

```sql
WITH october_events AS (
    SELECT user_id,
           COUNT(*) AS num_events
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2024-11-01 00:00:00'
    GROUP BY user_id
)
SELECT 
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY num_events) AS median_events,
    PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY num_events) AS percentile_90th,
    MAX(num_events) AS max_events_count
FROM october_events;
```

**Pros:** More efficient timestamp filtering, removed redundant WHERE  
**Cons:** None

---

### 🔄 Alternative Solution 2B: With Additional Percentiles

```sql
WITH october_events AS (
    SELECT user_id,
           COUNT(*) AS num_events
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2024-11-01 00:00:00'
    GROUP BY user_id
)
SELECT 
    MIN(num_events) AS min_events,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY num_events) AS percentile_25th,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY num_events) AS median_events,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY num_events) AS percentile_75th,
    PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY num_events) AS percentile_90th,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY num_events) AS percentile_95th,
    MAX(num_events) AS max_events_count
FROM october_events;
```

**Additional Value:** Full distribution analysis with quartiles

---

### 🔄 Alternative Solution 2C: Using PERCENTILE_DISC (Discrete)

```sql
WITH october_events AS (
    SELECT user_id,
           COUNT(*) AS num_events
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2024-11-01 00:00:00'
    GROUP BY user_id
)
SELECT 
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY num_events) AS median_events,
    PERCENTILE_DISC(0.9) WITHIN GROUP (ORDER BY num_events) AS percentile_90th,
    MAX(num_events) AS max_events_count
FROM october_events;
```

**Note:** `PERCENTILE_DISC` returns an actual value from the dataset, while `PERCENTILE_CONT` can interpolate. For event counts, both work well.

---

## Question 3: Top Shows by Device Type (Q4 2024)

### 📋 Problem
During the **fourth quarter of 2024** (October, November, December), which show titles generated the **highest number of resume events** on **each device type**?

### Expected Output
- `device_type`
- `show_title`
- `resume_count`

---

### ✅ Solution 3 (Original)

```sql
WITH q4_resume_counts AS (
    SELECT 
        device_type,
        show_title,
        COUNT(*) AS resume_count
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2025-01-01 00:00:00'
    GROUP BY device_type, show_title
),
ranked_shows AS (
    SELECT 
        device_type,
        show_title,
        resume_count,
        ROW_NUMBER() OVER (PARTITION BY device_type ORDER BY resume_count DESC) AS rank
    FROM q4_resume_counts
)
SELECT 
    device_type,
    show_title,
    resume_count
FROM ranked_shows
WHERE rank = 1
ORDER BY device_type;
```

**Explanation:**
- First CTE aggregates resume counts by device and show for Q4 2024
- Second CTE ranks shows within each device type using `ROW_NUMBER()`
- Final query filters to rank 1 (top show per device)
- Orders by device_type for clear presentation

**Pros:** Handles ties consistently (ROW_NUMBER picks one), clear multi-step logic  
**Cons:** Multiple CTEs, if ties exist only one show is returned

---

### 🔄 Alternative Solution 3A: DISTINCT ON (PostgreSQL-specific)

```sql
WITH q4_resume_counts AS (
    SELECT 
        device_type,
        show_title,
        COUNT(*) AS resume_count
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2025-01-01 00:00:00'
    GROUP BY device_type, show_title
)
SELECT DISTINCT ON (device_type)
    device_type,
    show_title,
    resume_count
FROM q4_resume_counts
ORDER BY device_type, resume_count DESC;
```

**Pros:** More concise, PostgreSQL-optimized, single CTE  
**Cons:** PostgreSQL-specific syntax, returns only one show if tied

---

### 🔄 Alternative Solution 3B: Using RANK() to Include Ties

```sql
WITH q4_resume_counts AS (
    SELECT 
        device_type,
        show_title,
        COUNT(*) AS resume_count
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2025-01-01 00:00:00'
    GROUP BY device_type, show_title
),
ranked_shows AS (
    SELECT 
        device_type,
        show_title,
        resume_count,
        RANK() OVER (PARTITION BY device_type ORDER BY resume_count DESC) AS rank
    FROM q4_resume_counts
)
SELECT 
    device_type,
    show_title,
    resume_count
FROM ranked_shows
WHERE rank = 1
ORDER BY device_type, show_title;
```

**Pros:** Returns ALL shows if there's a tie for #1  
**Cons:** May return multiple rows per device if tied

---

### 🔄 Alternative Solution 3C: With Percentage of Total

```sql
WITH q4_resume_counts AS (
    SELECT 
        device_type,
        show_title,
        COUNT(*) AS resume_count
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2025-01-01 00:00:00'
    GROUP BY device_type, show_title
),
device_totals AS (
    SELECT 
        device_type,
        SUM(resume_count) AS total_resumes
    FROM q4_resume_counts
    GROUP BY device_type
),
ranked_shows AS (
    SELECT 
        qrc.device_type,
        qrc.show_title,
        qrc.resume_count,
        dt.total_resumes,
        ROUND(100.0 * qrc.resume_count / dt.total_resumes, 2) AS percentage,
        ROW_NUMBER() OVER (PARTITION BY qrc.device_type ORDER BY qrc.resume_count DESC) AS rank
    FROM q4_resume_counts qrc
    JOIN device_totals dt ON qrc.device_type = dt.device_type
)
SELECT 
    device_type,
    show_title,
    resume_count,
    percentage AS percentage_of_device_total
FROM ranked_shows
WHERE rank = 1
ORDER BY device_type;
```

**Additional Value:** Shows what percentage of device's total resumes the top show represents

---

### 🔄 Alternative Solution 3D: Top 3 Shows per Device

```sql
WITH q4_resume_counts AS (
    SELECT 
        device_type,
        show_title,
        COUNT(*) AS resume_count
    FROM fct_user_resumptions
    WHERE resumption_timestamp >= TIMESTAMP '2024-10-01 00:00:00'
      AND resumption_timestamp < TIMESTAMP '2025-01-01 00:00:00'
    GROUP BY device_type, show_title
),
ranked_shows AS (
    SELECT 
        device_type,
        show_title,
        resume_count,
        ROW_NUMBER() OVER (PARTITION BY device_type ORDER BY resume_count DESC) AS rank
    FROM q4_resume_counts
)
SELECT 
    device_type,
    show_title,
    resume_count,
    rank
FROM ranked_shows
WHERE rank <= 3
ORDER BY device_type, rank;
```

**Additional Value:** Provides top 3 shows per device for deeper insights

---

## 📊 Key Takeaways

### Timestamp vs Date Handling
- **Using DATE()**: Simple but may not use indexes efficiently
- **Timestamp ranges**: `>= '2024-10-01 00:00:00' AND < '2024-10-08 00:00:00'` better for performance
- **Always exclude end date**: Use `< '2024-10-08'` instead of `<= '2024-10-07'` to avoid timezone issues

### Statistical Functions in PostgreSQL
- **PERCENTILE_CONT**: Continuous percentile (can interpolate between values)
- **PERCENTILE_DISC**: Discrete percentile (returns actual dataset value)
- **WITHIN GROUP (ORDER BY ...)**: Required syntax for ordered-set aggregate functions
- **Median**: Use PERCENTILE_CONT(0.5) or PERCENTILE_DISC(0.5)

### Window Functions for Ranking
- **ROW_NUMBER()**: Always returns 1 result per partition, even with ties
- **RANK()**: Returns multiple rows if tied (gaps in ranking: 1, 2, 2, 4)
- **DENSE_RANK()**: Returns multiple rows if tied (no gaps: 1, 2, 2, 3)
- **PARTITION BY**: Essential for "top N per group" queries

### Filtering Techniques
- **IN clause**: Efficient for multiple discrete values (`IN ('iOS', 'Android')`)
- **NOT IN**: Careful with NULLs - use `device_type != 'Desktop'` as alternative
- **Mobile vs non-mobile**: Define clearly in requirements

### Performance Optimization
1. **Question 1**: Timestamp range > DATE() function for large datasets
2. **Question 2**: CTE aggregation is efficient; redundant WHERE clause should be removed
3. **Question 3**: Single CTE with ranking is optimal; DISTINCT ON is fastest in PostgreSQL

### Business Insights
- **Q1**: Measures mobile engagement volume for specific time period
- **Q2**: Identifies power users and typical usage patterns through distribution
- **Q3**: Reveals content preferences by platform (iOS vs Android vs Desktop)

### Common Patterns
- **Time-based analysis**: Always use consistent timestamp filtering approach
- **Per-user metrics**: CTE to aggregate by user, then calculate statistics
- **Per-group rankings**: Window functions with PARTITION BY

### Practical Applications
- **Platform Strategy**: Understand iOS vs Android user behavior differences
- **Content Optimization**: Promote popular shows per platform
- **User Segmentation**: Use percentiles to identify power users for targeted features
- **App Performance**: Track resume event trends to measure app reliability

### SQL Techniques Covered
- Timestamp extraction and filtering
- IN clause for multiple value filtering
- PERCENTILE_CONT/PERCENTILE_DISC for statistical analysis
- WITHIN GROUP for ordered-set aggregates
- Window functions (ROW_NUMBER, RANK)
- PARTITION BY for grouped rankings
- Multiple CTEs for complex transformations
- PostgreSQL DISTINCT ON for efficient top-N queries
