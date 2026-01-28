# Case 06: WhatsApp User Engagement Analysis

## 📊 Business Context

You are a **Data Scientist on the WhatsApp consumer experience team** focusing on enhancing user interaction with call and group chat features. Your team aims to understand user engagement patterns with family-focused group chats, average call durations, and group chat participation levels. The end goal is to simplify the user interface and interaction flows based on these insights.

## 🗄️ Database Schema

### `fct_user_calls`
| Column | Type | Description |
|--------|------|-------------|
| user_id | INTEGER | User identifier |
| call_id | INTEGER | Unique call identifier |
| call_duration | INTEGER | Duration of call (in seconds/minutes) |
| call_date | DATE | Date of the call |

### `fct_group_chats`
| Column | Type | Description |
|--------|------|-------------|
| chat_id | INTEGER | Unique group chat identifier |
| user_id | INTEGER | User identifier |
| chat_name | VARCHAR | Name of the group chat |
| chat_creation_date | DATE | Date when chat was created |

---

## Question 1: Family Group Chat Users (April 2024)

### 📋 Problem
How many **distinct users** have participated in group chats with names containing the word **"family"**, where the chat was created in **April 2024**? 

This analysis will inform product managers about user engagement trends with family-focused chat groups.

### Expected Output
- Count of distinct users

---

### ✅ Solution 1 (Original)

```sql
SELECT COUNT(DISTINCT user_id) AS user_count
FROM fct_group_chats
WHERE chat_creation_date >= '2024-04-01' 
  AND chat_creation_date < '2024-05-01'
  AND LOWER(chat_name) LIKE '%family%';
```

**Explanation:**
- Uses `COUNT(DISTINCT user_id)` to count unique users
- Filters to April 2024 with date range
- `LOWER(chat_name) LIKE '%family%'` finds "family" in any case (Family, FAMILY, family)
- `%` wildcard matches any characters before/after "family"

**Pros:** Clear, case-insensitive, standard SQL pattern matching  
**Cons:** `LOWER()` function may not use indexes efficiently

---

### 🔄 Alternative Solution: Using ILIKE (PostgreSQL-specific)

```sql
SELECT COUNT(DISTINCT user_id) AS user_count
FROM fct_group_chats
WHERE chat_creation_date >= DATE '2024-04-01' 
  AND chat_creation_date < DATE '2024-05-01'
  AND chat_name ILIKE '%family%';
```

**Pros:** More concise, PostgreSQL's `ILIKE` is case-insensitive by default  
**Cons:** PostgreSQL-specific (not portable to other databases)

---

## Question 2: Average Total Call Duration per User (May 2024)

### 📋 Problem
To better understand user call behavior, we want to analyze the **total call duration per user** in May 2024. What is the **average total call duration across all users**?

### Expected Output
- `user_id`
- `total_call_duration`
- `avg_call_all_users`

---

### ✅ Solution 2 (Original)

```sql
WITH may_calls AS (
    SELECT user_id,
           SUM(call_duration) AS total_call_duration
    FROM fct_user_calls
    WHERE call_date >= '2024-05-01'
      AND call_date < '2024-06-01'
    GROUP BY user_id
)
SELECT user_id,
       total_call_duration,
       AVG(total_call_duration) OVER () AS avg_call_all_users
FROM may_calls;
```

**Explanation:**
- CTE aggregates total call duration per user for May 2024
- Main query uses `AVG() OVER ()` window function
- Window function calculates average across all users without GROUP BY
- Returns each user's total with the overall average for comparison

**Pros:** Shows per-user totals alongside overall average, window function is elegant  
**Cons:** Returns all users (could be many rows if only summary needed)

---

### 🔄 Alternative Solution: Summary Statistics Only

```sql
WITH may_calls AS (
    SELECT user_id,
           SUM(call_duration) AS total_call_duration
    FROM fct_user_calls
    WHERE call_date >= DATE '2024-05-01'
      AND call_date < DATE '2024-06-01'
    GROUP BY user_id
)
SELECT 
    COUNT(user_id) AS total_users,
    AVG(total_call_duration) AS avg_call_all_users,
    MIN(total_call_duration) AS min_call_duration,
    MAX(total_call_duration) AS max_call_duration
FROM may_calls;
```

**Pros:** Single summary row, provides additional context (min/max/count)  
**Cons:** Doesn't show individual user data

---

## Question 3: Max and Average Group Chat Participation (Q2 2024)

### 📋 Problem
What is the **maximum number of group chats** any user has participated in during the **second quarter of 2024** and how does this compare to the **average participation rate**? 

