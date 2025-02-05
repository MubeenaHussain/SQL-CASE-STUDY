## Business Task: 
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.
🙋‍♀️ View the Question  [here](https://8weeksqlchallenge.com/case-study-1/).


### Creation of "dannys_diner" Database
````
CREATE DATABASE dannys_diner;
USE dannys_diner;
CREATE TABLE sales
(
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);
```` 
````
INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);
````
````
INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
````
-- Database Creation Completed




### CASE STUDY QUESTIONS 

### Q1. What is the total amount each customer spent at the restaurant? 
````
SELECT s.customer_id, SUM(price) AS total_amount_spent
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id;
````

### Q2. How many days has each customer visited the restaurant?
````
SELECT customer_id , COUNT(DISTINCT order_date) AS days_visited
FROM sales
GROUP BY customer_id;
````
### Q3. What was the first item from the menu purchased by each customer?
-- Assuming that if the order_date is same for a customer, the item listed first in the table was purchased first.
-- Using ROW_NUMBER()
````
 SELECT S.customer_id,S.product_name AS first_product
FROM
(SELECT s.*,m.product_name,ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS rnk
FROM sales s
JOIN menu m
ON s.product_id = m.product_id) AS S
WHERE S.rnk = 1;
````
-- Using RANK()

````
SELECT S.customer_id,S.product_name AS first_product
FROM
(SELECT s.*,m.product_name,RANK() OVER (PARTITION BY customer_id ORDER BY order_date) AS rnk
FROM sales s
JOIN menu m
ON s.product_id = m.product_id) AS S
WHERE S.rnk = 1;
````
 ROW_NUMBER() and RANK(), both the window functions can be used to assign a rank to the rows of the table. However, the latter considers 
duplicate records and assigns them same rank whereas the former assigns unique ranks to such records.Here, we want to fetch the row of rank 1 from a window partitioned based on 'customer_id' ordered by 'order_date' to get the first product ordered by the customer.We have duplicate values of order_date for some customers as they may have ordered more than 1 item in an order.Now, using RANK() gives two values for customer 'A' and 'C' as both of them ordered two products in a single order due to which two rows got assigned the rank 1.Using ROW_NUMBER() gives a single result here as even for the same order_date, the product listed first in the table is assigned the rank 1.
--The first product purchased by customers A, B and C is sushi,curry and ramen, respectively.

### Q4. What is the most purchased item on the menu and how many times was it purchased by all customers? 
````
SELECT s.product_id,m.product_name,COUNT(s.product_id) as number_purchases
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
GROUP BY s.product_id,m.product_name;
````
-- Here, as we can see from the table, Ramen is the most purchased item on the menu and overall, it was purchased for 8 times.


### Q5. Which item was the most popular for each customer? 
````
SELECT X.customer_id,X.product_name,X.order_count
FROM
(SELECT s.customer_id,m.product_name,COUNT(s.product_id) as order_count,
RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC ) as rnk
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
GROUP BY m.product_name,s.customer_id) AS X
WHERE X.rnk = 1;
````
-- Ramen is the most popular item for customers 'A' and 'C' while 'B' likes all the three items equally.


### Q6. Which item was purchased first by the customer after they became a member?

--Let's assume that if the order_date and join_date are same for a member, they became the member first and then placed the order.
--Nested Query Approach
````
SELECT X.customer_id,X.product_name
FROM
(SELECT s.customer_id,m.product_name,RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date) as rnk
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
WHERE s.order_date >= me.join_date) AS X
WHERE X.rnk = 1;
````
CTE Approach
````
WITH members_first_item_cte AS
(
SELECT s.customer_id,m.product_name,RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date) as rnk
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
WHERE s.order_date >= me.join_date
)
SELECT customer_id,product_name
FROM members_first_item_cte
WHERE rnk = 1;
````
--A common application of Nested Query and CTE is that both of them can be used to create temporary tables which exist only within the scope of the query.


