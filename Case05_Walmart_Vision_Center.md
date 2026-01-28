# Case 05: Walmart Vision Center Eyewear Analysis

## 📊 Business Context

You are a **Data Analyst for the Walmart Vision Center team** focused on understanding customer preferences and product performance in eyewear. Your team is interested in analyzing how different styles and price points influence customer satisfaction and sales volume. By leveraging data, you aim to identify the most popular styles, assess customer satisfaction for top-selling items, and determine the best-performing product combinations to inform strategic decisions on inventory and pricing.

## 🗄️ Database Schema

### `fct_sales`
| Column | Type | Description |
|--------|------|-------------|
| sale_id | INTEGER | Unique sale transaction identifier |
| sale_date | DATE | Date of the sale |
| product_id | INTEGER | Foreign key to dim_products |
| quantity_sold | INTEGER | Number of units sold in transaction |
| sale_amount | DECIMAL | Total dollar amount of sale |
| customer_satisfaction | DECIMAL | Customer satisfaction rating for this sale |

### `dim_products`
| Column | Type | Description |
|--------|------|-------------|
| product_id | INTEGER | Unique product identifier |
| style | VARCHAR | Eyewear style (e.g., aviator, cat-eye, rectangular) |
| price | DECIMAL | Product price |

---

## Question 1: Top 3 Eyewear Styles by Units Sold (January 2024)

### 📋 Problem
Using sales data from **January 2024**, what are the **top 3 eyewear styles** by total units sold and what is the total number of units sold for each style? 

This query helps identify the most popular styles among customers.

### Expected Output
- `style`
- `total_units_sold`

Ordered by `total_units_sold` DESC

---

### ✅ Solution 1 (Original)

```sql
WITH jan_sales AS (
    SELECT style,
           SUM(quantity_sold) AS total_units_sold
    FROM fct_sales f
    LEFT JOIN dim_products d
        ON f.product_id = d.product_id
    WHERE sale_date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY style
),
ranked_styles AS (
    SELECT style, 
           total_units_sold,
           DENSE_RANK() OVER (ORDER BY total_units_sold DESC) AS "rank"
    FROM jan_sales 
)
SELECT style,
       total_units_sold
FROM ranked_styles
WHERE "rank" <= 3
ORDER BY total_units_sold DESC;
```

**Explanation:**
- First CTE aggregates total units sold per style for January
- Uses LEFT JOIN to include all sales (even if product info is missing)
- Second CTE applies DENSE_RANK to rank styles by units sold
- DENSE_RANK ensures no gaps if there are ties (e.g., 1, 2, 2, 3 not 1, 2, 2, 4)
- Filters to top 3 ranks
- Orders by units sold descending

**Pros:** DENSE_RANK handles ties well, returns top 3 even with ties  
**Cons:** Two CTEs add slight overhead, LEFT JOIN may not be necessary

---

### 🔄 Alternative Solution 1A: Using LIMIT (Simpler)

```sql
SELECT d.style,
       SUM(f.quantity_sold) AS total_units_sold
FROM fct_sales f
INNER JOIN dim_products d
    ON f.product_id = d.product_id
WHERE f.sale_date >= DATE '2024-01-01'
  AND f.sale_date < DATE '2024-02-01'
GROUP BY d.style
ORDER BY total_units_sold DESC
LIMIT 3;
```

**Pros:** Most concise, single query, very efficient  
**Cons:** Doesn't handle ties gracefully (stops at exactly 3 rows)

---

### 🔄 Alternative Solution 1B: Using ROW_NUMBER

```sql
WITH jan_sales AS (
    SELECT d.style,
           SUM(f.quantity_sold) AS total_units_sold
    FROM fct_sales f
    INNER JOIN dim_products d
        ON f.product_id = d.product_id
    WHERE f.sale_date >= DATE '2024-01-01'
      AND f.sale_date < DATE '2024-02-01'
    GROUP BY d.style
)
SELECT style,
       total_units_sold
FROM (
    SELECT style,
           total_units_sold,
           ROW_NUMBER() OVER (ORDER BY total_units_sold DESC) AS rn
    FROM jan_sales
) ranked
WHERE rn <= 3
ORDER BY total_units_sold DESC;
```

**Pros:** Guarantees exactly 3 rows  
**Cons:** If there's a 4-way tie for 3rd place, only returns one of them

---

### 🔄 Alternative Solution 1C: RANK vs DENSE_RANK Comparison

```sql
WITH jan_sales AS (
    SELECT d.style,
           SUM(f.quantity_sold) AS total_units_sold
    FROM fct_sales f
    INNER JOIN dim_products d
        ON f.product_id = d.product_id
    WHERE f.sale_date >= DATE '2024-01-01'
      AND f.sale_date < DATE '2024-02-01'
    GROUP BY d.style
)
SELECT style,
       total_units_sold,
       RANK() OVER (ORDER BY total_units_sold DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY total_units_sold DESC) AS dense_rank
FROM jan_sales
ORDER BY total_units_sold DESC;
```

