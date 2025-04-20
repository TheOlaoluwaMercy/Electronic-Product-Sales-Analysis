# Electronic-Product-Sales-Analysis
This project is a proof of my SQL window function skills.

Through a series of business-driven queries, the analysis answers key questions around profitability trends, customer purchase behaviour, and category performance.

Database diagram

Task 1:  Create a view that includes Order_Number, Line_Item, Order_Date, Quarter (YYYY-Q#), Month (YYYY-MM), Product_Name, Product_Category, Customer_Name, Store_Country, Quantity, and Profit (USD)—calculated as (Unit Price USD - Unit Cost USD) * Quantity — by joining relevant fields from the Sales, Customers, Products, and Stores tables.

  ```sql
  CREATE VIEW combo_data AS 
  SELECT Order_Number, 
          Line_Item, 
          Order_Date, 
          YEAR(Order_Date) AS Year, 
          CONCAT(YEAR(Order_Date), '-Q', DATEPART(QUARTER, Order_Date)) AS Quarter, 
          CONCAT(YEAR(Order_Date), '-', FORMAT(Order_Date, 'MM')) AS Month,
          Customers.Name AS Customer, 
          Stores.Country AS StoreCountry, 
          Products.Product_Name, 
          Products.Category, 
          (Products.Unit_Price_USD * Quantity) - (Products.Unit_Cost_USD * Quantity) AS Profit 
FROM Sales 
LEFT JOIN Customers ON Sales.CustomerKey = Customers.CustomerKey 
LEFT JOIN Stores ON Sales.StoreKey = Stores.StoreKey
LEFT JOIN Products ON Products.ProductKey = Sales.ProductKey; 
  ```

Task 2: Query the view `combo_data` to answer the following business questions
Question 1: Rank Product categories based on the total profit generated

  ```sql
  SELECT 
    Category,
    SUM(Profit) AS Total_Profit,
    RANK() OVER (ORDER BY SUM(Profit) DESC) AS Profit_Rank
FROM combo_data
GROUP BY Category;
  ```
Question 2: What is each product category’s share of overall profit?
```sql
SELECT 
	Category,
	(Total_Profit/SUM(Total_Profit) OVER()) * 100 AS profit_proportion
FROM (SELECT 
	Category,
	SUM(Profit) AS Total_Profit
FROM combo_data
GROUP BY Category) AS Product_summary
ORDER BY (Total_Profit/SUM(Total_Profit) OVER()) DESC;
```
Question 3: What is the running total of profit by quarter?
```sql
SELECT 
	Quarter,
	SUM(Total_Profit) Over(ORDER BY Quarter) AS Running_total
 FROM (SELECT
	Quarter, 
	SUM(Profit) AS Total_Profit
FROM combo_data
GROUP BY Quarter) AS Quarter_summary;
```
Question 4: What is the cumulative percentage of profit over quarters?
```sql
SELECT 
	Quarter,
	(SUM(Total_Profit) Over(ORDER BY Quarter)/SUM(Total_Profit) OVER()) * 100 AS Running_perc
 FROM (SELECT
	Quarter, 
	SUM(Profit) AS Total_Profit
FROM combo_data
GROUP BY Quarter) AS Quarter_summary;
```
Question 5: What is the 3-month moving average of monthly profit?
```sql
SELECT
	Month, 
	AVG(Total_Profit) OVER( ORDER BY MONTH ASC ROWS BETWEEN 2 PRECEDING AND CURRENT ROW ) AS Three_Month_Rolling_Average
FROM 
(SELECT 
	SUBSTRING(Month,1,4)  * 100 + SUBSTRING(Month,6,2) AS Month, 
	SUM(Profit) AS Total_Profit
FROM combo_data
GROUP BY Month) AS Month_agg
Order by Month;
```
Question 6: How does monthly profit change compared to the previous month?
```sql
SELECT 
	Month,
	Total_Profit,
	LAG(Total_Profit) OVER(ORDER BY Month) AS prev_month_profit,
	Total_Profit - LAG(Total_Profit) OVER(ORDER BY Month) AS diff,
	((Total_Profit - LAG(Total_Profit) OVER(ORDER BY Month))/LAG(Total_Profit) OVER(ORDER BY Month)) * 100 AS percent_var
FROM 
(SELECT 
	SUBSTRING(Month,1,4)  * 100 + SUBSTRING(Month,6,2) AS Month, 
	SUM(Profit) AS Total_Profit
FROM combo_data
GROUP BY Month) AS Month_agg;
```
Question 7: Which product categories in 2020 performed above or below the average?
```sql
SELECT 
	Category,
	CASE WHEN Total_Profit > AVG(Total_Profit) OVER() THEN 'ABOVE AVERAGE'
		WHEN Total_Profit < AVG(Total_Profit) OVER() THEN 'BELOW AVERAGE'
		ELSE 'AT PAR' END AS AVG_Comparison
FROM 
(SELECT 
	Category,
	SUM(Profit) AS Total_Profit
FROM combo_data
WHERE Year = 2020
GROUP BY Category) AS Category_summary;
```
Question 8: When did each customer place their first order?
```sql
SELECT
	DISTINCT Customer,
	FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC) AS first_patronage
FROM combo_data
ORDER BY Customer;
```
Question 9: When did each customer place their most recent order?
```sql
SELECT
	DISTINCT Customer,
	LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_patronage
FROM combo_data
ORDER BY Customer;
```
Question 10: How long did each customer actively patronize (in days)?
```sql
	SELECT
	DISTINCT Customer,
	FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC) AS first_patronage,
	LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_patronage,
	DATEDIFF(day, FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC),LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) ) + 1 as patronage_duration
FROM combo_data;
```
Question 11: What percentage of customers only ordered once (i.e., 1-day patronage)?
```sql
WITH patronage_data AS (SELECT
	DISTINCT Customer,
	FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC) AS first_patronage,
	LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_patronage,
	DATEDIFF(day, FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC),LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) ) + 1 AS patronage_duration
FROM combo_data)

SELECT COUNT(*) AS customer_that_patronized_for_1_day,
		(SELECT COUNT(*) FROM patronage_data) AS total_customers,
		COUNT(*) * 100/(SELECT COUNT(*) FROM patronage_data) AS percent_of_customers_that_patronized_for_1day
FROM patronage_data
WHERE patronage_duration = 1
```











