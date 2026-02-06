# Case 16: Amazon Devices Usage Patterns Analysis

## Business Context

As a Data Analyst on the Amazon Devices team, you are tasked with evaluating the usage patterns of Amazon services on devices like Echo, Fire TV, and Kindle. Your goal is to categorize device usage, assess overall engagement levels, and analyze the contribution of Prime Video and Amazon Music to total usage. This analysis will inform strategies to optimize service offerings and improve customer satisfaction.

## Database Schema

### fct_device_usage
- usage_id: INTEGER
- device_id: INTEGER
- service_id: INTEGER
- usage_duration_minutes: INTEGER
- usage_date: DATE

### dim_device
- device_id: INTEGER
- device_name: VARCHAR

### dim_service
- service_id: INTEGER
- service_name: VARCHAR

---

## Question 1: Total Usage Duration by Device Type (Q3 2024)

### Problem
The team wants to identify the total usage duration of the services for each device type by extracting the primary device category from the device name for the period from July 1, 2024 to September 30, 2024. The primary device category is derived from the first word of the device name.

### Expected Output
- device_type
- total_usage_minutes

### Solution

```sql
SELECT 
    SPLIT_PART(device_name, ' ', 1) AS device_type,
    SUM(usage_duration_minutes) AS total_usage_minutes
FROM fct_device_usage f
JOIN dim_device d 
    ON f.device_id = d.device_id
WHERE usage_date BETWEEN '2024-07-01' AND '2024-09-30'
GROUP BY SPLIT_PART(device_name, ' ', 1)
ORDER BY SUM(usage_duration_minutes) DESC;
```

**Explanation:**
- Uses SPLIT_PART() to extract the first word from device_name
- SPLIT_PART(string, delimiter, position) splits string by delimiter and returns the nth part
- Position 1 returns the first word (e.g., "Echo" from "Echo Dot")
- Joins fact table with device dimension to get device names
- Filters to Q3 2024 (July-September) using BETWEEN
- Groups by device_type and sums usage duration
- Orders by total usage descending to see most popular device types first

---

## Question 2: Device Usage Categorization (Q3 2024)

### Problem
The team also wants to label the usage of each device category into 'Low' or 'High' based on usage duration from July 1, 2024 to September 30, 2024. If the total usage time was less than 300 minutes, we'll categorize it as 'Low'. Otherwise, we'll categorize it as 'High'. Can you return a report with device ID, usage category and total usage time?

### Expected Output
- device_id
- usage_category
- total_usage_time

### Solution

```sql
WITH usage AS (
    SELECT device_id,
           SUM(usage_duration_minutes) AS total_usage_time
    FROM fct_device_usage
    WHERE usage_date BETWEEN '2024-07-01' AND '2024-09-30'
    GROUP BY device_id
)
SELECT device_id,
       CASE 
           WHEN total_usage_time < 300 THEN 'Low' 
           ELSE 'High' 
       END AS usage_category,
       total_usage_time
FROM usage;
```

**Explanation:**
- CTE aggregates total usage time per device for Q3 2024
- Main query applies CASE statement to categorize usage
- Devices with less than 300 minutes labeled as 'Low'
- Devices with 300+ minutes labeled as 'High'
- Returns device_id with its category and total usage time

---

## Question 3: Prime Video and Amazon Music Usage Percentage (Q3 2024)

### Problem
The team is considering bundling the Prime Video and Amazon Music subscription. They want to understand what percentage of total usage time comes from Prime Video and Amazon Music services respectively. Please use data from July 1, 2024 to September 30, 2024.

### Expected Output
- service_name
- usage_percentage

### Solution

```sql
WITH usage AS (
    SELECT service_name,
           SUM(usage_duration_minutes) AS total_usage
    FROM fct_device_usage f
    JOIN dim_service s 
        ON f.service_id = s.service_id
    WHERE f.usage_date >= DATE '2024-07-01'
      AND f.usage_date <= DATE '2024-09-30'
    GROUP BY service_name
), 
totals AS (
    SELECT SUM(total_usage) AS overall_usage
    FROM usage
)
SELECT
    u.service_name,
    ROUND(
        u.total_usage * 100.0 / NULLIF(t.overall_usage, 0),
        2
    ) AS usage_percentage
FROM usage u
CROSS JOIN totals t
WHERE u.service_name IN ('Prime Video', 'Amazon Music')
ORDER BY usage_percentage DESC;
```

**Explanation:**
- First CTE aggregates usage per service for Q3 2024
- Second CTE calculates total usage across all services
- Main query uses CROSS JOIN to combine each service with overall total
- Calculates percentage: (service_usage / total_usage) * 100
- NULLIF prevents division by zero
- Multiplies by 100.0 to get percentage
- ROUND to 2 decimal places for clean output
- Filters to only Prime Video and Amazon Music services
- Orders by usage percentage descending to show higher usage first
