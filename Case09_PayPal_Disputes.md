# Case 09: PayPal Refund Disputes Analysis

## 📊 Business Context

You are a **Product Analyst at PayPal** investigating customer refund dispute characteristics across transaction types. Your team wants to optimize the buyer protection process for more efficient resolutions. The goal is to analyze dispute metrics and develop targeted process improvements.

## 🗄️ Database Schema

### `fct_disputed_transactions`
| Column | Type | Description |
|--------|------|-------------|
| transaction_id | INTEGER | Unique transaction identifier |
| dispute_initiated_date | DATE | Date when dispute was initiated |
| refund_amount | DECIMAL | Amount refunded in dispute |
| resolution_time_days | INTEGER | Days taken to resolve dispute |
| product_type | VARCHAR | Type of product (e.g., DIG001, DIG002 for digital, PHY001 for physical) |
| transaction_category | VARCHAR | Category of transaction |

**Product Type Pattern**: Digital goods product types start with `'DIG'`

---

## Question 1: Average Refund Amount for Digital Goods (Oct 1-7, 2024)

### 📋 Problem
For disputes involving **digital goods** (where the product_type begins with **'DIG'**), what is the **average refund amount** for disputes initiated from **October 1st to October 7th, 2024**? 

This metric helps quantify the financial impact of digital goods disputes.

### Expected Output
- `avg_refund_amount`

---

### ✅ Solution 1 (Original)

```sql
SELECT AVG(refund_amount) AS avg_refund_amount
FROM fct_disputed_transactions
WHERE dispute_initiated_date >= '2024-10-01' 
  AND dispute_initiated_date < '2024-10-08' 
  AND product_type LIKE 'DIG%';
```

**Explanation:**
- Uses `AVG()` to calculate mean refund amount
- Filters to specific week using date range (Oct 1-7)
- Uses `LIKE 'DIG%'` pattern to match digital goods (any product_type starting with 'DIG')
- Single aggregation without grouping returns one value

**Pros:** Simple and straightforward, efficient single-pass query  
**Cons:** None - this is optimal for the requirement

---

### 🔄 Alternative Solution: With Additional Context

```sql
SELECT 
    AVG(refund_amount) AS avg_refund_amount,
    COUNT(*) AS total_disputes,
    SUM(refund_amount) AS total_refund_amount,
    MIN(refund_amount) AS min_refund,
    MAX(refund_amount) AS max_refund
FROM fct_disputed_transactions
WHERE dispute_initiated_date >= DATE '2024-10-01' 
  AND dispute_initiated_date < DATE '2024-10-08' 
  AND product_type LIKE 'DIG%';
```

**Pros:** Provides fuller context (count, sum, range) for business analysis  
**Cons:** More information than requested

---

## Question 2: Highest Average Resolution Time Among Top 5 Categories (October 2024)

### 📋 Problem
For disputes in **October 2024**:
1. First find the **top 5 transaction categories** by the number of disputes
2. Among these top 5 categories, identify the category with the **highest average dispute resolution time**

Report both the category name and its average resolution time to target process improvements.

### Expected Output
- `transaction_category`
- `avg_resol_time`

---

### ✅ Solution 2 (Original)

```sql
WITH top_5 AS (
    SELECT transaction_category,
           COUNT(*) AS dispute_number,
           AVG(resolution_time_days) AS avg_resol_time
    FROM fct_disputed_transactions
    WHERE dispute_initiated_date >= '2024-10-01' 
      AND dispute_initiated_date < '2024-11-01' 
    GROUP BY transaction_category
    ORDER BY COUNT(*) DESC, AVG(resolution_time_days) DESC
    LIMIT 5
),
ranking AS (
    SELECT transaction_category,
           avg_resol_time,
           ROW_NUMBER() OVER (ORDER BY avg_resol_time DESC) AS rn
    FROM top_5
)
SELECT transaction_category,
       avg_resol_time
FROM ranking
WHERE rn = 1;
```

