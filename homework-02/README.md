# Homework 2 — Dependent aggregates

[Assignment PDF](assignment.pdf) 

These reports compare each group's sales with aggregates from neighboring periods or peer groups. The median report uses cumulative counts to locate the middle observation under the assignment's odd-count assumption.

## Data setup

Use the [same 10,000-row sales dataset](../datasets/load_sales_10000.sql) as Homework 1. If you already ran the loader for Homework 1, reuse that database. Otherwise, follow the [PostgreSQL setup steps](../datasets/README.md). These queries use the transaction quantities to compare neighboring periods, calculate peer averages, and find product medians.

## 1. Sales between adjacent months' averages

Compute average sales by product and month across years. Join to the two adjacent calendar months, then count transactions between the two averages, including both endpoints. Product–month pairs with both averages and no qualifying transactions appear with a count of zero. January, December, and any other month missing a neighboring average are omitted.

```sql
WITH monthly_averages AS (
    SELECT prod, month, AVG(quant) AS avg_quantity
    FROM sales
    GROUP BY prod, month
),
neighbor_averages AS (
    SELECT
        current_month.prod,
        current_month.month,
        previous_month.avg_quantity AS previous_avg,
        following_month.avg_quantity AS following_avg
    FROM monthly_averages AS current_month
    JOIN monthly_averages AS previous_month
        ON previous_month.prod = current_month.prod
        AND previous_month.month = current_month.month - 1
    JOIN monthly_averages AS following_month
        ON following_month.prod = current_month.prod
        AND following_month.month = current_month.month + 1
    WHERE previous_month.avg_quantity IS NOT NULL
        AND following_month.avg_quantity IS NOT NULL
)
SELECT
    neighbors.prod AS product,
    neighbors.month,
    COUNT(s.quant) AS sales_count_between_avgs
FROM neighbor_averages AS neighbors
LEFT JOIN sales AS s
    ON s.prod = neighbors.prod
    AND s.month = neighbors.month
    AND (
        s.quant BETWEEN neighbors.previous_avg AND neighbors.following_avg
        OR s.quant BETWEEN neighbors.following_avg AND neighbors.previous_avg
    )
GROUP BY neighbors.prod, neighbors.month
ORDER BY product, neighbors.month;
```

## 2. Average sales before, during, and after each quarter

Compute one average per customer, product, and calendar quarter across years. Left self-joins retrieve the previous and following quarter averages and preserve `NULL` at the calendar boundaries or where a neighboring quarter has no sales.

```sql
WITH quarterly_sales AS (
    SELECT cust, prod, CEILING(month / 3.0) AS quarter, quant
    FROM sales
),
quarterly_averages AS (
    SELECT cust, prod, quarter, AVG(quant) AS avg_quantity
    FROM quarterly_sales
    GROUP BY cust, prod, quarter
)
SELECT
    current_quarter.cust AS customer,
    current_quarter.prod AS product,
    current_quarter.quarter,
    previous_quarter.avg_quantity AS before_avg,
    current_quarter.avg_quantity AS during_avg,
    following_quarter.avg_quantity AS after_avg
FROM quarterly_averages AS current_quarter
LEFT JOIN quarterly_averages AS previous_quarter
    ON previous_quarter.cust = current_quarter.cust
    AND previous_quarter.prod = current_quarter.prod
    AND previous_quarter.quarter = current_quarter.quarter - 1
LEFT JOIN quarterly_averages AS following_quarter
    ON following_quarter.cust = current_quarter.cust
    AND following_quarter.prod = current_quarter.prod
    AND following_quarter.quarter = current_quarter.quarter + 1
ORDER BY customer, product, current_quarter.quarter;
```

## 3. Average sales for the group and its peers

For each customer–product–state group, compare its average with sales of the same product to other customers and sales of other products to the same customer. Both comparisons stay within the same state and average individual transactions. A peer average is `NULL` when the peer group has no sales.

```sql
WITH group_averages AS (
    SELECT cust, prod, state, AVG(quant) AS product_avg
    FROM sales
    GROUP BY cust, prod, state
),
other_customer_averages AS (
    SELECT g.cust, g.prod, g.state, AVG(s.quant) AS other_customer_avg
    FROM group_averages AS g
    JOIN sales AS s
        ON s.prod = g.prod
        AND s.state = g.state
        AND s.cust <> g.cust
    GROUP BY g.cust, g.prod, g.state
),
other_product_averages AS (
    SELECT g.cust, g.prod, g.state, AVG(s.quant) AS other_product_avg
    FROM group_averages AS g
    JOIN sales AS s
        ON s.cust = g.cust
        AND s.state = g.state
        AND s.prod <> g.prod
    GROUP BY g.cust, g.prod, g.state
)
SELECT
    g.cust AS customer,
    g.prod AS product,
    g.state,
    g.product_avg,
    customers.other_customer_avg,
    products.other_product_avg
FROM group_averages AS g
LEFT JOIN other_customer_averages AS customers
    ON customers.cust = g.cust
    AND customers.prod = g.prod
    AND customers.state = g.state
LEFT JOIN other_product_averages AS products
    ON products.cust = g.cust
    AND products.prod = g.prod
    AND products.state = g.state
ORDER BY customer, product, g.state;
```

## 4. Median sales quantity by product

Count all transactions at or below each candidate quantity, including repeated quantities. For an odd number of sales, the median is the smallest candidate whose cumulative count reaches the middle position, `(sale_count + 1) / 2`. Comparing twice the cumulative count to `sale_count + 1` avoids integer-division rounding.

The supplied dataset contains four products with even transaction counts. For those products, this query returns the upper middle quantity. The assignment assumes odd counts.

```sql
WITH candidate_quantities AS (
    SELECT DISTINCT prod, quant
    FROM sales
    WHERE quant IS NOT NULL
),
cumulative_counts AS (
    SELECT candidate.prod, candidate.quant, COUNT(*) AS cumulative_count
    FROM candidate_quantities AS candidate
    JOIN sales AS s
        ON s.prod = candidate.prod
        AND s.quant <= candidate.quant
    GROUP BY candidate.prod, candidate.quant
),
product_counts AS (
    SELECT prod, COUNT(quant) AS sale_count
    FROM sales
    GROUP BY prod
)
SELECT
    counts.prod AS product,
    MIN(counts.quant) AS median_quantity
FROM cumulative_counts AS counts
JOIN product_counts AS totals ON totals.prod = counts.prod
WHERE 2 * counts.cumulative_count >= totals.sale_count + 1
GROUP BY counts.prod
ORDER BY product;
```
