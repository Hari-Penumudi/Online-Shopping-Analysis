# Online-Shopping-Analysis

## Project Overview

Target is a globally renowned brand and a prominent retailer in the United States. Target makes itself a preferred shopping destination by offering outstanding value, inspiration, innovation and an exceptional guest experience that no other retailer can deliver.

This particular business case focuses on the operations of Target in Brazil and provides insightful information about 100,000 orders placed between 2016 and 2018. The dataset offers a comprehensive view of various dimensions including the order status, price, payment and freight performance, customer location, product attributes, and customer reviews.

By analyzing this extensive dataset, it becomes possible to gain valuable insights into Target's operations in Brazil. The information can shed light on various aspects of the business, such as order processing, pricing strategies, payment and shipping efficiency, customer demographics, product characteristics, and customer satisfaction levels.

## Tools

- Excel - Data Cleaning
- SQL - Data Analysis

## Data Cleaning/Preparation
In this intial phase includes following steps:
1. Data loading and inspection
2. Handling missing values
3. Data cleaning and formatting

## Exploratory Data Analysis
In this exploration phase the dataset has been analysed for the following:
1. Data type of all columns in the "customers" table.
 ```sql
 SELECT column_name, data_type
 FROM `sql-portfolio-399905.TARGET.INFORMATION_SCHEMA.COLUMNS`
 WHERE table_name = 'Customers';
 ```
2. Get the time range between which the orders were placed.
 ```sql
SELECT
MIN(order_purchase_timestamp) AS min_order_timestamp,
MAX(order_purchase_timestamp) AS max_order_timestamp
FROM `TARGET.Orders`
```
3. Count the Cities & States of customers who ordered during the given period.
 ```sql
SELECT
COUNT(Distinct customer_city) AS city_count,
COUNT(DISTINCT customer_state) AS state_count
FROM `TARGET.Customers`
```

## Data Analysis
The data analysis phase is divided into three stages:
### Stage-1: In-depth Exploration
1. Is there a growing trend in the no. of orders placed over the past years?
```sql
SELECT EXTRACT(YEAR FROM order_purchase_timestamp) AS order_year, COUNT(*) AS order_count
FROM`TARGET.Orders`
GROUP BY order_year
ORDER BY order_year;
```
2. Can we see some kind of monthly seasonality in terms of the no. of orders being placed?
```sql
SELECT EXTRACT(MONTH FROM order_purchase_timestamp) AS month, COUNT(*) AS order_count
FROM `TARGET.Orders`
GROUP BY month
ORDER BY month;
```
3. During what time of the day, do the Brazilian customers mostly place their orders? (Dawn, Morning, Afternoon or Night)
0-6 hrs : Dawn
7-12 hrs : Mornings
13-18 hrs : Afternoon
19-23 hrs : Night
```sql
SELECT
  CASE
    WHEN EXTRACT(HOUR FROM order_purchase_timestamp) >= 0 AND EXTRACT(HOUR FROM order_purchase_timestamp) < 6 THEN 'Dawn'
    WHEN EXTRACT(HOUR FROM order_purchase_timestamp) >= 6 AND EXTRACT(HOUR FROM order_purchase_timestamp) < 12 THEN 'Mornings'
    WHEN EXTRACT(HOUR FROM order_purchase_timestamp) >= 12 AND EXTRACT(HOUR FROM order_purchase_timestamp) < 18 THEN 'Afternoon'
    ELSE 'Night' -- Assumes all other times are in the evening
  END AS time_of_day,
  COUNT(*) AS order_count
FROM
  `TARGET.Orders`
GROUP BY time_of_day
ORDER BY time_of_day;
```
### Stage-2: Evolution of E-commerce orders in the Brazil region
1.Get the month on month no. of orders placed in each state?
```sql
SELECT EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month, c.customer_state, COUNT(*) AS num_orders
FROM `TARGET.Customers` as c
JOIN `TARGET.Orders` as o ON o.customer_id = c.customer_id
GROUP By month, c.customer_state
ORDER BY month, c.customer_state;
```
2.How are the customers distributed across all the states?
```sql
select customer_state, count(distinct customer_id) as No_of_customers
from `TARGET.Customers`
group by customer_state
order by No_of_customers desc;
```
### Stage-3:Impact on Economy: Analyze the money movement by e-commerce by looking at order prices, freight and others.

