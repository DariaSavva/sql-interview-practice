# Case 03: Amazon Prime Member Engagement Analysis

## Business Context

As a Data Analyst on the Amazon Prime product analytics team, you are tasked with evaluating Prime member engagement with exclusive promotions. Your team is focused on understanding how members interact with special deals and early product access. The goal is to identify engagement patterns and target highly engaged members to enhance member value and drive higher engagement with these offerings.

## Database Schema

### fct_prime_deals
- deal_id: INTEGER
- member_id: INTEGER
- purchase_amount: DECIMAL
- purchase_date: DATE

---

## Question 1: January 2024 Member Engagement Metrics

### Problem
To assess the popularity of promotions among Prime members, answer the following:
- How many Prime members purchased deals in January 2024?
- What is the average number of deals purchased per member?

### Expected Output
- member_count
- avg_deals_per_member

### Solution

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
- Uses BETWEEN for date filtering (inclusive on both ends)
- Main query counts distinct members and averages their deal counts

---

## Question 2: Member Distribution by Purchase Frequency (February 2024)

### Problem
To gain insights into purchase patterns, what is the distribution of members based on the number of deals purchased in February 2024? Group the members into the following categories: 1-2 deals, 3-5 deals, and more than 5 deals.

### Expected Output
- members_distribution
- num_members

### Solution

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

---

## Question 3: Highly Engaged Members (Q1 2024)

### Problem
To target highly engaged members for tailored promotions, identify Prime members who purchased more than 5 exclusive deals between January 1st and March 31st, 2024. How many such members are there and what is their average total spend on these deals?

### Expected Output
- member_count
- avg_spend

### Solution

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
