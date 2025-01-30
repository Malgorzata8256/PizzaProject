
# PizzaProject

--
	1. Conceptualize the dataset
		a. Identify grain, measures and dimensions
		b. Identify critical vs. Non critical columns
		c. Understand definitions

Dataset includes information about orders made in 2015
Four tables:
 - Order details - includes 48.620 rows and four columns:
     - Order_details_ID - primary key, serial number
     - Order_ID - foreign key (reference to OrdersTable)
     - Pizza_ID - foreign key (reference to PizzasTable)
     - Quantity - number of ordered pizzas 
 - Orders - includes 21.350 rows and three columns:
     - Order_ID - primary key, serial number
     - Date - date when order was placed
     - Time - time when order was placed
 - Pizza types - includes 33 rows and four columns:
     - Pizza_type_ID - primary key, created based on the pizza name
     - Name - list of all pizzas in the menu
     - Category - pizzas' categories (chicken, classic, supreme, veggie)
     - Ingredients - list of ingredients for each pizza
 - Pizzas - includes 97 rows and four columns:
     - Pizza_ID - primary key, created based on combination of Pizza_type_ID and Size
     - Pizza_type_ID - foreign key (reference to PizzaTypesTable)
     - Size - pizza size (S/M/L/XL/XXL)
     - Price - each pizza price in USD based on the pizza type and pizza size
Tables are complete, there is no missing or invalid information in the dataset.
I created database through PgAdmin, created tables and imported all files (CSV) .
--

	2. Locate solvable issues 
		a. Solvalbe
			i. Formatting (daty, czas, itp. Formatowanie) - I created tables in PgAdmin, formatting is correct and in CSV files it was also correct)
			ii. Consisciency (np. US --> USA wszędzie)  - data in all tables in consistent
			iii. Duplicates - there are no duplicates
	4. Evaluate unsolvable - N/D
			i. Missing values
			ii. Non logical dates
			iii. Unexisting for example countries

 
	5. Augmentation - calendar table to add in PowerBI
		a. Dodawanie np. kolumn albo dodatkowych tabel

## Creating tables in PgAdmin
```sql
CREATE TABLE Orders
		(
order_id SERIAL PRIMARY KEY	,
-- Unique identifier for each order placed by a table
date	DATE,
-- Date the order was placed (entered into the system prior to cooking & serving)
time TIME
-- Time the order was placed (entered into the system prior to cooking & serving)
		);
CREATE TABLE PizzaTypes
		(
pizza_type_id	VARCHAR(50) PRIMARY KEY,
-- Unique identifier for each pizza type
name	VARCHAR(50),
-- Name of the pizza as shown in the menu
category VARCHAR(50)	,
-- Category that the pizza fall under in the menu (Classic, Chicken, Supreme, or Veggie)
ingredients VARCHAR(150)
-- Comma-delimited ingredients used in the pizza as shown in the menu (they all include Mozzarella Cheese, even if not specified  and they all include Tomato Sauce, unless another sauce is specified)
		);
CREATE TABLE Pizzas
		(
pizza_id	VARCHAR(50) PRIMARY KEY,
-- Unique identifier for each pizza (constituted by its type and size)
pizza_type_id	VARCHAR(50),
-- Foreign key that ties each pizza to its broader pizza type
size VARCHAR(50),
-- Size of the pizza (Small, Medium, Large, X Large, or XX Large)
price DECIMAL,
-- Price of the pizza in USD
FOREIGN KEY (pizza_type_id) REFERENCES PizzaTypes(pizza_type_id)
		);
CREATE TABLE OrderDetails 
		(
order_details_id INTEGER PRIMARY KEY	,
-- Unique identifier for each pizza placed within each order (pizzas of the same type and size are kept in the same row, and the quantity increases)
order_id	INTEGER,
-- Foreign key that ties the details in each order to the order itself
pizza_id	VARCHAR(50),
-- Foreign key that ties the pizza ordered to its details, like size and price
quantity INTEGER,
-- Quantity ordered for each pizza of the same type and size
FOREIGN KEY (order_id) REFERENCES Orders(order_id),
FOREIGN KEY (pizza_id) REFERENCES Pizzas(pizza_id) 
		);

```

SQL Queries

