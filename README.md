# Sales SQL Project
This project provides an in-depth analysis of sales, product performance, and customer behavior using SQL. The dataset comprises transactional sales records, customer details, and product information. The goal is to extract valuable insights that help understand trends, performance metrics, and customer segmentation for better business decision-making

## **Change-Over-Time Analysis**
This section examines how sales evolve over time, identifying trends, patterns, and seasonal variations. Key insights include:

* Yearly Sales Trends: Analyzing total sales per year to understand business growth.
* Monthly Trends: Drilling down into month-by-month performance to detect seasonal fluctuations. December emerges as the best-performing month, while February sees the lowest sales.
* Date Truncation: Using SQL date functions to refine time-based analysis, ensuring accurate and consistent reporting.
```sql
--how a measure evolves over time ~ Trends
SELECT 
order_date,
SUM(sales_amount) AS total_sales
FROM fact_sales
WHERE order_date IS NOT NULL
GROUP BY EXTRACT(YEAR FROM order_date)
ORDER BY EXTRACT(YEAR FROM order_date) ASC;

SELECT 
	EXTRACT(YEAR FROM order_date) AS year,
	SUM(sales_amount) AS total_sales,
	COUNT( DISTINCT customer_key) AS total_customers,
	SUM(quantity) AS total_quantity
FROM fact_sales
WHERE order_date IS NOT NULL
GROUP BY EXTRACT(YEAR FROM  order_date)
ORDER BY EXTRACT(YEAR FROM order_date) ASC;

--drill down to months 
--understanding seasonality and the trends patterns
--the best month is december and the worst is february

SELECT 
	EXTRACT(MONTH FROM order_date) AS month,
	SUM(sales_amount) AS total_sales,
	COUNT( DISTINCT customer_key) AS total_customers,
	SUM(quantity) AS total_quantity
FROM fact_sales
WHERE order_date IS NOT NULL
GROUP BY EXTRACT(MONTH FROM  order_date)
ORDER BY EXTRACT(MONTH FROM order_date) ASC;

--making it more specific by adding year information
SELECT 
	EXTRACT(YEAR FROM order_date) AS year,
	EXTRACT(MONTH FROM order_date) AS month,
	SUM(sales_amount) AS total_sales,
	COUNT( DISTINCT customer_key) AS total_customers,
	SUM(quantity) AS total_quantity
FROM fact_sales
WHERE order_date IS NOT NULL
GROUP BY EXTRACT(YEAR FROM order_date),
	EXTRACT(MONTH FROM  order_date)
ORDER BY EXTRACT(YEAR FROM order_date),
	EXTRACT(MONTH FROM order_date) ASC;

--Using date trunc
SELECT 
    TO_CHAR(DATE_TRUNC('month', order_date), 'YYYY-Month') AS month, -- Format as 'YYYY'
    SUM(sales_amount) AS total_sales,
    COUNT(DISTINCT customer_key) AS total_customers,
    SUM(quantity) AS total_quantity
FROM fact_sales
WHERE order_date IS NOT NULL
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY DATE_TRUNC('month', order_date) ASC;
```


## **Cumulative Analysis**


```sql
--aggregating data progressively over time
--helps to understand whether our business is growing or declining
--calculate the total sales per month and the running  total of sales over time

SELECT
    year,
    month,
    total_sales,
    SUM(total_sales) OVER (
        PARTITION BY year ORDER BY month ASC
    ) AS running_total_sales,
	AVG(avg_price) OVER (
        PARTITION BY year ORDER BY month ASC
    ) AS moving_avg_price
FROM (
    SELECT 
        EXTRACT(YEAR FROM order_date) AS year,
        EXTRACT(MONTH FROM order_date) AS month,
        SUM(sales_amount) AS total_sales,
		AVG(price) AS avg_price
    FROM fact_sales
    WHERE order_date IS NOT NULL
    GROUP BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)
) AS monthly_sales
ORDER BY year, month;
```
## **Perfomance Analysis**