1.Get the % increase in the cost of orders from year 2017 to 2018 (include months between Jan to Aug only).
```sql
SELECT
  EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month,
  round(
  (
    (
      SUM(CASE WHEN EXTRACT(YEAR FROM o.order_purchase_timestamp) = 2018 AND
      EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8 THEN
      p.payment_value END)
      -
      SUM(CASE WHEN EXTRACT(YEAR FROM o.order_purchase_timestamp) = 2017 AND
      EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8 THEN
      p.payment_value END)
    )
    /
    SUM(CASE WHEN EXTRACT(YEAR FROM o.order_purchase_timestamp) = 2017 AND
    EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8 THEN
    p.payment_value END)
  )*100,2) AS percent_increase
FROM
  `TARGET.Orders` o
JOIN
  `TARGET.Payments` p ON o.order_id = p.order_id
WHERE
  EXTRACT(YEAR FROM o.order_purchase_timestamp) IN (2017, 2018) AND
  EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8
GROUP BY month
ORDER BY month;
```

2. Calculate the Total & Average value of order price for each state.
```sql
select c.customer_state, 
      round(Sum(i.price + i.freight_value),2) as Total_Price, 
       Round(Avg(i.price + i.freight_value),2) as Avg_Price
from `TARGET.Orders` as o
join `TARGET.Order items` as i on o.order_id = i.order_id
join `TARGET.Customers` as c on c.customer_id = o.customer_id
group by c.customer_state
order by c.customer_state;
```

### Stage-4: Analysis based on sales, freight and delivery time.

1. Find the no. of days taken to deliver each order from the order’s purchase date as delivery time. Also, calculate the difference (in days) between the estimated & actual delivery date of an order. Do this in a single query.

```sql
select order_id, date_diff(order_delivered_customer_date,order_purchase_timestamp, day) as Time_taken_to_deliver,
                 date_diff(order_estimated_delivery_date,order_delivered_customer_date, day) as Diff_estimated_delivery
from `TARGET.Orders`
where DATE_DIFF(order_delivered_customer_date, order_purchase_timestamp, DAY) IS NOT NULL
order by Time_taken_to_deliver desc;
```

2. Calculating Mean Freight Value, Time to Delivery, and Difference in Estimated Delivery
```sql
SELECT
  c.customer_state,
  ROUND(AVG(i.freight_value), 2) AS mean_freight_value,
  ROUND(AVG(DATE_DIFF(o.order_delivered_customer_date, o.order_purchase_timestamp, DAY)), 2) AS time_to_delivery,
  ROUND(AVG(DATE_DIFF(o.order_estimated_delivery_date, o.order_delivered_customer_date, DAY)), 2) AS diff_estimated_delivery
FROM `TARGET.Orders` o
JOIN `TARGET.Order items` i ON o.order_id = i.order_id
JOIN `TARGET.Customers` c ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY mean_freight_value;
```
### Stage-5: Analysis based on the payments

1. Find the month on month no.of orders placed using different payment types.
```sql
SELECT p.payment_type, EXTRACT(YEAR FROM o.order_purchase_timestamp) AS YEAR,
       EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month,
       COUNT(DISTINCT o.order_id) AS order_count
FROM `TARGET.Orders` o
JOIN `TARGET.Payments` p ON o.order_id = p.order_id
GROUP BY YEAR, MONTH, p.payment_type
ORDER BY p.payment_type,YEAR, MONTH;
```
2. Find the no.of orders placed on the basis of the payment installments that have been paid.
```sql
select payment_installments,count(distinct order_id) as No_of_orders
from `TARGET.Payments`
group by payment_installments
order by payment_installments;
```