**Educational Value:** Shows difference between RANK (1,2,2,4) and DENSE_RANK (1,2,2,3)

---

## Question 2: Customer Rating for Highest-Selling Event per Style (January 2024)

### 📋 Problem
For each eyewear style sold in January 2024, what is the **customer rating for the sale event** that achieved the **highest number of units sold**? 

If there is a tie for highest units sold, return the event with the **highest customer satisfaction rating**.

This query helps us understand if our most active buyers are happy with their purchase.

### Expected Output
- `style`
- `quantity_sold`
- `customer_satisfaction`

---

### ✅ Solution 2 (Original)

```sql
WITH jan_sales AS (
    SELECT d.style,
           quantity_sold,
           customer_satisfaction
    FROM fct_sales f
    LEFT JOIN dim_products d
        ON f.product_id = d.product_id
    WHERE sale_date >= '2024-01-01' AND sale_date < '2024-02-01'
),
ranked_sales AS (
    SELECT s.style,
           quantity_sold,
           customer_satisfaction,
           ROW_NUMBER() OVER (PARTITION BY style 
                              ORDER BY quantity_sold DESC,
                                       customer_satisfaction DESC) AS "rn"
    FROM jan_sales s
)
SELECT r.style,
       quantity_sold,
       customer_satisfaction
FROM ranked_sales r
WHERE "rn" = 1;
```

**Explanation:**
- First CTE gets all January sales with style information
- Second CTE ranks sales within each style
- Orders by quantity_sold DESC (highest first), then customer_satisfaction DESC for tie-breaking
- ROW_NUMBER ensures exactly one row per style
- Final query filters to rank 1 (top sale per style)

**Pros:** Clean tie-breaking logic, guarantees one result per style  
**Cons:** Two CTEs add slight complexity

---

### 🔄 Alternative Solution 2A: DISTINCT ON (PostgreSQL-specific)

```sql
SELECT DISTINCT ON (d.style)
    d.style,
    f.quantity_sold,
    f.customer_satisfaction
FROM fct_sales f
INNER JOIN dim_products d
    ON f.product_id = d.product_id
WHERE f.sale_date >= DATE '2024-01-01' 
  AND f.sale_date < DATE '2024-02-01'
ORDER BY d.style, 
         f.quantity_sold DESC, 
         f.customer_satisfaction DESC;
```

**Pros:** Most concise, PostgreSQL-optimized, single query  
**Cons:** PostgreSQL-specific syntax

---

### 🔄 Alternative Solution 2B: Single CTE Approach

```sql
WITH ranked_sales AS (
    SELECT d.style,
           f.quantity_sold,
           f.customer_satisfaction,
           ROW_NUMBER() OVER (PARTITION BY d.style 
                              ORDER BY f.quantity_sold DESC,
                                       f.customer_satisfaction DESC) AS rn
    FROM fct_sales f
    INNER JOIN dim_products d
        ON f.product_id = d.product_id
    WHERE f.sale_date >= DATE '2024-01-01' 
      AND f.sale_date < DATE '2024-02-01'
)
SELECT style,
       quantity_sold,
       customer_satisfaction
FROM ranked_sales
WHERE rn = 1
ORDER BY style;
```

**Pros:** One fewer CTE, cleaner  
**Cons:** None - this is a solid optimization

---

### 🔄 Alternative Solution 2C: With Additional Context

```sql
WITH ranked_sales AS (
    SELECT d.style,
           f.quantity_sold,
           f.customer_satisfaction,
           f.sale_amount,
           COUNT(*) OVER (PARTITION BY d.style) AS total_sales_for_style,
           ROW_NUMBER() OVER (PARTITION BY d.style 
                              ORDER BY f.quantity_sold DESC,
                                       f.customer_satisfaction DESC) AS rn
    FROM fct_sales f
    INNER JOIN dim_products d
        ON f.product_id = d.product_id
    WHERE f.sale_date >= DATE '2024-01-01' 
      AND f.sale_date < DATE '2024-02-01'
)
SELECT style,
       quantity_sold,
       customer_satisfaction,
       sale_amount,
       total_sales_for_style
FROM ranked_sales
WHERE rn = 1
ORDER BY quantity_sold DESC;
```

**Additional Value:** Shows how many total sales each style had for context

---

## Question 3: Best Performing Style + Price Point Combination (January 2024)

### 📋 Problem
For each unique combination of **eyewear style and price point** in January 2024, calculate the **product performance score**.

