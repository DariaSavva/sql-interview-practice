# SQL Interview Practice

A collection of SQL problems and solutions from [InterviewMaster.ai](https://www.interviewmaster.ai/) practice sessions.

## 📋 Problem Cases

| Case | Topic | Questions | Difficulty | Key Concepts |
|------|-------|-----------|------------|--------------|
| [01](Case01_Amazon_Seller_Sales.md) | Amazon Seller Sales Analysis | 3 | Hard | Window Functions, CTEs, Date Operations, Cumulative Calculations |
| [02](Case02_Netflix_Marketing.md) | Netflix Marketing Efficiency | 3 | Medium | Aggregations, JOINs, Date Filtering, Division Safety |
| [03](Case03_Amazon_Prime_Engagement.md) | Amazon Prime Member Engagement | 3 | Medium | CTEs, Aggregations, CASE Statements, Bucketing |
| [04](Case04_Netflix_Mobile_Experience.md) | Netflix Mobile Experience | 3 | Hard | Percentiles, Window Functions, Timestamp Operations, Device Filtering |
| [05](Case05_Walmart_Vision_Center.md) | Walmart Vision Center Eyewear | 3 | Hard | DENSE_RANK, ROW_NUMBER, Multi-Column Ranking, Calculated Metrics |

## 🗂️ Repository Structure

```
sql-interview-practice/
├── README.md
├── Case01_Amazon_Seller_Sales.md
├── Case02_Netflix_Marketing.md
├── Case03_Amazon_Prime_Engagement.md
├── Case04_Netflix_Mobile_Experience.md
├── Case05_Walmart_Vision_Center.md
└── ... (more cases)
```

## 💡 How to Use

Each case file contains:
- **Business Context**: Real-world scenario
- **Database Schema**: Tables and columns
- **Questions**: Problem statements with expected outputs
- **Solutions**: Multiple approaches with your original solution + alternatives
- **Key Insights**: Performance notes and best practices

## 🎯 SQL Concepts Covered

- **Window Functions**: ROW_NUMBER, RANK, DENSE_RANK, SUM OVER, cumulative calculations
- **CTEs**: Common Table Expressions for complex queries
- **Date/Time Operations**: DATE_TRUNC, EXTRACT, BETWEEN, date ranges, timestamp handling
- **Aggregations**: SUM, COUNT, AVG, MAX, GROUP BY
- **Statistical Functions**: PERCENTILE_CONT, PERCENTILE_DISC, median, percentiles
- **JOINs**: INNER, LEFT, CROSS joins
- **Conditional Logic**: CASE statements, bucketing, IN clause
- **Ranking & Top-N**: Multiple ranking strategies with tie-breaking
- **Calculated Metrics**: Derived columns, performance scores
- **Advanced PostgreSQL**: DISTINCT ON, generate_series, LIMIT
- **Data Safety**: NULLIF for division, COALESCE for NULL handling

## 📊 Progress

- [x] Case 01: Amazon Seller Sales Analysis (Hard)
- [x] Case 02: Netflix Marketing Efficiency (Medium)
- [x] Case 03: Amazon Prime Member Engagement (Medium)
- [x] Case 04: Netflix Mobile Experience (Hard)
- [x] Case 05: Walmart Vision Center Eyewear (Hard)
- [ ] More cases coming soon...

## 🔧 Database

- **DBMS**: PostgreSQL
- **Platform**: [InterviewMaster.ai](https://www.interviewmaster.ai/)

---

**Note**: Each case includes both the original solution and optimized alternatives with detailed explanations.
