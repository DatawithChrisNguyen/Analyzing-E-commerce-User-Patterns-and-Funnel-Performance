# SQL-E-Commerce
Using SQL to extract and analyze key e-commerce and user behavior metrics from Google Analytics sample data. It covers traffic performance, conversion funnels, revenue breakdowns, and cohort analysis across various timeframes and dimensions. The queries are designed to support data-driven insights into user engagement and purchasing behavior.

## Dataset
Link to the dataset: [https://support.google.com/analytics/answer/3437719?hl=en]

## Description table
![](Images/E-Com-Des-Table.png)

## Problem Solving

### Query 01: calculate total visit, pageview, transaction for Jan, Feb, and March 2017 (order by month)
```
SELECT 
      FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
      COUNT(totals.visits) as visits,
      COUNT(totals.pageviews) as pageviews,
      COUNT(totals.transactions) as transactions    
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix between '0101' and '0331'
GROUP BY month
ORDER BY month;
```
![Result](Images/Query1.png)
### Query 02: Bounce rate per traffic source in July 2017 (Bounce_rate = num_bounce/total_visit) (order by total_visit DESC)
```
SELECT
    trafficSource.source as source,
    sum(totals.visits) as total_visits,
    sum(totals.Bounces) as total_no_of_bounces,
    (sum(totals.Bounces)/sum(totals.visits))* 100.00 as bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY source
ORDER BY total_visits DESC;
```
![Result](Images/Query2.png)
### Query 3: Revenue by traffic source by week, by month in June 2017
```
WITH month_revenue AS
      (SELECT
        'Month' AS time_type,
        FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', date)) AS time,
        trafficSource.source AS source,
        round(SUM(product.productRevenue) / 1000000,4) AS revenue
      FROM
        `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
      WHERE
        product.productRevenue IS NOT NULL
      GROUP BY
        source,
        time 
      ORDER BY source
      ),

  weekly_revenue AS (
    SELECT
      'Week' AS time_type,
      CASE EXTRACT(ISOWEEK FROM PARSE_DATE('%Y%m%d', date))
        WHEN 22 THEN "Week 1" --Date from 1st to 4th--
        WHEN 23 THEN "Week 2" --Date from 5th to 11th--
        WHEN 24 THEN "Week 3" --Date from 12th to 18th--
        WHEN 25 THEN "Week 4" --Date from 19th to 25th--
        WHEN 26 THEN "Week 5" --The rest--
        ELSE NULL
      END AS week_number,
      trafficSource.source AS source,
      ROUND(SUM(product.productRevenue) / 1000000, 4) AS revenue
    FROM
      `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
    WHERE
      product.productRevenue IS NOT NULL
    GROUP BY
      week_number, source, time_type
    ORDER BY source
  )


  SELECT *
FROM (
  SELECT * FROM month_revenue
  UNION ALL
  SELECT * FROM weekly_revenue
)
ORDER BY
  source,
  revenue DESC,
  time_type;

```
![Result](Images/Query3.png)
### Query 04: Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.
```
WITH purchased_pageviews AS (
  SELECT 
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    SUM(totals.pageviews) / COUNT(DISTINCT fullVisitorId) AS avg_pageviews_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits) AS hits,
    UNNEST(hits.product) AS product
  WHERE 
    totals.transactions >= 1
    AND product.productRevenue IS NOT NULL
    AND EXTRACT(MONTH FROM PARSE_DATE('%Y%m%d', date)) IN (6, 7)
    AND EXTRACT(YEAR FROM PARSE_DATE('%Y%m%d', date)) = 2017
  GROUP BY month
),
nonpurchased_pageviews AS (
  SELECT 
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    SUM(totals.pageviews) / COUNT(DISTINCT fullVisitorId) AS avg_pageviews_non_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits) AS hits,
    UNNEST(hits.product) AS product
  WHERE 
    totals.transactions IS NULL
    AND product.productRevenue IS NULL
    AND EXTRACT(MONTH FROM PARSE_DATE('%Y%m%d', date)) IN (6, 7)
    AND EXTRACT(YEAR FROM PARSE_DATE('%Y%m%d', date)) = 2017
  GROUP BY month
)
SELECT 
  p.month AS month,
  p.avg_pageviews_purchase,
  n.avg_pageviews_non_purchase
FROM purchased_pageviews AS p
FULL JOIN nonpurchased_pageviews AS n
  ON p.month = n.month
ORDER BY p.month;
```
![Result](Images/Query4.png)
### Query 05: Average number of transactions per user that purchased in July 2017
```
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
  SUM(totals.transactions) / COUNT(DISTINCT fullVisitorId) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits) AS hits,
  UNNEST(hits.product) AS product
WHERE
    totals.transactions >=1
    and productRevenue is not null
GROUP BY month;
```
![Result](Images/Query5.png)
### Query 06: Average amount of money spent per session. Only include purchaser data in July 2017
```
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
  (SUM(product.productRevenue) / SUM(totals.visits))/1000000 AS avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits) AS hits,
  UNNEST(hits.product) AS product
WHERE
  totals.transactions >= 1
  AND product.productRevenue IS NOT NULL
GROUP BY month;
```
![Result](Images/Query6.png)
### Query 07: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show the product name and the quantity ordered.
```
WITH henley_sessions AS (
  SELECT DISTINCT
    fullVisitorId,
    visitId
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
    UNNEST(hits) AS hits,
    UNNEST(hits.product) AS product
  WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
    AND product.productRevenue IS NOT NULL
    AND totals.transactions >= 1
)
SELECT
  other_products.v2ProductName AS other_purchased_product,
  SUM(other_products.productQuantity) AS quantity
FROM henley_sessions as hs
JOIN `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`as s
  ON hs.fullVisitorId = s.fullVisitorId AND hs.visitId = s.visitId,
  UNNEST(s.hits) AS hits,
  UNNEST(hits.product) AS other_products
WHERE other_products.v2ProductName != "YouTube Men's Vintage Henley"
  AND other_products.productRevenue IS NOT NULL
  AND s.totals.transactions >= 1
GROUP BY other_purchased_product
ORDER BY quantity DESC;

```
![Result](Images/Query7.png)
### "Query 08: Calculate cohort map from product view to addtocart to purchase in Jan, Feb, and March 2017. For example, 100% product view, then 40% add_to_cart, and 10% purchase.
Add_to_cart_rate = number of products added to cart/number of product views. Purchase_rate = number of product purchases/number of  product views. The output should be calculated at the product level."
```
WITH product_view AS(
  SELECT
    format_date("%Y%m", parse_date("%Y%m%d", date)) as month,
    count(product.productSKU) as num_product_view
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
  AND hits.eCommerceAction.action_type = '2'
  GROUP BY 1
),

add_to_cart AS(
  SELECT
    format_date("%Y%m", parse_date("%Y%m%d", date)) AS month,
    count(product.productSKU) as num_addtocart
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
  AND hits.eCommerceAction.action_type = '3'
  GROUP BY 1
),

purchase AS(
  SELECT
    format_date("%Y%m", parse_date("%Y%m%d", date)) AS month,
    count(product.productSKU) as num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
  AND hits.eCommerceAction.action_type = '6'
  AND product.productRevenue IS NOT NULL 
  GROUP BY 1
)

SELECT
    pv.*,
    num_addtocart,
    num_purchase,
    ROUND(num_addtocart*100/num_product_view,2) AS add_to_cart_rate,
    ROUND(num_purchase*100/num_product_view,2) AS purchase_rate
FROM product_view pv
LEFT JOIN add_to_cart a ON pv.month = a.month
LEFT JOIN purchase p ON pv.month = p.month
ORDER BY pv.month;
```
![Result](Images/Query8.png)

