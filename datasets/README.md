# Sales data for Homework 1 and 2

[load_sales_10000.sql](load_sales_10000.sql) creates the `sales` table and loads the 10,000 transactions provided with the course.

| Columns | Meaning |
| --- | --- |
| `cust`, `prod` | Customer and product |
| `day`, `month`, `year`, `date` | Transaction date and its calendar components |
| `state` | Two-letter state abbreviation |
| `quant` | Units sold in the transaction |

In pgAdmin, open the **Query Tool** for a fresh PostgreSQL database, open the script, and execute it once. Then verify the load:

```sql
SELECT COUNT(*) FROM sales;
```

The result should be **10,000**. Run each homework query separately in that same database.
