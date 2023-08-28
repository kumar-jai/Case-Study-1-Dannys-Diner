# Case-Study-1-Dannys-Diner
--1. What is the total amount each customer spent at the restaurant?

select customer_id,sum(price)
from sales s, menu m
where s.product_id = m.product_id
group by customer_id;

--2. How many days has each customer visited the restaurant?

select customer_id,count(distinct order_date)
from sales
group by customer_id;

--3. What was the first item from the menu purchased by each customer?

select customer_id, product_name
from
(select s.customer_id,product_name,order_date,
dense_rank() over(partition by s.customer_id order by order_date) as r
from sales s, menu m
where s.product_id = m.product_id)
where r=1
group by customer_id,product_name;

--4. What is the most purchased item on the menu and how many times was it purchased by all customers?

select product_name,count(s.product_id)
from sales s, menu m
where s.product_id = m.product_id
group by product_name
order by count(product_name) desc
limit 1;

--5. Which item was the most popular for each customer?

select customer_id,product_name,frequency
from
(select customer_id,product_name,count(m.product_id) as frequency,
dense_rank() over(partition by customer_id order by count(m.product_id) desc) as r
from menu m, sales s
where s.product_id = m.product_id
group by customer_id,product_name)
where r=1;

--6. Which item was purchased first by the customer after they became a member?

with cte as
(select m.customer_id,order_date,product_id,
row_number() over(partition by m.customer_id order by order_date) as r
from sales s, members m
where s.customer_id = m.customer_id
and order_date > join_date)

select customer_id,product_name
from cte , menu m
where cte.product_id = m.product_id
and r=1
order by customer_id;

--7. Which item was purchased just before the customer became a member?

with cte as
(select m.customer_id,order_date,product_id,
row_number() over(partition by m.customer_id order by order_date desc) as r
from sales s,members m
where s.customer_id = m.customer_id
and order_date < join_date)

select customer_id,product_name
from cte ,menu m
where cte.product_id = m.product_id
and r=1
order by customer_id;

--8. What is the total items and amount spent for each member before they became a member?

select s.customer_id,count(s.product_id),sum(me.price)
from sales s join members m
on s.customer_id = m.customer_id
and order_date < join_date
join menu me
on s.product_id = me.product_id
group by s.customer_id
order by s.customer_id;

--9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?

with cte as(
select product_id,
case
when product_id =1 then price*20
else price*10
end as points
from menu)
select customer_id,sum(points)
from sales s, cte
where s.product_id = cte.product_id
group by customer_id;

--10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

WITH cust_points 
AS(
select s.customer_id,s.order_date,mm.join_date,DATE(mm.join_date, '+6 day') AS end_promo,s.product_id,m.price,
case 
when s.product_id = 1
then m.price * 20 
when s.product_id != 1 AND 
(s.order_date BETWEEN mm.join_date AND DATE(mm.join_date,'+6 day'))
THEN (m.price * 20)
ELSE m.price * 10
END AS points
FROM sales s
join members mm
on s.customer_id = mm.customer_id
join menu m 
on s.product_id = m.product_id
where s.order_date <= '2021-01-31')

SELECT customer_id,sum(points) AS total
FROM cust_points
GROUP BY customer_id;

--Bonus Question - Join All The Things

select s.customer_id,order_date,product_name,price,
case
when join_date > order_date then "N"
when join_date <= order_date then "Y"
else "N" end as member
from sales s left join members m
on s.customer_id = m.customer_id
join menu me
where s.product_id = me.product_id;

--Bonus Question - Rank All the Things

WITH customers_data AS (
SELECT s.customer_id,s.order_date, m.product_name,m.price,
CASE
when mm.join_date > s.order_date THEN 'N'
when mm.join_date <= s.order_date THEN 'Y'
else 'N' END AS member_status
from sales s inner join members mm
on s.customer_id = mm.customer_id inner join menu m
on s.product_id = m.product_id
order BY mm.customer_id, s.order_date
)
select *, 
case
when member_status = 'N' then NULL
ELSE rank() OVER(PARTITION BY customer_id, member_status order by order_date) END AS ranking
FROM customers_data;
