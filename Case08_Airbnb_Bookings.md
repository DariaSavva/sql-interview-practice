# Case 08: Airbnb Booking Behaviors Analysis

## 📊 Business Context

You are a **Data Scientist on the Airbnb Stays team**, focusing on analyzing guest booking behaviors. Your team is investigating how **pricing transparency** and **cancellation policies** influence booking completion rates and daily booking volume. The goal is to provide insights into how these factors impact booking conversions and inform strategies to optimize guest satisfaction and booking success.

## 🗄️ Database Schema

### `fct_bookings`
| Column | Type | Description |
|--------|------|-------------|
| booking_id | INTEGER | Unique booking identifier |
| property_id | INTEGER | Property identifier |
| booking_date | DATE | Date when booking was made |
| completion_status | VARCHAR | Status: 'completed', 'cancelled', etc. |
| pricing_transparency_level | VARCHAR | Level: 'low', 'medium', 'high' |
| cancellation_policy | VARCHAR | Policy type: 'flexible', 'moderate', 'strict' |

---

## Question 1: Rolling 7-Day Completion Rate by Pricing Transparency (April 2024)

### 📋 Problem
What is the **average booking completion rate** for properties with **'low' and 'high' pricing transparency levels** during April 2024? 

Use a **7-day rolling window** to capture short-term trends. This analysis will help us understand the impact of pricing transparency on booking conversions.

### Expected Output
- `booking_date`
- `pricing_transparency_level`
- `rolling_completed`
- `rolling_total`
- `rolling_completion_rate`

---

### ✅ Solution 1 (Original)

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
- Uses conditional SUM with CASE to count completed bookings
- Second CTE applies rolling 7-day window using `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`
- Partitions by pricing_transparency_level to keep 'low' and 'high' separate
- Final query calculates completion rate with division safety check
- Returns detailed daily view of rolling metrics

**Pros:** Clear multi-step logic, safe division, shows all components of calculation  
**Cons:** Multiple CTEs and window functions can be expensive on large datasets

---

### 🔄 Alternative Solution: Single CTE with Direct Calculation

```sql
WITH daily_metrics AS (
    SELECT
        booking_date,
        pricing_transparency_level,
        COUNT(*) AS daily_total,
        SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END) AS daily_completed,
        SUM(COUNT(*)) OVER (
            PARTITION BY pricing_transparency_level
            ORDER BY booking_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_total,
        SUM(SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END)) OVER (
            PARTITION BY pricing_transparency_level
            ORDER BY booking_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_completed
    FROM fct_bookings
    WHERE booking_date >= DATE '2024-04-01'
      AND booking_date < DATE '2024-05-01'
      AND pricing_transparency_level IN ('low', 'high')
    GROUP BY booking_date, pricing_transparency_level
)
SELECT
    booking_date,
    pricing_transparency_level,
    rolling_completed,
    rolling_total,
    rolling_completed * 1.0 / NULLIF(rolling_total, 0) AS rolling_completion_rate
FROM daily_metrics
ORDER BY pricing_transparency_level, booking_date;
```

**Pros:** Single CTE, more concise, uses NULLIF for safer division  
**Cons:** Nested aggregations can be harder to read

---

## Question 2: Day-Over-Day Booking Changes by Cancellation Policy (April 2024)

### 📋 Problem
What are the **day-over-day changes** in number of bookings for properties with varying **cancellation policies** in April 2024? 

Compare each day with the previous day to reveal trends over time. This analysis will help us assess how cancellation policies influence booking volume on a daily basis.

### Expected Output
- `booking_date`
- `cancellation_policy`
- `total_bookings`
- `previous_daily_bookings`
- `daily_change`

---

### ✅ Solution 2 (Original)

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
- Uses `LAG()` function to access previous day's booking count
- Partitions by cancellation_policy to compare within same policy type
- Orders by booking_date to ensure chronological comparison
- Calculates daily_change by subtracting previous day from current day
- First day for each policy will have NULL for previous_daily_bookings

