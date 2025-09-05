# 🍕 Case Study #2 — Pizza Runner

This case study is part of the 8-Week SQL Challenge by Danny Ma.

The dataset simulates a pizza delivery business and focuses on analyzing:
- Delivery performance and efficiency
- Customer ordering patterns
- Revenue and sales trends
- Popular menu items

The goal is to practice SQL techniques like joins, aggregations, CTEs, and window functions to generate actionable business insights.

> [!NOTE] 
> Full problem description can be found [here](https://8weeksqlchallenge.com/case-study-2/)

# Solution

## Data cleaning and transforming
```
--updating data tables
UPDATE runner_orders
SET cancellation = ''
WHERE cancellation IS NULL OR cancellation = 'null';

UPDATE runner_orders
SET pickup_time = NULL, distance = NULL, duration = NULL
WHERE cancellation <> '';

UPDATE runner_orders
SET distance = REGEXP_REPLACE(distance, '[^0-9 .]+', '', 'g');

UPDATE runner_orders
SET duration = REGEXP_REPLACE(duration, '[^0-9]+', '', 'g');

ALTER TABLE runner_orders
ALTER COLUMN distance TYPE DECIMAL USING distance:: decimal,
ALTER COLUMN duration TYPE DECIMAL using duration:: decimal,
ALTER COLUMN pickup_time TYPE TIMESTAMP USING pickup_time:: timestamp;

UPDATE customer_orders
SET exclusions = ''
WHERE exclusions IS NULL OR exclusions = 'null';

UPDATE customer_orders
SET extras = ''
WHERE extras IS NULL OR extras = 'null';
```

## A. Pizza Metrics
1. How many pizzas were ordered?
```
SELECT COUNT(pizza_id) AS pizzas_num
    FROM customer_orders;
```

| pizzas_num |
| ---------- |
| 14         |

---
2. How many unique customer orders were made?
```
SELECT COUNT(DISTINCT order_id) AS unique_orders
    FROM customer_orders;
```

| unique_orders |
| ------------- |
| 10            |

---
3. How many successful orders were delivered by each runner?
```
SELECT runner_id, COUNT(cancellation) AS successful_orders_num
    FROM runner_orders
    WHERE cancellation = ''
    GROUP BY runner_id;
```

| runner_id | successful_orders_num |
| --------- | --------------------- |
| 1         | 4                     |
| 2         | 3                     |
| 3         | 1                     |

---
4. How many of each type of pizza was delivered?
```
SELECT pizza_name, COUNT(customer_orders.pizza_id) AS total_quantity
    FROM runner_orders JOIN customer_orders 
    ON runner_orders.order_id = customer_orders.order_id JOIN pizza_names
    ON customer_orders.pizza_id = pizza_names.pizza_id
    WHERE cancellation = ''
    GROUP BY pizza_name;
```

| pizza_name | total_quantity |
| ---------- | -------------- |
| Meatlovers | 9              |
| Vegetarian | 3              |

---
5. How many Vegetarian and Meatlovers were ordered by each customer?
```
SELECT customer_id, SUM(CASE WHEN pizza_name = 'Vegetarian' THEN 1 ELSE 0 END) AS vegetarian_quantity, SUM(CASE WHEN pizza_name = 'Meatlovers' THEN 1 ELSE 0 END) AS meatlovers_quantity
    FROM customer_orders JOIN pizza_names
    ON customer_orders.pizza_id = pizza_names.pizza_id
    GROUP BY customer_id
    ORDER BY customer_id ASC;
```

| customer_id | vegetarian_quantity | meatlovers_quantity |
| ----------- | ------------------- | ------------------- |
| 101         | 1                   | 2                   |
| 102         | 1                   | 2                   |
| 103         | 1                   | 3                   |
| 104         | 0                   | 3                   |
| 105         | 1                   | 0                   |

---
6. What was the maximum number of pizzas delivered in a single order?
```
    SELECT COUNT(customer_orders.pizza_id) AS max_quantity
    FROM runner_orders JOIN customer_orders 
    ON runner_orders.order_id = customer_orders.order_id
    WHERE cancellation = ''
    GROUP BY customer_orders.order_id
    ORDER BY COUNT(customer_orders.pizza_id) DESC
    FETCH FIRST ROW ONLY;
```

| max_quantity |
| ------------ |
| 3            |

---
7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```
SELECT customer_orders.customer_id, SUM(CASE WHEN exclusions <>'' OR extras <> '' THEN 1 ELSE 0 END) AS sum_of_changed_pizza, SUM(CASE WHEN exclusions = '' AND extras = '' THEN 1 ELSE 0 END) AS sum_of_unchanged_pizzas
    FROM runner_orders JOIN customer_orders 
    ON runner_orders.order_id = customer_orders.order_id
    WHERE cancellation = ''
    GROUP BY customer_orders.customer_id
    ORDER BY customer_orders.customer_id;
```

| customer_id | sum_of_changed_pizza | sum_of_unchanged_pizzas |
| ----------- | -------------------- | ----------------------- |
| 101         | 0                    | 2                       |
| 102         | 0                    | 3                       |
| 103         | 3                    | 0                       |
| 104         | 2                    | 1                       |
| 105         | 1                    | 0                       |

---
 8. How many pizzas were delivered that had both exclusions and extras?
```
SELECT SUM(CASE WHEN exclusions <>'' AND extras <> '' THEN 1 ELSE 0 END) AS total_sum_of_very_changed_pizzas 
    FROM runner_orders JOIN customer_orders 
    ON runner_orders.order_id = customer_orders.order_id
    WHERE cancellation = '';
````

| total_sum_of_very_changed_pizzas |
| -------------------------------- |
| 1                                |

---
9. What was the total volume of pizzas ordered for each hour of the day?
```
SELECT date_part('hour', order_time) AS each_hour, COUNT(pizza_id) AS volume_of_pizzas
    FROM customer_orders
    GROUP BY date_part('hour', order_time)
    ORDER BY date_part('hour', order_time);
```

| each_hour | volume_of_pizzas |
| --------- | ---------------- |
| 11        | 1                |
| 13        | 3                |
| 18        | 3                |
| 19        | 1                |
| 21        | 3                |
| 23        | 3                |

---
10. What was the volume of orders for each day of the week?
```
SELECT CASE date_part('dow', order_time)
        WHEN 0 THEN 'Sun'
        WHEN 1 THEN 'Mon'
        WHEN 2 THEN 'Tue'
        WHEN 3 THEN 'Wed'
        WHEN 4 THEN 'Thu'
        WHEN 5 THEN 'Fri'
        WHEN 6 THEN 'Sat'
        END AS day_of_week, COUNT(pizza_id) AS volume_of_pizzas
    FROM customer_orders
    GROUP BY date_part('dow', order_time)
    ORDER BY date_part('dow', order_time);
```

| day_of_week | volume_of_pizzas |
| ----------- | ---------------- |
| Wed         | 5                |
| Thu         | 3                |
| Fri         | 1                |
| Sat         | 5                |

## B. Runner and Customer Experience
1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```
SELECT date_part ('week', registration_date)AS week_num, COUNT(runner_id) AS num_of_runners
    FROM runners
    GROUP BY date_part ('week', registration_date)
    ORDER BY date_part ('week', registration_date);
```

| week_num | num_of_runners |
| -------- | -------------- |
| 1        | 1              |
| 2        | 1              |
| 53       | 2              |

2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```
SELECT runner_id, CAST(AVG(EXTRACT(EPOCH FROM (pickup_time - order_time)) / 60) AS DECIMAL(10,2)) AS avg_time_in_minutes
    FROM runner_orders JOIN customer_orders 
    ON runner_orders.order_id = customer_orders.order_id
    WHERE cancellation = ''
    GROUP BY runner_id
    ORDER BY runner_id;
```

| runner_id | avg_time_in_minutes |
| --------- | ------------------- |
| 1         | 15.68               |
| 2         | 23.72               |
| 3         | 10.47               |

---
3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```
WITH temp AS
    (
        SELECT customer_orders.order_id, COUNT(pizza_id) AS num_of_pizzas, EXTRACT(EPOCH FROM (pickup_time - order_time) / 60) AS time_in_minutes
      FROM runner_orders JOIN customer_orders 
      ON runner_orders.order_id = customer_orders.order_id
      WHERE cancellation = ''
      GROUP BY customer_orders.order_id, order_time, pickup_time
      ORDER BY customer_orders.order_id
      )
      SELECT corr(num_of_pizzas, time_in_minutes) AS corr_coef, corr(num_of_pizzas, time_in_minutes) / SQRT((1 - POW(corr(num_of_pizzas, time_in_minutes),2))/(COUNT(num_of_pizzas)-2)) AS t_value, 2.306 AS critical_value, CASE 
      WHEN corr(num_of_pizzas, time_in_minutes) / SQRT((1 - POW(corr(num_of_pizzas, time_in_minutes),2))/(COUNT(num_of_pizzas)-2)) > 2.306 THEN 'the relationship is statistically significant'
      ELSE 'the relationship is not statistically significant'
      END
      FROM temp;
```

| corr_coef          | t_value            | critical_value | case                                          |
| ------------------ | ------------------ | -------------- | --------------------------------------------- |
| 0.8356841187207869 | 3.7271685133750956 | 2.306          | the relationship is statistically significant |

---
4. What was the average distance travelled for each customer?
```
WITH distinct_customer_orders AS 
    (
    SELECT DISTINCT order_id, customer_id
      FROM customer_orders
    )
    SELECT customer_id, CAST(AVG(distance) AS DECIMAL(10,2)) AS avg_distance
    FROM runner_orders JOIN distinct_customer_orders 
    ON runner_orders.order_id = distinct_customer_orders.order_id
    WHERE cancellation = ''
    GROUP BY customer_id
    ORDER BY customer_id;
```

| customer_id | avg_distance |
| ----------- | ------------ |
| 101         | 20.00        |
| 102         | 18.40        |
| 103         | 23.40        |
| 104         | 10.00        |
| 105         | 25.00        |

---
5. What was the difference between the longest and shortest delivery times for all orders?
```
SELECT MAX(duration) - MIN(duration) AS diff
    FROM runner_orders
    WHERE cancellation = '';
```

| diff |
| ---- |
| 30   |

---
6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
```
SELECT runner_id, CAST(AVG(distance/(duration/60))AS DECIMAL(10,2)) AS avg_speed
    FROM runner_orders
    GROUP BY runner_id
    ORDER BY runner_id;
```

| runner_id | avg_speed |
| --------- | --------- |
| 1         | 45.54     |
| 2         | 62.90     |
| 3         | 40.00     |

---
7. What is the successful delivery percentage for each runner?
```
SELECT runner_id, CONCAT(100 * CAST(CAST(SUM(CASE WHEN cancellation = '' THEN 1 ELSE 0 END) AS FLOAT) / CAST(COUNT(order_id) AS FLOAT) AS DECIMAL (10,2)), '%') AS success_rate
    FROM runner_orders
    GROUP BY runner_id
    ORDER BY runner_id;
```

| runner_id | success_rate |
| --------- | ------------ |
| 1         | 100.00%      |
| 2         | 75.00%       |
| 3         | 50.00%       |

---
## C. Ingredient Optimisation
1. What are the standard ingredients for each pizza? 
```
WITH pizza_recipes_new AS
    (
    	SELECT pizza_id, CAST(UNNEST(STRING_TO_ARRAY(toppings, ', ')) AS INTEGER) AS topping_id
    	FROM pizza_recipes
    )
    SELECT pizza_name, STRING_AGG(topping_name, ', ') AS toppings_names
    FROM pizza_names JOIN pizza_recipes_new
    ON pizza_names.pizza_id = pizza_recipes_new.pizza_id JOIN pizza_toppings 
    ON pizza_recipes_new.topping_id = pizza_toppings.topping_id
    GROUP BY pizza_name;
```

| pizza_name | toppings_names                                                        |
| ---------- | --------------------------------------------------------------------- |
| Meatlovers | Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| Vegetarian | Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce            |

---
2. What was the most commonly added extra?
```
WITH customer_orders_new AS
    (
      SELECT CAST(UNNEST(STRING_TO_ARRAY(extras, ', ')) AS INTEGER) AS extra_id
      FROM customer_orders
    )
    SELECT topping_name
    FROM customer_orders_new JOIN pizza_toppings 
    ON extra_id = topping_id
    GROUP BY (topping_name)
    ORDER BY COUNT(extra_id) DESC
    FETCH NEXT 1 ROW ONLY;
```

| topping_name |
| ------------ |
| Bacon        |

---
3. What was the most common exclusion?
```
WITH customer_orders_new AS
    (
      SELECT CAST(UNNEST(STRING_TO_ARRAY(exclusions, ', ')) AS INTEGER) AS exclusions_id
      FROM customer_orders
    )
    SELECT topping_name
    FROM customer_orders_new JOIN pizza_toppings 
    ON exclusions_id = topping_id
    GROUP BY (topping_name)
    ORDER BY COUNT(exclusions_id) DESC
    FETCH NEXT 1 ROW ONLY;
```

| topping_name |
| ------------ |
| Cheese       |

---
4. Generate an order item for each record in the customers_orders table in the format of one of the following:
    -- Meat Lovers
    -- Meat Lovers - Exclude Beef
    -- Meat Lovers - Extra Bacon
    -- Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
```
WITH customer_orders_fixed AS
    (
      SELECT order_id, pizza_id, exclusions, CAST(UNNEST(STRING_TO_ARRAY(exclusions, ', ')) AS INTEGER) AS exclusion_id, extras, CAST(UNNEST(STRING_TO_ARRAY(extras, ', ')) AS INTEGER) AS extra_id
      FROM customer_orders
    ),
    exclusions AS 
    (
      SELECT exclusions AS exclude_id, STRING_AGG(DISTINCT topping_name, ', ') AS exclusion_names
      FROM customer_orders_fixed JOIN pizza_toppings
      ON exclusion_id = topping_id
      GROUP BY exclusions
    ),
    extras AS 
    (
      SELECT extras AS extra_id, STRING_AGG(DISTINCT topping_name, ', ') AS extras_names
      FROM customer_orders_fixed JOIN pizza_toppings
      ON extra_id = topping_id
      GROUP BY extras
    )
    SELECT order_id, CONCAT('-- ', pizza_name, CASE WHEN exclusions <> '' THEN CONCAT(' - Exclude ', exclusion_names) ELSE '' END, CASE WHEN extras <> '' THEN CONCAT(' - Extra ', extras_names) ELSE '' END) AS order_list
    FROM customer_orders JOIN pizza_names
    ON customer_orders.pizza_id = pizza_names.pizza_id LEFT JOIN extras
    ON customer_orders.extras = extras.extra_id LEFT JOIN exclusions
    ON customer_orders.exclusions = exclusions.exclude_id
    ORDER BY order_id;
```

| order_id | order_list                                                         |
| -------- | ------------------------------------------------------------------ |
| 1        | -- Meatlovers                                                      |
| 2        | -- Meatlovers                                                      |
| 3        | -- Vegetarian                                                      |
| 3        | -- Meatlovers                                                      |
| 4        | -- Meatlovers - Exclude Cheese                                     |
| 4        | -- Meatlovers - Exclude Cheese                                     |
| 4        | -- Vegetarian - Exclude Cheese                                     |
| 5        | -- Meatlovers - Extra Bacon                                        |
| 6        | -- Vegetarian                                                      |
| 7        | -- Vegetarian - Extra Bacon                                        |
| 8        | -- Meatlovers                                                      |
| 9        | -- Meatlovers - Exclude Cheese - Extra Bacon, Chicken              |
| 10       | -- Meatlovers - Exclude BBQ Sauce, Mushrooms - Extra Bacon, Cheese |
| 10       | -- Meatlovers                                                      |

---
5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
```
WITH pizza_recipes_fixed AS (
      SELECT pizza_id, temp.topping_id, topping_name
      FROM (
      SELECT pizza_id, CAST(UNNEST(STRING_TO_ARRAY(toppings, ', ')) AS INTEGER) AS topping_id
      FROM pizza_recipes
      ) temp
      JOIN pizza_toppings
      ON temp.topping_id = pizza_toppings.topping_id
    )
    SELECT order_id, CONCAT( pizza_name, ': ', ARRAY_TO_STRING(ARRAY
      (SELECT CASE 
                WHEN (topping_id IN (
                            SELECT topping_id
                            FROM pizza_recipes_fixed
                            WHERE pizza_id = customer_orders.pizza_id)
            AND
            position(CAST(pizza_toppings.topping_id AS TEXT) in extras) <> 0)
                THEN CONCAT('2x ', topping_name)
                ELSE topping_name
              END
      FROM pizza_toppings
      WHERE (topping_id IN (
                            SELECT topping_id
                            FROM pizza_recipes_fixed
                            WHERE pizza_id = customer_orders.pizza_id)
            OR
            position(CAST(pizza_toppings.topping_id AS TEXT) in extras) <> 0)
            AND
            position(CAST(pizza_toppings.topping_id AS TEXT) in exclusions) = 0
       ORDER BY topping_name
      ), ', '
    )) AS recipe
    FROM customer_orders JOIN pizza_names ON customer_orders.pizza_id = pizza_names.pizza_id;
```

| order_id | recipe                                                                               |
| -------- | ------------------------------------------------------------------------------------ |
| 10       | Meatlovers: BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 2        | Meatlovers: BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 3        | Meatlovers: BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 4        | Meatlovers: BBQ Sauce, Bacon, Beef, Chicken, Mushrooms, Pepperoni, Salami            |
| 4        | Meatlovers: BBQ Sauce, Bacon, Beef, Chicken, Mushrooms, Pepperoni, Salami            |
| 10       | Meatlovers: 2x Bacon, 2x Cheese, Beef, Chicken, Pepperoni, Salami                    |
| 5        | Meatlovers: 2x Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| 8        | Meatlovers: BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 1        | Meatlovers: BBQ Sauce, Bacon, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 9        | Meatlovers: 2x Bacon, 2x Chicken, BBQ Sauce, Beef, Mushrooms, Pepperoni, Salami      |
| 4        | Vegetarian: Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes                       |
| 7        | Vegetarian: Bacon, Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes        |
| 3        | Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes               |
| 6        | Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes               |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/7VcQKQwsS3CTkGRFG7vu98/3846)
