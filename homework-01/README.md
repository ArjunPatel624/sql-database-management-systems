# Homework 1 — Sales aggregation

[Assignment PDF](assignment.pdf) 

These five reports use grouped aggregates and join the results back to the detail rows when a report needs the product, date, or state associated with an extreme value. The walkthrough keeps that structure and uses descriptive CTE names and explicit joins.

## Data setup

The [sales loader](../datasets/load_sales_10000.sql) creates the `sales` table and loads 10,000 transactions. Load it once using the [PostgreSQL setup steps](../datasets/README.md). These reports aggregate those transactions by customer, product, calendar day, season, and quarter.

## 1. Minimum, maximum, and average sale by customer

Aggregate sales by customer, then join twice to recover the transactions at each extreme. Every tied minimum is paired with every tied maximum for the same customer.

```sql
WITH customer_totals AS (
    SELECT
        cust,
        MIN(quant) AS min_quantity,
        MAX(quant) AS max_quantity,
        ROUND(AVG(quant)) AS avg_quantity
    FROM sales
    GROUP BY cust
)
SELECT
    totals.cust AS customer,
    totals.min_quantity,
    low_sale.prod AS min_product,
    low_sale.date AS min_date,
    low_sale.state AS min_state,
    totals.max_quantity,
    high_sale.prod AS max_product,
    high_sale.date AS max_date,
    high_sale.state AS max_state,
    totals.avg_quantity
FROM customer_totals AS totals
JOIN sales AS low_sale
    ON low_sale.cust = totals.cust
    AND low_sale.quant = totals.min_quantity
JOIN sales AS high_sale
    ON high_sale.cust = totals.cust
    AND high_sale.quant = totals.max_quantity
ORDER BY customer, min_date, max_date;
```

## 2. Busiest and slowest sales days

Sum quantities by calendar day, then find each month's maximum and minimum daily totals. Joining on year, month, and total recovers the matching days, including ties.

```sql
WITH daily_totals AS (
    SELECT year, month, day, SUM(quant) AS total_quantity
    FROM sales
    GROUP BY year, month, day
),
monthly_extremes AS (
    SELECT
        year,
        month,
        MAX(total_quantity) AS busiest_total_quantity,
        MIN(total_quantity) AS slowest_total_quantity
    FROM daily_totals
    GROUP BY year, month
)
SELECT
    monthly.year,
    monthly.month,
    busiest.day AS busiest_day,
    monthly.busiest_total_quantity,
    slowest.day AS slowest_day,
    monthly.slowest_total_quantity
FROM monthly_extremes AS monthly
JOIN daily_totals AS busiest
    ON busiest.year = monthly.year
    AND busiest.month = monthly.month
    AND busiest.total_quantity = monthly.busiest_total_quantity
JOIN daily_totals AS slowest
    ON slowest.year = monthly.year
    AND slowest.month = monthly.month
    AND slowest.total_quantity = monthly.slowest_total_quantity
ORDER BY monthly.year, monthly.month, busiest_day, slowest_day;
```

## 3. Most- and least-purchased products

Count transactions per customer and product. The highest and lowest counts determine the favorites among products that customer purchased.

```sql
WITH purchase_counts AS (
    SELECT cust, prod, COUNT(*) AS purchase_count
    FROM sales
    GROUP BY cust, prod
),
customer_extremes AS (
    SELECT
        cust,
        MAX(purchase_count) AS most_purchases,
        MIN(purchase_count) AS fewest_purchases
    FROM purchase_counts
    GROUP BY cust
)
SELECT
    extremes.cust AS customer,
    most_purchased.prod AS most_favorite_product,
    least_purchased.prod AS least_favorite_product
FROM customer_extremes AS extremes
JOIN purchase_counts AS most_purchased
    ON most_purchased.cust = extremes.cust
    AND most_purchased.purchase_count = extremes.most_purchases
JOIN purchase_counts AS least_purchased
    ON least_purchased.cust = extremes.cust
    AND least_purchased.purchase_count = extremes.fewest_purchases
ORDER BY customer, most_favorite_product, least_favorite_product;
```

