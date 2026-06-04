# Monday Coffee Expansion SQL Project

![Company Logo](logo.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?
```sql
SELECT 
	city_name,
	CAST(ROUND((population * 0.25)/1000000,2) AS NUMERIC(10,2)) AS coffee_consumer_in_millions,
	city_rank
FROM [dbo].[city]
ORDER BY coffee_consumer_in_millions DESC;
```

2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
```sql
SELECT
	ct.city_id,
	ct.city_name,
	SUM(s.total) AS Total_Revenue
FROM [dbo].[sales] s
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
JOIN [dbo].[city] ct
ON c.city_id = ct.city_id
WHERE YEAR(s.sale_date) = 2023 AND DATEPART(QUARTER,s.sale_date) = 4 
GROUP BY ct.city_id,ct.city_name;
```

3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?
```sql
SELECT
	p.product_id,
	p.product_name,
	COUNT(s.sale_id) AS Total_Units
FROM [dbo].[sales] s
RIGHT JOIN [dbo].[products] p
ON s.product_id = p.product_id
GROUP BY p.product_id,p.product_name
ORDER BY Total_Units DESC;
```

4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
```sql
SELECT 
	ct.city_name,
	SUM(total) AS Total_Revenue,
	COUNT(DISTINCT s.customer_id) AS total_customers,
	ROUND(SUM(total) / COUNT(DISTINCT s.customer_id),2) AS avg_sale_per_cust
FROM [dbo].[sales] s
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
JOIN [dbo].[city] ct
ON c.city_id = ct.city_id
GROUP BY ct.city_name
ORDER BY Total_Revenue DESC;
```

5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers.
```sql
WITH city_list AS (
	SELECT
		city_name,
		population,
		CAST(ROUND((population * 0.25)/1000000,2) AS NUMERIC(10,2)) AS estimated_coffee_consumers_in_millions
	FROM [dbo].[city]
),
customers_table AS (
	SELECT
		ct.city_name,
		COUNT(DISTINCT s.customer_id) AS unique_customers
	FROM [dbo].[sales] s
	JOIN [dbo].[customers] c
	ON s.customer_id = c.customer_id
	JOIN [dbo].[city] ct
	ON ct.city_id = c.city_id
	GROUP BY ct.city_name
)
SELECT
	cl.city_name,
	cl.population,
	cl.estimated_coffee_consumers_in_millions,
	ct.unique_customers
FROM customers_table ct
JOIN city_list cl
ON ct.city_name = cl.city_name;
```

6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?
```sql
SELECT
	*
FROM (
SELECT 
	ct.city_name,
	p.product_name,
	COUNT(s.sale_id) AS total_orders,
	DENSE_RANK() OVER(PARTITION BY ct.city_name ORDER BY COUNT(s.sale_id) DESC) AS ranking
FROM [dbo].[sales] s
JOIN [dbo].[products] p
ON s.product_id = p.product_id
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
JOIN [dbo].[city] ct
ON c.city_id = ct.city_id
GROUP BY ct.city_name,p.product_name
)t
WHERE ranking <= 3;
```

7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products?
```sql
SELECT
	ct.city_name,
	COUNT(DISTINCT s.customer_id) AS Unique_Customers
FROM [dbo].[sales] s
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
RIGHT JOIN [dbo].[city] ct
ON ct.city_id = c.city_id
WHERE s.product_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14)
GROUP BY ct.city_name;
```

8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer
```sql
WITH city_table AS (
SELECT
	ct.city_name,
	SUM(total) AS Total_Revenue,
	COUNT(DISTINCT s.customer_id) AS Unique_Customers,
	ROUND(SUM(total) / COUNT(DISTINCT s.customer_id),2) AS avg_revenue_per_customer
FROM [dbo].[sales] s
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
JOIN [dbo].[city] ct
ON ct.city_id = c.city_id
GROUP BY ct.city_name
),
city_rent AS (
	SELECT
		city_name,
		estimated_rent
	FROM [dbo].[city]
)
SELECT
	cr.city_name,
	ct.Total_Revenue,
	ct.Unique_Customers,
	ct.avg_revenue_per_customer,
	cr.estimated_rent,
	ROUND((cr.estimated_rent / ct.Unique_Customers),2) AS avg_rent_per_customer
FROM city_rent cr
LEFT JOIN city_table ct
ON cr.city_name = ct.city_name
ORDER BY avg_revenue_per_customer DESC;
```

9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).
```sql
WITH monthly_sales AS (
SELECT
	ct.city_name,
	YEAR(s.sale_date) AS yr,
	MONTH(s.sale_date) AS mth,
	SUM(s.total) AS Total_sale
FROM [dbo].[sales] s
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
JOIN [dbo].[city] ct
ON ct.city_id = c.city_id
GROUP BY ct.city_name, YEAR(s.sale_date), MONTH(s.sale_date)
),
growth_ratio AS (
	SELECT
		city_name,
		yr,
		mth,
		Total_sale AS Current_Month_Sale,
		LAG(Total_sale) OVER(PARTITION BY city_name ORDER BY yr,mth) AS Previous_Month_Sale 
	FROM monthly_sales
)
SELECT
	city_name,
	yr,
	mth,
	Current_Month_Sale,
	Previous_Month_Sale,
	ROUND((Current_Month_Sale - Previous_Month_Sale) * 100.0 / Previous_Month_Sale,2) AS growth_ratio
FROM growth_ratio
WHERE Previous_Month_Sale IS NOT NULL;
```

10. **Market Potential Analysis**  
    Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated  coffee consumer
```sql
WITH city_table AS (
SELECT
	ct.city_name,
	SUM(total) AS Total_Revenue,
	COUNT(DISTINCT s.customer_id) AS Unique_Customers,
	ROUND(SUM(total) / COUNT(DISTINCT s.customer_id),2) AS avg_revenue_per_customer
FROM [dbo].[sales] s
JOIN [dbo].[customers] c
ON s.customer_id = c.customer_id
JOIN [dbo].[city] ct
ON ct.city_id = c.city_id
GROUP BY ct.city_name
),
city_rent AS (
	SELECT
		city_name,
		estimated_rent,
		CAST(ROUND((population * 0.25)/1000000,2)AS NUMERIC(10,2)) AS estimated_coffee_consumer_in_millions
	FROM [dbo].[city]
)
SELECT
	cr.city_name,
	ct.Total_Revenue,
	cr.estimated_rent AS total_rent,
	ct.Unique_Customers,
	cr.estimated_coffee_consumer_in_millions,
	ct.avg_revenue_per_customer,
	ROUND(cr.estimated_rent/ct.Unique_Customers,2) AS avg_rent_per_customer
FROM city_rent cr
LEFT JOIN city_table ct
ON cr.city_name = ct.city_name
ORDER BY ct.Total_Revenue DESC;
```   

## Recommendations
After analyzing the data, the recommended top three cities for new store openings are:

**City 1: Pune**  
1. Average rent per customer is very low.  
2. Highest total revenue.  
3. Average sales per customer is also high.

**City 2: Delhi**  
1. Highest estimated coffee consumers at 7.7 million.  
2. Highest total number of customers, which is 68.  
3. Average rent per customer is 330 (still under 500).

**City 3: Jaipur**  
1. Highest number of customers, which is 69.  
2. Average rent per customer is very low at 156.  
3. Average sales per customer is better at 11.6k.

---
