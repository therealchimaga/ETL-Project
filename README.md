# ETL Data Pipeline & Data Warehouse Project

**Author:** Rufaro Chimaga  
**Course:** Database Management (IA626/IA637) - Prof. Boris Jukić  
**Institution:** Clarkson University  
**Date:** November 2024

---

## 📋 Project Overview

This project implements a comprehensive **Extract, Transform, Load (ETL) pipeline** for a retail business intelligence system, demonstrating advanced database management techniques including:

- Multi-tier data warehouse architecture (Source → Staging → Data Warehouse)
- Slowly Changing Dimensions (SCD Type 2) implementation
- Automated data refresh procedures
- Fact table incremental loading strategies
- Star schema dimensional modeling

**Business Context:** The system processes sales and rental transactions for a retail business (ZAGIMORE), enabling business intelligence analysis across products, customers, stores, and time dimensions.

---

## 🏗️ Architecture

### Three-Tier Data Warehouse Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SOURCE DATABASE                           │
│              (chimagr_ZAGIMORE)                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ Products │  │Customers │  │  Stores  │                 │
│  │  Sales   │  │ Rentals  │  │  Regions │                 │
│  └──────────┘  └──────────┘  └──────────┘                 │
└────────────────────┬────────────────────────────────────────┘
                     │ ETL Process
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              DATA STAGING (chimagr_ZAGIMORE_DS)             │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │ Product_Dimension│  │Customer_Dimension│               │
│  │ Store_Dimension  │  │Calendar_Dimension│               │
│  │    (with SCD)    │  │   Revenue Facts  │               │
│  └──────────────────┘  └──────────────────┘               │
└────────────────────┬────────────────────────────────────────┘
                     │ Load Process
                     ▼
┌─────────────────────────────────────────────────────────────┐
│          DATA WAREHOUSE (chimagr_ZAGIMORE_DW)               │
│  ┌──────────────────────────────────────────┐              │
│  │         Star Schema                      │              │
│  │                                          │              │
│  │    Product ← Revenue Facts → Customer   │              │
│  │       ↓            ↓           ↓         │              │
│  │    Store       Calendar                 │              │
│  └──────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema

### Dimensional Model (Star Schema)

#### **Dimension Tables**

1. **Product_Dimension**
   - Attributes: ProductKey, ProductID, ProductName, ProductSalesPrice, ProductDailyRentalPrice, ProductWeeklyRental, VendorID, VendorName, CategoryID, CategoryName, ProductType
   - SCD Type 2 Fields: DVF (Date Valid From), DVU (Date Valid Until), CurrentStatus

2. **Customer_Dimension**
   - Attributes: CustomerKey, CustomerID, CName, CZip
   - SCD Type 2 Fields: DVF, DVU, CurrentStatus

3. **Store_Dimension**
   - Attributes: StoreKey, StoreID, StoreZip, RegionID, RegionName
   - SCD Type 2 Fields: DVF, DVU, CurrentStatus

4. **Calendar_Dimension**
   - Attributes: Calendar_Key, FullDate, MonthYear, Year
   - Coverage: 10,000 days from 2013-01-01

#### **Fact Table**

**Revenue**
- Measures: RevenueGenerated, UnitSolds
- Descriptive Attributes: RevenueType, TID (Transaction ID)
- Foreign Keys: CustomerKey, StoreKey, ProductKey, Calendar_Key

---

## 🔄 ETL Process Components

### 1. **Extract Phase**
- Identifies new/changed records in source database
- Uses timestamp-based change detection
- Creates intermediate staging tables (IPD, CPD, IntermediateFact)

### 2. **Transform Phase**
- Applies business rules and calculations
- Revenue calculation: `Price × Quantity` or `Price × Duration`
- Handles three revenue types:
  - Sales (one-time purchases)
  - RentalDaily (daily rental rate)
  - RentalWeekly (weekly rental rate)

### 3. **Load Phase**
- Inserts new dimension members
- Updates existing records (SCD Type 2)
- Loads facts with proper foreign key relationships

---

## 📊 Slowly Changing Dimensions (SCD Type 2)

### Implementation Strategy

**Why SCD Type 2?**
Historical tracking of dimension changes is critical for accurate trend analysis. When product prices, customer addresses, or other attributes change, we preserve the historical values.

### How It Works

