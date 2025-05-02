# dinahkissiowusu-s_project

Customer Sales ETL Pipeline (SSIS + SSMS):
1. Overview:
This project is an end-to-end ETL pipeline built using SQL Server Integration Services (SSIS) to load a Customer Sales Data Warehouse (SalesDW) from the AdventureWorks sample database. ThE goal of the project is to create a centralised, cleaned dataset that is ready for analytical reporting on customer and sales performance.

Source: AdventureWorks2022 (OLTP) database

ETL Tool: SSIS (SQL Server Integration Services)

Target: SalesDW (star schema) created in SQL Server Management Studio (SSMS) with one fact table and four (4) dimension table

2. Features:
a. ETL pipeline using SSIS packages.

b. SQL Script for raw data extraction to create dimension tables and in extracting key columns from different tables.

c. Data transformation logic implemented in SSIS Data Flow Tasks and SQL scripts.

d. Clean dimensional model with fact and dimension tables.

3. ETL Process:
a. Extract
 Extract data from key AdventureWorks tables  such as:
 
( Sales.SalesOrderHeader
 Sales.SalesOrderDetail
 Sales.Customer
 Production.Product
 Person.Person
 Sales.SalesTerritory) using joins to create dimension and fact tables.

 4. Schemas:
Dimension Tables:
a. DimCustomer
 Sources: Sales.Customer, Person.Person, Person.EmailAddress, Person.PersonPhone,
Person.BusinessEntity, Sales.Store

Columns:
 CustomerKey (Surrogate Key)
 CustomerID (Business Key)
 FirstName
 LastName
 EmailAddress
 PhoneNumber
 StoreName
 CompanyName
 ModifiedDate

i. First create DimCustomer table in SSMS:
CREATE TABLE DimCustomer (
    CustomerKey INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID INT,
    FirstName NVARCHAR(50),
    Fullname NVARCHAR(100),
    LastName NVARCHAR(50),
    EmailAddress NVARCHAR(100),
    PhoneNumber NVARCHAR(25),
    StoreName NVARCHAR(100),
    CompanyName NVARCHAR(100),
    ModifiedDate DATETIME
);

ii. Write a join statement to extract key columns from the various table in SSMS:
SQL Syntax for DimCustomer table:
SELECT c.CustomerID,
       c.ModifiedDate,
	   p.FirstName,
	   p.LastName,
	   e.EmailAddress,
	   st.Name as StoreName,
	   ph.PhoneNumber
  FROM [AdventureWorks2022].[Sales].[Customer] c
   left join Sales.Store st on c.CustomerID = st.BusinessEntityID
   left join person.Person p on c.CustomerID = p.BusinessEntityID
   left join person.EmailAddress e on c.CustomerID = e.BusinessEntityID
   left join person.PersonPhone ph on c.CustomerID = ph.BusinessEntityID
   left join person.BusinessEntity be on c.CustomerID = be.BusinessEntityID

b. DimProduct
 Sources: Production.Product, Production.ProductSubcategory,
Production.ProductCategory, Production.ProductModel

Columns:
 ProductKey
 ProductID
 Name
 ProductNumber
 Color
 StandardCost
 ListPrice
 Size
 Weight
 ProductSubcategory
 ProductCategory
 ProductModel
 ModifiedDate

i.First create DimProduct table in SSMS:
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductID INT,
    Name NVARCHAR(100),
    ProductNumber NVARCHAR(50),
    Color NVARCHAR(25),
    StandardCost MONEY,
    ListPrice MONEY,
    Size NVARCHAR(10),
    Weight DECIMAL(8,2),
    ProductSubcategory NVARCHAR(50),
    ProductCategory NVARCHAR(50),
    ProductModel NVARCHAR(100),
    ModifiedDate DATETIME
);

ii. Write a join statement to extract key columns from the various table in SSMS:
SQL Syntax for DimProduct table:
SELECT p.ProductID,
             p.Name,
              p.ProductNumber,
	   p.color,
	   p.StandardCost,
	   p.ListPrice,
	   p.Size,
	   p.Weight,
	   p.ModifiedDate,
	   pm.Name as ProductModel,
	   ps.Name as ProductSubcategory,
	   pc.Name as ProductCategory
  FROM [AdventureWorks2022].[Production].[Product] p
  left join Production.ProductModel pm on p.ProductModelID = pm.ProductModelID
  left join Production.ProductSubcategory ps on p.ProductSubcategoryID = ps.ProductSubcategoryID
  left join Production.ProductCategory pc on ps.ProductCategoryID = pc.ProductCategoryID