**Pros:** Clean use of LAG function, clear day-over-day comparison  
**Cons:** LAG is computed twice (could use subquery to compute once)

---

### 🔄 Alternative Solution: Optimized with Single LAG Call

```sql
WITH daily_bookings AS (
    SELECT booking_date,
           cancellation_policy,
           COUNT(*) AS total_bookings
    FROM fct_bookings
    WHERE booking_date >= DATE '2024-04-01' 
      AND booking_date < DATE '2024-05-01'
    GROUP BY booking_date, cancellation_policy
),
with_lag AS (
    SELECT booking_date,
           cancellation_policy,
           total_bookings,
           LAG(total_bookings) OVER (
               PARTITION BY cancellation_policy
               ORDER BY booking_date
           ) AS previous_daily_bookings
    FROM daily_bookings
)
SELECT booking_date,
       cancellation_policy,
       total_bookings,
       previous_daily_bookings,
       total_bookings - previous_daily_bookings AS daily_change,
       CASE 
           WHEN previous_daily_bookings IS NULL THEN NULL
           ELSE ROUND(100.0 * (total_bookings - previous_daily_bookings) / previous_daily_bookings, 2)
       END AS pct_change
FROM with_lag
ORDER BY booking_date, cancellation_policy;
```

**Pros:** Computes LAG once, adds percentage change for deeper insights  
**Cons:** Extra CTE level

---

## Question 3: Completion Rate Comparison by Transparency and Policy (April 2024)

### 📋 Problem
What is the **percentage difference in booking completion rates** between properties with **high pricing transparency** and those with **low transparency** in April 2024, when also accounting for differing **cancellation policies**?

Compute the booking completion rates separately for high and low transparency levels within each cancellation policy, then calculate the percentage difference between them to compare the results.

### Expected Output
- `cancellation_policy`
- `high_completion_rate`
- `low_completion_rate`
- `pct_difference_completion_rate`

---

### ✅ Solution 3 (Original)

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
- Uses conditional aggregation to count completed bookings
- Second CTE pivots data using MAX/CASE pattern
- Transforms rows (high/low) into columns for easy comparison
- Final query calculates percentage difference: (high - low) / low
- Safe division check to avoid division by zero
- Returns one row per cancellation policy

**Pros:** Clear pivoting logic, easy to compare high vs low side-by-side  
**Cons:** Pivoting with MAX/CASE can be verbose

---

### 🔄 Alternative Solution: Using FILTER Clause (PostgreSQL 9.4+)

```sql
WITH policy_rates AS (
    SELECT
        cancellation_policy,
        COUNT(*) FILTER (WHERE pricing_transparency_level = 'high') AS high_total,
        SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END) 
            FILTER (WHERE pricing_transparency_level = 'high') AS high_completed,
        COUNT(*) FILTER (WHERE pricing_transparency_level = 'low') AS low_total,
        SUM(CASE WHEN completion_status = 'completed' THEN 1 ELSE 0 END) 
            FILTER (WHERE pricing_transparency_level = 'low') AS low_completed
    FROM fct_bookings
    WHERE booking_date >= DATE '2024-04-01'
      AND booking_date < DATE '2024-05-01'
      AND pricing_transparency_level IN ('low', 'high')
    GROUP BY cancellation_policy
)
SELECT
    cancellation_policy,
    high_completed * 1.0 / NULLIF(high_total, 0) AS high_completion_rate,
    low_completed * 1.0 / NULLIF(low_total, 0) AS low_completion_rate,
    (high_completed * 1.0 / NULLIF(high_total, 0) - 
     low_completed * 1.0 / NULLIF(low_total, 0)) / 
    NULLIF(low_completed * 1.0 / NULLIF(low_total, 0), 0) AS pct_difference_completion_rate
FROM policy_rates
ORDER BY cancellation_policy;
```

