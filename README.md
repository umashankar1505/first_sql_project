# first_sql_project(unguided)
# Case Study: Customer Insights for Danny's Restaurant

Danny wants to gain deeper insights into his customers' visiting patterns, spending habits, and menu preferences to provide a more personalized experience.
These insights will also help him decide whether to expand the existing customer loyalty program.

## Objectives
- Understand customer behavior and preferences.
- Analyze total revenue, transaction trends, and favorite menu items.
- Differentiate spending patterns between loyalty members and non-members.
- Prepare datasets for the team to inspect without needing SQL expertise.

## Schema
# SQL Schema for Danny's Diner

This SQL script creates the schema and tables for `dannys_diner` with sample data for `sales`, `menu`, and `members`.

```sql
-- Create Schema and Set Search Path
CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

-- Create Sales Table
CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

-- Insert Data into Sales Table
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

-- Create Menu Table
CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

-- Insert Data into Menu Table
INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');

-- Create Members Table
CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

-- Insert Data into Members Table
INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');


## Queries to work on 
# Customer Insights SQL Queries

This file contains SQL queries designed to extract insights about customer behavior and restaurant performance.

```sql
-- 1. Total Amount Each Customer Spent
SELECT 
    customer_id,
    SUM(price) AS total
FROM 
    sales AS s
JOIN 
    menu AS m
ON 
    s.product_id = m.product_id
GROUP BY 
    customer_id;

-- 2. Number of Days Each Customer Visited
SELECT 
    customer_id,
    COUNT(order_date) AS days_visited
FROM 
    sales
GROUP BY 
    customer_id;

-- 3. First Item Purchased by Each Customer
SELECT 
    customer_id, 
    product_name
FROM (
    SELECT 
        customer_id, 
        order_date, 
        product_name,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date ASC) AS ron
    FROM 
        sales AS s
    JOIN 
        menu AS m
    ON 
        s.product_id = m.product_id
) AS first_prod
WHERE 
    ron = 1;

-- 4. Most Purchased Item on the Menu
SELECT 
    product_id,
    COUNT(product_id) AS purchasecount
FROM 
    sales
GROUP BY 
    product_id
ORDER BY 
    purchasecount DESC;

-- 5. Most Popular Item Per Customer
WITH itempopularity AS (
    SELECT 
        customer_id,
        COUNT(product_id) AS cnt,
        RANK() OVER (PARTITION BY customer_id ORDER BY COUNT(product_id) DESC) AS rn
    FROM 
        sales
    GROUP BY 
        customer_id
)
SELECT 
    customer_id, 
    cnt
FROM 
    itempopularity
WHERE 
    rn = 1;

-- 6. First Item Purchased After Membership
SELECT 
    m.customer_ID, 
    s.product_ID, 
    n.product_name
FROM 
    sales AS s
JOIN 
    members AS m 
ON 
    m.customer_id = s.customer_id
JOIN 
    menu AS n 
ON 
    n.product_id = s.product_id
WHERE 
    m.join_date = s.order_date;

-- 7. Item Purchased Just Before Membership
WITH cte AS (
    SELECT 
        m.customer_ID, 
        s.product_ID, 
        n.product_name,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY product_id DESC) AS rn
    FROM 
        sales AS s
    JOIN 
        members AS m
    ON 
        m.customer_id = s.customer_id
    JOIN 
        menu AS n
    ON 
        n.product_id = s.product_id
    WHERE 
        m.join_date > s.order_date
)
SELECT 
    customer_id, 
    product_id, 
    product_name
FROM 
    cte
WHERE 
    rn = 1;

-- 8. Total Items and Amount Spent Before Membership
WITH cte AS (
    SELECT 
        s.customer_id, 
        COUNT(s.product_id) AS cnt, 
        SUM(n.price) AS total,
        ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC) AS rn
    FROM 
        sales AS s
    JOIN 
        members AS m
    ON 
        m.customer_id = s.customer_id
    JOIN 
        menu AS n
    ON 
        n.product_id = s.product_id
    WHERE 
        m.join_date > s.order_date
    GROUP BY 
        s.customer_id
)
SELECT 
    customer_id, 
    cnt, 
    total
FROM 
    cte
WHERE 
    rn = 1;

-- 9. Points Calculation for Each Customer
WITH cte AS (
    SELECT 
        s.customer_id,
        SUM(m.price) AS total_amount,
        SUM(
            CASE
                WHEN m.product_name = "sushi" THEN m.price * 10 * 2
                ELSE m.price * 10
            END
        ) AS total_points
    FROM 
        sales AS s
    JOIN 
        menu AS m
    ON 
        s.product_id = m.product_id
    GROUP BY 
        s.customer_id
)
SELECT 
    customer_id, 
    total_amount, 
    total_points
FROM 
    cte;

-- 10. Points Earned in the First Week After Membership
WITH cte AS (
    SELECT 
        s.customer_id,
        s.product_id,
        m.price,
        n.join_date,
        CASE
            WHEN order_date BETWEEN join_date AND DATE_ADD(join_date, INTERVAL 6 DAY) THEN m.price * 10 * 2
            WHEN m.product_name = "sushi" THEN m.price * 10 * 2
            ELSE m.price * 10
        END AS total_points
    FROM 
        sales AS s
    JOIN 
        menu AS m
    ON 
        s.product_id = m.product_id
    JOIN 
        members AS n
    ON 
        n.customer_id = s.customer_id
    WHERE 
        s.order_date <= "2025-01-31"
)
SELECT 
    *
FROM 
    cte;


