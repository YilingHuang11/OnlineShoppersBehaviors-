```
-- Describe the table
DESC online_shopper;

-- Select top 5 rows
SELECT * FROM online_shopper 
LIMIT 5;

-- Total rows 
SELECT COUNT(1) AS Num_Of_Rows
FROM online_shopper;

-- Calculate date range
SELECT MIN(time) AS start_date
        ,  MAX(time) AS end_date 
FROM online_shopper;

'''
```

# 1  Data preparation

'''

```
-- STEP 1 Delete unnecessary column user_geohash 
ALTER TABLE online_shopper DROP user_geohash;

-- STEP 2 Add two more columns [date] and [hour] for further analysis 
ALTER TABLE online_shopper ADD date date NULL;
ALTER TABLE online_shopper ADD hour int(2) NULL;
UPDATE online_shopper
SET date = DATE_FORMAT(time, '%Y-%m-%d')
    , hour = HOUR(time);

-- Preview result 
SELECT * FROM online_shopper LIMIT 5;

-- STEP 3 ADD A NEW TABLE behavior for better interpretation of behavior type
DROP TABLE IF EXISTS behavior;
CREATE TABLE behavior(id int, behavior varchar(12));
INSERT INTO behavior(id, behavior) 
VALUES (1, 'pv'), (2, 'buy'), (3, 'cart'), (4, 'fav'); -- pv: page view, uv: unique visitor, 
SELECT * FROM behavior;

-- STEP 4 Join two tables based on behavior attribute and drop duplicates 
DROP VIEW IF EXISTS trade_view;
CREATE VIEW trade_view AS (
    SELECT DISTINCT t.*
    FROM
    (
                SELECT O.user_id
                    , O.item_id
                    , b.behavior 
                    , O.item_category
                    , O.date
                    , O.`hour`
                FROM online_shopper AS O
                JOIN behavior AS b 
            ON b.id = O.behavior_type 
    ) AS t
);
-- Results
SELECT * FROM trade_view LIMIT 5;

-- Total  row numbers after dropping duplicates 
SELECT COUNT(1) AS ROW_COUNT FROM temp_trade;
-- only 46844 records left 

-- STEP 5 Create view for further use and plotting
'''
```

# 2 Key Metric

# 2.1 PV, UV, PV_PCT

```
Daily number of PV, UV , PV_PCT calculationBO
'''

 DROP VIEW IF EXISTS pv_uv_daily_view;
 CREATE VIEW pv_uv_daily_view AS
        SELECT date
         ,SUM(IF(behavior='pv',1,0)) AS 'pv_num'
         ,COUNT(DISTINCT user_id) AS 'uv_num'
         ,ROUND(
          SUM(IF(behavior='pv',1,0))
          /
                    COUNT(DISTINCT user_id),1) AS 'pv_pct'
    FROM trade_view
    GROUP BY date
    ORDER BY date
;
-- Results
SELECT * FROM pv_uv_daily_view ;  -- view can be referred in views but temporary table cannot 

'''
```

# 2.2 Bounce Rate

```
Daily Bounce Rate calculation 
-- STEP 1 Calculate all sessions include PV
                    PV definition: each user views pages in each day and hour 
-- STEP 2 Calculate Bounce from above defined sessions
                    Bounce definition: when a user opens a single page on website and then exits without triggering any other requests during that session
-- STEP 3 Calculate Bounce Rate 
                    BOUNCE RATE = BOUNCED SESSIONS/TOTAL SESSIONS * 100%