**Product performance score** is defined as:
```
total_units_sold × average_customer_satisfaction
```

Which combination has the **highest product performance score**?

**Note**: To derive the price point for each sale, divide `sale_amount` by `quantity_sold`.

This query will inform strategic decisions by identifying the mix of style and price point that best drives both sales and customer satisfaction.

### Expected Output
- `style`
- `price_point`
- `product_performance_score`

---

### ✅ Solution 3 (Original)

```sql
WITH january_sales AS (
    SELECT
        p.style,
        (s.sale_amount / s.quantity_sold) AS price_point,
        s.quantity_sold,
        s.customer_satisfaction
    FROM fct_sales s
    JOIN dim_products p
        ON s.product_id = p.product_id
    WHERE s.sale_date >= DATE '2024-01-01'
      AND s.sale_date < DATE '2024-02-01'
),
aggregated_performance AS (
    SELECT
        style,
        price_point,
        SUM(quantity_sold) AS total_units_sold,
        AVG(customer_satisfaction) AS avg_customer_satisfaction
    FROM january_sales
    GROUP BY style, price_point
),
scored_performance AS (
    SELECT
        style,
        price_point,
        total_units_sold,
        avg_customer_satisfaction,
        total_units_sold * avg_customer_satisfaction AS product_performance_score
    FROM aggregated_performance
)
SELECT
    style,
    price_point,
    product_performance_score
FROM scored_performance
ORDER BY product_performance_score DESC
LIMIT 1;
```

**Explanation:**
- First CTE calculates price point per sale (sale_amount / quantity_sold)
- Second CTE aggregates by style + price_point combination
- Third CTE calculates the performance score
- Final query returns the top-performing combination
- Uses LIMIT 1 to get the highest score

**Pros:** Very clear step-by-step logic, easy to understand each calculation  
**Cons:** Three CTEs could be condensed

---

### 🔄 Alternative Solution 3A: Condensed Two-CTE Approach

```sql
WITH january_sales AS (
    SELECT
        p.style,
        (s.sale_amount / s.quantity_sold) AS price_point,
        s.quantity_sold,
        s.customer_satisfaction
    FROM fct_sales s
    JOIN dim_products p
        ON s.product_id = p.product_id
    WHERE s.sale_date >= DATE '2024-01-01'
      AND s.sale_date < DATE '2024-02-01'
),
performance_scores AS (
    SELECT
        style,
        price_point,
        SUM(quantity_sold) AS total_units_sold,
        AVG(customer_satisfaction) AS avg_customer_satisfaction,
        SUM(quantity_sold) * AVG(customer_satisfaction) AS product_performance_score
    FROM january_sales
    GROUP BY style, price_point
)
SELECT
    style,
    price_point,
    product_performance_score
FROM performance_scores
ORDER BY product_performance_score DESC
LIMIT 1;
```

**Pros:** One fewer CTE, more concise  
**Cons:** Calculation happens in GROUP BY context (still correct)

---

### 🔄 Alternative Solution 3B: Single CTE with Inline Calculation

```sql
WITH performance_scores AS (
    SELECT
        p.style,
        (s.sale_amount / s.quantity_sold) AS price_point,
        SUM(s.quantity_sold) AS total_units_sold,
        AVG(s.customer_satisfaction) AS avg_customer_satisfaction,
        SUM(s.quantity_sold) * AVG(s.customer_satisfaction) AS product_performance_score
    FROM fct_sales s
    JOIN dim_products p
        ON s.product_id = p.product_id
    WHERE s.sale_date >= DATE '2024-01-01'
      AND s.sale_date < DATE '2024-02-01'
    GROUP BY p.style, (s.sale_amount / s.quantity_sold)
)
SELECT
    style,
    price_point,
    product_performance_score
FROM performance_scores
ORDER BY product_performance_score DESC
LIMIT 1;
```

**Pros:** Most concise, single CTE  
**Cons:** Price point calculation in GROUP BY may be less readable

---

### 🔄 Alternative Solution 3C: Top 5 Combinations with Rankings

```sql
WITH january_sales AS (
    SELECT
        p.style,
        (s.sale_amount / s.quantity_sold) AS price_point,
        s.quantity_sold,
        s.customer_satisfaction
    FROM fct_sales s
    JOIN dim_products p
        ON s.product_id = p.product_id
    WHERE s.sale_date >= DATE '2024-01-01'
      AND s.sale_date < DATE '2024-02-01'
),
performance_scores AS (
    SELECT
        style,
        price_point,
        SUM(quantity_sold) AS total_units_sold,
        AVG(customer_satisfaction) AS avg_customer_satisfaction,
        SUM(quantity_sold) * AVG(customer_satisfaction) AS product_performance_score
    FROM january_sales
    GROUP BY style, price_point
)
SELECT
    style,
    ROUND(price_point::numeric, 2) AS price_point,
    total_units_sold,
    ROUND(avg_customer_satisfaction::numeric, 2) AS avg_satisfaction,
    ROUND(product_performance_score::numeric, 2) AS performance_score,
    RANK() OVER (ORDER BY product_performance_score DESC) AS rank
FROM performance_scores
ORDER BY product_performance_score DESC
LIMIT 5;
```

