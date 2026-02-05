# Case 08: Airbnb Booking Behaviors Analysis

## Business Context

You are a Data Scientist on the Airbnb Stays team, focusing on analyzing guest booking behaviors. Your team is investigating how pricing transparency and cancellation policies influence booking completion rates and daily booking volume.

## Database Schema

### fct_bookings
- booking_id: INTEGER
- property_id: INTEGER
- booking_date: DATE
- completion_status: VARCHAR
- pricing_transparency_level: VARCHAR
- cancellation_policy: VARCHAR

---

## Question 1: Rolling 7-Day Completion Rate by Pricing Transparency (April 2024)

### Problem
What is the average booking completion rate for properties with 'low' and 'high' pricing transparency levels during April 2024? Use a 7-day rolling window to capture short-term trends.

### Expected Output
- booking_date
- pricing_transparency_level
- rolling_completed
- rolling_total
- rolling_completion_rate

### Solution

```sql
WITH daily_bookings AS (
    SELECT
        booking_date,
        pricing_transparency_level,
        COUNT(*) AS total_bookings,
        SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END) AS completed_bookings
    FROM fct_bookings
    WHERE booking_date >= DATE '2024-04-01'
      AND booking_date < DATE '2024-05-01'
      AND pricing_transparency_level IN ('low', 'high')
    GROUP BY booking_date, pricing_transparency_level
),
rolling_completion AS (
    SELECT
        booking_date,
        pricing_transparency_level,
        SUM(completed_bookings) OVER (
            PARTITION BY pricing_transparency_level
            ORDER BY booking_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_completed,
        SUM(total_bookings) OVER (
            PARTITION BY pricing_transparency_level
            ORDER BY booking_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_total
    FROM daily_bookings
)
SELECT
    booking_date,
    pricing_transparency_level,
    rolling_completed,
    rolling_total,
    CASE
        WHEN rolling_total = 0 THEN NULL
        ELSE rolling_completed * 1.0 / rolling_total
    END AS rolling_completion_rate
FROM rolling_completion
ORDER BY pricing_transparency_level, booking_date;
```

**Explanation:**
- First CTE aggregates daily bookings and completed bookings by transparency level
- Second CTE applies rolling 7-day window using ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
- Final query calculates completion rate with division safety check

---

## Question 2: Day-Over-Day Booking Changes by Cancellation Policy (April 2024)

### Problem
What are the day-over-day changes in number of bookings for properties with varying cancellation policies in April 2024?

### Expected Output
- booking_date
- cancellation_policy
- total_bookings
- previous_daily_bookings
- daily_change

### Solution

```sql
WITH daily_bookings AS (
    SELECT booking_date,
           cancellation_policy,
           COUNT(*) AS total_bookings
    FROM fct_bookings
    WHERE booking_date >= '2024-04-01' 
      AND booking_date < '2024-05-01'
    GROUP BY booking_date, cancellation_policy
)
SELECT booking_date,
       cancellation_policy,
       total_bookings,
       LAG(total_bookings) OVER (
           PARTITION BY cancellation_policy
           ORDER BY booking_date
       ) AS previous_daily_bookings,
       total_bookings - LAG(total_bookings) OVER (
           PARTITION BY cancellation_policy
           ORDER BY booking_date
       ) AS daily_change
FROM daily_bookings
ORDER BY booking_date, cancellation_policy;
```

**Explanation:**
- CTE aggregates daily bookings by cancellation policy
- Uses LAG() to access previous day's booking count
- Calculates daily_change by subtracting previous day from current day

---

## Question 3: Completion Rate Comparison by Transparency and Policy (April 2024)

### Problem
What is the percentage difference in booking completion rates between properties with high pricing transparency and those with low transparency in April 2024, when also accounting for differing cancellation policies?

### Expected Output
- cancellation_policy
- high_completion_rate
- low_completion_rate
- pct_difference_completion_rate

### Solution

```sql
WITH daily_bookings AS (
    SELECT
        cancellation_policy,
        pricing_transparency_level,
        COUNT(*) AS total_bookings,
        SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END) AS completed_bookings,
        (SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END) * 1.0) / COUNT(*) AS completion_rate
    FROM fct_bookings
    WHERE booking_date >= DATE '2024-04-01'
      AND booking_date < DATE '2024-05-01'
      AND pricing_transparency_level IN ('low', 'high')
    GROUP BY cancellation_policy, pricing_transparency_level
),
pivoted_rates AS (
    SELECT cancellation_policy,
           MAX(CASE WHEN pricing_transparency_level = 'high' THEN completion_rate END) AS high_completion_rate,
           MAX(CASE WHEN pricing_transparency_level = 'low' THEN completion_rate END) AS low_completion_rate
    FROM daily_bookings
    GROUP BY cancellation_policy
)
SELECT cancellation_policy,
       high_completion_rate,
       low_completion_rate,
       CASE 
           WHEN low_completion_rate = 0 THEN NULL 
           ELSE (high_completion_rate - low_completion_rate) * 1.0 / low_completion_rate 
       END AS pct_difference_completion_rate
FROM pivoted_rates
ORDER BY cancellation_policy;
```

**Explanation:**
- First CTE calculates completion rates for each policy + transparency combination
- Second CTE pivots data using MAX/CASE pattern
- Final query calculates percentage difference: (high - low) / low
