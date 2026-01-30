# Case 10: Apple Philanthropic Initiatives Analysis

## 📊 Business Context

As a **Data Analyst on Apple's Corporate Social Responsibility team**, you are tasked with evaluating the effectiveness of recent philanthropic initiatives. Your focus is on understanding participant engagement across different communities and programs. The insights you gather will guide strategic decisions for resource allocation and future program expansions.

## 🗄️ Database Schema

### `fct_philanthropic_initiatives`
| Column | Type | Description |
|--------|------|-------------|
| program_id | INTEGER | Program identifier |
| community_id | INTEGER | Foreign key to dim_community |
| event_date | DATE | Date of the event |
| participants | INTEGER | Number of participants |
| program_name | VARCHAR | Name of the program |

### `dim_community`
| Column | Type | Description |
|--------|------|-------------|
| community_id | INTEGER | Unique community identifier |
| community_name | VARCHAR | Name of the community |
| region | VARCHAR | Geographic region |

---

## Question 1: Participant Summary by Community and Program (January 2024)

### 📋 Problem
Apple's Corporate Social Responsibility team wants a **summary report** of philanthropic initiatives in **January 2024**. 

Please compile a report that **aggregates participant numbers** by community and by program.

### Expected Output
- `program_name`
- `community_name`
- `participants_count`

---

### ✅ Solution 1 (Original)

```sql
SELECT f.program_name,
       d.community_name,
       SUM(participants) AS participants_count
FROM fct_philanthropic_initiatives f
JOIN dim_community d
    ON f.community_id = d.community_id
WHERE event_date >= '2024-01-01' 
  AND event_date < '2024-02-01'
GROUP BY f.program_name, d.community_name;
```

**Explanation:**
- Joins fact table with dimension table to get community names
- Filters to January 2024 using date range
- Groups by both program_name and community_name (multi-dimensional grouping)
- Sums participants across all events for each program-community combination
- Returns one row per unique program-community pair

**Pros:** Clear multi-dimensional aggregation, straightforward JOIN  
**Cons:** None - this is optimal for the requirement

---

### 🔄 Alternative Solution: With Region and Event Count

```sql
SELECT 
    f.program_name,
    d.community_name,
    d.region,
    COUNT(*) AS event_count,
    SUM(f.participants) AS participants_count,
    AVG(f.participants) AS avg_participants_per_event
FROM fct_philanthropic_initiatives f
JOIN dim_community d
    ON f.community_id = d.community_id
WHERE f.event_date >= DATE '2024-01-01' 
  AND f.event_date < DATE '2024-02-01'
GROUP BY f.program_name, d.community_name, d.region
ORDER BY participants_count DESC;
```

**Pros:** Provides additional context (region, event count, averages) for deeper insights  
**Cons:** More columns than requested

---

## Question 2: Program Details with Earliest Event Date (February 2024)

### 📋 Problem
The team is reviewing the execution of **February 2024** philanthropic programs. 

For each initiative, provide details along with the **earliest event date recorded within each program campaign** to understand start timings.

### Expected Output
- `program_id`
- `program_name`
- `event_date`
- `min_event_date` (earliest date for that program)

---

### ✅ Solution 2 (Original)

```sql
SELECT program_id,
       program_name,
       event_date,
       MIN(event_date) OVER (PARTITION BY program_name) AS min_event_date
FROM fct_philanthropic_initiatives
WHERE event_date >= '2024-02-01' 
  AND event_date < '2024-03-01'
ORDER BY program_name;
```

**Explanation:**
- Uses window function `MIN(event_date) OVER (PARTITION BY program_name)`
- Partitions by program_name to find earliest date within each program
- Shows every event row with its program's minimum date for comparison
- No GROUP BY - returns all detail rows
- Useful for seeing which events happened first vs later in each program

**Pros:** Shows all event details while including program-level minimum date  
**Cons:** Returns many rows (one per event); may be redundant if only summary needed

---

### 🔄 Alternative Solution: Aggregated Program Summary

```sql
SELECT 
    program_id,
    program_name,
    MIN(event_date) AS min_event_date,
    MAX(event_date) AS max_event_date,
    COUNT(*) AS total_events,
    SUM(participants) AS total_participants
FROM fct_philanthropic_initiatives
WHERE event_date >= DATE '2024-02-01' 
  AND event_date < DATE '2024-03-01'
GROUP BY program_id, program_name
ORDER BY program_name;
```

**Pros:** Cleaner summary (one row per program), includes useful date range and totals  
**Cons:** Doesn't show individual event details

---

## Question 3: Maximum Participation by Program (First Week of March 2024)

### 📋 Problem
For a refined analysis of initiatives held during the **first week of March 2024** (March 1-7), include for each program the **maximum participation count** recorded in any event. 

This information will help highlight the highest engagement levels within each campaign.

### Expected Output
- `program_id`
- `program_name`
- `max_participants`

---

### ✅ Solution 3 (Original)

```sql
SELECT
    program_id,
    program_name,
    MAX(participants) AS max_participants
FROM fct_philanthropic_initiatives
WHERE event_date >= DATE '2024-03-01'
  AND event_date < DATE '2024-03-08'
GROUP BY program_id, program_name;
```

