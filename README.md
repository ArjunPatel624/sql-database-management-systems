# Database Management Systems | SQL

SQL coursework  for **CS/CPE 442 — Database Management Systems I** 

| Assignment | Reports | SQL techniques |
| --- | --- | --- |
| [Homework 1 — Sales aggregation](homework-01/README.md) | Customer extrema, busiest and slowest sales days, purchase frequency, seasonal averages, quarterly maxima | Common table expressions, grouped aggregates, joins that recover transaction details |
| [Homework 2 — Dependent aggregates](homework-02/README.md) | Adjacent-month comparisons, quarter-over-quarter averages, peer averages, product medians | Self-joins, outer joins, exclusion predicates, cumulative counts |

Each folder pairs the assignment PDF and submitted work with a readable SQL walkthrough. The walkthroughs use the course's `sales` table and its restrictions on aggregate functions and window functions.

## Running the queries

Load the shared [course sales dataset](datasets/load_sales_10000.sql) in PostgreSQL, then run each SQL block separately in the same database. The [dataset guide](datasets/README.md) includes setup instructions and a row-count check.