**Explanation:**
- First CTE gets top 5 categories by dispute count
- Orders by COUNT(*) DESC to get highest volume categories
- Secondary sort by AVG(resolution_time_days) DESC for consistent ordering
- LIMIT 5 restricts to top 5 categories
- Second CTE ranks these 5 categories by average resolution time
- Final query filters to rank 1 (highest average resolution time)

**Pros:** Clear two-step logic, handles ties in dispute count gracefully  
**Cons:** Two CTEs add slight overhead

---

### 🔄 Alternative Solution: Single CTE with Direct Ordering

```sql
WITH top_5_categories AS (
    SELECT transaction_category,
           COUNT(*) AS dispute_number,
           AVG(resolution_time_days) AS avg_resol_time
    FROM fct_disputed_transactions
    WHERE dispute_initiated_date >= DATE '2024-10-01' 
      AND dispute_initiated_date < DATE '2024-11-01' 
    GROUP BY transaction_category
    ORDER BY COUNT(*) DESC
    LIMIT 5
)
SELECT transaction_category,
       avg_resol_time
FROM top_5_categories
ORDER BY avg_resol_time DESC
LIMIT 1;
```

**Pros:** Single CTE, more concise  
**Cons:** Functionally equivalent to original

---

## Question 3: Physical Goods Percentage in Highest Resolution Time Quartile (October 2024)

### 📋 Problem
Segment all disputes from **October 2024** into **quartiles** based on the resolution time. What **percentage of disputes** in the **highest resolution time quartile** (4th quartile) involve **physical goods** (i.e. product_type values **not starting with 'DIG'**)? 

This analysis will guide recommendations to reduce overall dispute resolution time.

### Expected Output
- `percent_physical_disputes`

---

### ✅ Solution 3 (Original)

```sql
WITH oct_disputes AS (
    SELECT product_type,
           resolution_time_days,
           NTILE(4) OVER (ORDER BY resolution_time_days) AS ntiles
    FROM fct_disputed_transactions
    WHERE dispute_initiated_date >= '2024-10-01' 
      AND dispute_initiated_date < '2024-11-01' 
)
SELECT
    100.0 * SUM(CASE WHEN product_type NOT LIKE 'DIG%' THEN 1 ELSE 0 END)
          / COUNT(*) AS percent_physical_disputes
FROM oct_disputes
WHERE ntiles = 4;
```

**Explanation:**
- CTE uses `NTILE(4)` to divide disputes into 4 equal-sized groups (quartiles)
- Orders by resolution_time_days so quartile 4 = highest resolution times
- Filters to quartile 4 (ntiles = 4) in main query
- Uses conditional aggregation to count physical goods (NOT LIKE 'DIG%')
- Calculates percentage: (physical goods count / total count) × 100
- Multiplies by 100.0 to get percentage

**Pros:** Clear use of NTILE for quartile segmentation, straightforward percentage calculation  
**Cons:** None - this is a solid solution

---

### 🔄 Alternative Solution: With Breakdown by Product Type

```sql
WITH oct_disputes AS (
    SELECT product_type,
           resolution_time_days,
           NTILE(4) OVER (ORDER BY resolution_time_days) AS quartile,
           CASE 
               WHEN product_type LIKE 'DIG%' THEN 'Digital'
               ELSE 'Physical'
           END AS product_category
    FROM fct_disputed_transactions
    WHERE dispute_initiated_date >= DATE '2024-10-01' 
      AND dispute_initiated_date < DATE '2024-11-01' 
),
quartile_4 AS (
    SELECT 
        product_category,
        COUNT(*) AS dispute_count
    FROM oct_disputes
    WHERE quartile = 4
    GROUP BY product_category
)
SELECT 
    product_category,
    dispute_count,
    100.0 * dispute_count / SUM(dispute_count) OVER () AS percentage
FROM quartile_4
ORDER BY product_category;
```

**Pros:** Shows breakdown for both digital and physical, more detailed analysis  
**Cons:** Returns 2 rows instead of single percentage value

---

## 📊 Key Takeaways

### NTILE Function for Quartile Segmentation
```sql
NTILE(n) OVER (ORDER BY column)
```
- **Purpose**: Divides rows into `n` equal-sized groups
- **Use Cases**: Quartiles (4), quintiles (5), deciles (10), percentiles (100)
- **Ordering**: ORDER BY determines which values go into which group
- **Distribution**: Attempts equal distribution; if not divisible evenly, first groups get extra rows

