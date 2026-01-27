# Case 03: Amazon Prime Member Engagement Analysis

## 📊 Business Context

As a **Data Analyst on the Amazon Prime product analytics team**, you are tasked with evaluating Prime member engagement with exclusive promotions. Your team is focused on understanding how members interact with special deals and early product access. The goal is to identify engagement patterns and target highly engaged members to enhance member value and drive higher engagement with these offerings.

## 🗄️ Database Schema

### `fct_prime_deals`
| Column | Type | Description |
|--------|------|-------------|
| deal_id | INTEGER | Unique deal transaction identifier |
| member_id | INTEGER | Prime member identifier |
| purchase_amount | DECIMAL | Amount spent on the deal |
| purchase_date | DATE | Date of deal purchase |

---

## Question 1: January 2024 Member Engagement Metrics

### 📋 Problem
To assess the **popularity of promotions** among Prime members, answer the following:
- How many Prime members purchased deals in January 2024?
- What is the average number of deals purchased per member?

### Expected Output
- `member_count` (total members who purchased)
- `avg_deals_per_member` (average deals per member)

---

### ✅ Solution 1 (Original)

```sql
WITH deals_per_member AS (
    SELECT member_id,
           COUNT(*) AS number_of_deals
    FROM fct_prime_deals
    WHERE purchase_date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY member_id
)
SELECT COUNT(dpm.member_id) AS member_count,
       AVG(dpm.number_of_deals) AS avg_deals_per_member
FROM deals_per_member dpm;
```

**Explanation:**
- CTE aggregates deals per member for January 2024
- Uses `BETWEEN` for date filtering (inclusive on both ends)
- Main query counts distinct members and averages their deal counts
- Clean separation of concerns with CTE

**Pros:** Clear logic, easy to understand  
**Cons:** BETWEEN includes both endpoints (need to ensure Jan 31 23:59:59 is included)

---

### 🔄 Alternative Solution 1A: Using Date Range (More Precise)

```sql
WITH deals_per_member AS (
    SELECT member_id,
           COUNT(*) AS number_of_deals
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-01-01'
      AND purchase_date < DATE '2024-02-01'
    GROUP BY member_id
)
SELECT COUNT(dpm.member_id) AS member_count,
       AVG(dpm.number_of_deals) AS avg_deals_per_member
FROM deals_per_member dpm;
```

**Pros:** No ambiguity with end date, standard date range pattern  
**Cons:** Slightly more verbose

---

### 🔄 Alternative Solution 1B: Without CTE (Single Query)

```sql
SELECT 
    COUNT(DISTINCT member_id) AS member_count,
    COUNT(*)::DECIMAL / COUNT(DISTINCT member_id) AS avg_deals_per_member
FROM fct_prime_deals
WHERE purchase_date >= DATE '2024-01-01'
  AND purchase_date < DATE '2024-02-01';
```

**Pros:** More concise, single pass through data  
**Cons:** Less readable, manual average calculation

---

### 🔄 Alternative Solution 1C: With Additional Metrics

```sql
WITH deals_per_member AS (
    SELECT member_id,
           COUNT(*) AS number_of_deals,
           SUM(purchase_amount) AS total_spent
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-01-01'
      AND purchase_date < DATE '2024-02-01'
    GROUP BY member_id
)
SELECT 
    COUNT(dpm.member_id) AS member_count,
    AVG(dpm.number_of_deals) AS avg_deals_per_member,
    AVG(dpm.total_spent) AS avg_spend_per_member,
    MIN(dpm.number_of_deals) AS min_deals,
    MAX(dpm.number_of_deals) AS max_deals
FROM deals_per_member dpm;
```

**Additional Value:** Provides broader engagement context with spend and range metrics

---

## Question 2: Member Distribution by Purchase Frequency (February 2024)

### 📋 Problem
To gain insights into **purchase patterns**, what is the distribution of members based on the number of deals purchased in February 2024? 

Group members into the following categories:
- **1-2 deals**
- **3-5 deals**
- **More than 5 deals** (> 5 deals)

### Expected Output
- `members_distribution` (category label)
- `num_members` (count of members in each category)

---

### ✅ Solution 2 (Original)

```sql
WITH feb_deals AS (
    SELECT member_id,
           COUNT(*) AS deals_purchased
    FROM fct_prime_deals
    WHERE purchase_date BETWEEN '2024-02-01' AND '2024-02-29'
    GROUP BY member_id
)
SELECT 
    CASE 
        WHEN fd.deals_purchased <= 2 THEN '1-2 deals'
        WHEN fd.deals_purchased <= 5 THEN '3-5 deals'
        ELSE '> 5 deals'
    END AS members_distribution,
    COUNT(*) AS num_members      
FROM feb_deals fd
GROUP BY CASE 
    WHEN fd.deals_purchased <= 2 THEN '1-2 deals'
    WHEN fd.deals_purchased <= 5 THEN '3-5 deals'
    ELSE '> 5 deals'
END
ORDER BY members_distribution;
```

