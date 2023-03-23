# Target-SQL-Project
Querries 
1) Data type of columns in a table
->  Select Data_Type,column_name,Table_name from target_sql_project.INFORMATION_SCHEMA.COLUMNS

2) Time period for which the data is given
->  with table1 as
   (SELECT MIN(order_purchase_timestamp) AS min_pur_time,
    MAX(order_estimated_delivery_date) AS max_pur_time
    FROM `target_sql_project.orders` )
    select*, datetime_diff((max_pur_time),(min_pur_time),day) as count_of_days
    from table1

3) Cities and States of customers ordered during the given period
-> select customer_state,customer_city, count(customer_id) customer_id_count,count(customer_unique_id)as customer_unique_count
   from `target_sql_project.customers`
   group by 1,2
   order by 1,2

4) Is there a growing trend on e-commerce in Brazil? How can we describe a complete scenario? Can we see some seasonality with peaks at specific months?
->  SELECT *, ROUND(((PRESENT_COUNT - PREVIOUS_COUNT) /PREVIOUS_COUNT ) * 100, 2) AS order_growth_rate FROM  
   (SELECT *, LAG(PRESENT_COUNT) OVER(ORDER BY YEAR, MONTH) AS PREVIOUS_COUNT FROM   
    (SELECT EXTRACT(YEAR FROM order_purchase_timestamp) AS YEAR, EXTRACT(MONTH FROM order_purchase_timestamp) AS MONTH, COUNT(*) AS PRESENT_COUNT  
    FROM `target_sql_project.orders`
    WHERE order_status = 'delivered' 
    GROUP BY 1, 2 ORDER BY 1, 2) AS BASE1 ORDER BY YEAR, MONTH) AS BASE2;
    
5) What time do Brazilian customers tend to buy (Dawn, Morning, Afternoon or Night)?
->  SELECT COUNTIF((TIME(order_purchase_timestamp) >= '05:00:00' AND TIME(order_purchase_timestamp) < '06:00:00')) AS dawn_orders_count, 
    COUNTIF((TIME(order_purchase_timestamp) >= '06:00:00' AND TIME(order_purchase_timestamp) < '12:00:00')) AS morning_orders_count, 
    COUNTIF((TIME(order_purchase_timestamp) >= '12:00:00' AND TIME(order_purchase_timestamp) < '18:00:00')) AS afternoon_orders_count, 
    COUNTIF((TIME(order_purchase_timestamp) >= '18:00:00' AND TIME(order_purchase_timestamp) <= '23:59:59') OR (TIME(order_purchase_timestamp) >= '00:00:00' AND TIME(order_purchase_timestamp) < '05:00:00')) AS night_orders_count 
    FROM `target_sql_project.orders`;
    
6) Get month on month orders by states.
->  SELECT *, ROUND(((orders_count - prev_orders_count) / prev_orders_count) * 100, 2) AS orders_count_growth_rate FROM 
    (SELECT *, LAG(orders_count) OVER(PARTITION BY customer_state, customer_city ORDER BY YEAR, MONTH) AS prev_orders_count FROM  
    (SELECT C.customer_state, C.customer_city, YEAR, MONTH, COUNT(*) AS orders_count FROM `target_sql_project.customers` AS C 
     JOIN 
    (SELECT *, EXTRACT(MONTH FROM order_purchase_timestamp) AS MONTH, EXTRACT(YEAR FROM order_purchase_timestamp) AS YEAR FROM `target_sql_project.orders` 
     WHERE order_status = 'delivered') AS A ON C.customer_id = A.customer_id GROUP BY 1, 2, 3, 4)) AS B;
     
7) Distribution of customers across the states in Brazil.
->  SELECT customer_state, customer_city ,
    COUNT(DISTINCT customer_id) AS count_customer_id,
    COUNT(DISTINCT customer_unique_id) AS count_customer_unique_id
    FROM `target_sql_project.customers`
    GROUP BY  customer_state, customer_city
    ORDER BY  customer_state, customer_city
    
