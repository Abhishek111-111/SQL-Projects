#  Pizza Runner :pizza:

### Introduction

Danny was scrolling through his Instagram feed when something really caught his eye - “80s Retro Styling and Pizza Is The Future!”

Danny was sold on the idea, but he knew that pizza alone was not going to help him get seed funding to expand his new Pizza Empire - so he had one more genius idea to combine with it - he was going to Uberize it - and so Pizza Runner was launched!

Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers.

****
#### 1. How many of each type of pizza was delivered?
```sql
SELECT p.pizza_name
	,count(c.order_id)
FROM customer_orders c
LEFT JOIN pizza_names p ON c.pizza_id = p.pizza_id
LEFT JOIN runner_orders r ON c.order_id = r.order_id
WHERE pickup_time != 'null'
	AND distance != 'null'
	AND duration != 'null'
GROUP BY pizza_name;
```
![image](https://user-images.githubusercontent.com/45015990/201479786-84593f73-6e43-4d4c-ae24-26ca2178f182.png)

#### 2. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT c.customer_id
	,pizza_name
	,count(c.pizza_id) AS num_of_pizzas_ordered
FROM customer_orders c
LEFT JOIN pizza_names p ON c.pizza_id = p.pizza_id
GROUP BY c.customer_id
	,c.pizza_id
ORDER BY 1;
```
![image](https://user-images.githubusercontent.com/45015990/201480542-4856b03d-18f9-436a-b1e4-5ded6a8e35d8.png)

#### 3. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
SELECT customer_id
	,(
		CASE 
			WHEN exclusions
				OR extras IN (
					1
					,12
					)
				THEN 'change'
			ELSE 'no_changes'
			END
		) AS changes
	,count(c.order_id) AS num_of_pizzas
FROM customer_orders c
INNER JOIN runner_orders r ON c.order_id = r.order_id
WHERE r.pickup_time != 'null'
	AND r.distance != 'null'
	AND r.duration != 'null'
GROUP BY 1
	,2
ORDER BY 1;
```

![image](https://user-images.githubusercontent.com/45015990/201480626-5cfa94e4-a0be-4511-afe7-3ae928a0f022.png)

#### 4. What was the total volume of pizzas ordered for each hour of the day?
```sql
WITH times
AS (
	SELECT *
		,day(order_time) AS days
		,hour(order_time) AS `hour`
	FROM customer_orders
	)
SELECT date(order_time) as `date`
	,`hour` as `time`
	,count(order_id) AS num_of_orders
FROM times
GROUP BY days
	,`hour`
ORDER BY 1;
```
![image](https://user-images.githubusercontent.com/45015990/201480697-855af4ae-0759-4efa-8871-f2be8ec8eef3.png)

#### 5. What was the volume of orders for each day of the week?
```sql
WITH `week`
AS (
	SELECT *
		,weekday(order_time) AS week_days
	FROM customer_orders
	)
SELECT date(order_time) as `date`
	,`week_days` 
	,count(order_id) AS num_of_orders
FROM `week`
GROUP BY 
	`week_days`
ORDER BY 2;
```
![image](https://user-images.githubusercontent.com/45015990/201480758-aa95113a-4f22-418e-af90-f35482bd0b17.png)

#### 6. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
WITH cte
AS (
	SELECT c.order_id
		,c.order_time
		,r.runner_id
		,r.pickup_time
		,minute(timediff(pickup_time, order_time)) AS a
	FROM customer_orders c
	INNER JOIN runner_orders r ON c.order_id = r.order_id
	)
SELECT runner_id
	,CONCAT (
		round(avg(a))
		,' Min '
		) AS avg_time
FROM cte
GROUP BY 1;
```

![image](https://user-images.githubusercontent.com/45015990/201480831-5afe190f-8d27-4ee2-a1c3-2b524a65d1bc.png)

#### 7. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
SELECT 
count(c.pizza_id) as num_of_pizzas
		,minute(timediff(pickup_time, order_time)) AS minutes_taken
	FROM customer_orders c
	INNER JOIN runner_orders r ON c.order_id = r.order_id
    where pickup_time != 'null'
    group by c.order_id
    order by 1 desc;
```

![image](https://user-images.githubusercontent.com/45015990/201480979-3289e16c-564f-422b-920c-39024838830e.png)

#### 8. What was the average distance travelled for each customer?
```sql
SELECT c.customer_id
	,CONCAT (
		round(avg(r.distance), 2)
		,' Km '
		) AS avg_distance
FROM runner_orders r
INNER JOIN customer_orders c ON c.order_id = r.order_id
WHERE r.pickup_time != 'null'
	AND r.distance != 'null'
	AND r.duration != 'null'
GROUP BY c.customer_id
ORDER BY 1;
```
![image](https://user-images.githubusercontent.com/45015990/201481053-25c294f9-030b-4a6d-8cab-8e1d28b9b9e1.png)


#### 9. What is the successful delivery percentage for each runner?
```sql
WITH cte1
AS (
	SELECT *
		,count(order_id) AS a
	FROM runner_orders
	GROUP BY runner_id
	)
	,cte2
AS (
	SELECT *
		,count(order_id) AS b
	FROM runner_orders
	WHERE pickup_time != 'null'
		AND distance != 'null'
		AND duration != 'null'
	GROUP BY runner_id
	)
SELECT cte2.runner_id
	,round(100 * (b / a), 1) AS sucess_delivery_percent
FROM cte1
	,cte2
GROUP BY 1
ORDER BY 1;
```
![image](https://user-images.githubusercontent.com/45015990/201481142-55b715f6-61d7-4d44-965f-b6615b4cde9b.png)

#### 10. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
WITH cte
AS (
	SELECT c.order_id
		,p.pizza_name
		,count(c.order_id) AS total_pizzas
	FROM customer_orders c
	LEFT JOIN pizza_names p ON c.pizza_id = p.pizza_id
	INNER JOIN runner_orders r ON c.order_id = r.order_id
	WHERE r.pickup_time != 'null'
		AND r.distance != 'null'
		AND r.duration != 'null'
	GROUP BY 2
	)
SELECT sum(CASE 
			WHEN pizza_name = 'Meatlovers'
				THEN (total_pizzas * 12)
			ELSE (
					CASE 
						WHEN pizza_name = 'Vegetarian'
							THEN (total_pizzas * 10)
						ELSE 0
						END
					)
			END) AS total_revenue
FROM cte;
```

![image](https://user-images.githubusercontent.com/45015990/201481233-a707e64d-bac4-4bec-a724-abccddbddfd4.png)



