# Case 10: Apple Philanthropic Initiatives Analysis

## Business Context

As a Data Analyst on Apple's Corporate Social Responsibility team, you are tasked with evaluating the effectiveness of recent philanthropic initiatives. Your focus is on understanding participant engagement across different communities and programs.

## Database Schema

### fct_philanthropic_initiatives
- program_id: INTEGER
- community_id: INTEGER
- event_date: DATE
- participants: INTEGER
- program_name: VARCHAR

### dim_community
- community_id: INTEGER
- community_name: VARCHAR
- region: VARCHAR

---

## Question 1: Participant Summary by Community and Program (January 2024)

### Problem
Apple's Corporate Social Responsibility team wants a summary report of philanthropic initiatives in January 2024. Please compile a report that aggregates participant numbers by community and by program.

### Expected Output
- program_name
- community_name
- participants_count

### Solution

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

---

## Question 2: Program Details with Earliest Event Date (February 2024)

### Problem
The team is reviewing the execution of February 2024 philanthropic programs. For each initiative, provide details along with the earliest event date recorded within each program campaign to understand start timings.

### Expected Output
- program_id
- program_name
- event_date
- min_event_date

### Solution

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
- Uses window function MIN(event_date) OVER (PARTITION BY program_name)
- Partitions by program_name to find earliest date within each program
- Shows every event row with its program's minimum date for comparison
- No GROUP BY - returns all detail rows

---

## Question 3: Maximum Participation by Program (First Week of March 2024)

### Problem
For a refined analysis of initiatives held during the first week of March 2024, include for each program the maximum participation count recorded in any event.

### Expected Output
- program_id
- program_name
- max_participants

### Solution

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