### Q7. Which item was purchased just before the customer became a member? 
````
SELECT X.customer_id,X.product_name,X.order_date,X.join_date
FROM
(SELECT s.customer_id,m.product_name,s.order_date,me.join_date,RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) as rnk
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
WHERE s.order_date < me.join_date) AS X
WHERE X.rnk = 1;
````
--So 'A' bought sushi and curry before becoming a member. And 'B' bought sushi before becoming a member.


### Q8. What is the total items and amount spent for each member before they became a member? 
````
SELECT s.customer_id, COUNT(s.product_id) AS total_items, SUM(m.price) AS amount_spent
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
WHERE s.order_date < me.join_date
GROUP BY s.customer_id
ORDER BY s.customer_id;
````
### What is the total items and amount spent for each member for each item before they became a member? 
````
SELECT s.customer_id, m.product_name,COUNT(s.product_id) AS total_items, SUM(m.price) AS amount_spent
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
WHERE s.order_date < me.join_date
GROUP BY m.product_name,s.customer_id
ORDER BY s.customer_id,m.product_name;
````
--To aggregate at different levels of granularity of data, use columns in the GROUP BY clause.
--Increase the number of columns OR Keep adding columns to the GROUP BY clause to increase the level of granularity (Highest to lowest level)


### Q9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have? 
````
SELECT X.customer_id, SUM(X.points) AS total_points
FROM
(SELECT s.customer_id, 
CASE WHEN m.product_name = 'sushi' THEN ( 2*m.price*10)
ELSE m.price*10
END AS points
FROM sales s
JOIN menu m
ON s.product_id = m.product_id) AS X
GROUP BY X.customer_id
ORDER BY X.customer_id;
````
-- Case Statement inside Sum
````
SELECT s.customer_id, 
SUM(CASE WHEN m.product_name = 'sushi' THEN ( 2*m.price*10)
ELSE m.price*10
END )AS points
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id;
````

### Q10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many point  do customer A and B have at the end of January? 
````
WITH total_points_cte AS
(
SELECT s.customer_id,s.order_date,m.price,m.product_name,me.join_date, DATEADD(day,6,join_date) AS first_week
FROM sales s
JOIN menu m
ON s.product_id =m.product_id
JOIN members me
ON s.customer_id = me.customer_id
)
SELECT customer_id, 
SUM(CASE WHEN order_date BETWEEN join_date AND first_week THEN (2*price*10)
         WHEN (order_date NOT BETWEEN join_date AND first_week) AND product_name ='sushi' THEN (2*price*10)
		 ELSE (price*10)
		 END) AS total_points
FROM total_points_cte
GROUP BY customer_id;
````
### BONUS QUESTION : Join all the Things 
````
 SELECT s.customer_id,s.order_date,m.product_name,m.price,
 CASE WHEN (s.order_date >= me.join_date) AND (s.customer_id = me.customer_id) THEN 'Y'
 ELSE 'N'
 END AS member
 FROM sales s
 JOIN menu m
 ON s.product_id = m.product_id
 LEFT JOIN members me
 ON s.customer_id = me.customer_id;
````
### BONUS QUESTION: Rank all the Things 
````
  WITH cte AS
 (
 SELECT s.customer_id,s.order_date,m.product_name,m.price,
 CASE WHEN (s.order_date >= me.join_date) AND (s.customer_id = me.customer_id) THEN 'Y'
 ELSE 'N'
 END AS member
 FROM sales s
 JOIN menu m
 ON s.product_id = m.product_id
 LEFT JOIN members me
 ON s.customer_id = me.customer_id
 )
 SELECT *,
 CASE WHEN member = 'N' THEN 'null'
 ELSE RANK() OVER (PARTITION BY customer_id ORDER BY order_date)
 END AS ranking
 FROM cte
 ORDER BY customer_id;
 ````
 
 ----------------------------------------------------------------------------------------------------------------------------------------------------------------------