### Pattern Matching with LIKE
- **Prefix matching**: `LIKE 'DIG%'` - starts with DIG
- **Suffix matching**: `LIKE '%END'` - ends with END
- **Contains**: `LIKE '%MIDDLE%'` - contains MIDDLE
- **Negation**: `NOT LIKE 'DIG%'` - doesn't start with DIG
- **Case sensitivity**: `LIKE` is case-sensitive, `ILIKE` is case-insensitive (PostgreSQL)

### Top-N with Secondary Filtering Pattern
1. **Find top N by one metric** (e.g., top 5 by dispute count)
2. **Among those N, find extreme by another metric** (e.g., highest resolution time)
3. **Pattern**: Use LIMIT for first filter, then rank or order for second

### Percentage Calculations
- **Formula**: `100.0 * numerator / denominator`
- **Multiply by 100.0** (not 100) to ensure floating-point division
- **Safety**: Use NULLIF if denominator could be zero
- **Interpretation**: Result is percentage (e.g., 65.5 = 65.5%)

### Conditional Aggregation Pattern
```sql
SUM(CASE WHEN condition THEN 1 ELSE 0 END)
```
- Counts rows meeting a condition
- Alternative: `COUNT(*) FILTER (WHERE condition)` in PostgreSQL
- Use in percentage calculations: conditional sum / total count

### Date Filtering for Months
- **Full Month**: `>= '2024-10-01' AND < '2024-11-01'`
- **Week Range**: `>= '2024-10-01' AND < '2024-10-08'` (Oct 1-7)
- **Consistency**: Always use `>=` and `<` pattern for clean boundaries

### Performance Considerations
- **Question 1**: Simple aggregation, very efficient
- **Question 2**: Two limits applied (LIMIT 5 then LIMIT 1), efficient
- **Question 3**: NTILE requires full sort, acceptable for monthly data

### Business Insights
- **Q1**: Quantifies financial exposure from digital goods disputes in specific timeframe
- **Q2**: Identifies which high-volume categories have longest resolution times
- **Q3**: Reveals if physical goods dominate the slowest-to-resolve disputes

### Common Patterns in This Case
1. **Prefix pattern matching**: Identifying product types by prefix
2. **Multi-stage filtering**: Top N → Find extreme within that N
3. **Quartile analysis**: Segment by metric, analyze characteristics of segments
4. **Percentage calculations**: Compare subgroups as percentages

### SQL Techniques Covered
- AVG aggregation for means
- LIKE pattern matching with wildcards
- NOT LIKE for negation
- NTILE for equal-sized segmentation
- Multi-CTE queries for complex logic
- ROW_NUMBER for ranking
- LIMIT for top-N queries
- Conditional aggregation with CASE
- Percentage calculations
- Date range filtering

### Practical Applications
- **Financial Analysis**: Track refund amounts by product type
- **Process Optimization**: Identify bottlenecks (long resolution times)
- **Resource Allocation**: Focus on high-volume, high-resolution-time categories
- **Product Strategy**: Compare digital vs physical goods performance
- **Risk Assessment**: Monitor dispute patterns by timeframe
- **Performance Metrics**: Quartile analysis for outlier identification

### NTILE vs Other Functions
| Function | Purpose | Output |
|----------|---------|--------|
| **NTILE(4)** | Equal-sized groups | 1, 2, 3, 4 (quartiles) |
| **PERCENTILE_CONT** | Specific percentile value | Single number |
| **RANK** | Ranking with gaps | 1, 2, 2, 4, 5 |
| **ROW_NUMBER** | Sequential numbering | 1, 2, 3, 4, 5 |

### Advanced Concepts
- **Multi-stage aggregation**: Aggregate → Rank → Filter pattern
- **Quartile interpretation**: Q4 = top 25% by resolution time (slowest)
- **Product categorization**: Using naming conventions (DIG prefix) for segmentation
- **Comparative analysis**: Looking at subgroup percentages within segments
