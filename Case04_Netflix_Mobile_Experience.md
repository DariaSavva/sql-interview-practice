# Case 04: Netflix Mobile Experience Analysis

## Business Context

As a Data Analyst on the Netflix Mobile Experience team, you are investigating how users resume watching shows on mobile devices to enhance the seamless viewing experience. Your goal is to analyze user engagement through resume events, understand the distribution of these events per user, and identify the most popular shows for resumption on different mobile platforms.

## Database Schema

### fct_user_resumptions
- user_id: INTEGER
- device_type: VARCHAR (iOS, Android, Desktop)
- show_title: VARCHAR
- resumption_timestamp: TIMESTAMP

---

## Question 1: Mobile Resume Events (Week of Oct 1-7, 2024)

### Problem
For the week from October 1st to October 7th, 2024, how many total resume events occurred on mobile devices?

### Expected Output
- number_events

### Solution

```sql
SELECT COUNT(*) AS number_events
FROM fct_user_resumptions
WHERE DATE(resumption_timestamp) BETWEEN '2024-10-01' AND '2024-10-07'
  AND device_type IN ('iOS', 'Android');
```

**Explanation:**
- Uses DATE() to extract date from timestamp
- BETWEEN is inclusive on both ends
- Filters to mobile devices only using IN ('iOS', 'Android')
- Counts all matching rows

---

## Question 2: Resume Event Distribution per User (October 2024)

### Problem
For October 2024, analyze the distribution of resume events per user. Calculate the median, 90th percentile, and maximum number of resume events across users who had at least 1 resume event.

### Expected Output
- median_events
- 90th_percentile
- max_events_count

### Solution

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
- Uses PERCENTILE_CONT() for continuous percentile calculation
- 0.5 percentile = median (50th percentile)
- 0.9 percentile = 90th percentile

---

## Question 3: Top Shows by Device Type (Q4 2024)

### Problem
During the fourth quarter of 2024, which show titles generated the highest number of resume events on each device type?

### Expected Output
- device_type
- show_title
- resume_count

### Solution

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
- Second CTE ranks shows within each device type using ROW_NUMBER()
- Final query filters to rank 1 (top show per device)