**Explanation:**
- CTE counts deals per member in February 2024
- Main query buckets members into 3 categories using CASE
- Groups by the same CASE expression to count members per bucket
- Orders alphabetically by distribution label

**Pros:** Clear bucketing logic, readable CASE statement  
**Cons:** CASE expression repeated in GROUP BY (required in standard SQL)

---

### 🔄 Alternative Solution 2A: Ordering by Bucket Logic

```sql
WITH feb_deals AS (
    SELECT member_id,
           COUNT(*) AS deals_purchased
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-02-01'
      AND purchase_date < DATE '2024-03-01'
    GROUP BY member_id
)
SELECT 
    CASE 
        WHEN fd.deals_purchased <= 2 THEN '1-2 deals'
        WHEN fd.deals_purchased <= 5 THEN '3-5 deals'
        ELSE '> 5 deals'
    END AS members_distribution,
    COUNT(*) AS num_members      
FROM feb_deals fd
GROUP BY CASE 
    WHEN fd.deals_purchased <= 2 THEN '1-2 deals'
    WHEN fd.deals_purchased <= 5 THEN '3-5 deals'
    ELSE '> 5 deals'
END
ORDER BY 
    CASE 
        WHEN MAX(fd.deals_purchased) <= 2 THEN 1
        WHEN MAX(fd.deals_purchased) <= 5 THEN 2
        ELSE 3
    END;
```

**Pros:** Logical ordering (1-2, 3-5, >5) instead of alphabetical  
**Cons:** More complex ORDER BY

---

### 🔄 Alternative Solution 2B: Using Subquery for Cleaner CASE

```sql
WITH feb_deals AS (
    SELECT member_id,
           COUNT(*) AS deals_purchased
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-02-01'
      AND purchase_date < DATE '2024-03-01'
    GROUP BY member_id
),
categorized AS (
    SELECT 
        CASE 
            WHEN deals_purchased <= 2 THEN '1-2 deals'
            WHEN deals_purchased <= 5 THEN '3-5 deals'
            ELSE '> 5 deals'
        END AS members_distribution
    FROM feb_deals
)
SELECT 
    members_distribution,
    COUNT(*) AS num_members
FROM categorized
GROUP BY members_distribution
ORDER BY members_distribution;
```

**Pros:** CASE expression only written once, more maintainable  
**Cons:** Extra CTE level

---

### 🔄 Alternative Solution 2C: With Percentage Distribution

```sql
WITH feb_deals AS (
    SELECT member_id,
           COUNT(*) AS deals_purchased
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-02-01'
      AND purchase_date < DATE '2024-03-01'
    GROUP BY member_id
),
distribution AS (
    SELECT 
        CASE 
            WHEN deals_purchased <= 2 THEN '1-2 deals'
            WHEN deals_purchased <= 5 THEN '3-5 deals'
            ELSE '> 5 deals'
        END AS members_distribution,
        COUNT(*) AS num_members
    FROM feb_deals
    GROUP BY CASE 
        WHEN deals_purchased <= 2 THEN '1-2 deals'
        WHEN deals_purchased <= 5 THEN '3-5 deals'
        ELSE '> 5 deals'
    END
)
SELECT 
    members_distribution,
    num_members,
    ROUND(100.0 * num_members / SUM(num_members) OVER (), 2) AS percentage
FROM distribution
ORDER BY members_distribution;
```

**Additional Value:** Shows percentage of total members in each bucket for better insights

---

## Question 3: Highly Engaged Members (Q1 2024)

### 📋 Problem
To target **highly engaged members** for tailored promotions, identify Prime members who purchased **more than 5 exclusive deals** between January 1st and March 31st, 2024.

Answer the following:
- How many such members are there?
- What is their average total spend on these deals?

### Expected Output
- `member_count` (count of highly engaged members)
- `avg_spend` (average total spend per member)

---

### ✅ Solution 3 (Original)

```sql
WITH jan_march_deals AS (
    SELECT member_id,
           COUNT(*) AS num_deals,
           SUM(purchase_amount) AS total_spend
    FROM fct_prime_deals
    WHERE purchase_date BETWEEN '2024-01-01' AND '2024-03-31'
    GROUP BY member_id
)
SELECT COUNT(jm.member_id) AS member_count,
       AVG(total_spend) AS avg_spend
FROM jan_march_deals jm
WHERE jm.num_deals > 5;
```

**Explanation:**
- CTE aggregates deals and spend per member for Q1 2024
- Filters to members with more than 5 deals
- Calculates count of these members and their average spend
- Uses BETWEEN for date range (inclusive)

**Pros:** Clear logic, readable, efficient filtering  
**Cons:** None - this is a solid solution!

---

### 🔄 Alternative Solution 3A: Using HAVING Clause

```sql
WITH jan_march_deals AS (
    SELECT member_id,
           COUNT(*) AS num_deals,
           SUM(purchase_amount) AS total_spend
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-01-01'
      AND purchase_date < DATE '2024-04-01'
    GROUP BY member_id
    HAVING COUNT(*) > 5
)
SELECT COUNT(member_id) AS member_count,
       AVG(total_spend) AS avg_spend
FROM jan_march_deals;
```