```sql
-- Example: Product price changes from $30 to $35

-- Before Change:
ProductKey | ProductID | Price | DVF        | DVU        | CurrentStatus
1          | 1X2       | 30.00 | 2013-01-01 | 2040-01-01 | C (Current)

-- After ETL:
ProductKey | ProductID | Price | DVF        | DVU        | CurrentStatus
1          | 1X2       | 30.00 | 2013-01-01 | 2025-03-25 | N (Not Current)
2          | 1X2       | 35.00 | 2025-03-26 | 2040-01-01 | C (Current)
```

### Change Detection Process

1. **Identify Changes**
   ```sql
   -- Find products with price changes
   CREATE TABLE IPD AS
   SELECT p.productid
   FROM product p, Product_Dimension pd
   WHERE p.productid = pd.ProductID
   AND p.productprice != pd.ProductSalesPrice
   AND pd.CurrentStatus = 'C';
   ```

2. **Expire Old Records**
   ```sql
   UPDATE Product_Dimension
   SET DVU = CURRENT_DATE - 1, CurrentStatus = 'N'
   WHERE ProductId IN (SELECT * FROM IPD);
   ```

3. **Insert New Records**
   ```sql
   INSERT INTO Product_Dimension (...)
   SELECT ..., CURRENT_DATE AS DVF, '2040-01-01' AS DVU, 'C' AS CurrentStatus
   FROM product WHERE productid IN (SELECT * FROM IPD);
   ```

---

## 🔧 Automated Procedures

### 1. **DailyProductRefresh()**
**Purpose:** Adds new products to the dimension  
**Frequency:** Daily  
**Logic:**
- Checks for new ProductIDs not in dimension
- Loads both Sales and Rental products
- Maintains separate records for same product in different categories

```sql
CALL DailyProductRefresh();
```

### 2. **Daily_Store_Refresh()**
**Purpose:** Adds new store locations  
**Frequency:** Daily  
**Logic:**
- Identifies new StoreIDs
- Joins with Region table for region names
- Loads to staging, then warehouse

### 3. **Daily_Customer_Refresh()**
**Purpose:** Adds new customers  
**Frequency:** Daily  
**Logic:**
- Checks for new CustomerIDs
- Extracts customer demographic info
- Handles SCD Type 2 for address changes

### 4. **Daily_Regular_Fact_Refresh()**
**Purpose:** Loads daily transactions  
**Frequency:** Daily  
**Logic:**
- Extracts transactions after last load date
- Processes sales, daily rentals, weekly rentals
- Calculates revenue: `Price × Quantity/Duration`
- Loads facts with proper dimension keys

### 5. **LateFactRefresh()**
**Purpose:** Catches missed transactions  
**Frequency:** Weekly/On-demand  
**Logic:**
- Identifies TIDs not in fact table
- Useful for late-arriving data
- Ensures data completeness

---

## 🚀 Usage Examples

### Initial Setup

```sql
-- 1. Create databases
CREATE DATABASE chimagr_ZAGIMORE;        -- Source
CREATE DATABASE chimagr_ZAGIMORE_DS;     -- Staging
CREATE DATABASE chimagr_ZAGIMORE_DW;     -- Warehouse

-- 2. Create dimension and fact tables (see schema above)

-- 3. Populate Calendar dimension
CALL populateCalendar();

-- 4. Initial data load
CALL DailyProductRefresh();
CALL Daily_Store_Refresh();
CALL Daily_Customer_Refresh();
CALL Daily_Regular_Fact_Refresh();
```

### Daily Operations

```sql
-- Run each morning to process previous day's data
CALL DailyProductRefresh();
CALL Daily_Store_Refresh();
CALL Daily_Customer_Refresh();
CALL Daily_Regular_Fact_Refresh();
```

### Handling Price Changes

```sql
-- When product prices change:
UPDATE product SET productprice = 35.00 WHERE productid = '1X2';

-- ETL automatically:
-- 1. Detects the change
-- 2. Expires old record (DVU = yesterday)
-- 3. Creates new record (DVF = today)
-- 4. Maintains history for accurate reporting
```

---

## 📈 Business Intelligence Capabilities

### Sample Queries

**1. Product Performance Analysis**
```sql
SELECT 
    p.ProductName,
    SUM(r.RevenueGenerated) AS TotalRevenue,
    SUM(r.UnitSolds) AS TotalUnitsSold
FROM Revenue r
JOIN Product_Dimension p ON r.ProductKey = p.ProductKey
WHERE p.CurrentStatus = 'C'
GROUP BY p.ProductName
ORDER BY TotalRevenue DESC;
```

