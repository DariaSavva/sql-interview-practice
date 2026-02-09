# Case 18: Apple Supply Chain Procurement Analysis

## Business Context

As a Data Analyst on the Supply Chain Procurement Team in Apple, you are tasked with assessing supplier performance to ensure reliable delivery of critical components. Your goal is to identify the most active suppliers, understand which suppliers dominate in specific manufacturing regions, and pinpoint any gaps in supply to the Asia region. By leveraging data, you will help optimize vendor selection strategies and mitigate potential supply chain risks.

## Database Schema

### supplier_deliveries
- supplier_id: INTEGER
- delivery_date: DATE
- component_count: INTEGER
- manufacturing_region: VARCHAR

### suppliers
- supplier_id: INTEGER
- supplier_name: VARCHAR

---

## Question 1: Top 5 Most Active Suppliers (October 2024)

### Problem
We need to know who our most active suppliers are. Identify the top 5 suppliers based on the total volume of components delivered in October 2024.

### Expected Output
- supplier_name
- total_volume

### Solution

```sql
SELECT supplier_name,
       SUM(component_count) AS total_volume
FROM supplier_deliveries f
JOIN suppliers d
    ON f.supplier_id = d.supplier_id
WHERE delivery_date >= '2024-10-01' 
  AND delivery_date < '2024-11-01'
GROUP BY supplier_name
ORDER BY SUM(component_count) DESC
LIMIT 5;
```

**Explanation:**
- Joins deliveries fact table with suppliers dimension to get supplier names
- Filters to October 2024 using date range
- Groups by supplier_name and sums component counts
- Orders by total volume descending to get highest first
- LIMIT 5 returns only the top 5 suppliers
- Shows which suppliers are most active by delivery volume

---

## Question 2: Top Supplier per Region (November 2024)

### Problem
For each region, find the supplier ID that delivered the highest number of components in November 2024. This will help us understand which supplier is handling the most volume per market.

### Expected Output
- manufacturing_region
- supplier_id
- total_volume

### Solution

```sql
WITH nov_volume AS (
    SELECT manufacturing_region,
           supplier_id,
           SUM(component_count) AS total_volume
    FROM supplier_deliveries
    WHERE delivery_date >= '2024-11-01' 
      AND delivery_date < '2024-12-01'
    GROUP BY supplier_id, manufacturing_region
),
ranking AS (
    SELECT manufacturing_region,
           supplier_id,
           total_volume,
           ROW_NUMBER() OVER (
               PARTITION BY manufacturing_region 
               ORDER BY total_volume DESC
           ) AS rn
    FROM nov_volume
)
SELECT manufacturing_region,
       supplier_id,
       total_volume
FROM ranking
WHERE rn = 1;
```

**Explanation:**
- First CTE aggregates total component volume per supplier per region for November 2024
- Second CTE applies ROW_NUMBER() partitioned by region
- Orders by total_volume DESC within each region
- Main query filters to rn = 1 to get the top supplier in each region
- Shows regional leaders in supply volume

---

## Question 3: Suppliers Not Delivering to Asia (December 2024)

### Problem
We need to identify potential gaps in our supply chain for Asia. List all suppliers by name who have not delivered any components to the 'Asia' manufacturing region in December 2024.

### Expected Output
- supplier_id
- supplier_name

### Solution A: Using NOT IN

```sql
SELECT
    s.supplier_id,
    s.supplier_name
FROM suppliers s
WHERE s.supplier_id NOT IN (
    SELECT d.supplier_id
    FROM supplier_deliveries d
    WHERE d.manufacturing_region = 'Asia'
      AND d.delivery_date >= DATE '2024-12-01'
      AND d.delivery_date < DATE '2025-01-01'
);
```

**Explanation:**
- Subquery finds all supplier_ids that delivered to Asia in December 2024
- NOT IN excludes those suppliers from the result
- Returns suppliers who are not in the Asia delivery list
- Note: NOT IN can have issues with NULL values

### Solution B: Using LEFT JOIN with NULL Check

```sql
SELECT
    s.supplier_id,
    s.supplier_name
FROM suppliers s
LEFT JOIN supplier_deliveries d
    ON s.supplier_id = d.supplier_id
   AND d.manufacturing_region = 'Asia'
   AND d.delivery_date >= DATE '2024-12-01'
   AND d.delivery_date < DATE '2025-01-01'
WHERE d.supplier_id IS NULL;
```

**Explanation:**
- LEFT JOIN includes all suppliers, matching with Asia deliveries if they exist
- Join conditions include the region and date filters
- WHERE d.supplier_id IS NULL finds suppliers with no matching Asia deliveries
- NULL-safe approach, generally preferred over NOT IN

### Solution C: Using NOT EXISTS

```sql
SELECT
    s.supplier_id,
    s.supplier_name
FROM suppliers s
WHERE NOT EXISTS (
    SELECT 1
    FROM supplier_deliveries d
    WHERE d.supplier_id = s.supplier_id
      AND d.manufacturing_region = 'Asia'
      AND d.delivery_date >= DATE '2024-12-01'
      AND d.delivery_date < DATE '2025-01-01'
);
```

**Explanation:**
- NOT EXISTS checks if any matching row exists in the subquery
- Correlated subquery checks for each supplier if they delivered to Asia
- Returns suppliers for which no matching delivery exists
- Often the most efficient approach for anti-joins
- Handles NULL values correctly

### Comparison of Anti-Join Methods

**NOT IN:**
- Simple syntax
- Can be slow on large datasets
- Issues with NULL values in subquery

**LEFT JOIN with NULL:**
- More explicit about the anti-join logic
- Generally good performance
- NULL-safe

**NOT EXISTS:**
- Often best performance
- Short-circuits when first match is found
- Preferred for large datasets
- NULL-safe
