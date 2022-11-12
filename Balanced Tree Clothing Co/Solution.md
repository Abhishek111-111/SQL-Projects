# Balanced Tree Clothing Co. :mountain_snow:

## Introduction

Balanced Tree Clothing Company prides themselves on providing an optimised range of clothing and lifestyle wear for the modern adventurer!

Danny, the CEO of this trendy fashion company has asked you to assist the team’s merchandising teams analyse their sales performance and generate a basic financial report to share with the wider business.

Full description: [Case Study #7 - Balanced Tree Clothing Co.](https://8weeksqlchallenge.com/case-study-7/)


*********

#### 1. What are the 25th, 50th and 75th percentile values for the revenue per transaction?

```sql
WITH cte2
AS (
	WITH cte1 AS (
			SELECT *
				,round(sum(qty * price), 2) AS revenue_per_txn
			FROM sales
			GROUP BY txn_id
			)
	SELECT *
		,round((
				percent_rank() OVER (
					ORDER BY revenue_per_txn
					)
				) * 100) AS p
	FROM cte1
	ORDER BY 2 DESC
	)
SELECT *
FROM (
	SELECT round(avg(revenue_per_txn), 1) AS 25th_percentile_value
	FROM cte2
	WHERE p = 25
	) AS a
	,(
		SELECT round(avg(revenue_per_txn), 1) AS 50th_percentile_value
		FROM cte2
		WHERE p = 50
		) AS b
	,(
		SELECT round(avg(revenue_per_txn), 1) AS 75th_percentile_value
		FROM cte2
		WHERE p = 75
		) AS c;
```

![image](https://user-images.githubusercontent.com/45015990/201464474-7733aa34-9ffc-458c-a692-0448c9b6e3ee.png)
****
#### 2. What is the percentage split of all transactions for members vs non-members?

```sql
WITH txn
AS (
	SELECT *
		,sum(qty * price) AS txn
	FROM sales
	GROUP BY member
	)
SELECT round(((a.txn) / (a.txn + b.txn) * 100), 2) AS member_transactions
	,round(((b.txn) / (a.txn + b.txn) * 100), 2) AS non_member_transactions
FROM (
	SELECT *
	FROM txn
	WHERE `member` = 't'
	) AS a
	,(
		SELECT *
		FROM txn
		WHERE `member` = 'f'
		) AS b;
```

![image](https://user-images.githubusercontent.com/45015990/201464871-21565738-6f4d-4175-a7a6-573e02db6e3c.png)

****
#### 3. What is the average revenue for member transactions and non-member transactions?

```sql
WITH cte
AS (
	SELECT *
		,(sum(qty * price)) AS revenue
	FROM sales
	GROUP BY txn_id
		,`member`
	)
SELECT *
FROM (
	SELECT round(avg(revenue), 2) AS member_revenue
	FROM cte
	WHERE `member` = 't'
	) AS a
	,(
		SELECT round(avg(revenue), 2) AS non_member_revenue
		FROM cte
		WHERE `member` = 'f'
		) AS b;
```

![image](https://user-images.githubusercontent.com/45015990/201464941-9b1bb2b9-d506-4956-8c27-04d7285a454c.png)

***
#### 4. What are the top 3 products by total revenue before discount?
```sql
WITH rank_cte
AS (
	WITH revenue_cte AS (
			SELECT *
				,sum(qty * price) AS revenue
			FROM sales
			GROUP BY 1
			)
	SELECT *
		,rank() OVER (
			ORDER BY revenue DESC
			) AS `rank`
	FROM revenue_cte
	)
SELECT pd.product_name
	,c.revenue
FROM rank_cte c
INNER JOIN product_details pd ON c.prod_id = pd.product_id
WHERE `rank` IN (
		1
		,2
		,3
		)
ORDER BY `revenue` DESC;
```

![image](https://user-images.githubusercontent.com/45015990/201465041-feb6e635-cc9a-49b8-8721-d363f9126907.png)
***
#### 5. What is the total quantity, revenue and discount for each segment?
```sql
SELECT pd.segment_name
	,sum(s.qty) AS total_quantity
	,sum(s.price * s.qty) AS total_revenue
	,sum(s.discount) AS total_discount
FROM sales s
LEFT JOIN product_details pd ON s.prod_id = pd.product_id
GROUP BY 1;
```

![image](https://user-images.githubusercontent.com/45015990/201465347-6ce582a7-0db1-4cfa-a821-852e792595b4.png)

***
#### 6. What is the top selling product for each segment?
```sql
WITH rank_cte
AS (
	WITH quantity_cte AS (
			SELECT pd.segment_name
				,pd.product_name
				,sum(s.qty) AS total_quantity
			FROM sales s
			LEFT JOIN product_details pd ON s.prod_id = pd.product_id
			GROUP BY 1
				,2
			)
	SELECT *
		,rank() OVER (
			PARTITION BY segment_name ORDER BY total_quantity DESC
			) AS `rank`
	FROM quantity_cte
	)
SELECT segment_name
	,product_name
FROM rank_cte
WHERE `rank` = 1;
```

![image](https://user-images.githubusercontent.com/45015990/201465418-d1662319-686d-45b8-b7ab-e7ecbdaaae3a.png)


***
#### 7. What is the percentage split of revenue by product for each segment?
```sql
SELECT pd.segment_name
	,pd.product_name
	,concat(round(100 * (sum(s.qty * s.price) / sum(sum(s.qty * s.price)) OVER (PARTITION BY pd.segment_name)),2), '%') AS revenue
FROM sales s
LEFT JOIN product_details pd ON s.prod_id = pd.product_id
GROUP BY 2;
```

![image](https://user-images.githubusercontent.com/45015990/201465553-a64d09de-71f3-4b4c-bedc-2be681ab07c8.png)
****

#### 8. What is the percentage split of total revenue by category?
```sql
SELECT pd.category_name
	,round(100*(sum(s.qty * s.price) / sum(sum(s.qty * s.price)) OVER ()),1) AS revenue
FROM sales s
LEFT JOIN product_details pd ON s.prod_id = pd.product_id
GROUP BY 1;
```

![image](https://user-images.githubusercontent.com/45015990/201465700-b84d330f-a0d7-486b-9e13-295d87255e6c.png)
***

#### 9. What is the total transaction “penetration” for each product? (hint: penetration = number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)
```sql
WITH join_cte
AS (
	WITH txn_cte AS (
			SELECT *
				,count(txn_id) OVER (PARTITION BY prod_id) AS num_of_txn
			FROM sales
			)
		,total_txn_cte AS (
			SELECT count(DISTINCT txn_id) AS total_txn
			FROM sales
			)
	SELECT *
		,round(100 * (num_of_txn / total_txn), 2) AS penetration
	FROM txn_cte
		,total_txn_cte
	GROUP BY 1
	)
SELECT pd.product_name
	,j.penetration
FROM join_cte j
INNER JOIN product_details pd ON j.prod_id = pd.product_id;
```

![image](https://user-images.githubusercontent.com/45015990/201465998-1be29e89-5271-432c-8ac7-b94a47c94116.png)