c. DimSalesTerritory
 Source: Sales.SalesTerritory

Columns:
 TerritoryKey
 TerritoryID
 Name
 CountryRegionCode
 Group
 SalesYTD
 SalesLastYear
 CostYTD
 CostLastYear

i.First create DimSalesTerritory table in SSMS:
CREATE TABLE DimSalesTerritory (
    TerritoryKey INT IDENTITY(1,1) PRIMARY KEY,
    TerritoryID INT,
    Name NVARCHAR(50),
    CountryRegionCode NVARCHAR(10),
    Group NVARCHAR(50),
    SalesYTD MONEY,
    SalesLastYear MONEY,
    CostYTD MONEY,
    CostLastYear MONEY
);

d. DimDate
Custom date dimension using a generated date table covering the range of OrderDate

Columns:
 DateKey (YYYYMMDD)
 FullDate
 Day
 Month
 MonthName
 Quarter
 Year
 DayOfWeek
 WeekOfYear


i.First create DimDate table in SSMS:
CREATE TABLE DimDate (
    DateKey INT PRIMARY KEY,
    FullDate DATE,
    Day INT,
    Month INT,
    MonthName NVARCHAR(20),
    Quarter INT,
    Year INT,
    DayOfWeek INT,
    WeekOfYear INT
);

ii. SQL Syntax for DimDate table:
DECLARE @StartDate DATE = '2011-01-01';
DECLARE @EndDate DATE = '2014-12-31';

WITH DateSequence AS (
    SELECT @StartDate AS FullDate
    UNION ALL
    SELECT DATEADD(DAY, 1, FullDate)
    FROM DateSequence
    WHERE DATEADD(DAY, 1, FullDate) <= @EndDate
)
INSERT INTO DimDate (
    DateKey,
    FullDate,
    Day,
    Month,
    MonthName,
    Quarter,
    Year,
    DayOfWeek,
    WeekOfYear
)
SELECT 
    CONVERT(INT, FORMAT(FullDate, 'yyyyMMdd')) AS DateKey,
    FullDate,
    DAY(FullDate),
    MONTH(FullDate),
    DATENAME(MONTH, FullDate),
    DATEPART(QUARTER, FullDate),
    YEAR(FullDate),
    DATEPART(WEEKDAY, FullDate),
    DATEPART(WEEK, FullDate)
FROM DateSequence
OPTION (MAXRECURSION 4000);

e. Fact Table:
FactSales
Source: Sales.SalesOrderHeader and Sales.SalesOrderDetail

Columns:
 SalesOrderID
 SalesOrderDetailID
 ProductKey (FK to DimProduct)
 CustomerKey (FK to DimCustomer)
 TerritoryKey (FK to DimSalesTerritory)
 OrderDateKey (FK to DimDate)
 UnitPrice
 OrderQty
 TotalDue (from SalesOrderHeader)
 LineTotal (from SalesOrderDetail)

i. First create FactSales table in SSMS:
CREATE TABLE FactSales (
    SalesOrderID INT,
    SalesOrderDetailID INT,
    ProductKey INT,
    CustomerKey INT,
    TerritoryKey INT,
    OrderDateKey INT,
    UnitPrice MONEY,
    OrderQty INT,
    TotalDue MONEY,
    LineTotal MONEY,
    PRIMARY KEY (SalesOrderID, SalesOrderDetailID)
);

f. Data Warehouse Design:
Star Schema Includes:
FactSales

g. Transformations:
 Derived Column: Concatenate first name and last name, compute TotalPrice = UnitPrice
* OrderQty.
 Data Conversion: Convert all data types incompatible with destination schema.
 Lookup: Use Lookup transformations to retrieve surrogate keys from dimension tables.
 SCD Type 1: Implement for DimCustomer using the Slowly Changing Dimension
transformation.

g. Loading:
 Load into SalesDW tables.
 Load dimension table first, then the fact table.

h. Prerequisites:
i. SQL Server + SSMS (SQL Server Management Studio)

ii. SSDT (SQL Server Data Tools) or Visual Studio with SSIS extension

iii. AdventureWorks2022 (OLTP) database installed

i. How to Run:
Open the .dtsx package in SSDT or Visual Studio.

Deploy and execute the SSIS package.

i. Use Cases
Business reporting and dashboards
Sales performance analysis
Customer segmentation and insights
Data preparation for predictive modeling

j. Author
Your Name – @dinahgborglah