''' 
DROP VIEW IF EXISTS bounce_rate_daily_view;
CREATE VIEW bounce_rate_daily_view AS(
        WITH sessions AS(
                -- calculate all visits 
                SELECT user_id
                         , date
                         , hour
                         , COUNT(behavior) AS 'pv_num'
              FROM trade_view
                WHERE behavior = 'pv'
                GROUP BY user_id
                             , date
                             , hour
        )
        SELECT date
                 , COUNT(1) AS 'total_visits'
                 , SUM(CASE WHEN pv_num = 1 THEN 1 ELSE 0 END) AS 'bounced_num'
                 , CAST(
                                SUM(CASE WHEN pv_num = 1 THEN 1 ELSE 0 END)
                                /
                                COUNT(1) *100 AS DECIMAL(10,2)) AS 'bounce_rate %'
        FROM sessions
        GROUP BY date 
        ORDER BY date
);

-- Results
SELECT * FROM bounce_rate_daily_view;

'''
```

# 2.3 Daily Order Numbers

```
Daily Order Numbers Calculation
'''
DROP VIEW IF EXISTS orders_daily_view;
CREATE VIEW orders_daily_view AS(
        SELECT date
                 , COUNT(behavior) AS orders 
      FROM trade_view
        WHERE behavior = 'buy'
        GROUP BY date
        ORDER BY date
);

'''
```

# 2.4 uv_to_buy_rate

```
uv_to_buy_rate Calculation
'''
DROP VIEW IF EXISTS uv_to_buy_view;
CREATE VIEW uv_to_buy_view AS (
    SELECT p.date
             , P.uv_num
             , O.orders
             , CAST(
                            O.orders/P.uv_num * 100 AS DECIMAL(10, 2)
                            ) AS 'uv_to_buy_rate %'
    FROM pv_uv_daily_view AS P 
    LEFT JOIN orders_daily_view AS O
        ON P.date = O.date
    GROUP BY p.date 
);

SELECT * FROM uv_to_buy_view;

'''
```

# 3 People -  User Analysis

# 3.1 daily new acitive users

```
-- STEP 1: Calculate all active users\' first login date
-- STEP 2: Filter the first date of data collected 
-- STEP 3: GROUP BY first login date and calculate new active users
'''
DROP VIEW IF EXISTS new_active_users_daily_view;
CREATE VIEW new_active_users_daily_view AS(
SELECT T.first_login_date
         , COUNT(1) AS new_active_users
FROM (
        SELECT 
                   user_id
                 , MIN(date) AS first_login_date
        FROM trade_view
        GROUP BY user_id
) AS T
WHERE T.first_login_date > 
(
        SELECT MIN(date) 
        FROM trade_view
)
GROUP BY T.first_login_date 
ORDER BY T.first_login_date
);

'''
```

# 3.2 User Retention Analysis

```
-- STEP 1: Active users: no matter what behabiors they have. Find the users and relative dates 
-- STEP 2: Self Join to itself for further calculating retentaion rates in the required time intervals
-- STEP 3: Filter the date range to larger than right table date
-- STEP 4: Group by active date
-- STEP 5: Calculate active users by groups
-- STEP 6: According to date difference to calculate 1st, 2nd, 3rd, 4th, 5th, 6th, 7th, 15th, 30th remaining users numbers
-- STEP 7: Create a common expression to store the above result 
-- STEP 8: Calculate 1st, 2nd, 3rd, 4th, 5th, 6th, 7th, 15th, 30th user retention rates
'''

-- CREATE VIEW USING COMMON TABLE EXPRESSION 
DROP VIEW IF EXISTS retention_rate_view;
CREATE VIEW retention_rate_view AS(
WITH temp_remain_num AS(
SELECT T2.date
         , COUNT(DISTINCT (T1.user_id)) AS uv_num
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 1 THEN 1 ELSE 0 END) AS uv_remain1
-- method 2 COUNT(DISTINCT IF(DATEDIFF(T1.date, T2.date)=1, T.user_id, NULL) AS uv_remain1
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 2 THEN 1 ELSE 0 END) AS uv_remain2
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 3 THEN 1 ELSE 0 END) AS uv_remain3
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 4 THEN 1 ELSE 0 END) AS uv_remain4
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 5 THEN 1 ELSE 0 END) AS uv_remain5
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 6 THEN 1 ELSE 0 END) AS uv_remain6
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 7 THEN 1 ELSE 0 END) AS uv_remain7
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 15 THEN 1 ELSE 0 END) AS uv_remain15
         , SUM(CASE WHEN DATEDIFF(T1.date, T2.date) = 30 THEN 1 ELSE 0 END) AS uv_remain30
FROM
(
    SELECT user_id
             , date
    FROM trade_view
    GROUP BY user_id
             , date
) AS T1
LEFT JOIN
(
    SELECT user_id
             , date
    FROM trade_view
    GROUP BY user_id
                 , date
) AS T2
ON T1.user_id = T2.user_id
WHERE T1.date >= T2.date
GROUP BY T2.date
)
SELECT date
         , uv_num
         , CONCAT(CAST(uv_remain1/uv_num*100 AS DECIMAL(18,2)), '%') AS day1_retention_rate
         , CONCAT(CAST(uv_remain2/uv_num*100 AS DECIMAL(18,2)), '%') AS day2_retention_rate
         , CONCAT(CAST(uv_remain3/uv_num*100 AS DECIMAL(18,2)), '%') AS day3_retention_rate
         , CONCAT(CAST(uv_remain4/uv_num*100 AS DECIMAL(18,2)), '%') AS day4_retention_rate
         , CONCAT(CAST(uv_remain5/uv_num*100 AS DECIMAL(18,2)), '%') AS day5_retention_rate
         , CONCAT(CAST(uv_remain6/uv_num*100 AS DECIMAL(18,2)), '%') AS day6_retention_rate
         , CONCAT(CAST(uv_remain7/uv_num*100 AS DECIMAL(18,2)), '%') AS day7_retention_rate
         , CONCAT(CAST(uv_remain15/uv_num*100 AS DECIMAL(18,2)), '%') AS day15_retention_rate
         , CONCAT(CAST(uv_remain30/uv_num*100 AS DECIMAL(18,2)), '%') AS day30_retention_rate
FROM temp_remain_num
);
```

'''

