# Case 12: Uber/Lyft Rides Performance Analysis

## Business Context

As a Data Analyst on the Rides Performance team, you are investigating how to reduce driver idle time and improve trip allocation strategies. The goal is to analyze driver trip patterns and identify opportunities to maximize earnings per active hour. Your analysis will help develop targeted recommendations to increase driver productivity and overall earnings.

## Database Schema

### fct_trips
- trip_id: INTEGER
- driver_id: INTEGER
- trip_start_time: TIMESTAMP
- trip_end_time: TIMESTAMP
- earnings: DECIMAL

---

## Question 1: Average Trip Earnings per Active Hour (October 2024)

### Problem
Based on October 2024 data, what was the average trip earnings per active hour? We want to calculate the trip earnings per hour for each driver, and then get the average across all drivers.

### Expected Output
- avg_trip_earnings_per_active_hour

### Solution

```sql
WITH driver_october_stats AS (
    SELECT driver_id,
           SUM(EXTRACT(EPOCH FROM (trip_end_time - trip_start_time)) / 3600) AS active_hours,
           SUM(earnings) AS total_ernings
    FROM fct_trips
    WHERE trip_start_time >= TIMESTAMP '2024-10-01'
      AND trip_start_time < TIMESTAMP '2024-11-01'
    GROUP BY driver_id
),
driver_earnings_per_hour AS (  
    SELECT driver_id,
           total_ernings / active_hours AS earnings_per_hour
    FROM driver_october_stats
    WHERE active_hours > 0
)
SELECT AVG(earnings_per_hour) AS avg_trip_earnings_per_active_hour
FROM driver_earnings_per_hour;
```

**Explanation:**
- First CTE calculates total active hours and earnings per driver for October
- Uses EXTRACT(EPOCH FROM (trip_end_time - trip_start_time)) to get seconds, then divides by 3600 for hours
- EPOCH extracts timestamp as seconds since 1970-01-01, allowing timestamp arithmetic
- Second CTE calculates earnings per hour for each driver
- Filters out drivers with 0 active hours to avoid division by zero
- Final query averages earnings per hour across all drivers

---

## Question 2: Idle Time Between Consecutive Trips (October 2024)

### Problem
For each driver, identify the next trip following a trip and calculate the idle time between the previous trip's end and that next trip's start. Look only at trips that started in October 2024. This analysis will pinpoint specific downtime intervals that could be reduced to enhance ride allocation.

### Expected Output
- driver_id
- trip_id
- trip_start_time
- trip_end_time
- next_trip_start_time
- idle_time_minutes

### Solution

```sql
SELECT driver_id,
       trip_id,
       trip_start_time,
       trip_end_time,
       LEAD(trip_start_time) OVER (
           PARTITION BY driver_id
           ORDER BY trip_start_time
       ) AS next_trip_start_time,
       EXTRACT(EPOCH FROM (
           LEAD(trip_start_time) OVER (
               PARTITION BY driver_id
               ORDER BY trip_start_time
           ) - trip_end_time
       )) / 60.0 AS idle_time_minutes
FROM fct_trips
WHERE trip_start_time >= TIMESTAMP '2024-10-01'
  AND trip_start_time < TIMESTAMP '2024-11-01';
```

**Explanation:**
- Uses LEAD() to get the start time of the next trip for each driver
- PARTITION BY driver_id keeps trips separate per driver
- ORDER BY trip_start_time ensures chronological ordering
- Calculates idle time: next trip start - current trip end
- EXTRACT(EPOCH FROM ...) / 60.0 converts seconds to minutes
- Last trip for each driver will have NULL for next_trip_start_time and idle_time_minutes

---

## Question 3: Top 2 Drivers with Shortest Average Idle Time (October 2024)

### Problem
For October 2024, identify the top 2 drivers with the shortest average idle time between consecutive trips. We aim to reach out to these drivers to understand their strategies for minimizing idle time.

### Expected Output
- driver_id
- avg_idle_time

### Solution

```sql
WITH idle_time AS (
    SELECT
        driver_id,
        trip_id,
        trip_start_time,
        trip_end_time,
        LEAD(trip_start_time) OVER (
            PARTITION BY driver_id
            ORDER BY trip_start_time
        ) AS next_trip_start_time,
        EXTRACT(EPOCH FROM (
            LEAD(trip_start_time) OVER (
                PARTITION BY driver_id
                ORDER BY trip_start_time
            ) - trip_end_time
        )) / 60.0 AS idle_time_minutes
    FROM fct_trips
    WHERE trip_start_time >= TIMESTAMP '2024-10-01'
      AND trip_start_time < TIMESTAMP '2024-11-01'
    ORDER BY driver_id, trip_start_time
)
SELECT driver_id,
       AVG(idle_time_minutes) AS avg_idle_time
FROM idle_time
GROUP BY driver_id
ORDER BY AVG(idle_time_minutes) ASC
LIMIT 2;
```

**Explanation:**
- CTE calculates idle time between consecutive trips for each driver
- Uses LEAD() to get next trip start time within each driver's trips
- Calculates idle time in minutes using EXTRACT(EPOCH ...) / 60.0
- Main query averages idle time per driver
- Orders by average idle time ascending (shortest first)
- LIMIT 2 returns top 2 drivers with shortest average idle time
- NULL idle times (last trip per driver) are excluded from AVG automatically
