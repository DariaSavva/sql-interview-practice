# Case 19: Stripe Connect Payment Performance Analysis

## Business Context

As a Data Analyst on the Stripe Connect team, you are tasked with evaluating payment performance across various marketplace segments. Your team is focused on understanding how payout fees and compliance costs differ between platform types to identify opportunities for optimizing fee structures and compliance strategies. Your goal is to provide insights that will help streamline costs and improve operational efficiency across segments.

## Database Schema

### fct_transactions
- transaction_id: INTEGER
- marketplace_segment: VARCHAR
- transaction_date: DATE
- payout_fee: DECIMAL
- compliance_cost: DECIMAL

---

## Question 1: Average Payout Fee per Marketplace Segment (April 2024)

### Problem
What is the average payout fee per transaction for each marketplace segment during April 2024? This result will provide an initial benchmark for fee structures by segment.

### Expected Output
- marketplace_segment
- avg_payout_fee

### Solution

```sql
SELECT marketplace_segment,
       AVG(payout_fee) AS avg_payout_fee
FROM fct_transactions
WHERE transaction_date BETWEEN '2024-04-01' AND '2024-04-30'
GROUP BY marketplace_segment;
```

**Explanation:**
- Filters to April 2024 using BETWEEN (inclusive on both ends)
- Groups by marketplace_segment to calculate averages per segment
- Uses AVG() to compute mean payout fee for each segment
- Provides benchmark for comparing fee structures across segments

---

## Question 2: Total Compliance Costs per Marketplace Segment (April 2024)

### Problem
For each marketplace segment, what is the total compliance costs in April 2024? This metric will help evaluate the efficiency of current compliance strategies.

### Expected Output
- marketplace_segment
- total_cost

### Solution

```sql
SELECT marketplace_segment,
       SUM(compliance_cost) AS total_cost
FROM fct_transactions
WHERE transaction_date BETWEEN '2024-04-01' AND '2024-04-30'
GROUP BY marketplace_segment;
```

**Explanation:**
- Filters to April 2024 using BETWEEN
- Groups by marketplace_segment
- Uses SUM() to aggregate all compliance costs per segment
- Shows total compliance overhead for each segment

---

## Question 3: Lowest Fee and Compliance Cost Segment (April 2024)

### Problem
We know that one of the marketplace segments exhibited both the lowest total compliance overhead and the lowest average payout fee structures in April 2024. Can you identify which one it is?

### Expected Output
- marketplace_segment
- avg_fee
- total_cost

### Solution

```sql
WITH apr_segments AS (
    SELECT marketplace_segment,
           AVG(payout_fee) AS avg_fee,
           SUM(compliance_cost) AS total_cost
    FROM fct_transactions
    WHERE transaction_date BETWEEN '2024-04-01' AND '2024-04-30'
    GROUP BY marketplace_segment
),
ranking AS (
    SELECT marketplace_segment,
           avg_fee,
           total_cost,
           ROW_NUMBER() OVER (ORDER BY avg_fee ASC, total_cost ASC) AS rn
    FROM apr_segments
)
SELECT marketplace_segment,
       avg_fee, 
       total_cost
FROM ranking
WHERE rn = 1;
```

**Explanation:**
- First CTE calculates both average payout fee and total compliance cost per segment
- Second CTE applies ROW_NUMBER() with multi-criteria ordering
- Orders by avg_fee ASC (lowest first), then total_cost ASC (lowest first)
- This prioritizes segments with lowest fees, using compliance cost as tie-breaker
- Main query filters to rn = 1 to get the segment with both lowest metrics
- Identifies the most cost-efficient segment for optimization insights