# 3.3 ACTIVE USER DISTRIBUTION

```
'''
SELECT date
         , hour
         , COUNT(*) AS active_users
FROM trade_view
GROUP BY date
             , hour
ORDER BY date
             , hour; 

'''
```

# 3.4 Different behaviors by hour

```
'''
SELECT hour
          ,COUNT(CASE WHEN behavior='pv' THEN 1 ELSE NULL END) AS pv_nums
            ,COUNT(CASE WHEN behavior='fav' THEN 1 ELSE NULL END) AS fav_nums
            ,COUNT(CASE WHEN behavior='cart' THEN 1 ELSE NULL END) AS cart_nums
            ,COUNT(CASE WHEN behavior='buy' THEN 1 ELSE NULL END) AS buy_nums
FROM trade_view
GROUP BY hour
ORDER BY hour;

'''
```

# 3.5 RFM analysis

```
Since we don\'t have monetary part, we focus on recency(R) and frequency(F) 
-- STEP 1: Calculate users last order date and distance from '2019-12-18' 
-- STEP 2: Set the difference days to 5 bins and assign related scores.
-- STEP 3: Calculate users purchase frequency
-- STEP 4: Calculate F score, R score average value
-- STEP 5: classify users by average value
'''
-- R Score 
DROP VIEW IF EXISTS r_score_view;
CREATE VIEW r_score_view AS 
SELECT  user_id
            , MAX(date) AS last_order_date
            , (SELECT MAX(date) FROM trade_view) AS Today
            , DATEDIFF((SELECT MAX(date) FROM trade_view), MAX(date)) AS date_diff
            , NTILE(5) OVER(ORDER BY DATEDIFF(
                                (SELECT MAX(date) FROM trade_view), 
                                            MAX(date)) DESC
                                                ) AS r_scores
FROM trade_view
WHERE behavior = 'buy'
GROUP BY user_id;

-- F Score
DROP VIEW IF EXISTS f_score_view;
CREATE VIEW f_score_view AS
SELECT user_id
            ,COUNT(*) AS purchase_times
            ,NTILE(5) OVER(ORDER BY COUNT(*) ASC) AS f_scores
FROM trade_view
WHERE behavior = 'buy'
GROUP BY user_id;

-- R score average value & F score average value
SELECT AVG(R_Scores) AS AVG_R FROM r_score_view;
SELECT AVG(f_scores) AS AVG_F FROM f_score_view;

-- classify users
DROP VIEW IF EXISTS user_class_view;
CREATE VIEW user_class_view AS 
SELECT r.user_id
     , r.r_scores
         , f.f_scores
         , (CASE WHEN r.r_scores > 3 AND f.f_scores > 3 THEN 'Champions'
                         WHEN r.r_scores > 3 AND f.f_scores <=3 THEN 'Potential Loyalists'
                         WHEN r.r_scores <= 3 AND f.f_scores > 3 THEN 'At Risk Customers'
                         ELSE 'Can\'t Lose Them'
                         END) AS user_class 
FROM r_score_view AS r
JOIN f_score_view AS f
ON r.user_id = f.user_id 

-- Calculate each class user number and their percentages 
```

# method 1 using common table expression