**Additional Value:** Shows top 5 combinations with detailed metrics for strategic analysis

---

### 🔄 Alternative Solution 3D: Using Product Table Price (Alternative Approach)

```sql
WITH performance_scores AS (
    SELECT
        p.style,
        p.price AS price_point,
        SUM(s.quantity_sold) AS total_units_sold,
        AVG(s.customer_satisfaction) AS avg_customer_satisfaction,
        SUM(s.quantity_sold) * AVG(s.customer_satisfaction) AS product_performance_score
    FROM fct_sales s
    JOIN dim_products p
        ON s.product_id = p.product_id
    WHERE s.sale_date >= DATE '2024-01-01'
      AND s.sale_date < DATE '2024-02-01'
    GROUP BY p.style, p.price
)
SELECT
    style,
    price_point,
    product_performance_score
FROM performance_scores
ORDER BY product_performance_score DESC
LIMIT 1;
```

**Note:** Uses product table price instead of calculated price from sales. Choose based on business requirements.

---

## 📊 Key Takeaways

### Ranking Functions Comparison

| Function | Behavior with Ties | Use Case |
|----------|-------------------|----------|
| **ROW_NUMBER()** | 1, 2, 3, 4, 5 (no ties) | Need exactly N rows |
| **RANK()** | 1, 2, 2, 4, 5 (gaps) | Want to show ties but skip ranks |
| **DENSE_RANK()** | 1, 2, 2, 3, 4 (no gaps) | Want to show ties without gaps |

### When to Use Each Ranking Function
- **Top N with no ties**: Use `LIMIT` (simplest)
- **Top N ranks with ties**: Use `DENSE_RANK` (original solution for Q1)
- **Exactly N rows**: Use `ROW_NUMBER()` (Q2)
- **Show all ties clearly**: Use `RANK()`

### Multi-Column Ordering for Tie-Breaking
```sql
ORDER BY primary_column DESC, tiebreaker_column DESC
```
- Question 2 demonstrates this perfectly
- First sorts by quantity_sold, then by customer_satisfaction
- Ensures deterministic results

### Calculated Metrics Best Practices
- **Division safety**: Consider `NULLIF(quantity_sold, 0)` for production
- **Rounding**: Use `ROUND(value::numeric, 2)` for cleaner output
- **Step-by-step CTEs**: Make complex calculations readable
- **Performance scores**: Combine multiple metrics into single KPI

### JOIN Types Matter
- **INNER JOIN**: Only include matching records (recommended for product data)
- **LEFT JOIN**: Include all sales even if product info is missing
- Choose based on data quality expectations

### Date Filtering Consistency
- Original solution uses `BETWEEN '2024-01-01' AND '2024-01-31'`
- Alternative uses `>= '2024-01-01' AND < '2024-02-01'`
- Both work, but the second is more standard and timezone-safe

### Performance Considerations
1. **Question 1**: `LIMIT` approach is fastest for simple top-N
2. **Question 2**: `DISTINCT ON` is fastest in PostgreSQL
3. **Question 3**: Fewer CTEs generally better, but readability matters

### Business Insights
- **Q1**: Identifies most popular styles for inventory planning
- **Q2**: Validates that high-volume sales have good customer satisfaction
- **Q3**: Finds optimal style+price combinations for marketing and merchandising

### Common Patterns in This Case
1. **Aggregate then rank**: Common pattern (Q1, Q2)
2. **Multi-column ranking**: Essential for tie-breaking (Q2)
3. **Calculated metrics**: Derived columns for business KPIs (Q3)
4. **Top-N queries**: Multiple approaches depending on tie handling

### SQL Techniques Covered
- DENSE_RANK vs RANK vs ROW_NUMBER
- PARTITION BY for per-group rankings
- Multi-column ORDER BY for tie-breaking
- Calculated columns in CTEs
- Multiple aggregations (SUM, AVG)
- Composite performance metrics
- LIMIT for top-N results
- PostgreSQL DISTINCT ON optimization
- Division for price point calculation
- Rounding numeric values

### Practical Applications
- **Product Analytics**: Identify best-selling items and categories
- **Customer Satisfaction**: Link satisfaction to purchase patterns
- **Pricing Strategy**: Find optimal price points by segment
- **Inventory Management**: Stock popular style+price combinations
- **Marketing**: Target promotions to high-performing products
