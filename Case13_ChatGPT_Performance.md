# Case 13: ChatGPT Query Performance Analysis

## Business Context

As a Product Analyst on the ChatGPT team, you are tasked with understanding how query complexity affects user engagement and system performance. Your team is particularly interested in identifying which complexity levels provide the best balance between user satisfaction and response efficiency. By analyzing these patterns, your goal is to recommend optimizations for enhancing user experience while maintaining system performance.

## Database Schema

### fct_queries
- query_id: INTEGER
- user_id: INTEGER
- complexity_level: INTEGER
- response_time_seconds: FLOAT
- query_date: DATE

### fct_user_engagement
- query_id: INTEGER
- satisfaction_score: FLOAT

---

## Question 1: Average Response Time by Complexity Level (January 2024)

### Problem
What is the average response time for each query complexity level in January 2024?

### Expected Output
- complexity_level
- avg_response_time

### Solution

```sql
SELECT complexity_level,
       AVG(response_time_seconds) AS avg_response_time
FROM fct_queries 
WHERE query_date >= '2024-01-01' 
  AND query_date < '2024-02-01'
GROUP BY complexity_level;
```

**Explanation:**
- Filters to January 2024 using date range
- Groups by complexity_level to calculate averages per level
- Uses AVG() to compute mean response time for each complexity level

---

## Question 2: Average Satisfaction Score for Slow Queries (January 2024)

### Problem
For each query complexity level, what is the average user satisfaction score for queries that took more than 2 seconds to respond in January 2024?

### Expected Output
- complexity_level
- avg_satisf_score

### Solution

```sql
SELECT complexity_level,
       AVG(satisfaction_score) AS avg_satisf_score
FROM fct_queries q
JOIN fct_user_engagement u
    ON q.query_id = u.query_id
WHERE query_date >= '2024-01-01' 
  AND query_date < '2024-02-01' 
  AND response_time_seconds > 2.0
GROUP BY complexity_level;
```

**Explanation:**
- Joins queries table with engagement table on query_id
- Filters to January 2024 and queries with response time over 2 seconds
- Groups by complexity_level
- Calculates average satisfaction score per complexity level for slow queries

---

## Question 3: Optimal Complexity Level (January 2024)

### Problem
We want to identify the complexity level that optimizes both user engagement and system efficiency. Rank average user satisfaction (high to low) and average response time (low to high) across different complexity levels in January 2024. Which level has the best average satisfaction and response time ranking?

### Expected Output
- complexity_level
- avg_satisf_score
- avg_response_time

### Solution

```sql
WITH jan_levels AS (
    SELECT complexity_level,
           AVG(satisfaction_score) AS avg_satisf_score,
           AVG(response_time_seconds) AS avg_response_time
    FROM fct_queries q
    JOIN fct_user_engagement u
        ON q.query_id = u.query_id
    WHERE query_date >= '2024-01-01' 
      AND query_date < '2024-02-01'
    GROUP BY complexity_level  
),
ranked_levels AS (  
    SELECT complexity_level,
           avg_satisf_score,
           avg_response_time,
           RANK() OVER (ORDER BY avg_satisf_score DESC, avg_response_time ASC) AS rn
    FROM jan_levels
)
SELECT complexity_level,
       avg_satisf_score,
       avg_response_time
FROM ranked_levels
WHERE rn = 1;
```

**Explanation:**
- First CTE calculates average satisfaction and response time per complexity level
- Joins queries with engagement to get satisfaction scores
- Second CTE ranks complexity levels using RANK()
- Orders by satisfaction DESC (higher is better) and response time ASC (lower is better)
- Multi-criteria ranking prioritizes satisfaction first, then response time for ties
- Final query filters to rank 1 (best performing complexity level)