```
WITH user_num AS(
SELECT user_class
            ,COUNT(1) AS user_num
FROM user_class_view
GROUP BY user_class
ORDER BY user_num DESC
),
total_users AS(
    SELECT SUM(user_num) AS total_users
    FROM user_num
)
SELECT 
        user_class
    , user_num
    , ROUND(user_num/ total_users, 3) AS percentage
FROM user_num, total_users
```

# method 2 using windows function

```
SELECT user_class
         , COUNT(1)  AS user_class_num
         , ROUND(COUNT(1) / SUM(COUNT(1)) OVER(), 3) AS Percentage 
FROM user_class_view
GROUP BY user_class
ORDER BY Percentage DESC 

'''
```

# 4  Product Analysis

```
'''
DROP VIEW IF EXISTS product_order_view;
CREATE VIEW product_order_view AS 
SELECT item_category
          ,SUM(CASE WHEN behavior='pv' THEN 1 ELSE 0 END) AS 'pv'
            ,SUM(CASE WHEN behavior='fav' THEN 1 ELSE 0 END) AS 'fav'
            ,SUM(CASE WHEN behavior='cart' THEN 1 ELSE 0 END) AS 'cart'
            ,SUM(CASE WHEN behavior='buy' THEN 1 ELSE 0 END) AS 'buy'
            ,COUNT(DISTINCT CASE WHEN behavior='buy' THEN user_id ELSE NULL END)
             /COUNT(DISTINCT user_id) AS order_rate
FROM trade_view
GROUP BY item_category
ORDER BY order_rate asc;

WITH order_rate AS(
SELECT order_rate
            ,COUNT(*) AS item_count
FROM product_order_view
GROUP BY order_rate
)

-- order rate total item counts 
SELECT SUM(item_count) AS TOTALS
FROM order_rate
WHERE order_rate >= 0.5;

'''
As we can see that 2587 items have order rate 0% wheras we have only 49 items\' order rates greater than 0.5. 
We should analyze whether the products themselves have low conversion rate or the prices are not competitive or the products are warehouse stocks etc.,
''' 

'''
```

# 5  Place Analysis - Purchase Path

```
-- STEP 1: Find Behaviors that lead buy behavior (using lag function)
-- STEP 2: Find purchase path (concat the behaviors together)
-- STEP 3: Calculate different purchase path counts
'''
DROP VIEW IF EXISTS purchase_path_view;
CREATE VIEW purchase_path_view AS 
SELECT t.*
            ,CONCAT(
                            IFNULL(t.lag4, 'NA'), '-'
                        , IFNULL(t.lag3, 'NA'), '-'
                        , IFNULL(t.lag2, 'NA'), '-'
                        , IFNULL(t.lag1, 'NA'), '-'
                        , IFNULL(t.lag0, 'NA')) AS purchase_path
FROM(
SELECT user_id
            ,item_id
            ,behavior
            ,LAG(behavior, 4) OVER(PARTITION BY user_id, item_id ORDER BY date) AS lag4
            ,LAG(behavior, 3) OVER(PARTITION BY user_id, item_id ORDER BY date) AS lag3
            ,LAG(behavior, 2) OVER(PARTITION BY user_id, item_id ORDER BY date) AS lag2
            ,LAG(behavior, 1) OVER(PARTITION BY user_id, item_id ORDER BY date) AS lag1
            ,LAG(behavior, 0) OVER(PARTITION BY user_id, item_id ORDER BY date) AS lag0
FROM trade_view
) t
WHERE t.behavior = 'buy';

SELECT purchase_path
            ,COUNT(1) AS path_count
FROM purchase_path_view
GROUP BY purchase_path
ORDER BY path_count DESC;

-- calculate the total count of behaviors path including 'fav' and 'cart'
WITH fav_cart_path_count AS(
SELECT pv
        purchase_path
    , COUNT(1) AS purchase_path_count
FROM purchase_path_view
WHERE purchase_path LIKE '%fav%'
        OR purchase_path LIKE '%cart%'
GROUP BY purchase_path
)

SELECT SUM(purchase_path_count) AS fav_cart_path_totals
FROM fav_cart_path_count; 
'''
Customers purchase directly 728 times, while purchase via adding to cart and saving as favorites only 6 times in total. We should improve the adding to cart and saving as favorites these two features. 
''' 
```
