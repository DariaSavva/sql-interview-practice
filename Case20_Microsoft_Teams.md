# Case 20: Microsoft Teams Collaboration Analysis

## Business Context

You are a Product Analyst on the Microsoft Teams collaboration platform. Your team is focused on enhancing user engagement with chat features to streamline communication and reduce the need for switching between applications. By analyzing user interaction data, you aim to identify key usage patterns and prioritize feature improvements that will drive user satisfaction and efficiency.

## Database Schema

### fct_chat_interactions
- user_id: INTEGER
- message_id: INTEGER
- feature_used: VARCHAR
- interaction_date: DATE

---

## Question 1: Average Chat Messages per User (October 2024)

### Problem
For October 2024, what is the average number of chat messages sent per user? This baseline metric will help us understand overall user engagement.

### Expected Output
- avg_chat_number_per_user

### Solution

```sql
SELECT COUNT(*) / 
       COUNT(DISTINCT user_id) AS avg_chat_number_per_user
FROM fct_chat_interactions
WHERE interaction_date BETWEEN '2024-10-01' AND '2024-10-31';
```

**Explanation:**
- Filters to October 2024 using BETWEEN
- COUNT(*) gives total number of messages
- COUNT(DISTINCT user_id) gives total number of unique users
- Dividing total messages by total users gives average messages per user
- Simple arithmetic division, not using AVG() aggregation

---

## Question 2: Top 5 Chat Features by Unique Users (October 2024)

### Problem
In October 2024, which are the top five chat features ranked by the number of unique users interacting with them? Identifying these features will help prioritize enhancements to reduce context-switching.

### Expected Output
- feature_used
- unique_users

### Solution

```sql
SELECT feature_used,
       COUNT(DISTINCT user_id) AS unique_users
FROM fct_chat_interactions
WHERE interaction_date BETWEEN '2024-10-01' AND '2024-10-31'
GROUP BY feature_used
ORDER BY COUNT(DISTINCT user_id) DESC
LIMIT 5;
```

**Explanation:**
- Filters to October 2024
- Groups by feature_used to analyze each feature separately
- COUNT(DISTINCT user_id) counts unique users per feature
- Orders by unique user count descending to get most popular features first
- LIMIT 5 returns only top 5 features
- Helps identify which features drive most engagement

---

## Question 3: Highly Engaged Users Using Reply Feature (October 2024)

### Problem
For October 2024, what percentage of highly engaged users (those sending more than 50 messages) used the 'reply' feature at least once? This metric directly informs recommendations for feature enhancements that boost engagement and minimize context-switching.

### Expected Output
- pct_highly_engaged_users_using_reply

### Solution

```sql
WITH user_message_count AS (
    SELECT user_id,
           COUNT(*) AS total_messages
    FROM fct_chat_interactions
    WHERE interaction_date BETWEEN '2024-10-01' AND '2024-10-31'
    GROUP BY user_id
),
highly_engaged_users AS (
    SELECT user_id
    FROM user_message_count
    WHERE total_messages > 50
),
reply_users AS (
    SELECT DISTINCT user_id
    FROM fct_chat_interactions
    WHERE interaction_date >= DATE '2024-10-01'
      AND interaction_date < DATE '2024-11-01'
      AND feature_used = 'reply'
)
SELECT COUNT(DISTINCT r.user_id) * 100.0
       / COUNT(DISTINCT h.user_id) AS pct_highly_engaged_users_using_reply
FROM highly_engaged_users h
LEFT JOIN reply_users r
    ON h.user_id = r.user_id;
```

**Explanation:**
- First CTE counts total messages per user in October 2024
- Second CTE identifies highly engaged users (more than 50 messages)
- Third CTE identifies users who used the 'reply' feature at least once
- Main query uses LEFT JOIN to connect highly engaged users with reply users
- LEFT JOIN ensures all highly engaged users are counted in denominator
- COUNT(DISTINCT r.user_id) counts highly engaged users who also used reply
- COUNT(DISTINCT h.user_id) counts all highly engaged users
- Multiplies by 100.0 to get percentage
- Shows feature adoption among power users