8)  Get % increase in cost of orders from 2017 to 2018 (include months between Jan to Aug only) - You can use “payment_value” column in payments table.
->  WITH TABLE1 AS
    (SELECT ROUND(SUM(price + freight_value), 2) AS total_cost_2017
     FROM (SELECT O.*, OI.* FROM `target_sql_project.orders` AS O JOIN `target_sql_project.order_items` AS OI ON
     O.order_id = OI.order_id
     WHERE O.order_status = 'delivered' AND (EXTRACT(YEAR FROM O.order_purchase_timestamp) = 2017) AND
     EXTRACT(MONTH FROM O.order_purchase_timestamp) BETWEEN 1 AND 8)),
     TABLE2 AS (SELECT ROUND(SUM(price + freight_value),2) AS total_cost_2018
     FROM (SELECT O.*, OI.* FROM `target_sql_project.orders` AS O JOIN `target_sql_project.order_items` AS OI ON O.order_id = OI.order_id
     WHERE O.order_status = 'delivered' AND (EXTRACT(YEAR FROM O.order_purchase_timestamp) = 2018) AND
     EXTRACT(MONTH FROM O.order_purchase_timestamp) BETWEEN 1 AND 8))
     SELECT T1.total_cost_2017, T2.total_cost_2018, ROUND(((T2.total_cost_2018 - T1.total_cost_2017) /
     T1.total_cost_2017) * 100, 2) AS cost_growth_rate
     FROM TABLE1 AS T1 CROSS JOIN TABLE2 AS T2;
     
9)  Mean & Sum of price and freight value by customer state.
->  SELECT C.customer_state, ROUND(AVG(OI.price + OI.freight_value), 2) AS avg_cost, ROUND(SUM(price +
    freight_value), 2) AS sum_cost
    FROM `target_sql_project.customers` AS C JOIN `target_sql_project.orders` AS O ON C.customer_id = O.customer_id
      JOIN
    `target_sql_project.order_items` AS OI ON O.order_id = OI.order_id
     WHERE O.order_status = 'delivered'
     GROUP BY C.customer_state;
     
8)  Calculate days between purchasing, delivering and estimated delivery
->   SELECT order_id, order_purchase_timestamp, order_delivered_customer_date, order_estimated_delivery_date, 
     (TIMESTAMP_DIFF(order_delivered_customer_date,order_purchase_timestamp, DAY)) AS time_to_delivery, 
     (TIMESTAMP_DIFF(order_estimated_delivery_date, order_purchase_timestamp, DAY)) AS diff_estimated_delivery, 
     (TIMESTAMP_DIFF(order_estimated_delivery_date, order_delivered_customer_date, DAY)) AS diff_estdel_actdel 
      FROM `target_sql_project.orders`
      WHERE order_status = 'delivered';
      
9)  Find time_to_delivery & diff_estimated_delivery. Formula for the same given below:
    a)time_to_delivery = order_purchase_timestamp-order_delivered_customer_date
    b)diff_estimated_delivery = order_estimated_delivery_date-order_delivered_customer_date
->  SELECT order_id, order_purchase_timestamp, order_delivered_customer_date, order_estimated_delivery_date, 
    (TIMESTAMP_DIFF(order_delivered_customer_date,order_purchase_timestamp, DAY)) AS time_to_delivery, 
    (TIMESTAMP_DIFF(order_estimated_delivery_date, order_delivered_customer_date, DAY)) AS diff_estimated_delivery 
     FROM `target_sql_project.orders`
     WHERE order_status = 'delivered';
     
10)  Group data by state, take mean of freight_value, time_to_delivery, diff_estimated_delivery.
->   SELECT C.customer_state,  
      AVG(OI.freight_value) AS avg_freight_value,  
      AVG(BASE1.time_to_delivery) AS avg_time_to_delivery,  
      AVG(BASE1.diff_estimated_delivery) AS avg_diff_estimated_delivery 
      FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *,  
      TIMESTAMP_DIFF(order_delivered_customer_date, order_purchase_timestamp, DAY) AS time_to_delivery, 
      TIMESTAMP_DIFF(order_estimated_delivery_date, order_purchase_timestamp, DAY) AS diff_estimated_delivery, 
      FROM `target_sql_project.orders`
      WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      JOIN `target_sql_project.order_items` AS OI ON BASE1.order_id = OI.order_id 
      GROUP BY C.customer_state;
      
11.1)  Top 5 states with highest freight value - sort in desc/asc limit 5.
->   SELECT C.customer_state, ROUND(AVG(OI.freight_value), 2) AS avg_freight_value,  
     FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *,  
      FROM `target_sql_project.orders` 
      WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      JOIN 
      `target_sql_project.order_items` AS OI ON BASE1.order_id = OI.order_id 
      GROUP BY C.customer_state 
      ORDER BY AVG(OI.freight_value) DESC LIMIT 5;
      