This insight will guide decisions on simplifying the chat interface for both heavy and average users.

### Expected Output
- `max_group_chats_per_user`
- `avg_group_chats_per_user`

---

### ✅ Solution 3 (Original)

```sql
WITH q2_chats AS (
    SELECT user_id,
           COUNT(chat_id) AS total_chats
    FROM fct_group_chats
    WHERE chat_creation_date >= '2024-04-01' 
      AND chat_creation_date < '2024-07-01'
    GROUP BY user_id
)
SELECT MAX(total_chats) AS max_group_chats_per_user,
       AVG(total_chats) AS avg_group_chats_per_user
FROM q2_chats;
```

**Explanation:**
- CTE counts group chats per user for Q2 2024 (April, May, June)
- Main query finds maximum chats (power user) and average chats (typical user)
- Uses date range for Q2: April 1 to June 30
- Simple aggregations on the per-user counts

**Pros:** Clear two-step process, easy to understand  
**Cons:** None - this is a solid solution

---

### 🔄 Alternative Solution: With Additional Percentiles

```sql
WITH q2_chats AS (
    SELECT user_id,
           COUNT(chat_id) AS total_chats
    FROM fct_group_chats
    WHERE chat_creation_date >= DATE '2024-04-01' 
      AND chat_creation_date < DATE '2024-07-01'
    GROUP BY user_id
)
SELECT 
    MAX(total_chats) AS max_group_chats_per_user,
    AVG(total_chats) AS avg_group_chats_per_user,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_chats) AS median_chats,
    PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY total_chats) AS percentile_90th
FROM q2_chats;
```

**Pros:** Provides fuller picture of distribution (median and 90th percentile)  
**Cons:** More complex output, may be more than needed for simple comparison

---

## 📊 Key Takeaways

### String Pattern Matching in PostgreSQL
- **LIKE**: Case-sensitive pattern matching (`'%family%'`)
- **ILIKE**: Case-insensitive pattern matching (PostgreSQL-specific)
- **LOWER() + LIKE**: Portable case-insensitive approach
- **Performance**: `ILIKE` is optimized in PostgreSQL, but neither may use indexes efficiently

### Window Functions vs Aggregations
- **Window Function**: `AVG() OVER ()` - computes across all rows without collapsing
- **Regular Aggregation**: `AVG()` - collapses to single row
- **Use Case**: Window functions great for showing detail + summary in same result

### Date Range Best Practices
- **Standard Pattern**: `>= '2024-04-01' AND < '2024-05-01'` 
- **Explicit Casting**: `DATE '2024-04-01'` for clarity
- **Q2 2024**: April 1 to June 30 = `>= '2024-04-01' AND < '2024-07-01'`

### CTE Benefits for Multi-Step Analysis
1. **Question 2**: Aggregate per user → Calculate overall average
2. **Question 3**: Count per user → Find max and average
3. **Pattern**: First aggregate at granular level, then summarize

### Performance Considerations
- **Question 1**: String matching with `LIKE` can be slow on large datasets
- **Question 2**: Window function efficient for showing all users with average
- **Question 3**: Two-step aggregation (CTE then aggregate) is standard and efficient

### Business Insights
- **Q1**: Measures adoption of family-focused features
- **Q2**: Identifies typical call behavior and outliers
- **Q3**: Segments users (power users vs average users) for UX optimization

### User Segmentation Strategies
From Question 3 results, you can segment users:
- **Power Users**: Near or at max_group_chats_per_user
- **Average Users**: Around avg_group_chats_per_user
- **Light Users**: Below average

This informs interface design:
- Power users need advanced features
- Average users need streamlined experience
- Light users need simple onboarding

### SQL Techniques Covered
- Pattern matching with LIKE/ILIKE
- LOWER() for case-insensitive comparison
- COUNT(DISTINCT) for unique counts
- Window functions without PARTITION BY
- CTEs for two-step aggregations
- Date range filtering for quarters
- MAX and AVG aggregations
- Statistical functions (PERCENTILE_CONT)

### Common Patterns
1. **Filter → Aggregate → Summarize**: Used in all 3 questions
2. **Date-based analysis**: Consistent across questions
3. **Per-user metrics**: Central to understanding engagement
4. **Comparison metrics**: Max vs average, individual vs overall

### Practical Applications
- **Product Development**: Understand feature usage patterns
- **UX Design**: Tailor interface to different user segments
- **User Research**: Identify power users for beta testing
- **Marketing**: Target family-focused campaigns based on Q1 insights
- **Infrastructure**: Plan capacity based on call duration patterns
