# Case 15: Game Balance Analysis

## Business Context

You are a Product Analyst working with the Game Balance team to evaluate competitive match performance. Your team seeks to understand the current meta of weapon and legend combinations in competitive play. The goal is to identify underutilized and overperforming loadouts to inform potential game balance adjustments.

## Database Schema

### fct_matches
- match_id: INTEGER
- weapon_id: INTEGER
- legend_id: INTEGER
- match_date: DATE

### dim_legends
- legend_id: INTEGER
- legend_name: VARCHAR

### dim_weapons
- weapon_id: INTEGER
- weapon_name: VARCHAR

---

## Question 1: Match Count by Weapon-Legend Combination (October 2024)

### Problem
What is the total number of matches played using each weapon and legend combination in October 2024?

### Expected Output
- weapon_name
- legend_name
- matches_cnt

### Solution

```sql
SELECT weapon_name,
       legend_name,
       COUNT(*) AS matches_cnt
FROM fct_matches f
LEFT JOIN dim_legends l 
    ON f.legend_id = l.legend_id
LEFT JOIN dim_weapons w 
    ON f.weapon_id = w.weapon_id
WHERE match_date >= '2024-10-01' 
  AND match_date < '2024-11-01'
GROUP BY weapon_name, legend_name;
```

**Explanation:**
- Joins fact table with both dimension tables to get names
- Uses LEFT JOIN to include matches even if dimension data is missing
- Filters to October 2024 using date range
- Groups by weapon_name and legend_name to count matches per combination
- COUNT(*) aggregates total matches for each unique pairing

---

## Question 2: Peak Weekly Matches per Weapon (October 2024)

### Problem
For each weapon, what is the highest number of matches played in a week during October 2024? Return the weapon name and the count of matches.

### Expected Output
- weapon_name
- max_weekly_matches

### Solution

```sql
WITH weekly_weapon_counts AS (
    SELECT
        w.weapon_name,
        DATE_TRUNC('week', f.match_date) AS week_start,
        COUNT(*) AS weekly_match_count
    FROM fct_matches f
    JOIN dim_weapons w
        ON f.weapon_id = w.weapon_id
    WHERE f.match_date >= DATE '2024-10-01'
      AND f.match_date < DATE '2024-11-01'
    GROUP BY
        w.weapon_name,
        DATE_TRUNC('week', f.match_date)
)
SELECT
    weapon_name,
    MAX(weekly_match_count) AS max_weekly_matches
FROM weekly_weapon_counts
GROUP BY weapon_name;
```

**Explanation:**
- CTE aggregates match counts by weapon and week
- Uses DATE_TRUNC('week', match_date) to group dates into weeks
- Counts matches per weapon per week
- Main query finds the maximum weekly count for each weapon
- Shows peak popularity week for each weapon in October

---

## Question 3: Least and Most Utilized Weapon-Legend Combinations (October 2024)

### Problem
Identify the least and most utilized weapon ID and legend ID combinations based on the total matches played in October 2024.

### Expected Output
- weapon_id
- legend_id
- combination_category ('least_utilized' or 'most_utilized')

### Solution

```sql
WITH combinations AS (
    SELECT weapon_id,
           legend_id,
           COUNT(*) AS match_cnt
    FROM fct_matches
    WHERE match_date >= '2024-10-01' 
      AND match_date < '2024-11-01'
    GROUP BY weapon_id, legend_id
)
SELECT weapon_id,
       legend_id,
       CASE 
           WHEN least_utilized = 1 THEN 'least_utilized'
           WHEN most_utilized = 1 THEN 'most_utilized'
       END AS combination_category
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY match_cnt ASC) AS least_utilized,
           ROW_NUMBER() OVER (ORDER BY match_cnt DESC) AS most_utilized
    FROM combinations
) t
WHERE least_utilized = 1 
   OR most_utilized = 1;
```

**Explanation:**
- First CTE counts matches for each weapon-legend combination
- Subquery applies two ROW_NUMBER() window functions:
  - least_utilized: ranks by match_cnt ASC (lowest first)
  - most_utilized: ranks by match_cnt DESC (highest first)
- Main query filters to combinations where either rank = 1
- CASE statement labels each as 'least_utilized' or 'most_utilized'
- Returns both extremes in a single result set for comparison
- Helps identify underutilized and overperforming loadouts for balance adjustments
