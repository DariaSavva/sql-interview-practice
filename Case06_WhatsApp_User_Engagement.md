# Case 06: WhatsApp User Engagement Analysis

## Business Context

You are a Data Scientist on the WhatsApp consumer experience team focusing on enhancing user interaction with call and group chat features. Your team aims to understand user engagement patterns with family-focused group chats, average call durations, and group chat participation levels.

## Database Schema

### fct_user_calls
- user_id: INTEGER
- call_id: INTEGER
- call_duration: INTEGER
- call_date: DATE

### fct_group_chats
- chat_id: INTEGER
- user_id: INTEGER
- chat_name: VARCHAR
- chat_creation_date: DATE

---

## Question 1: Family Group Chat Users (April 2024)

### Problem
How many distinct users have participated in group chats with names containing the word "family", where the chat was created in April 2024?

### Expected Output
- user_count

### Solution

```sql
SELECT COUNT(DISTINCT user_id) AS user_count
FROM fct_group_chats
WHERE chat_creation_date >= '2024-04-01' 
  AND chat_creation_date < '2024-05-01'
  AND LOWER(chat_name) LIKE '%family%';
```

**Explanation:**
- Uses COUNT(DISTINCT user_id) to count unique users
- Filters to April 2024 with date range
- LOWER(chat_name) LIKE '%family%' finds "family" in any case

---

## Question 2: Average Total Call Duration per User (May 2024)

### Problem
To better understand user call behavior, we want to analyze the total call duration per user in May 2024. What is the average total call duration across all users?

### Expected Output
- user_id
- total_call_duration
- avg_call_all_users

### Solution

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
- Main query uses AVG() OVER () window function
- Window function calculates average across all users without GROUP BY
- Returns each user's total with the overall average for comparison

---

## Question 3: Max and Average Group Chat Participation (Q2 2024)

### Problem
What is the maximum number of group chats any user has participated in during the second quarter of 2024 and how does this compare to the average participation rate?

### Expected Output
- max_group_chats_per_user
- avg_group_chats_per_user

### Solution

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