```sql
--helps meausre succes and compare perfomance
--[Current Measure]- Target[Meausure]

--Task
--Analyse the yearly perfomance of products by comparing 
--each product's sales to  both its average sales perfomance and the previous year's sales

WITH yearly_product_sales
AS
	(SELECT 
	EXTRACT(YEAR FROM f.order_date) AS year,
	p.product_name,
	SUM(f.sales_amount) AS yearly_sales_amount
	FROM fact_sales AS f
	LEFT JOIN dim_products AS p
	ON f.product_key = p.product_key
	WHERE order_date IS NOT NULL
	GROUP BY year, p.product_name )
SELECT
	year,
	product_name,
	yearly_sales_amount,
	AVG(yearly_sales_amount) OVER ( PARTITION BY product_name) AS avg_sales,
	yearly_sales_amount-(AVG(yearly_sales_amount) 
	OVER ( PARTITION BY product_name)) AS difference_avg,
	CASE
		WHEN yearly_sales_amount-(AVG(yearly_sales_amount) 
		OVER ( PARTITION BY product_name))>0 THEN 'ABOVE AVERAGE'
		WHEN yearly_sales_amount-(AVG(yearly_sales_amount) 
		OVER ( PARTITION BY product_name)) <0 THEN 'BELOW AVERAGE'
		ELSE 'AVERAGE'  
		END AS avg_change,
		--year-over-year analysis
		-- good for long term trends analysis
	LAG(yearly_sales_amount) OVER (PARTITION BY product_name ORDER BY year) AS previous_year_sales,
	yearly_sales_amount-LAG(yearly_sales_amount) OVER (PARTITION BY product_name ORDER BY year) AS difference_py,
	CASE
		WHEN yearly_sales_amount-LAG(yearly_sales_amount) OVER (PARTITION BY product_name ORDER BY year)>0 THEN 'INCREASINGE'
		WHEN yearly_sales_amount-LAG(yearly_sales_amount) OVER (PARTITION BY product_name ORDER BY year) <0 THEN 'DECREASING'
		ELSE 'NO CHANGE'  
		END AS py_change
FROM yearly_product_sales
ORDER BY product_name, year
```


## **Part-to-Whole Analysis**

```sql
--Analyse how an individual part is perfomimg compared to overall
--allowing us to understand which category has the greatest impact on the business
--[Measure]/Total[Measure]*100  (visualize it like pie chart)

--Task
--which category contributes the most to overall sales

WITH category_sales 
AS 
	(SELECT 
		p.category,
		SUM(sales_amount) AS total_sales
	FROM fact_sales AS f
	LEFT JOIN dim_products AS p
	ON p.product_key = f.product_key
	GROUP BY category)
SELECT 
	category, 
	total_sales,
	SUM( total_sales) OVER () AS overall_sales,
	ROUND(total_sales::numeric /(SUM( total_sales) OVER ())*100,2) AS percentage_of_total
FROM category_sales
ORDER BY total_sales DESC;
--overrelying on one category, can be dangerous
```

## **Data Segmentation**

```sql
-- grup the data based on a specific range
--helps understand the correlation between two measures
--[Measure] By [Measure] (not dimension) ( Total Products by Sales Range / total Customers by Age)
-- task : segment products into cost ranges and count how many products fal into each segment

WITH product_segments
AS
	(SELECT 
		product_key,
		product_name,
		cost,
		CASE
			WHEN cost <100 THEN 'BELOW 100'
			WHEN cost BETWEEN 100 AND 500 THEN 'BETWEEN 100 AND 500'
			WHEN cost BETWEEN 500 AND 1000 THEN ' BETWEEN 500 AND 1000'
			ELSE 'ABOVE 1000'
			END AS cost_range
	FROM dim_products)
SELECT 
	cost_range,
	COUNT(product_key) AS total_products
FROM product_segments
GROUP BY cost_range
ORDER BY total_products DESC;

/* Group customers into three segments based on their spending behavior:
-VIP : Customers with at least 12 months of history and spemding more than €5000 
-Regular: Customers with at least 12 months history but spending €5000 or less
-New: Customers with a lifespan less than 12 months
And find the total number of customers by each group*/

WITH customer_spending
AS
	(SELECT 
		c.customer_key,
		SUM(f.sales_amount) AS total_spending,
		MIN(order_date) AS first_order,
		MAX(order_date) AS last_order,
		EXTRACT(YEAR FROM AGE(MAX(order_date), MIN(order_date))) * 12 +
		EXTRACT(MONTH FROM AGE(MAX(order_date), MIN(order_date))) AS lifespan
	FROM fact_sales f
	LEFT JOIN dim_customers c
	ON f.customer_key= c.customer_key
	GROUP BY c.customer_key)
SELECT
customer_segment,
COUNT(customer_key) AS total_customers,
SUM(total_spending) AS total_spent
FROM
(SELECT
	customer_key,
	total_spending,
	lifespan,
	CASE
		WHEN lifespan >= 12 AND total_spending >5000 THEN 'VIP'
		WHEN lifespan >= 12 AND total_spending <= 5000 THEN 'Regular'
		ELSE 'New'
		END AS customer_segment
FROM customer_spending)
GROUP BY customer_segment
ORDER BY total_customers DESC;
```

## **Customer report**

