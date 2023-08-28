# Case-Study-1-Dannys-Diner
--1.What is the total amount each customer spent at the restaurant?
select customer_id,sum(price)
from sales s, menu m
where s.product_id = m.product_id
group by customer_id;