**2. Monthly Revenue Trends**
```sql
SELECT 
    c.Year, c.MonthYear,
    SUM(r.RevenueGenerated) AS MonthlyRevenue
FROM Revenue r
JOIN Calendar_Dimension c ON r.Calendar_Key = c.Calendar_Key
GROUP BY c.Year, c.MonthYear
ORDER BY c.Year, c.MonthYear;
```

**3. Historical Price Analysis (SCD Type 2)**
```sql
SELECT 
    ProductID, ProductName, ProductSalesPrice,
    DVF AS PriceEffectiveDate,
    DVU AS PriceEndDate,
    CurrentStatus
FROM Product_Dimension
WHERE ProductID = '1X2'
ORDER BY DVF DESC;
```

**4. Regional Performance**
```sql
SELECT 
    s.RegionName,
    COUNT(DISTINCT r.TID) AS Transactions,
    SUM(r.RevenueGenerated) AS TotalRevenue
FROM Revenue r
JOIN Store_Dimension s ON r.StoreKey = s.StoreKey
GROUP BY s.RegionName
ORDER BY TotalRevenue DESC;
```

---

## 🛠️ Technical Implementation

### Key Design Decisions

1. **Three-Tier Architecture**
   - Separation of concerns: source, staging, warehouse
   - Staging area enables complex transformations
   - Clean warehouse for optimized querying

2. **SCD Type 2 Implementation**
   - Preserves complete history
   - Enables time-based analysis
   - DateValidFrom/DateValidUntil tracking

3. **Incremental Loading**
   - Only processes new/changed data
   - Uses timestamps and flags (PDLoaded, f_loaded)
   - Efficient for large datasets

4. **Surrogate Keys**
   - Auto-incrementing keys for dimensions
   - Decouples warehouse from source systems
   - Enables proper SCD handling

### Data Quality Controls

- **Referential Integrity:** Foreign key constraints enforce relationships
- **Timestamp Tracking:** ExtractionTimestamp for audit trail
- **Load Flags:** Boolean flags prevent duplicate loads
- **Change Detection:** Compares source vs. staging to identify changes

---

## 📊 Performance Considerations

### Optimization Strategies

1. **Indexes**
   ```sql
   CREATE INDEX idx_product_id ON Product_Dimension(ProductID);
   CREATE INDEX idx_customer_id ON Customer_Dimension(CustomerID);
   CREATE INDEX idx_date ON Calendar_Dimension(FullDate);
   ```

2. **Partitioning** (for large datasets)
   ```sql
   -- Partition Revenue by year
   ALTER TABLE Revenue PARTITION BY RANGE (Calendar_Key);
   ```

3. **Batch Processing**
   - Daily procedures batch changes
   - Reduces transaction overhead
   - Scheduled during off-peak hours

---

## 🎓 Learning Outcomes

This project demonstrates mastery of:

✅ **Database Design**
- Star schema dimensional modeling
- Normalization vs. denormalization trade-offs
- Surrogate key design

✅ **ETL Development**
- Extract strategies (incremental, full)
- Complex transformations (revenue calculations)
- Load patterns (SCD Type 2)

✅ **SQL Proficiency**
- Advanced joins and subqueries
- Stored procedures and functions
- Transaction management

✅ **Data Warehousing Concepts**
- Fact and dimension tables
- Slowly changing dimensions
- Temporal data handling

✅ **Business Intelligence**
- Analytical query design
- Historical trend analysis
- Performance optimization

---

## 🔍 Future Enhancements

- [ ] Implement SCD Type 1 for certain attributes
- [ ] Add aggregate fact tables for performance
- [ ] Build OLAP cubes for multidimensional analysis
- [ ] Create ETL error logging and monitoring
- [ ] Develop data quality dashboards
- [ ] Implement change data capture (CDC)
- [ ] Add data lineage tracking

---

## 📚 References

- Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit*
- Date, C. J. (2003). *An Introduction to Database Systems*
- Course Materials: IA626/IA637 Database Management

---

## 📧 Contact

**Rufaro Chimaga**  
Graduate Student, Applied Data Science  
Clarkson University

- Email: chimagr@clarkson.edu
- LinkedIn: [linkedin.com/in/rufaro-chimaga](https://www.linkedin.com/in/rufaro-chimaga)
- GitHub: [@chimagr](https://github.com/chimagr)

---

## ⚖️ Disclaimer

This project was developed as part of academic coursework at Clarkson University. The data and business scenarios are for educational purposes only.

**Acknowledgment:** Special thanks to Professor Boris Jukić for guidance on ETL design and data warehousing concepts.

---

**Last Updated:** November 2024  
**Course:** Database Management (IA626/IA637)  
**Project Partner:** Lerato Pahla
