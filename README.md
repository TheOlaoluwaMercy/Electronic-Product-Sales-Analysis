# Electronic-Product-Sales-Analysis
This project is a proof of my SQL window function skills.

Through a series of business-driven queries, the analysis answers key questions around profitability trends, customer purchase behaviour, and category performance.

**Database diagram**
![DB Diagram](https://github.com/user-attachments/assets/1f2c1df0-ec57-4d0a-bebb-eeb93508f433)

**Task 1:  Create a view that includes Order_Number, Line_Item, Order_Date, Quarter (YYYY-Q#), Month (YYYY-MM), Product_Name, Product_Category, Customer_Name, Store_Country, Quantity, and Profit (USD)—calculated as (Unit Price USD - Unit Cost USD) * Quantity — by joining relevant fields from the Sales, Customers, Products, and Stores tables.**

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

**Task 2: Query the view `combo_data` to answer the following business questions**

**Question 1: Rank Product categories based on the total profit generated****

  ```sql
  SELECT 
    Category,
    SUM(Profit) AS Total_Profit,
    RANK() OVER (ORDER BY SUM(Profit) DESC) AS Profit_Rank
FROM combo_data
GROUP BY Category;
  ```
![Question 1](https://github.com/user-attachments/assets/7cbdcd63-d136-4386-88e1-9924195e962d)


**Question 2: What is each product category’s share of overall profit?**
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
![Question 2](https://github.com/user-attachments/assets/60d7cf47-f7d6-44f2-9d3f-fc73f4606cc7)


**Question 3: What is the running total of profit by quarter?**
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
![Question 3](https://github.com/user-attachments/assets/c1f75eb3-7ad5-49b8-a7bd-e43d19471e1d)


**Question 4: What is the cumulative percentage of profit over quarters?**
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
![Question 4](https://github.com/user-attachments/assets/fd91f48c-2329-47e5-b363-4e9fdecb0bdc)


**Question 5: What is the 3-month moving average of monthly profit?**
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
![Question 5](https://github.com/user-attachments/assets/33c540e5-df29-4cb1-b779-ace5a4ebb889)


**Question 6: How does monthly profit change compared to the previous month?**
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
![Question 6](https://github.com/user-attachments/assets/9ef9b959-e5de-479b-a8dc-7fef924d5ccb)


**Question 7: Which product categories in 2020 performed above or below the average?**
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
![Question 7](https://github.com/user-attachments/assets/cf275c95-5182-4e5c-8122-7c77780612dc)


**Question 8: When did each customer place their first order?**
```sql
SELECT
	DISTINCT Customer,
	FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC) AS first_patronage
FROM combo_data
ORDER BY Customer;
```
![Question 8](https://github.com/user-attachments/assets/76478280-0ef2-49ab-b700-a342f88a8cba)


**Question 9: When did each customer place their most recent order?**
```sql
SELECT
	DISTINCT Customer,
	LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_patronage
FROM combo_data
ORDER BY Customer;
```
![Question 9](https://github.com/user-attachments/assets/0142230d-f198-4aac-a36c-89ee2d9bfce0)


**Question 10: How long did each customer actively patronize (in days)?**
```sql
	SELECT
	DISTINCT Customer,
	FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC) AS first_patronage,
	LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_patronage,
	DATEDIFF(day, FIRST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC),LAST_VALUE(Order_Date) OVER(PARTITION BY Customer ORDER BY Order_Date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) ) + 1 as patronage_duration
FROM combo_data;
```
![Question 10](https://github.com/user-attachments/assets/7651499d-eebe-4ff9-9916-a51f6459c826)


**Question 11: What percentage of customers only ordered once (i.e., 1-day patronage)?**
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
![Question 11](https://github.com/user-attachments/assets/366ee9c3-d6e4-43e2-b894-588e300f43b8)