11.2)  Top 5 states with lowest average freight value - sort in desc/asc limit 5.
 ->  SELECT C.customer_state, ROUND(AVG(OI.freight_value), 2) AS avg_freight_value,  
      FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *,  
      FROM `target_sql_project.orders` 
      WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      JOIN 
      `target_sql_project.order_items` AS OI ON BASE1.order_id = OI.order_id 
      GROUP BY C.customer_state 
      ORDER BY AVG(OI.freight_value) ASC 
      LIMIT 5;
      
12.1)  Top 5 states with highest average time to delivery
 ->   SELECT C.customer_state,  ROUND(AVG(BASE1.time_to_delivery), 2) AS avg_time_to_delivery,  
      FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *,  TIMESTAMP_DIFF(order_delivered_customer_date, order_purchase_timestamp, DAY) AS time_to_delivery, 
      FROM `target_sql_project.orders` 
      WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      GROUP BY C.customer_state 
      ORDER BY AVG(BASE1.time_to_delivery) DESC LIMIT 5
      
12.2) Top 5 states with lowest average time to delivery
->    SELECT C.customer_state, ROUND(AVG(BASE1.time_to_delivery), 2) AS avg_time_to_delivery,  
      FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *,TIMESTAMP_DIFF(order_delivered_customer_date, order_purchase_timestamp, DAY) AS time_to_delivery 
      FROM `target_sql_project.orders` 
      WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      GROUP BY C.customer_state 
      ORDER BY AVG(BASE1.time_to_delivery) ASC 
      LIMIT 5;
      
13.1) Top 5 states where delivery is really fast compared to estimated date
->    SELECT C.customer_state, ROUND(AVG(BASE1.diff_estdel_actdel), 2) AS avg_daydiff_estdel_actdel 
      FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *, TIMESTAMP_DIFF(order_estimated_delivery_date, order_delivered_customer_date, DAY) AS diff_estdel_actdel 
      FROM `target_sql_project.orders` WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      GROUP BY C.customer_state 
      ORDER BY AVG(BASE1.diff_estdel_actdel) DESC LIMIT 5;
      
13.2) Top 5 states where delivery is really not so fast compared to estimated date
->    SELECT C.customer_state, ROUND(AVG(BASE1.diff_estdel_actdel), 2) AS avg_daydiff_estdel_actdel 
      FROM `target_sql_project.customers` AS C 
      JOIN 
      (SELECT *, TIMESTAMP_DIFF(order_estimated_delivery_date, order_delivered_customer_date, DAY) AS diff_estdel_actdel 
      FROM `target_sql_project.orders` 
      WHERE order_status = 'delivered') AS BASE1 ON C.customer_id = BASE1.customer_id 
      GROUP BY C.customer_state 
      ORDER BY AVG(BASE1.diff_estdel_actdel) ASC LIMIT 5
      
14)  Month over Month count of orders for different payment types
->   SELECT BASE2.*, ROUND(((BASE2.payment_type_count - BASE2.prev_count) / BASE2.prev_count) * 100, 2)
     AS count_growth_rate_percent 
     FROM  
     (SELECT payment_type, YEAR, MONTH, payment_type_count, LAG(payment_type_count) OVER(PARTITION BY payment_type ORDER BY YEAR, MONTH) AS prev_count 
     FROM 
     (SELECT DISTINCT P.payment_type, O.YEAR, O.MONTH, COUNT(*) OVER (PARTITION BY P.payment_type, O.YEAR, O.MONTH ORDER BY O.YEAR, O.MONTH) AS
     payment_type_count 
     FROM `target_sql_project.payments` AS P  
     JOIN 
     (SELECT order_id, EXTRACT(MONTH FROM order_purchase_timestamp) AS MONTH, EXTRACT(YEAR FROM order_purchase_timestamp) AS YEAR, 
     FROM `target_sql_project.orders`
     WHERE order_status = 'delivered') AS O ON P.order_id = O.order_id) AS BASE1) AS BASE2 
 
15)  Count of orders based on the no. of payment installments.
->   SELECT P.payment_installments, COUNT(*) AS orders_count 
     FROM `target_sql_project.payments` AS P 
     JOIN   
    (SELECT * 
     FROM `target_sql_project.orders`
     WHERE order_status = 'delivered') AS O ON P.order_id = O.order_id 
     GROUP BY 1;
 
 


      


    
  
 
 
 
 
 
     