**Pros:** Filters during aggregation (potentially more efficient)  
**Cons:** Slight change in query structure

---

### 🔄 Alternative Solution 3B: Without CTE

```sql
SELECT 
    COUNT(DISTINCT member_id) AS member_count,
    AVG(total_spend) AS avg_spend
FROM (
    SELECT member_id,
           SUM(purchase_amount) AS total_spend
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-01-01'
      AND purchase_date < DATE '2024-04-01'
    GROUP BY member_id
    HAVING COUNT(*) > 5
) highly_engaged;
```

**Pros:** More concise, inline subquery  
**Cons:** Less readable than CTE version

---

### 🔄 Alternative Solution 3C: With Additional Engagement Metrics

```sql
WITH jan_march_deals AS (
    SELECT member_id,
           COUNT(*) AS num_deals,
           SUM(purchase_amount) AS total_spend,
           AVG(purchase_amount) AS avg_deal_value,
           MAX(purchase_date) AS last_purchase_date
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-01-01'
      AND purchase_date < DATE '2024-04-01'
    GROUP BY member_id
    HAVING COUNT(*) > 5
)
SELECT 
    COUNT(member_id) AS member_count,
    AVG(total_spend) AS avg_total_spend,
    AVG(num_deals) AS avg_deals_count,
    AVG(avg_deal_value) AS avg_deal_value,
    MIN(num_deals) AS min_deals,
    MAX(num_deals) AS max_deals
FROM jan_march_deals;
```

**Additional Value:** Comprehensive engagement profile for targeting strategy

---

### 🔄 Alternative Solution 3D: With Member Segmentation

```sql
WITH jan_march_deals AS (
    SELECT member_id,
           COUNT(*) AS num_deals,
           SUM(purchase_amount) AS total_spend
    FROM fct_prime_deals
    WHERE purchase_date >= DATE '2024-01-01'
      AND purchase_date < DATE '2024-04-01'
    GROUP BY member_id
),
highly_engaged AS (
    SELECT *,
           CASE 
               WHEN num_deals > 10 THEN 'Super Engaged'
               WHEN num_deals > 5 THEN 'Highly Engaged'
               ELSE 'Regular'
           END AS engagement_tier
    FROM jan_march_deals
    WHERE num_deals > 5
)
SELECT 
    engagement_tier,
    COUNT(member_id) AS member_count,
    AVG(total_spend) AS avg_spend,
    AVG(num_deals) AS avg_deals
FROM highly_engaged
GROUP BY engagement_tier
ORDER BY engagement_tier DESC;
```

**Additional Value:** Segments highly engaged members into tiers for targeted marketing

---

## 📊 Key Takeaways

### Date Filtering Best Practices
- **BETWEEN vs Date Range**: `BETWEEN '2024-01-01' AND '2024-01-31'` is inclusive on both ends
- **Standard Pattern**: `>= '2024-01-01' AND < '2024-02-01'` is more precise and avoids edge cases
- **Consistency**: Choose one pattern and use it throughout your codebase

### Aggregation Patterns
- **Count distinct members**: `COUNT(DISTINCT member_id)` or CTE approach
- **Average of counts**: Calculate individual counts first, then average
- **HAVING vs WHERE**: Use HAVING for filtering on aggregated values

### Bucketing and Categorization
- **CASE statements**: Essential for creating custom categories
- **Repeating CASE**: Standard SQL requires same CASE in GROUP BY and SELECT
- **Ordering buckets**: Use numeric CASE in ORDER BY for logical ordering

### CTEs for Clarity
- **Question 1**: CTE makes two-step calculation clear (deals per member → overall stats)
- **Question 2**: CTE separates aggregation from categorization
- **Question 3**: CTE enables filtering on aggregated values cleanly

### Business Insights
- **Q1**: Measures overall engagement breadth (how many members) and depth (deals per member)
- **Q2**: Identifies engagement distribution to target different member segments
- **Q3**: Finds VIP members for premium promotions and retention programs

### Performance Considerations
1. **Question 1**: Both CTE and single-query approaches are efficient for this use case
2. **Question 2**: Adding an extra CTE level (Solution 2B) has minimal overhead and improves maintainability
3. **Question 3**: HAVING clause (Solution 3A) can be more efficient than WHERE on CTE results

### Practical Applications
- **Engagement Metrics**: Track active user counts and activity levels
- **Cohort Analysis**: Bucket users by behavior for targeted strategies
- **Loyalty Programs**: Identify high-value customers for VIP treatment
- **Churn Prevention**: Monitor engagement levels to identify at-risk members

### SQL Techniques Covered
- Common Table Expressions (WITH clause)
- Date range filtering (BETWEEN, >= and <)
- CASE statements for bucketing
- GROUP BY with CASE expressions
- HAVING clause for filtered aggregations
- Window functions for percentages (OVER clause)
- Multiple aggregations in single query
