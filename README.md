/*
	@author: alfonsohdez
	@date: 03/12/2018
*/

-- Creates DWH --
CREATE DATABASE PointSalesDwh;
GO

USE PointSalesDwh;
GO

-- Category Dimension --
CREATE TABLE dbo.Dim_Category
(
	CategoryId INT PRIMARY KEY,
	[Name] NVARCHAR(50) NOT NULL
);
GO

WITH Category_CTE AS
(
	SELECT
		c.CategoryId
	FROM GenericPointSales.Sales.OrderDetail AS sod
		INNER JOIN GenericPointSales.Sales.[Order] AS so
			ON so.OrderId = sod.OrderId
		INNER JOIN GenericPointSales.Production.Product AS p
			ON sod.ProductId = p.ProductId
		INNER JOIN GenericPointSales.Production.Category AS c
			ON p.CategoryId = c.CategoryId
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY c.CategoryId
)
INSERT INTO dbo.Dim_Category
SELECT
	pc.CategoryId,
	pc.CategoryName
FROM GenericPointSales.Production.Category AS pc
	INNER JOIN Category_CTE AS cte
		ON pc.CategoryId = cte.CategoryId;
GO

-- Product Dimension --
CREATE TABLE dbo.Dim_Product
(
	ProductId INT PRIMARY KEY,
	[Name] NVARCHAR(50) NOT NULL,
	CategoryId INT NOT NULL,
	CONSTRAINT FK_DimCategory_DimProduct_CategoryId FOREIGN KEY (CategoryId) REFERENCES dbo.Dim_Category(CategoryId)
);
GO

WITH Product_CTE AS
(
	SELECT
		sod.ProductId
	FROM GenericPointSales.Sales.OrderDetail AS sod
		INNER JOIN GenericPointSales.Sales.[Order] AS so
			ON sod.OrderId = so.OrderId
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY sod.ProductId
)
INSERT INTO dbo.Dim_Product
SELECT
	p.ProductId,
	p.ProductName,
	p.CategoryId
FROM GenericPointSales.Production.Product AS p
	INNER JOIN Product_CTE AS cte
		ON p.ProductId = cte.ProductId;
GO

-- Country Dimension --
CREATE TABLE dbo.Dim_Country
(
	CountryCode NVARCHAR(5) PRIMARY KEY,
	[Name] NVARCHAR(60) NOT NULL
);
GO

WITH Country_CTE AS
(
	SELECT
		c.CountryCode
	FROM GenericPointSales.Sales.[Order] AS so
		INNER JOIN GenericPointSales.Sales.SalesTerritory AS st
			ON so.TerritoryId = st.TerritoryId
		INNER JOIN GenericPointSales.Territory.Country AS c
			ON st.CountryCode = c.CountryCode
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY c.CountryCode
)
INSERT INTO dbo.Dim_Country
SELECT
	c.CountryCode,
	c.CountryName
FROM GenericPointSales.Territory.Country AS c
	INNER JOIN Country_CTE AS cte
		ON c.CountryCode = cte.CountryCode;
GO

-- Territory Dimension --
CREATE TABLE dbo.Dim_Territory
(
	TerritoryId INT PRIMARY KEY,
	[Name] NVARCHAR(50) NOT NULL,
	CountryCode NVARCHAR(5) NOT NULL,
	CONSTRAINT FK_DimCountry_DimTerritory_CountryCode FOREIGN KEY (CountryCode) REFERENCES dbo.Dim_Country(CountryCode)
);
GO

WITH Territory_CTE AS
(
	SELECT
		so.TerritoryId
	FROM GenericPointSales.Sales.[Order] AS so
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY so.TerritoryId
)
INSERT INTO dbo.Dim_Territory
SELECT
	st.TerritoryId,
	st.[Name],
	st.CountryCode
FROM GenericPointSales.Sales.SalesTerritory AS st
	INNER JOIN Territory_CTE AS cte
		ON st.TerritoryId = cte.TerritoryId;
GO

-- Customer Dimension --
CREATE TABLE dbo.Dim_Customer
(
	CustomerId INT PRIMARY KEY,
	FirstName NVARCHAR(50) NOT NULL,
	LastName NVARCHAR(50) NOT NULL
);
GO

