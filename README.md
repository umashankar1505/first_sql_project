# first_sql_project(unguided)
# Case Study: Customer Insights for Danny's Restaurant

Danny wants to gain deeper insights into his customers' visiting patterns, spending habits, and menu preferences to provide a more personalized experience.
These insights will also help him decide whether to expand the existing customer loyalty program.

## Objectives
- Understand customer behavior and preferences.
- Analyze total revenue, transaction trends, and favorite menu items.
- Differentiate spending patterns between loyalty members and non-members.
- Prepare datasets for the team to inspect without needing SQL expertise.

## Queries to work on 

- What is the total amount each customer spent at the restaurant?
  select customer_id,sum(price) as total
from sales as s
join menu as m
on s.product_id=m.product_id
group by customer_id;
  
-How many days has each customer visited the restaurant?
 select customer_id,count(order_date) as days_visited
from sales
group by customer_id;

-What was the first item from the menu purchased by each customer?
 select customer_id, product_name
from(
     select customer_id, order_date,product_name,row_number() over (partition by customer_id order by order_date asc) as ron
     from sales as s
	 join menu as m
     on s.product_id=m.product_id) as first_prod
     where ron=1;
     
-What is the most purchased item on the menu and how many times was it purchased by all customers? 
 select product_id,count(product_id) as purchasecount
from sales as s
group by product_id
order by purchasecount desc;

-Which item was the most popular for each customer?
 with itempopularity as(
select customer_id,count(product_id) as cnt,
rank() over (partition by customer_id order by count(product_id) desc) as rn
from sales
group by customer_id
)
select customer_id,cnt
from itempopularity
where rn=1;

-Which item was purchased first by the customer after they became a member?
 SELECT 
    m.customer_ID, s.product_ID, n.product_name
FROM
    sales AS s
        JOIN
    members AS m ON m.customer_id = s.customer_id
        JOIN
    menu AS n ON n.product_id = s.product_id
WHERE
    m.join_date = s.order_date;
    
-Which item was purchased just before the customer became a member?
 with cte as(
select m.customer_ID,s.product_ID,n.product_name,
row_number() over(partition by customer_id order by product_id desc) as rn
from sales as s
join members as m
on m.customer_id=s.customer_id
join menu as n
on n.product_id=s.product_id
where m.join_date<s.order_date
)
select customer_id,product_id,product_name
from cte
where rn=1;

-What is the total items and amount spent for each member before they became a member?
 with cte as(
select s.customer_id,count(s.product_id) as cnt,sum(n.price) as total,                         
row_number() over(partition by s.customer_id order by count(s.product_id) desc) as rn
from sales as s
join members as m
on m.customer_id=s.customer_id
join menu as n
on n.product_id=s.product_id
where m.join_date<s.order_date
 GROUP BY s.customer_id
)
select customer_id,cnt,total
from cte
where rn=1;

-If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
 with cte as (
     select s.customer_id,
     sum(m.price) as total_amount,
     sum(
         case
	     when m.product_name="sushi" then m.price*10*2
	     else m.price*10
	     end
     ) as total_points
    from sales as s
    join menu as m
    on s.product_id=m.product_id
    group by s.customer_id
    )
select customer_id,total_amount,total_points
from cte;

- In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
  with cte as (
     select s.customer_id,
     s.product_id,
     m.price,
     n.join_date,
	 case
         when order_date between join_date and date_add(join_date,interval 6 day) then m.price*10*2
	     when m.product_name="sushi" then m.price*10*2
	     else m.price*10
	     end as total_points
    from sales as s
    join menu as m
    on s.product_id=m.product_id
    join members as n
    on n.customer_id=s.customer_id
    where s.order_date<="2025-01-31"
    )

select *
from cte;
