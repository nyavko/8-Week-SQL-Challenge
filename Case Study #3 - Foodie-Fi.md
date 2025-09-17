# 🍰 Case Study #3 — Foodie-Fi
![Case Study 3 Image](https://8weeksqlchallenge.com/images/case-study-designs/3.png)

This case study is part of the 8-Week SQL Challenge by Danny Ma.

The dataset simulates a subscription-based streaming service Foodie-Fi, offering cooking/food-related video content, and focuses on analyzing:
- Customer subscription journeys and plan transitions
- Retention and churn behavior
- Revenue performance across different plans
- Upgrade and downgrade patterns

The goal is to practice SQL techniques such as joins, aggregations, CTEs, window functions, and date-based calculations to generate actionable business insights for a subscription business model.
> [!NOTE]
Full problem description and instructions can be found [here](https://8weeksqlchallenge.com/case-study-3/)

# Solution
The case study is divided into several parts: first understanding customer journeys; then more quantitative analyses; finally a challenge/extension around payments.

## A. Customer Journey -is not included here, as it is not directly related to SQL.

## B. Data Analysis Questions
Since the dataset only contains two tables, it is convenient to create a SQL view that joins them together.
```
    CREATE VIEW full_info AS
    SELECT customer_id, start_date, plans.plan_id, plan_name, price
    FROM subscriptions JOIN plans 
    ON subscriptions.plan_id = plans.plan_id;
```

  1. How many customers has Foodie-Fi ever had?
```
    SELECT COUNT(DISTINCT customer_id) AS unique_customers
    FROM subscriptions;
```
| unique_customers |
| ---------------- |
| 1000             |

2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value
```
    SELECT to_char(start_date,'Month') as month_name, COUNT(start_date) AS trial_num
    FROM full_info
    WHERE plan_name = 'trial'
    GROUP BY to_char(start_date,'Month');
```
| month_name | trial_num |
| ---------- | --------- |
| January    | 88        |
| February   | 68        |
| November   | 75        |
| September  | 87        |
| August     | 88        |
| October    | 79        |
| December   | 84        |
| June       | 79        |
| March      | 94        |
| May        | 88        |
| April      | 81        |
| July       | 89        |

3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name
``` SELECT plan_id, plan_name, COUNT(*) AS count_of_events
    FROM full_info
    WHERE start_date >= '01-01-2021'
    GROUP BY plan_id, plan_name
    ORDER BY plan_id;
```
| plan_id | plan_name     | count_of_events |
| ------- | ------------- | --------------- |
| 1       | basic monthly | 8               |
| 2       | pro monthly   | 60              |
| 3       | pro annual    | 63              |
| 4       | churn         | 71              |

4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place? 
```
SELECT SUM(CASE WHEN plan_name = 'churn' THEN 1 ELSE 0 END) AS churned_customer_count, 	CONCAT(ROUND(SUM(CASE WHEN plan_name = 'churn' THEN 1 ELSE 0 END) / COUNT(DISTINCT customer_id)::numeric(10,1) * 100, 1), '%') AS churned_rate
    FROM full_info;
```
| churned_customer_count | churned_rate |
| ---------------------- | ------------ |
| 307                    | 30.7%        |

5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number
``` SELECT COUNT(*) as customer_count, CONCAT(ROUND(COUNT(*) / (SELECT COUNT(DISTINCT customer_id) FROM full_info)::numeric(10,0)*100, 0), '%') AS immediate_churn_rate
    FROM (SELECT start_date, LEAD(start_date,1) OVER (PARTITION BY customer_id ORDER BY start_date) AS churn_date
          FROM full_info
          WHERE plan_id = 0 OR plan_id = 4) AS x
    WHERE churn_date - start_date = 7;
```
| customer_count | immediate_churn_rate |
| -------------- | -------------------- |
| 92             | 9%                   |

6. What is the number and percentage of customer plans after their initial free trial?
``` WITH next_plans AS 
    (
      SELECT *
    FROM ( SELECT customer_id, plan_name, LEAD(plan_name,1) OVER (PARTITION BY customer_id ORDER BY start_date) AS new_plan FROM full_info) AS x
    WHERE plan_name = 'trial'
     )
     SELECT new_plan AS plan_name, COUNT(new_plan) AS plan_count, ROUND(COUNT(new_plan):: numeric / ( SELECT COUNT(*) FROM next_plans) *100, 1) AS plan_rate
     FROM next_plans
     GROUP BY new_plan;
```
| plan_name     | plan_count | plan_rate |
| ------------- | ---------- | --------- |
| pro annual    | 37         | 3.7       |
| pro monthly   | 325        | 32.5      |
| churn         | 92         | 9.2       |
| basic monthly | 546        | 54.6      |

7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?
``` WITH up_to_date_plans AS 
    (
      SELECT DISTINCT customer_id, FIRST_VALUE(plan_name) OVER (PARTITION BY customer_id ORDER BY start_date DESC) AS plan_name
    FROM full_info
    WHERE start_date <= '2020-12-31'
    )
    SELECT plan_name, COUNT(plan_name) AS plan_count, ROUND(COUNT(plan_name):: numeric / ( SELECT COUNT(*) FROM up_to_date_plans) *100, 1) AS plan_rate
     FROM up_to_date_plans
     GROUP BY plan_name;
```
| plan_name     | plan_count | plan_rate |
| ------------- | ---------- | --------- |
| pro annual    | 195        | 19.5      |
| trial         | 19         | 1.9       |
| churn         | 236        | 23.6      |
| pro monthly   | 326        | 32.6      |
| basic monthly | 224        | 22.4      |

8. How many customers have upgraded to an annual plan in 2020?
```
    WITH curr_plans AS
    (
      SELECT customer_id, plan_name AS curr_plan, LAG(plan_name,1) OVER (PARTITION BY customer_id ORDER BY start_date) AS prev_plan 
    FROM full_info
    WHERE start_date > '2019-12-31' AND start_date < '2021-01-01'
     )
     SELECT COUNT(customer_id) AS customer_count
     FROM curr_plans
     WHERE curr_plan = 'pro annual' AND prev_plan IS NOT NULL;
```
| customer_count |
| -------------- |
| 195            |

9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
``` WITH trial_info AS
    (
      SELECT customer_id, start_date AS trial_date
      FROM full_info
      WHERE plan_id = 0
     ),
     annual_info AS
    (
      SELECT customer_id, start_date AS annual_date
      FROM full_info
      WHERE plan_id = 3
     )
     SELECT AVG(annual_date - trial_date) AS avg_days_to_annual_plan
     FROM trial_info JOIN annual_info
     ON trial_info.customer_id = annual_info.customer_id;
```
| avg_days_to_annual_plan |
| ----------------------- |
| 104.6201550387596899    |

10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
``` WITH trial_info AS
    (
      SELECT customer_id, start_date AS trial_date
      FROM full_info
      WHERE plan_id = 0
     ),
     annual_info AS
    (
      SELECT customer_id, start_date AS annual_date
      FROM full_info
      WHERE plan_id = 3
     ), 
     days_dist AS
     (
       SELECT trial_info.customer_id, CONCAT((annual_date - trial_date) / 30 * 30 + 1, ' - ', 30 + (annual_date - trial_date)/30*30, ' days') AS days_to_annual_plan
     FROM trial_info JOIN annual_info
     ON trial_info.customer_id = annual_info.customer_id
     )
     SELECT days_to_annual_plan, COUNT(*) AS customer_count
     FROM days_dist
     GROUP BY days_to_annual_plan
     ORDER BY COUNT(*) DESC;
```
| days_to_annual_plan | customer_count |
| ------------------- | -------------- |
| 1 - 30 days         | 48             |
| 121 - 150 days      | 43             |
| 91 - 120 days       | 35             |
| 151 - 180 days      | 35             |
| 61 - 90 days        | 33             |
| 181 - 210 days      | 27             |
| 31 - 60 days        | 25             |
| 241 - 270 days      | 5              |
| 211 - 240 days      | 4              |
| 301 - 330 days      | 1              |
| 331 - 360 days      | 1              |
| 271 - 300 days      | 1              |

11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
```
WITH basic_info AS
    (
      SELECT customer_id, start_date AS basic_date
      FROM full_info
      WHERE plan_id = 1
     ),
     pro_info AS
    (
      SELECT customer_id, start_date AS pro_date
      FROM full_info
      WHERE plan_id = 2
     )
     SELECT COUNT(*) AS customer_count
     FROM basic_info JOIN pro_info
     ON basic_info.customer_id = pro_info.customer_id
     WHERE basic_date > pro_date;
```
| customer_count |
| -------------- |
| 0              |


[View on DB Fiddle](https://www.db-fiddle.com/f/rHJhRrXy5hbVBNJ6F6b9gJ/16)
