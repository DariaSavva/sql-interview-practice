# Case 05: Walmart Vision Center Eyewear Analysis

## Business Context

You are a Data Analyst for the Walmart Vision Center team focused on understanding customer preferences and product performance in eyewear. Your team is interested in analyzing how different styles and price points influence customer satisfaction and sales volume.

## Database Schema

### fct_sales
- sale_id: INTEGER
- sale_date: DATE
- product_id: INTEGER
- quantity_sold: INTEGER
- sale_amount: DECIMAL
- customer_satisfaction: DECIMAL

### dim_products
- product_id: INTEGER
- style: VARCHAR
- price: DECIMAL

---

## Question 1: Top 3 Eyewear Styles by Units Sold (January 2024)

### Problem
Using sales data from January 2024, what are the top 3 eyewear styles by total units sold and what is the total number of units sold for each style?

### Expected Output
- style
- total_units_sold

### Solution

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
- Second CTE applies DENSE_RANK to rank styles by units sold
- DENSE_RANK ensures no gaps if there are ties
- Filters to top 3 ranks

---

## Question 2: Customer Rating for Highest-Selling Event per Style (January 2024)

### Problem
For each eyewear style sold in January 2024, what is the customer rating for the sale event that achieved the highest number of units sold? If there is a tie for highest units sold, return the event with the highest customer satisfaction rating.

### Expected Output
- style
- quantity_sold
- customer_satisfaction

### Solution

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
- Orders by quantity_sold DESC, then customer_satisfaction DESC for tie-breaking
- ROW_NUMBER ensures exactly one row per style

---

## Question 3: Best Performing Style + Price Point Combination (January 2024)

### Problem
For each unique combination of eyewear style and price point in January 2024, calculate the product performance score (total_units_sold × average_customer_satisfaction). Which combination has the highest product performance score?

Note: To derive the price point for each sale, divide sale_amount by quantity_sold.

### Expected Output
- style
- price_point
- product_performance_score

### Solution

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
