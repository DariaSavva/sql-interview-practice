# SQL Interview Practice

A collection of SQL problems and solutions from InterviewMaster.ai practice sessions.

## Problem Cases

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
| [10](Case10_Apple_Philanthropy.md) | Apple Philanthropic Initiatives | 3 | Medium | Window Functions, JOINs, MIN/MAX Aggregations, Multi-Dimensional Grouping |
| [11](Case11_Stripe_Capital.md) | Stripe Capital Lending | 3 | Hard | LAG-based Growth Calculations, Month-over-Month Analysis, Complex Pivoting, RANK |
| [12](Case12_Rides_Performance.md) | Uber/Lyft Rides Performance | 3 | Hard | Timestamp Calculations, LEAD Function, EXTRACT EPOCH, Idle Time Analysis |
| [13](Case13_ChatGPT_Performance.md) | ChatGPT Query Performance | 3 | Medium | Multi-Criteria Ranking, JOINs, Aggregations |
| [14](Case14_X_Advertising.md) | X (Twitter) Advertising Campaigns | 3 | Hard | Data Cleaning, Growth Rate Calculations, Pivoting, Multiple ROW_NUMBER |
| [15](Case15_Game_Balance.md) | Game Balance Analysis | 3 | Hard | Multi-Dimensional JOINs, Weekly Aggregations, Dual ROW_NUMBER for Extremes |

## Repository Structure

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
├── Case10_Apple_Philanthropy.md
├── Case11_Stripe_Capital.md
├── Case12_Rides_Performance.md
├── Case13_ChatGPT_Performance.md
├── Case14_X_Advertising.md
└── Case15_Game_Balance.md
```

## How to Use

Each case file contains:
- Business Context
- Database Schema
- Questions with problem statements
- SQL Solutions

## SQL Concepts Covered

- Window Functions: ROW_NUMBER, RANK, DENSE_RANK, SUM OVER, AVG OVER, LAG, LEAD, NTILE
- CTEs: Common Table Expressions
- Aggregations: SUM, COUNT, AVG, MAX, MIN, GROUP BY
- JOINs: INNER, LEFT, CROSS
- Date Operations: DATE_TRUNC, EXTRACT, BETWEEN
- Timestamp Calculations: EXTRACT EPOCH, timestamp differences
- Statistical Functions: PERCENTILE_CONT, PERCENTILE_DISC
- String Operations: LIKE, ILIKE, LOWER, UPPER
- Data Cleaning: UPPER, LOWER, TRIM
- Pivoting and conditional aggregation
- Rolling windows and time series analysis
- Multi-criteria ranking
- Growth rate calculations
- Finding extremes with dual ROW_NUMBER

## Progress

Total: 45 SQL interview questions
Difficulty: 9 Hard, 6 Medium

## Database

- DBMS: PostgreSQL
- Platform: InterviewMaster.ai