## counting value of sold pizzas
```sql
SELECT od.pizza_id,
       od.quantity,
       p.price,
       od.quantity * p.price AS sales_value
FROM orderdetails od INNER JOIN pizzas p ON od.pizza_id = p.pizza_id;
```

## verification which pizza size is the most popular (by number of order units) and % share
```sql

SELECT p.size,
       SUM(od.quantity) AS pizza_by_size,
       ROUND(SUM(od.quantity) * 100.0 / SUM(SUM(od.quantity)) OVER (),2) AS percentage_share_by_size
FROM orderdetails od 
LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
GROUP BY p.size
ORDER BY pizza_by_size DESC;
```

## verification which pizza category is the most popular (by number of order units) and % share
```sql

SELECT  pt.category,
        SUM(od.quantity) AS pizza_by_category,
        ROUND(SUM(od.quantity) * 100.0 / SUM(SUM(od.quantity)) OVER (),2) AS percentage_share_by_category
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN pizzatypes AS pt ON p.pizza_type_id = pt.pizza_type_id
GROUP BY pt.category
ORDER BY pizza_by_category DESC;
```

## verification which pizza (name) is the most popular (by number of order units) and % share
```sql

SELECT  pt.name,
        SUM(od.quantity) AS pizza_by_name,
        OUND(SUM(od.quantity) * 100.0 / SUM(SUM(od.quantity)) OVER (),2) AS percentage_share_by_name
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN pizzatypes AS pt ON p.pizza_type_id = pt.pizza_type_id
GROUP BY pt.name
ORDER BY pizza_by_name DESC;
```

## the best selling pizza in each category
```sql
WITH CTE AS
		(
		SELECT  pt.name,
			pt.category,
			SUM(od.quantity) OVER (PARTITION BY pt.name, pt.category) AS best_pizza_by_category
		FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
			LEFT JOIN pizzatypes AS pt ON p.pizza_type_id = pt.pizza_type_id
		),

	CTE2 AS
		(
		SELECT  name,
			category, 
			best_pizza_by_category,
			DENSE_RANK() OVER (PARTITION BY category ORDER BY best_pizza_by_category DESC ) AS ranking_by_category
		FROM CTE
		)
SELECT DISTINCT(name),
	category, 
	best_pizza_by_category
FROM CTE2
WHERE ranking_by_category =1;
```

## the best selling pizza in each size
```sql
WITH CTE AS
		(
		SELECT  distinct(pt.name),
			p.size,
			SUM(od.quantity) OVER (PARTITION BY pt.name, p.size) AS best_pizza_by_size
		FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
			LEFT JOIN pizzatypes AS pt ON p.pizza_type_id = pt.pizza_type_id
		),
	CTE2 AS 
		(
		SELECT  name,
			size, 
			best_pizza_by_size,
		DENSE_RANK() OVER (PARTITION BY size ORDER BY best_pizza_by_size DESC ) AS ranking_by_size
		FROM CTE
		)
		
SELECT  DISTINCT(name),
	size,
	best_pizza_by_size
FROM CTE2
WHERE ranking_by_size = 1;
```

##  daily sales
```sql

SELECT o.date, 
       SUM(od.quantity * p.price) AS daily_sales
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id
	LEFT JOIN orders o ON o.order_id = od.order_id
GROUP BY o.date
ORDER BY o.date;
```

## seasonality check
```sql

SELECT TO_CHAR(o.date, 'Day') AS day_of_week, 
       COUNT(*) AS order_count,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage_share
FROM orders o
GROUP BY day_of_week
ORDER BY order_count DESC;
```

## value of pizzas sold each month 
```sql

SELECT DISTINCT (TO_CHAR (o.date, 'MM')) AS month,
	SUM(od.quantity * p.price) OVER (PARTITION BY TO_CHAR (o.date, 'MM'))  AS sales_value_by_month
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
ORDER BY month ;
```

## value of pizzas by quarter
```sql

SELECT DISTINCT (TO_CHAR (o.date, 'Q')) AS quarter,
	SUM(od.quantity * p.price) OVER (PARTITION BY TO_CHAR (o.date, 'Q'))  AS sales_value_by_quarter
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
ORDER BY quarter ;
```

