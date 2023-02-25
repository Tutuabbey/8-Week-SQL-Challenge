**Schema (PostgreSQL v13)**

---
**1. What is the total amount each customer spent at the restaurant?**

    SELECT
       customer_id,
       SUM(price) AS amount
    FROM sales
    INNER JOIN menu
    USING (product_id)
    GROUP BY customer_id
    ORDER BY customer_id;

| customer_id | amount |
| ----------- | ------ |
| A           | 76     |
| B           | 74     |
| C           | 36     |

---
**2. How many days has each customer visited the restaurant?**

    SELECT customer_id, COUNT(DISTINCT(order_date)) AS no_of_visits
    From sales
    GROUP BY customer_id
    ORDER BY no_of_visits DESC;

| customer_id | no_of_visits |
| ----------- | ------------ |
| B           | 6            |
| A           | 4            |
| C           | 2            |

---
**3. What was the first item from the menu purchased by each customer?**

    WITH first_purchase AS 
    (
        SELECT 
            *,
            ROW_NUMBER () OVER (PARTITION BY customer_id ORDER BY order_date ) AS purchase_number            
        FROM sales
    )
    SELECT 
        customer_id, product_name
    FROM first_purchase 
    INNER JOIN menu
    USING(product_id)
    WHERE purchase_number = 1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

---
**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**

    SELECT product_name, COUNT(product_id) AS product
    FROM menu 
    JOIN sales
    USING(product_id)
    GROUP BY product_name
    ORDER BY COUNT(product_id) DESC
    LIMIT 1;

| product_name | product |
| ------------ | ------- |
| ramen        | 8       |

---
**5. Which item was the most popular for each customer?**

    WITH purchase_counts AS 
    (
      SELECT s.customer_id, m.product_name, COUNT(*) AS purchase_quantity,
             DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY COUNT(*) DESC) AS rank
      FROM sales s
      INNER JOIN menu m 
      USING(product_id)
      GROUP BY s.customer_id, m.product_name
    )
    SELECT customer_id, product_name, purchase_quantity
    FROM purchase_counts
    WHERE rank = 1;

| customer_id | product_name | purchase_quantity |
| ----------- | ------------ | ----------------- |
| A           | ramen        | 3                 |
| B           | ramen        | 2                 |
| B           | curry        | 2                 |
| B           | sushi        | 2                 |
| C           | ramen        | 3                 |

---
**6. Which item was purchased first by the customer after they became a member?**

    WITH member_purchase AS 
    (
    SELECT *,
    ROW_NUMBER () OVER(PARTITION BY customer_id ORDER BY order_date) AS rank
    FROM sales
    INNER JOIN menu USING(product_id)
    INNER JOIN members USING(customer_id)
      WHERE order_date >= join_date
      ORDER BY customer_id, order_date
      )
     SELECT customer_id, product_name
     FROM member_purchase
     WHERE rank = 1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| B           | sushi        |

---
**7. Which item was purchased just before the customer became a member?**

    With before_member_purchase AS (
    SELECT *,
    RANK () OVER(PARTITION BY customer_id ORDER BY order_date DESC) AS rank
    FROM sales
    INNER JOIN menu USING(product_id)
    INNER JOIN members USING(customer_id)
      WHERE order_date < join_date
     ORDER BY customer_id, order_date
      )
      
     SELECT customer_id, product_name
     FROM before_member_purchase
     WHERE rank = 1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| A           | curry        |
| B           | sushi        |

---
**8. What is the total items and amount spent for each member before they became a member?**

    SELECT customer_id, sum(price) AS total_spent_before_membership
    FROM sales
    INNER JOIN menu using(product_id)
    INNER JOIN members using(customer_id)
      WHERE order_date < join_date
      GROUP BY customer_id
     ORDER BY customer_id;

| customer_id | total_spent_before_membership |
| ----------- | ----------------------------- |
| A           | 25                            |
| B           | 40                            |

---
**9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**

    WITH customer_points AS
    (
      SELECT *,
      	CASE
      		WHEN product_id = 1 THEN price * 20
      		ELSE price * 10
      	END AS points
      FROM menu
      )
      SELECT customer_id, sum(points)
      FROM customer_points
      INNER JOIN sales
      USING(product_id)
     GROUP BY customer_id
     ORDER BY customer_id;

| customer_id | sum |
| ----------- | --- |
| A           | 860 |
| B           | 940 |
| C           | 360 |

---
**10. In the first week after a customer joins the program (including their join date)
they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**

    WITH dates AS 
    (
      SELECT *, 
      join_date + INTERVAL '6 DAY' AS valid_date, 
      DATE_TRUNC('month', '2021-01-31'::DATE) + INTERVAL '1 MONTH - 1 DAY' AS last_date
    FROM members
    
    ) 
    SELECT Customer_id, 
           SUM(
    	   CASE 
    	  WHEN product_ID = 1 THEN price *20
          WHEN order_date between join_date and valid_date Then price *2 * 10
    	  ELSE price * 10
    	  END 
    	  ) as Points
    FROM Dates
    JOIN Sales USING(customer_id)
    JOIN Menu USING(product_id)
    WHERE order_date < last_date
    GROUP BY customer_id;

| customer_id | points |
| ----------- | ------ |
| A           | 1370   |
| B           | 820    |

---
