#🍜 Case Study #1 — Danny’s Diner
![Case Study 1 Image](https://8weeksqlchallenge.com/images/case-study-designs/1.png)

This case study is part of the 8-Week SQL Challenge by Danny Ma.
The dataset simulates a small Japanese restaurant, Danny’s Diner, and explores how to use SQL to answer business questions about:
- Customer visiting patterns
- Spending habits
- Menu preferences
- Loyalty and rewards.

> [!NOTE]
Full problem description and instructions can be found [here](https://8weeksqlchallenge.com/case-study-1/)

#Solution

1. What is the total amount each customer spent at the restaurant?
    ```SELECT
      customer_id, SUM(price) AS total_sum
    FROM dannys_diner.sales JOIN dannys_diner.menu
    ON dannys_diner.sales.product_id = dannys_diner.menu.product_id
    GROUP BY customer_id
    ORDER BY customer_id ASC;```

| customer_id | total_sum |
| ----------- | --------- |
| A           | 76        |
| B           | 74        |
| C           | 36        |

2. How many days has each customer visited the restaurant?
    ```SELECT
      customer_id, COUNT(DISTINCT order_date) AS visits_num
    FROM dannys_diner.sales JOIN dannys_diner.menu
    ON dannys_diner.sales.product_id = dannys_diner.menu.product_id
    GROUP BY customer_id
    ORDER BY customer_id ASC;```

| customer_id | visits_num |
| ----------- | ---------- |
| A           | 4          |
| B           | 6          |
| C           | 2          |

---
**Query #3**

    
    
    -- 3. What was the first item from the menu purchased by each customer?
    SELECT DISTINCT ON (customer_id)
       customer_id, order_date, STRING_AGG(product_name, ', ') AS first_order
    FROM dannys_diner.sales JOIN dannys_diner.menu
    ON dannys_diner.sales.product_id = dannys_diner.menu.product_id
    GROUP BY customer_id, order_date
    ORDER BY customer_id ASC, order_date ASC;

| customer_id | order_date | first_order  |
| ----------- | ---------- | ------------ |
| A           | 2021-01-01 | sushi, curry |
| B           | 2021-01-01 | curry        |
| C           | 2021-01-01 | ramen, ramen |

---
**Query #4**

    
    
    -- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
    SELECT
      product_name, COUNT(sales.product_id) AS order_quantity
    FROM dannys_diner.sales JOIN dannys_diner.menu
    ON dannys_diner.sales.product_id = dannys_diner.menu.product_id
    GROUP BY product_name
    ORDER BY COUNT(sales.product_id) DESC
    FETCH FIRST 1 ROW WITH TIES;

| product_name | order_quantity |
| ------------ | -------------- |
| ramen        | 8              |

---
**Query #5**

    
    
    -- 5. Which item was the most popular for each customer?
    WITH ranked_quantity AS
    (  SELECT
          customer_id, product_name, COUNT(sales.product_id) AS order_quantity,       RANK () OVER (
              PARTITION BY customer_id
              ORDER BY COUNT(sales.product_id) DESC
          ) quantity_rank
      FROM dannys_diner.sales JOIN dannys_diner.menu
      ON dannys_diner.sales.product_id = dannys_diner.menu.product_id
      GROUP BY customer_id, product_name
    )
    SELECT
      customer_id, STRING_AGG(product_name,', '), order_quantity
    FROM ranked_quantity
    WHERE quantity_rank = 1
    GROUP BY customer_id, order_quantity
    ORDER BY customer_id ASC;

| customer_id | string_agg          | order_quantity |
| ----------- | ------------------- | -------------- |
| A           | ramen               | 3              |
| B           | ramen, curry, sushi | 2              |
| C           | ramen               | 3              |

---
**Query #6**

    
    
    -- 6. Which item was purchased first by the customer after they became a member?
    SELECT DISTINCT ON (members.customer_id)
      members.customer_id, STRING_AGG(product_name, ', ') AS first_order_since_membership
    FROM members JOIN sales
    ON members.customer_id = sales.customer_id JOIN menu
    ON sales.product_id = menu.product_id
    WHERE order_date > join_date
    GROUP BY members.customer_id, order_date
    ORDER BY members.customer_id ASC, order_date ASC;

| customer_id | first_order_since_membership |
| ----------- | ---------------------------- |
| A           | ramen                        |
| B           | sushi                        |

---
**Query #7**

    
    
    -- 7. Which item was purchased just before the customer became a member?
    SELECT DISTINCT ON (members.customer_id)
      members.customer_id, STRING_AGG(product_name, ', ') AS first_order_since_membership
    FROM members JOIN sales
    ON members.customer_id = sales.customer_id JOIN menu
    ON sales.product_id = menu.product_id
    WHERE order_date < join_date
    GROUP BY members.customer_id, order_date
    ORDER BY members.customer_id ASC, order_date DESC;

| customer_id | first_order_since_membership |
| ----------- | ---------------------------- |
| A           | sushi, curry                 |
| B           | sushi                        |

---
**Query #8**

    
    
    -- 8. What is the total items and amount spent for each member before they became a member?
    SELECT
      members.customer_id, COUNT(sales.product_id) AS total_quantity, SUM(price) AS total_price
    FROM members JOIN sales
    ON members.customer_id = sales.customer_id JOIN menu
    ON sales.product_id = menu.product_id
    WHERE order_date < join_date
    GROUP BY members.customer_id
    ORDER BY members.customer_id ASC;

| customer_id | total_quantity | total_price |
| ----------- | -------------- | ----------- |
| A           | 2              | 25          |
| B           | 3              | 40          |

---
**Query #9**

    
    
    -- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
    SELECT
      customer_id,
        SUM( CASE WHEN product_name='sushi'
        THEN price * 20
        ELSE price * 10
            END ) AS total_points
    FROM dannys_diner.sales JOIN dannys_diner.menu
    ON dannys_diner.sales.product_id = dannys_diner.menu.product_id
    GROUP BY customer_id
    ORDER BY customer_id ASC;

| customer_id | total_points |
| ----------- | ------------ |
| A           | 860          |
| B           | 940          |
| C           | 360          |

---
**Query #10**

    
    
    -- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
    SELECT
      members.customer_id,
        SUM (CASE 
             WHEN ((order_date::date - join_date::date > 0) AND (order_date::date - join_date::date < 7)) OR product_name = 'sushi'
        THEN price * 20
        ELSE price * 10
            END ) AS total_points
    FROM members JOIN sales
    ON members.customer_id = sales.customer_id JOIN menu
    ON sales.product_id = menu.product_id
    WHERE order_date < '2021-02-01'
    --WHERE EXTRACT(MONTH FROM order_date) = 1
    GROUP BY members.customer_id
    ORDER BY members.customer_id ASC;

| customer_id | total_points |
| ----------- | ------------ |
| A           | 1220         |
| B           | 820          |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138)

