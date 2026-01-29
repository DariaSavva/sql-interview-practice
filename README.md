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
| [06](Case06_WhatsApp_User_Engagement.md) | WhatsApp User Engagement | 3 | Medium | String Matching, Window Functions, Aggregations |
| [07](Case07_Google_Ads_Performance.md) | Google Ads Performance | 3 | Hard | Rolling Windows, ROWS BETWEEN, ROI Calculations, Comparative Analysis |
| [08](Case08_Airbnb_Bookings.md) | Airbnb Booking Behaviors | 3 | Hard | Rolling Completion Rates, LAG Function, Pivoting, Multi-Dimensional Analysis |
| [09](Case09_PayPal_Disputes.md) | PayPal Refund Disputes | 3 | Medium | NTILE, Pattern Matching, Top-N Analysis, Quartile Segmentation |

## 🗂️ Repository Structure

```
sql-interview-practice/
├── README.md
├── Case01_Amazon_Seller_Sales.md
├── Case02_Netflix_Marketing.md
├── Case03_Amazon_Prime_Engagement.md
├── Case04_Netflix_Mobile_Experience.md
├── Case05_Walmart_Vision_Center.md
├── Case06_WhatsApp_User_Engagement.md
├── Case07_Google_Ads_Performance.md
├── Case08_Airbnb_Bookings.md
├── Case09_PayPal_Disputes.md
└── ... (more cases)
```

## 💡 How to Use

Each case file contains:
- **Business Context**: Real-world scenario
- **Database Schema**: Tables and columns
- **Questions**: Problem statements with expected outputs
- **Solutions**: Your original solution + 1 alternative approach
- **Key Insights**: Performance notes and best practices

## 🎯 SQL Concepts Covered

- **Window Functions**: ROW_NUMBER, RANK, DENSE_RANK, SUM OVER, AVG OVER, LAG, NTILE, cumulative calculations, rolling windows
- **Frame Clauses**: ROWS BETWEEN, RANGE BETWEEN for sliding windows
- **CTEs**: Common Table Expressions for complex queries
- **Date/Time Operations**: DATE_TRUNC, EXTRACT, BETWEEN, date ranges, timestamp handling
- **Aggregations**: SUM, COUNT, AVG, MAX, GROUP BY
- **Statistical Functions**: PERCENTILE_CONT, PERCENTILE_DISC, NTILE, median, percentiles, quartiles
- **String Operations**: LIKE, ILIKE, LOWER, pattern matching, prefix matching
- **JOINs**: INNER, LEFT, CROSS joins
- **Conditional Logic**: CASE statements, bucketing, IN clause, comparative analysis
- **Pivoting**: MAX/CASE pattern for transforming rows to columns
- **Time Series**: LAG/LEAD for day-over-day analysis
- **Ranking & Top-N**: Multiple ranking strategies with tie-breaking, LIMIT
- **Calculated Metrics**: Derived columns, performance scores, ROI calculations, completion rates
- **Advanced PostgreSQL**: DISTINCT ON, generate_series, LIMIT
- **Data Safety**: NULLIF for division, COALESCE for NULL handling

## 📊 Progress

- [x] Case 01: Amazon Seller Sales Analysis (Hard)
- [x] Case 02: Netflix Marketing Efficiency (Medium)
- [x] Case 03: Amazon Prime Member Engagement (Medium)
- [x] Case 04: Netflix Mobile Experience (Hard)
- [x] Case 05: Walmart Vision Center Eyewear (Hard)
- [x] Case 06: WhatsApp User Engagement (Medium)
- [x] Case 07: Google Ads Performance (Hard)
- [x] Case 08: Airbnb Booking Behaviors (Hard)
- [x] Case 09: PayPal Refund Disputes (Medium)
- [ ] More cases coming soon...

## 🔧 Database

- **DBMS**: PostgreSQL
- **Platform**: [InterviewMaster.ai](https://www.interviewmaster.ai/)

---

**Note**: Each case includes both the original solution and an alternative approach with detailed explanations.