WITH Customer_CTE AS
(
	SELECT
		so.CustomerId
	FROM GenericPointSales.Sales.[Order] AS so
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY so.CustomerId
)
INSERT INTO dbo.Dim_Customer
SELECT
	c.CustomerId,
	c.FirstName,
	c.LastName
FROM GenericPointSales.Sales.Customers AS c
	INNER JOIN Customer_CTE AS cte
		ON C.CustomerId = CTE.CustomerId;
GO

-- Employee Dimension --
CREATE TABLE dbo.Dim_Employee
(
	EmployeeId INT PRIMARY KEY,
	FirstName NVARCHAR(50) NOT NULL,
	LastName NVARCHAR(50) NOT NULL
);
GO

WITH Employee_CTE AS
(
	SELECT
		so.EmployeeId
	FROM GenericPointSales.Sales.[Order] AS so
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY so.EmployeeId
)
INSERT INTO dbo.Dim_Employee
SELECT
	e.EmployeeId,
	e.FirstName,
	e.LastName
FROM GenericPointSales.HR.Employees AS e
	INNER JOIN Employee_CTE AS cte
		ON e.EmployeeId = cte.EmployeeId
GO

-- Year Dimension --
CREATE TABLE dbo.Dim_Year
(
	YearId INT PRIMARY KEY,
	[Year] INT NOT NULL
);
GO

INSERT INTO dbo.Dim_Year
SELECT
	ROW_NUMBER() OVER (ORDER BY YEAR(so.OrderDate)) AS YearId,
	YEAR(so.OrderDate) AS [Year]
FROM GenericPointSales.Sales.[Order] AS so
WHERE so.ShippedDate IS NOT NULL
GROUP BY YEAR(so.OrderDate);
GO

-- Month Dimension --
CREATE TABLE dbo.Dim_Month
(
	MonthId INT PRIMARY KEY,
	[Month] NVARCHAR(30) NOT NULL
);
GO

INSERT INTO dbo.Dim_Month (MonthId, [Month])
SELECT
	c.OrderMonth AS MonthId,
	DATENAME(MONTH, DATEADD(MONTH, c.OrderMonth, -1 )) AS [Month]
FROM (SELECT MONTH(so.OrderDate) AS OrderMonth
		FROM GenericPointSales.Sales.[Order] AS so
		WHERE so.ShippedDate IS NOT NULL
		GROUP BY MONTH(so.OrderDate)) AS c;
GO

-- WeekDay Dimension --
CREATE TABLE dbo.Dim_WeekDay
(
	WeekDayId INT PRIMARY KEY,
	[WeekDay] NVARCHAR(30) NOT NULL
);
GO

INSERT INTO dbo.Dim_WeekDay (WeekDayId, [WeekDay])
VALUES
(1, N'Monday'),
(2, N'Tuesday'),
(3, N'Wednesday'),
(4, N'Thursday'),
(5, N'Friday'),
(6, N'Saturday'),
(7, N'Sunday');
GO

-- Week Dimension --
CREATE TABLE dbo.Dim_Week
(
	WeekId INT PRIMARY KEY
);
GO

INSERT INTO dbo.Dim_Week (WeekId)
VALUES
(1),(2),(3),(4)

-- Time Dimension --
CREATE TABLE dbo.Dim_Time
(
	TimeId INT PRIMARY KEY,
	YearId INT NOT NULL,
	MonthId INT NOT NULL,
	WeekId INT NOT NULL,
	WeekDayId INT NOT NULL,
	RawDate DATE NOT NULL,
	CONSTRAINT FK_DimYear_DimTime_YearId FOREIGN KEY (YearId) REFERENCES dbo.Dim_Year(YearId),
	CONSTRAINT FK_DimMonth_DimTime_MonthId FOREIGN KEY (MonthId) REFERENCES dbo.Dim_Month(MonthId),
	CONSTRAINT FK_DimWeek_DimTime_WeekId FOREIGN KEY (WeekId) REFERENCES dbo.Dim_Week(WeekId),
	CONSTRAINT FK_DimWeekDay_DimTime_WeekDayId FOREIGN KEY (WeekDayId) REFERENCES dbo.Dim_WeekDay(WeekDayId)
);
GO