```sql
/*
===============================================================================
Customer Report
===============================================================================
Purpose:
    - This report consolidates key customer metrics and behaviors

Highlights:
    1. Gathers essential fields such as names, ages, and transaction details.
	2. Segments customers into categories (VIP, Regular, New) and age groups.
    3. Aggregates customer-level metrics:
	   - total orders
	   - total sales
	   - total quantity purchased
	   - total products
	   - lifespan (in months)
    4. Calculates valuable KPIs:
	    - recency (months since last order)
		- average order value
		- average monthly spend
===============================================================================
*/

CREATE VIEW report_customers
AS
WITH base_query
AS
	(SELECT 
		f.order_number,
		f.product_key,
		f.order_date,
		f.sales_amount,
		f.quantity,
		c.customer_key,
		c.customer_number,
		CONCAT(c.first_name,' ',c.last_name) AS customer_name,
	EXTRACT (YEAR FROM AGE(c.birthdate)) AS age
	FROM fact_sales AS f
	LEFT JOIN dim_customers AS c
	ON c.customer_key = f.customer_key),
customer_aggregation AS 
	(SELECT
		customer_key,
		customer_number,
		customer_name,
		age,
		COUNT(DISTINCT order_number) AS total_orders,
		SUM(sales_amount) AS total_sales,
		SUM(quantity) AS total_quantity,
		COUNT(DISTINCT product_key)AS total_products,
		MAX(order_date) AS last_order_date,
		EXTRACT(YEAR FROM AGE(MAX(order_date), MIN(order_date))) * 12 +
				EXTRACT(MONTH FROM AGE(MAX(order_date), MIN(order_date))) AS lifespan
	FROM base_query
	GROUP BY 
		customer_key,
		customer_number,
		customer_name,
		age)
SELECT
	customer_key,
	customer_number,
	customer_name,
	age,
	CASE 
		WHEN age < 20 THEN 'under 20'
		WHEN age BETWEEN 20 and 29 THEN '20-29'
		WHEN age BETWEEN 30 and 39 THEN '30-39'
		WHEN age BETWEEN 40 and 49 THEN '40-49'
		ELSE '50 and above'
		END AS age_group,
	CASE
			WHEN lifespan >= 12 AND total_sales >5000 THEN 'VIP'
			WHEN lifespan >= 12 AND total_sales <= 5000 THEN 'Regular'
			ELSE 'New'
			END AS customer_segment,
	total_orders,
	total_sales,
	total_quantity,
	total_products,
	last_order_date,
	(EXTRACT(YEAR FROM AGE(NOW(), last_order_date)) * 12) + 
	EXTRACT(MONTH FROM AGE(NOW(), last_order_date)) AS recency,
	lifespan,
	CASE 
		WHEN total_orders = 0 THEN 0
		ELSE
		total_sales/total_orders
		END AS avg_order_value,
	CASE 
		WHEN lifespan = 0 THEN total_sales
		ELSE ROUND(total_sales/lifespan, 2)
		END AS avg_monthly_spend
FROM customer_aggregation
```


## **Products report**

```sql
/*
===============================================================================
Product Report
===============================================================================
Purpose:
    - This report consolidates key product metrics and behaviors.

Highlights:
    1. Gathers essential fields such as product name, category, subcategory, and cost.
    2. Segments products by total sales to identify High-Performers, Mid-Range, or Low-Performers.
    3. Aggregates product-level metrics:
       - total orders
       - total sales
       - total quantity sold
       - total customers (unique)
       - lifespan (in months)
    4. Calculates valuable KPIs:
       - recency (months since last sale)
       - average order revenue (AOR)
       - average monthly revenue
===============================================================================
*/

DROP VIEW IF EXISTS  report_products; 
CREATE VIEW report_products
AS
	WITH product_aggregation
	AS
		(SELECT 
			p.product_key,
			p.product_id,
			p.product_name,
			p.category,
			p.subcategory,
			p.cost,
			f.order_date,
			MAX(f.order_date) AS last_order_date,
			COUNT(DISTINCT f.customer_key) AS total_unique_customers,
			SUM(f.sales_amount) AS total_sales,
			SUM(f.quantity) AS total_quantity,
			COUNT( DISTINCT f.order_number) AS total_orders,
			EXTRACT(YEAR FROM AGE(MAX(f.order_date), MIN(f.order_date))) * 12 +
						EXTRACT(MONTH FROM AGE(MAX(f.order_date), MIN(f.order_date))) AS lifespan
		FROM fact_sales AS f
		LEFT JOIN dim_products AS p
		ON p.product_key = f.product_key
		GROUP BY 
			p.product_key,
			p.product_id,
			p.product_number,
			p.product_name,
			p.category,
			p.subcategory,
			p.cost,
			f.order_date)
	SELECT 
			product_key,
			product_id,
			product_name,
			category,
			subcategory,
			cost,
			total_unique_customers,
			total_sales,
			CASE
				WHEN total_sales < 5000 THEN 'Low Perfomance'
				WHEN  total_sales BETWEEN 5000 AND 10000 THEN  'Mid Perfomance'
				ELSE 'High Perfomance'
				END AS product_segments,
			total_quantity,
			total_orders,
			lifespan,
			last_order_date,
			(EXTRACT(YEAR FROM AGE(NOW(), last_order_date)) * 12) + 
			 EXTRACT(MONTH FROM AGE(NOW(), last_order_date)) AS recency,
			 CASE 
			WHEN total_orders = 0 THEN 0
			ELSE
			total_sales/total_orders
			END AS avg_order_value,
		CASE 
			WHEN lifespan = 0 THEN total_sales
			ELSE ROUND(total_sales/lifespan, 2)
			END AS avg_monthly_spend
	FROM product_aggregation
	ORDER BY total_sales DESC;
```