## 4. Seasonal and overall sales statistics

Aggregate each season across years. Use overall customer–product totals as the starting point so a season with no transactions appears as `NULL`.

```sql
WITH spring_sales AS (
    SELECT cust, prod, ROUND(AVG(quant)) AS avg_quantity
    FROM sales
    WHERE month IN (3, 4, 5)
    GROUP BY cust, prod
),
summer_sales AS (
    SELECT cust, prod, ROUND(AVG(quant)) AS avg_quantity
    FROM sales
    WHERE month IN (6, 7, 8)
    GROUP BY cust, prod
),
fall_sales AS (
    SELECT cust, prod, ROUND(AVG(quant)) AS avg_quantity
    FROM sales
    WHERE month IN (9, 10, 11)
    GROUP BY cust, prod
),
winter_sales AS (
    SELECT cust, prod, ROUND(AVG(quant)) AS avg_quantity
    FROM sales
    WHERE month IN (12, 1, 2)
    GROUP BY cust, prod
),
overall_sales AS (
    SELECT
        cust,
        prod,
        ROUND(AVG(quant)) AS avg_quantity,
        SUM(quant) AS total_quantity,
        COUNT(*) AS transaction_count
    FROM sales
    GROUP BY cust, prod
)
SELECT
    overall.cust AS customer,
    overall.prod AS product,
    spring.avg_quantity AS spring_avg,
    summer.avg_quantity AS summer_avg,
    fall.avg_quantity AS fall_avg,
    winter.avg_quantity AS winter_avg,
    overall.avg_quantity AS overall_avg,
    overall.total_quantity,
    overall.transaction_count
FROM overall_sales AS overall
LEFT JOIN spring_sales AS spring
    ON spring.cust = overall.cust AND spring.prod = overall.prod
LEFT JOIN summer_sales AS summer
    ON summer.cust = overall.cust AND summer.prod = overall.prod
LEFT JOIN fall_sales AS fall
    ON fall.cust = overall.cust AND fall.prod = overall.prod
LEFT JOIN winter_sales AS winter
    ON winter.cust = overall.cust AND winter.prod = overall.prod
ORDER BY customer, product;
```

## 5. Quarterly maximum quantities and dates

Group transactions by product and calendar quarter across years. Recover each matching date within the same quarter, then place the four quarter results side by side. A missing quarter appears as `NULL`, and ties across quarters produce all combinations of matching dates.

```sql
WITH quarterly_sales AS (
    SELECT prod, CEILING(month / 3.0) AS quarter, quant, date
    FROM sales
),
quarterly_maxima AS (
    SELECT prod, quarter, MAX(quant) AS max_quantity
    FROM quarterly_sales
    GROUP BY prod, quarter
),
maximum_dates AS (
    SELECT DISTINCT maxima.prod, maxima.quarter, maxima.max_quantity, s.date
    FROM quarterly_maxima AS maxima
    JOIN quarterly_sales AS s
        ON s.prod = maxima.prod
        AND s.quarter = maxima.quarter
        AND s.quant = maxima.max_quantity
),
products AS (
    SELECT DISTINCT prod
    FROM sales
)
SELECT
    products.prod AS product,
    q1.max_quantity AS q1_max,
    q1.date AS q1_max_date,
    q2.max_quantity AS q2_max,
    q2.date AS q2_max_date,
    q3.max_quantity AS q3_max,
    q3.date AS q3_max_date,
    q4.max_quantity AS q4_max,
    q4.date AS q4_max_date
FROM products
LEFT JOIN maximum_dates AS q1 ON q1.prod = products.prod AND q1.quarter = 1
LEFT JOIN maximum_dates AS q2 ON q2.prod = products.prod AND q2.quarter = 2
LEFT JOIN maximum_dates AS q3 ON q3.prod = products.prod AND q3.quarter = 3
LEFT JOIN maximum_dates AS q4 ON q4.prod = products.prod AND q4.quarter = 4
ORDER BY product, q1_max_date, q2_max_date, q3_max_date, q4_max_date;
```