**Explanation:**
- Filters to first week of March (March 1-7)
- Uses MAX() aggregation to find highest participant count per program
- Groups by program_id and program_name to get one row per program
- Returns program identifiers with their peak participation

**Pros:** Clean aggregation, one row per program, efficient  
**Cons:** None - this is optimal for the requirement

---

### 🔄 Alternative Solution: With Peak Event Details

```sql
WITH ranked_events AS (
    SELECT
        program_id,
        program_name,
        event_date,
        participants,
        ROW_NUMBER() OVER (
            PARTITION BY program_id 
            ORDER BY participants DESC, event_date
        ) AS rn
    FROM fct_philanthropic_initiatives
    WHERE event_date >= DATE '2024-03-01'
      AND event_date < DATE '2024-03-08'
)
SELECT
    program_id,
    program_name,
    participants AS max_participants,
    event_date AS peak_event_date
FROM ranked_events
WHERE rn = 1
ORDER BY max_participants DESC;
```

**Pros:** Shows which specific event had the peak participation (includes date)  
**Cons:** More complex, uses ROW_NUMBER for what could be a simple MAX

---

## 📊 Key Takeaways

### Multi-Dimensional Grouping
```sql
GROUP BY dimension1, dimension2
```
- **Question 1**: Groups by both program AND community
- Creates one row per unique combination
- Essential for cross-tabulation and multi-dimensional analysis
- Think of it as a pivot table in SQL

### Window Functions vs GROUP BY
**Window Functions (Question 2)**:
```sql
MIN(column) OVER (PARTITION BY group_column)
```
- Returns value for each row
- No row reduction
- Shows detail + aggregated value side-by-side

**GROUP BY Aggregation (Questions 1 & 3)**:
```sql
SELECT group_column, MIN(column)
GROUP BY group_column
```
- Returns one row per group
- Reduces row count
- Summary-focused

### When to Use Each Approach
- **Window Function**: Need detail rows + group-level metrics
- **GROUP BY**: Need only summary (one row per group)

### Date Range Patterns
- **January**: `>= '2024-01-01' AND < '2024-02-01'`
- **February**: `>= '2024-02-01' AND < '2024-03-01'`
- **First Week of March**: `>= '2024-03-01' AND < '2024-03-08'`
- **Consistency**: Always use `>= start AND < end` pattern

### MIN and MAX Aggregations
- **MIN**: Find earliest date, smallest value, first alphabetically
- **MAX**: Find latest date, largest value, last alphabetically
- **Use Cases**: Peak values, date ranges, extremes
- **With OVER**: Apply within partitions without grouping

### JOIN Best Practices
- **INNER JOIN**: Only include matching records (used in Q1)
- **Column Qualification**: Use table aliases (f.column, d.column) for clarity
- **Join Key**: Always verify correct foreign key relationship

### Performance Considerations
- **Question 1**: Standard JOIN + GROUP BY, efficient
- **Question 2**: Window function scans all rows in partition
- **Question 3**: Simple aggregation, very efficient

### Business Insights
- **Q1**: Identifies which program-community combinations have highest engagement
- **Q2**: Shows program launch timing and duration
- **Q3**: Highlights peak engagement moments for replication

### Common Patterns in This Case
1. **Dimension table enrichment**: JOIN to get descriptive names
2. **Time-bounded analysis**: Filter to specific months/weeks
3. **Multi-level aggregation**: Summary across multiple dimensions
4. **Peak value identification**: MAX to find highest engagement

### SQL Techniques Covered
- INNER JOIN for dimension enrichment
- Multi-dimensional GROUP BY
- SUM, MIN, MAX aggregations
- Window functions (MIN OVER)
- PARTITION BY for grouped window functions
- Date range filtering
- Table aliases for readability
- ORDER BY for result organization

### Practical Applications
- **Resource Allocation**: Direct resources to high-participation program-community pairs
- **Program Timing**: Use earliest event dates to plan future campaigns
- **Success Metrics**: Peak participation shows maximum program reach
- **Regional Analysis**: Can add region grouping for geographic insights
- **Trend Analysis**: Compare months to identify seasonal patterns
- **Impact Reporting**: Aggregate metrics for stakeholder reports

### Query Optimization Tips
- **Index on event_date**: Helps with date range filters
- **Index on foreign keys**: Improves JOIN performance
- **Covering indexes**: Consider indexes that include commonly grouped columns
- **Statistics**: Keep table statistics updated for accurate query plans

### Window Functions with MIN/MAX
```sql
-- Shows each event with its program's range
MIN(event_date) OVER (PARTITION BY program_name) AS first_event,
MAX(event_date) OVER (PARTITION BY program_name) AS last_event
```
- Useful for comparing individual events to program boundaries
- No GROUP BY needed
- Each row gets the aggregated values for its partition

### Reporting Considerations
- **Question 1**: Perfect for pivot table or cross-tab report
- **Question 2**: Good for timeline visualization
- **Question 3**: Ideal for bar chart showing peak engagement by program
