# Target Sales Data Analysis


## 1	Import the dataset and do usual exploratory analysis steps like checking the structure & characteristics of the dataset
1.1	Data type of all columns in the "customers" table
•	Customer_id: String
•	Customer_unique_id: String
•	Customer_zip_code_prefix: Integer
•	Customer_city: string
•	Customer_state: string

1.2	Get the time range between which the orders were placed:
Query:
select 
max(order_purchase_timestamp)latest_purchase_date,
min(order_purchase_timestamp)earliest_purchase_date,
datetime_diff(max(order_purchase_timestamp),min(order_delivered_carrier_date),hour)/24 order_period_in_days
from `scaler-dsml-sql-405717.target.orders`

Result:
 ![image](https://github.com/user-attachments/assets/ad6afea0-529c-4f37-833d-fde023cdde78)

Observation: 
The latest purchase was made on 2018-10-17 17:30:18 UTC and the earliest purchase was made on 2016-09-04 21:15:19. The time period is 739.25 days.

1.3	Count the Cities & States of customers who ordered during the given period 
Query:
select count(distinct C.customer_city)city_count,count(distinct C.customer_state)state_count from `scaler-dsml-sql-405717.target.orders`O
left join `scaler-dsml-sql-405717.target.customers`C
on O.customer_id=C.customer_id
where C.customer_state is not null and C.customer_city is not null

 ![image](https://github.com/user-attachments/assets/4116de08-3824-49b4-a60a-d0ab021a91e6)


Observation: During this time period, total number of cities are 4119 and total number of states are 27


## 2	IN depth exploration:
2.1	Is there a growing trend in the number of orders placed over the past years.
Query:
create view `scaler-dsml-sql-405717.target.month_wise_orders` as 
(select extract(year from order_purchase_timestamp)year,
format_date("%b",order_purchase_timestamp)month,
count(*)no_of_orders
from `scaler-dsml-sql-405717.target.orders`
group by year,month
order by year,parse_date("%b",month)
);

select *,round(100*((no_of_orders/lag(no_of_orders,1)over (order by year,parse_date("%b",month)))-1),1) change_in_orders_per_month,from 
`scaler-dsml-sql-405717.target.month_wise_orders`
order by year, parse_date("%b",month)

Results:
 ![image](https://github.com/user-attachments/assets/5365bd58-368a-425a-9200-80236cdc3615)


Observation: There is a growing trend in the number of orders till Jan 2018 and then the number of orders have settled in the range of 6000 to 7300 range and then there is a fall in the number of orders from sep 2018 onwards 

2.2	Can we see some kind of monthly seasonality in terms of the no. of orders being placed?
select month,year,round(100*((no_of_orders/lag(no_of_orders,1)over (order by year,parse_date("%b",month)))-1),1) percentage_change_in_orders_per_month,from 
`scaler-dsml-sql-405717.target.month_wise_orders`
order by parse_date("%b",month),year

Results:
![image](https://github.com/user-attachments/assets/9150b932-0ea9-47e8-a6f0-c253e1b8cafc)

 

Observation: There is no such obvious seasonality in the number of orders. However, it isobserved that there is a drop in sales in the month of april and june by around 10% and increase in the month of august by around 3 to 7%.

2.3	During what time of the day, do the Brazilian customers mostly place their orders?
Query: 
with base as (
select *,
case when hour between 0 and 6 
then "dawn"
when hour between 7 and 12 
then "mornings"
when hour between 13 and 18
then "afternoon"
when hour between 19 and 23
then "night"
end as time_
 from 
(
select extract(hour from order_purchase_timestamp)hour,
order_id from `scaler-dsml-sql-405717.target.orders`)a
)

select time_,count(*)no_of_orders from base
group by time_
order by 2 desc

Results: 
 ![image](https://github.com/user-attachments/assets/d2d76f1f-8b4b-4152-b070-db19cf617a30)


Observation: maximum of the orders are placed in afternoon with 38135 orders. Then night, morning and dawn with 28331, 27733 and 5242 respectively. 

## 3	Evolution of E-commerce orders in the Brazil region
3.1	Get the month on month no. of orders placed in each state?
Query: 
with base as(
select C.customer_state as state,
extract(year from O.order_purchase_timestamp)year,
extract(month from O.order_purchase_timestamp)month,
order_id
 from `scaler-dsml-sql-405717.target.orders`O
 left join `scaler-dsml-sql-405717.target.customers`C
 on C.customer_id=O.customer_id)

 select state,year,month,count(order_id)no_of_orders
 from base
 group by state,year,month
 order by state,year,month

Results:
![image](https://github.com/user-attachments/assets/7d5667f9-4b93-4c47-888b-2d07b241134f)

 
Observation : The month on month orders for each state is as shown

3.2	 How are the customers distributed across all the states?
Query: 
select count(customer_id)no_customer,customer_state state from `scaler-dsml-sql-405717.target.customers`
group by state
order by no_customer desc

results: 
![image](https://github.com/user-attachments/assets/4c625f1c-f7e4-4b87-a3d7-6ab0cc0c8af3)

 
Observations: maximum number of customer are from state SP with 41746 and minimum number of customers are from state RR with 46 customers. 

## 4	Impact on Economy: Analyze the money movement by e-commerce by looking at order prices, freight and others.
4.1	Get the % increase in the cost of orders from year 2017 to 2018 (include months between Jan to Aug only).
Query: 
with base as (
select O.order_id,O.order_purchase_timestamp, P.payment_value from `scaler-dsml-sql-405717.target.orders`O
left join `scaler-dsml-sql-405717.target.payments`P on
O.order_id=P.order_id
where date(O.order_purchase_timestamp) between "2017-01-01" and "2017-08-01"
or date(O.order_purchase_timestamp) between "2018-01-01" and "2018-08-01"),
year_cost as(
select year,sum(payment_value)total_payment from(
select extract(year from order_purchase_timestamp)year,payment_value
from base)a
group by a.year
order by year)

select round(100*((lead(total_payment)over (order by year)/total_payment)-1),2) percentage_increase from year_cost
limit 1

Results:
![image](https://github.com/user-attachments/assets/fbb87609-af60-4e45-8456-984990ea7a93)

 
Observation: There is an increase of 155.66% in cost from 2017 to 2018 including only Jan to aug months. 

4.2	Calculate the Total & Average value of order price for each state.
Query: 
with state_wise as(
select O.order_id,C.customer_state,OI.* from `scaler-dsml-sql-405717.target.orders` O
left join `scaler-dsml-sql-405717.target.customers` C
using (customer_id)
left join `scaler-dsml-sql-405717.target.order_items` OI
using (order_id))

select customer_state,round(sum(price),2)total_price,round(avg(price),2)avg_price
from state_wise
group by 1
order by 2 desc
--order by 3 desc (order by avg price)


Results: 
 ![image](https://github.com/user-attachments/assets/ce6c61da-81ca-4461-8851-4d7b006998cd)


Observation: 
The state with highest total price is SP with 5202955.05.
The state with highest avg price is PB with 191.48.

4.3	Calculate the Total & Average value of order freight for each state.
Query:
with state_wise as(
select O.order_id,C.customer_state,OI.* from `scaler-dsml-sql-405717.target.orders` O
left join `scaler-dsml-sql-405717.target.customers` C
using (customer_id)
left join `scaler-dsml-sql-405717.target.order_items` OI
using (order_id))

select customer_state,round(sum(freight_value),2)total_freight_value,round(avg(freight_value),2)avg_freight_value
from state_wise
group by 1
order by 2 desc
-- order by 3 desc

Results: 
![image](https://github.com/user-attachments/assets/4960217b-03d5-4093-922a-bdb9683d39b2)

 
Observations: 
The state with highest total freight value is SP with 718723.07.
The state with highest avg freight value is RR with 42.9 

## 5	Analysis based on sales, freight and delivery time
5.1	Find the no. of days taken to deliver each order from the order’s purchase date as delivery time. Also, calculate the difference (in days) between the estimated & actual delivery date of an order. 
Query:
select order_id,date(order_purchase_timestamp)order_date,date(order_delivered_customer_date)delivery_date,
date(order_estimated_delivery_date)est_date,date_diff(order_delivered_customer_date ,order_purchase_timestamp,day) as time_to_deliver,
date_diff(order_estimated_delivery_date,order_delivered_customer_date,day ) as diff_est
from `scaler-dsml-sql-405717.target.orders`
where order_delivered_customer_date is not null
limit 100

Results: 
 ![image](https://github.com/user-attachments/assets/26eba0b8-3bc8-47b2-906f-7e11bc309bca)


Observations: The days taken to deliver and the difference in estimated days and delivered date is shown above. A positive day in diff_est indicates that the order was delivered before the estimated date.

5.2	Find out the top 5 states with the highest & lowest average freight value.
Query:
with state_wise as(
select O.order_id,C.customer_state,OI.* from `scaler-dsml-sql-405717.target.orders` O
left join `scaler-dsml-sql-405717.target.customers` C
using (customer_id)
left join `scaler-dsml-sql-405717.target.order_items` OI
using (order_id))

select customer_state,round(sum(freight_value),2)total_freight_value,round(avg(freight_value),2)avg_freight_value
from state_wise
group by 1
order by 3 desc
limit 5

results:
![image](https://github.com/user-attachments/assets/5928df0b-6d43-4fc2-aa01-e52079b7124b)

 
Observation: The top 5 state with highest freight value are RR,PB,RO,AC and PI

For top 5 lowest:
Query:
with state_wise as(
select O.order_id,C.customer_state,OI.* from `scaler-dsml-sql-405717.target.orders` O
left join `scaler-dsml-sql-405717.target.customers` C
using (customer_id)
left join `scaler-dsml-sql-405717.target.order_items` OI
using (order_id))

select customer_state,round(sum(freight_value),2)total_freight_value,round(avg(freight_value),2)avg_freight_value
from state_wise
group by 1
order by 3 
limit 5

Results:
![image](https://github.com/user-attachments/assets/d5cb5a3e-17c4-4464-b969-f591e71f18dd)

 
Observation:
The top 5 state with lowest freight value are SP,PR,MG,RJ and DF.

5.3	Find out the top 5 states with the highest & lowest average delivery time.
Query:
with deliver_days as(
select order_id,date(order_purchase_timestamp)order_date,date(order_delivered_customer_date)delivery_date,
date(order_estimated_delivery_date)est_date,date_diff(order_delivered_customer_date ,order_purchase_timestamp,day) as time_to_deliver,
date_diff(order_estimated_delivery_date,order_delivered_customer_date,day ) as diff_est,
customer_state as state
from `scaler-dsml-sql-405717.target.orders`
left join `scaler-dsml-sql-405717.target.customers`
using (customer_id)
where order_delivered_customer_date is not null
)

select * from(
select *,rank() over (order by avg_delivery_time desc) rnk from (
select state,round(avg(time_to_deliver))avg_delivery_time from deliver_days
group by state
order by avg_delivery_time )a)b
where rnk<6
order by rnk

Results:
![image](https://github.com/user-attachments/assets/bb20548c-81d3-44dd-951f-7f0d4e8afd59)

 
Observation:
The top 5 states with highest average delivery time are RR, AP, AM, AL and PA.

Top 5 lowest:
Query:
with deliver_days as(
select order_id,date(order_purchase_timestamp)order_date,date(order_delivered_customer_date)delivery_date,
date(order_estimated_delivery_date)est_date,date_diff(order_delivered_customer_date ,order_purchase_timestamp,day) as time_to_deliver,
date_diff(order_estimated_delivery_date,order_delivered_customer_date,day ) as diff_est,
customer_state as state
from `scaler-dsml-sql-405717.target.orders`
left join `scaler-dsml-sql-405717.target.customers`
using (customer_id)
where order_delivered_customer_date is not null
)

select * from(
select *,rank() over (order by avg_delivery_time ) rnk from (
select state,round(avg(time_to_deliver))avg_delivery_time from deliver_days
group by state
order by avg_delivery_time )a)b
where rnk<6
order by rnk

Results: 
![image](https://github.com/user-attachments/assets/7f3f1e3b-70d1-4da2-884c-202a2face7ff)

 
Observation:
The top 5 states with lowest average delivery time are SP, MG, PR, DF and SC.

5.4	Find out the top 5 states where the order delivery is really fast as compared to the estimated date of delivery.

Query: 
with deliver_days as(
select order_id,date(order_purchase_timestamp)order_date,date(order_delivered_customer_date)delivery_date,
date(order_estimated_delivery_date)est_date,
date(order_estimated_delivery_date)est_date,
date_diff(order_delivered_customer_date ,order_purchase_timestamp,day) as act_days_to_deliver,
date_diff(order_estimated_delivery_date,order_purchase_timestamp,day ) as est_days_to_deliver,
customer_state as state
from `scaler-dsml-sql-405717.target.orders`
left join `scaler-dsml-sql-405717.target.customers`
using (customer_id)
where order_delivered_customer_date is not null
),

avg_days as(
select state ,
round(avg(act_days_to_deliver))avg_act_days_to_deliver,
round(avg(est_days_to_deliver))avg_est_days_to_deliver
from deliver_days
group by state)

select state from(
select *,rank()over (order by diff_in_avg desc)rnk from(
select *, (avg_est_days_to_deliver-avg_act_days_to_deliver)diff_in_avg
from avg_days)a)b
where rnk<6
order by rnk

Results:
 ![image](https://github.com/user-attachments/assets/90db137e-192d-4e79-a8ba-4738763021dd)



Observation: 
The top 5 states with fastest delivery are as shown. These observation are made based on the difference in average delivery date and average estimated date for each state. 
 
## 6	Analysis based on the payments:
6.1	Find the month on month no. of orders placed using different payment types.
Query: 
with base as (
select O.order_id,extract(year from O.order_purchase_timestamp)year, 
format_date("%b", O.order_purchase_timestamp)month,
P.payment_type from
`scaler-dsml-sql-405717.target.orders`O
left join `scaler-dsml-sql-405717.target.payments`P
using(order_id)
where payment_type is not null
)

select payment_type,year,month,count(order_id)no_of_orders
from base
group by 1,2,3
order by 1,2,parse_date("%b",month)

Results:
 ![image](https://github.com/user-attachments/assets/e6ea6f3f-6a6f-4f57-96c1-dc9d846e8095)


Observation: It is observed that the orders done by UPI, Credit card and Debit card increases with time, whereas the number of orders for vouchers increase till jan 2018 and then decrease gradually.
The most number of orders are done using credit card for the month of  2017 Nov.

6.2	Find the no. of orders placed on the basis of the payment installments that have been paid.
Query: 
select payment_installments,count(order_id) no_of_orders from 
`scaler-dsml-sql-405717.target.orders`O
left join `scaler-dsml-sql-405717.target.payments` P
using(order_id)
where payment_installments is not null
group by payment_installments
order by 2 desc

Results:
![image](https://github.com/user-attachments/assets/410bd3d3-bb0e-424f-b279-1d0e12940dcb)

 
select review_score, customer_state, count(*) from `scaler-dsml-sql-405717.target.delivery_complaints`
group by 1,2
order by 1,3 desc,2

 

Recommendation:
Delivery complaints:
Query: 
create view `scaler-dsml-sql-405717.target.delivery_complaints` as(
select O.order_id,O.customer_id,rev.review_score,rev.review_comment_title,O.order_delivered_customer_date,
O.order_estimated_delivery_date,C.customer_state
from `scaler-dsml-sql-405717.target.orders`O
left join
`scaler-dsml-sql-405717.target.order_reviews` rev
using(order_id)
left join
`scaler-dsml-sql-405717.target.customers`C
using(customer_id)
where rev.review_score<3
and rev.review_comment_title like "%deliver%");

select review_score,customer_state,count(*) from `scaler-dsml-sql-405717.target.delivery_complaints`
group by 1,2
order by 1,3 desc,2
 Result: 
 
Observation:  It is observed that the delivery is not on time in the state SP as per the reviews.
Query:
select customer_state,seller_state ,count(review_score)no_of_comments from(
select order_id,review_score, order_delivered_customer_date,order_estimated_delivery_date,
customer_state,S.seller_id,S.seller_state,review_comment_title
from scaler-dsml-sql-405717.target.delivery_complaints
left join scaler-dsml-sql-405717.target.order_items OI
using(order_id)
left join scaler-dsml-sql-405717.target.sellers S
using(seller_id)
where customer_state="SP" and review_score<=2)
group by 1,2
order by 3 desc,1,2

Result:  
![image](https://github.com/user-attachments/assets/61b0c033-9796-4b21-a6fb-cd194b3ffedd)


Observation: Most of the delivery issues are coming from orders where both seller and customer state is SP. This is happening because highest orders are from state SP. It would be recommended to prioritise in state delivery time in state SP or increase the delivery agents in the state SP.


# Recommendations
Delivery Improvement: Prioritize deliveries within the SP state to reduce complaints related to delivery delays.
Increase Delivery Agents: Particularly in SP to handle the high order volume.