WITH DateGroup_CTE AS
(
	SELECT so.OrderDate
	FROM GenericPointSales.Sales.[Order] AS so
	WHERE so.ShippedDate IS NOT NULL
	GROUP BY so.OrderDate
),
WeekCalculation_CTE AS
(
	SELECT
		YEAR(tc.OrderDate) AS [Year],
		MONTH(tc.OrderDate) AS [Month],
		(
			CAST(DATENAME(WEEK,tc.OrderDate) AS INT)- CAST( DATENAME(WEEK,DATEADD(DD,1-DAY(tc.OrderDate),tc.OrderDate)) AS INT)+1
		) AS WeekId,
		DATENAME(dw, tc.OrderDate) AS [WeekDay],
		tc.OrderDate
	FROM DateGroup_CTE AS tc
),
Time_CTE AS
(
	SELECT
		t.[Year],
		t.[Month],
		CASE
			WHEN t.WeekId > 4 THEN 4
			WHEN t.WeekId <= 0 THEN 1
			ELSE t.WeekId
		END AS WeekId,
		t.[WeekDay],
		t.OrderDate
	FROM WeekCalculation_CTE AS t
)
INSERT INTO dbo.Dim_Time
SELECT
	ROW_NUMBER() OVER (ORDER BY cte.OrderDate) AS TimeId,
	dimY.YearId,
	dimM.MonthId,
	dimW.WeekId,
	dimWd.WeekDayId,
	cte.OrderDate AS RawDate
FROM Time_CTE AS cte
	INNER JOIN dbo.Dim_Year AS dimY
		ON cte.[Year] = dimY.[Year]
	INNER JOIN dbo.Dim_Month AS dimM
		ON cte.[Month] = dimM.MonthId
	INNER JOIN dbo.Dim_Week AS dimW
		ON cte.WeekId = dimW.WeekId
	INNER JOIN dbo.Dim_WeekDay AS dimWd
		ON cte.[WeekDay] = dimWd.[WeekDay];
GO

-- Sales Fact --
CREATE TABLE dbo.Fact_Sales
(
	FactSalesId INT PRIMARY KEY,
	ProductId INT NOT NULL,
	CustomerId INT NOT NULL,
	EmployeeId INT NOT NULL,
	TerritoryId INT NOT NULL,
	TimeId INT NOT NULL,
	Quantity INT NOT NULL,
	UnitPrice MONEY NOT NULL,
	CONSTRAINT FK_DimProduct_FactSales_ProductId FOREIGN KEY (ProductId) REFERENCES dbo.Dim_Product(ProductId),
	CONSTRAINT FK_DimCustomer_FactSales_CustomerId FOREIGN KEY (CustomerId) REFERENCES dbo.Dim_Customer(CustomerId),
	CONSTRAINT FK_DimEmployee_FactSales_EmployeeId FOREIGN KEY (EmployeeId) REFERENCES dbo.Dim_Employee(EmployeeId),
	CONSTRAINT FK_DimTerritory_FactSales_TerritoryId FOREIGN KEY (TerritoryId) REFERENCES dbo.Dim_Territory(TerritoryId),
	CONSTRAINT FK_DimTime_FactSales_TimeId FOREIGN KEY (TimeId) REFERENCES dbo.Dim_Time(TimeId)
);
GO

INSERT INTO dbo.Fact_Sales
SELECT
	ROW_NUMBER() OVER (ORDER BY dp.ProductId, dc.CustomerId, de.EmployeeId, dte.TerritoryId, dt.TimeId) AS FactSalesId,
	dp.ProductId,
	dc.CustomerId,
	de.EmployeeId,
	dte.TerritoryId,
	dt.TimeId,
	sod.Quantity,
	sod.UnitPrice
FROM GenericPointSales.Sales.[Order] AS so
	INNER JOIN  GenericPointSales.Sales.OrderDetail AS sod
		ON so.OrderId = sod.OrderId
	INNER JOIN dbo.Dim_Product AS dp
		ON dp.ProductId = sod.ProductId
	INNER JOIN dbo.Dim_Customer AS dc
		ON dc.CustomerId = so.CustomerId
	INNER JOIN dbo.Dim_Employee AS de
		ON de.EmployeeId = so.EmployeeId
	INNER JOIN dbo.Dim_Territory AS dte
		ON dte.TerritoryId = so.TerritoryId
	INNER JOIN dbo.Dim_Time AS dt
		ON dt.RawDate = so.OrderDate;
GO