**Pros:** Single CTE using FILTER clause, more PostgreSQL-idiomatic  
**Cons:** Calculation repeated multiple times, can be harder to read

---

## 📊 Key Takeaways

### Rolling Windows for Time Series Analysis
- **7-Day Window**: `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`
- **Use Cases**: Smoothing daily volatility, identifying trends, comparing periods
- **Pattern**: Aggregate daily → Apply rolling window → Calculate rate
- **Partitioning**: Essential to keep groups (e.g., transparency levels) separate

### LAG Function for Day-Over-Day Analysis
```sql
LAG(column) OVER (PARTITION BY group_column ORDER BY date_column)
```
- **Purpose**: Access previous row's value within partition
- **Opposite**: `LEAD()` accesses next row's value
- **Use Cases**: Day-over-day changes, growth rates, trends
- **Null Handling**: First row in partition returns NULL

### Pivoting Techniques in PostgreSQL
**Method 1: MAX/CASE Pattern (Original)**
```sql
MAX(CASE WHEN condition THEN value END) AS column_name
```
**Method 2: FILTER Clause (Alternative)**
```sql
COUNT(*) FILTER (WHERE condition)
```
- Both transform rows into columns
- FILTER is more concise for PostgreSQL
- MAX/CASE is more portable across databases

### Multi-Dimensional Analysis Pattern
1. **Aggregate at granular level** (policy × transparency)
2. **Transform for comparison** (pivot to columns)
3. **Calculate comparative metrics** (percentage difference)
4. **Interpret results** (higher/lower performance)

### Completion Rate Calculations
- **Formula**: `completed_bookings / total_bookings`
- **Safety**: Always check for division by zero
- **Rolling**: Calculate totals first, then divide
- **Comparison**: Use percentage difference for relative comparison

### Percentage Difference vs Percentage Change
- **Percentage Difference**: `(new - old) / old`
- **Example**: If low = 0.6, high = 0.75, diff = (0.75-0.6)/0.6 = 0.25 = 25%
- **Interpretation**: High transparency has 25% better completion rate

### Performance Considerations
- **Question 1**: Multiple window functions can be expensive; consider materializing daily_bookings
- **Question 2**: LAG computed twice in original; alternative computes once
- **Question 3**: Pivoting with MAX/CASE is standard and efficient

### Business Insights
- **Q1**: Track how pricing transparency affects completion rates over time
- **Q2**: Identify booking volume trends by cancellation policy
- **Q3**: Quantify impact of transparency on completion rates by policy type

### Common Patterns in This Case
1. **Conditional Aggregation**: `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`
2. **Rolling Calculations**: Apply window function after daily aggregation
3. **Pivoting**: Transform categorical variable into columns
4. **Safe Division**: Always use NULLIF or CASE to prevent division by zero

### SQL Techniques Covered
- Rolling windows with ROWS BETWEEN
- LAG function for time series comparison
- Conditional aggregation with CASE
- Pivoting with MAX/CASE pattern
- FILTER clause for conditional aggregation (PostgreSQL)
- Multi-level CTEs for complex transformations
- Safe division with NULLIF and CASE
- PARTITION BY for independent group analysis
- Percentage calculations and comparisons

### Practical Applications
- **A/B Testing**: Compare completion rates between different transparency levels
- **Trend Analysis**: Identify patterns in booking volume over time
- **Policy Optimization**: Determine which cancellation policies perform best
- **Segmentation**: Analyze performance across multiple dimensions
- **Dashboard Metrics**: Rolling completion rates for monitoring
- **Strategic Planning**: Data-driven decisions on pricing transparency

### Advanced Concepts
- **Multi-dimensional pivoting**: Transforming 2+ categories into columns
- **Rolling rate calculations**: Avoid averaging rates; sum numerator and denominator first
- **Comparative metrics**: Expressing differences as percentages for relative comparison
- **Temporal analysis**: Combining window functions (LAG, rolling windows) for rich insights