## number of pizzas ordered by time buckets (buckets to create)
```sql

SELECT  CASE WHEN o.time BETWEEN '07:00:01' AND '12:00:00' THEN 'MORNING'
		WHEN o.time BETWEEN '12:00:01' AND '18:00:00' THEN 'AFTERNOON'
		WHEN o.time BETWEEN '18:00:01' AND '23:59:59' THEN 'EVENING'
		ELSE 'OTHER' END AS time_buckets,
	COUNT(od.quantity),
	ROUND(SUM(od.quantity) * 100.0 / SUM(SUM(od.quantity)) OVER (),2) AS percentage_share_by_bucket
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
GROUP BY time_buckets;
```

## number of orders
```sql

SELECT COUNT(*) AS orders_quantity
FROM orderdetails;
```

## number of orders monthly
```sql

SELECT DISTINCT (TO_CHAR (o.date, 'MM')) AS month,
	SUM(od.quantity) OVER (PARTITION BY TO_CHAR (o.date, 'MM'))  AS number_of_pizzas_monthly
FROM orderdetails od LEFT JOIN orders o ON o.order_id = od.order_id
ORDER BY month ;
```

## number of orders quaterly
```sql

SELECT DISTINCT (TO_CHAR (o.date, 'Q')) AS quarter,
	SUM(od.quantity) OVER (PARTITION BY TO_CHAR (o.date, 'Q'))  AS number_of_pizzas_quaterly
FROM orderdetails od LEFT JOIN orders o ON o.order_id = od.order_id
ORDER BY quarter ;
```

## AVG value of order
```sql

SELECT ROUND (SUM(od.quantity * p.price) / COUNT (od.quantity),2) AS AOV
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id;
```

## AVG order value by month
```sql

SELECT  TO_CHAR (o.date, 'MM') as month,
	ROUND (SUM(od.quantity * p.price) / COUNT (od.quantity),2) AS AOV_monthly
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
GROUP BY month
ORDER BY month;
```

## AVG order value by quarter
```sql

SELECT  TO_CHAR (o.date, 'Q') as quarter,
	ROUND (SUM(od.quantity * p.price) / COUNT (od.quantity),2) AS AOV_quaterly
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
GROUP BY quarter
ORDER BY quarter;
```

## AVG order value by time bucket
```sql

SELECT  CASE WHEN o.time BETWEEN '07:00:01' AND '12:00:00' THEN 'MORNING'
		WHEN o.time BETWEEN '12:00:01' AND '18:00:00' THEN 'AFTERNOON'
		WHEN o.time BETWEEN '18:00:01' AND '23:59:59' THEN 'EVENING'
		ELSE 'OTHER' END AS time_buckets,
	ROUND (SUM(od.quantity * p.price) / COUNT (od.quantity),2) AS AOV_time_buckets
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
GROUP BY time_buckets;
```

## AVG order value by hour
```sql

SELECT  EXTRACT(HOUR FROM o.time) as hour,
	ROUND (SUM(od.quantity * p.price) / COUNT (od.quantity),2) AS AOV_hourly
FROM orderdetails od LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id 
	LEFT JOIN orders o ON o.order_id = od.order_id
GROUP BY hour
ORDER BY hour;
```

## avg number of pizzas in order
```sql

SELECT ROUND(AVG(order_size),2)
FROM
		(
		SELECT order_id,
			SUM(quantity) AS order_size
		 FROM orderdetails
		 GROUP BY order_id
		) AS sub;
```

## pizzas combinations
```sql

SELECT a.pizza_id AS pizza_1, 
       b.pizza_id AS pizza_2, 
       COUNT(*) AS pair_count
FROM orderdetails a JOIN orderdetails b ON a.order_id = b.order_id
					AND a.pizza_id < b.pizza_id
GROUP BY pizza_1, pizza_2
ORDER BY pair_count DESC
LIMIT 10;
```

## the biggest orders
```sql

SELECT od.order_id, 
       SUM(od.quantity * p.price) AS order_value
FROM orderdetails od 
LEFT JOIN pizzas p ON od.pizza_id = p.pizza_id
GROUP BY od.order_id
ORDER BY order_value DESC
LIMIT 10;
```

## ingredients popularity
```sql

SELECT TRIM(UNNEST(STRING_TO_ARRAY(ingredients, ','))) AS ingredient, 
       COUNT(*) AS count
FROM pizzatypes
GROUP BY ingredient
ORDER BY count DESC;
```

