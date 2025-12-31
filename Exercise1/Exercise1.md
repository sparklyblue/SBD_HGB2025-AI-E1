# Assignment 1

*Lea Treml*

## Exercise 1

```shell-console
DROP TABLE IF EXISTS orders_big;

CREATE TABLE orders_big (
  id SERIAL PRIMARY KEY,
  customer_name TEXT,
  product_category TEXT,
  quantity INT,
  price_per_category FLOAT,
  order_date DATE,
  country TEXT
);

\COPY orders_big(customer_name,product_category,quantity,price_per_category,order_date,country)
FROM '/ecommerce/orders_1M.csv' DELIMITER ',' CSV HEADER;

A. What is the single item with the highest price_per_category?

Select * FROM orders_big ORDER BY price_per_category desc LIMIT 1;
   id   | customer_name | product_category | quantity | price_per_category | order_date | country 
--------+---------------+------------------+----------+---------------+------------+---------
 841292 | Emma Brown    | Automotive       |        3 |          2000 | 2024-10-11 | Italy
(1 row)

B. What are the top 3 products with the highest total quantity sold across all orders?

Select product_category, sum(quantity) AS quantity FROM orders_big GROUP BY product_category ORDER BY sum(quantity) desc LIMIT 3;
 product_category | quantity 
------------------+----------
 Health & Beauty  |   300842
 Electronics      |   300804
 Toys             |   300598
(3 rows)

C. What is the total revenue per product category?
(Revenue = price_per_unit × quantity)

SELECT product_category, SUM(revenue) FROM (SELECT product_category, (quantity * price_per_category) as revenue FROM orders_big) sub GROUP BY product_category;
product_category |        sum
------------------+--------------------
 Automotive       |  306589798.8600006
 Books            | 12731976.040000025
 Electronics      |       241525009.45
 Fashion          | 31566368.220000055
 Grocery          |  15268355.66000006
 Health & Beauty  |  46599817.89000019
 Home & Garden    |  78023780.09000021
 Office Supplies  | 38276061.639999785
 Sports           |  61848990.83000006
 Toys             |  23271039.02000004

D. Which customers have the highest total spending?

SELECT customer_name, SUM(revenue) FROM (SELECT customer_name, (quantity * price_per_category) as revenue FROM orders_big) sub GROUP BY customer_name ORDER BY SUM(revenue) desc LIMIT 10;
 customer_name  |        sum        
----------------+-------------------
 Carol Taylor   | 991179.1800000002
 Nina Lopez     | 975444.9500000002
 Daniel Jackson | 959344.4800000003
 Carol Lewis    | 947708.5700000001
 Daniel Young   | 946030.1400000005
 Alice Martinez |         935100.02
 Ethan Perez    | 934841.2400000002
 Leo Lee        | 934796.4799999997
 Eve Young      | 933176.8600000003
 Ivy Rodriguez  | 925742.6400000004

```

## Exercise 2
*Why this query is bad to begin with*
- self-join on low cardinality attribute 
- n rows -> n^2 join combinations
- super large table for 1 million rows
- OLTP optimized for smaller, indexed lookups - not massive joins like this

### Performance Improvements:

#### Query Rewriting: 
The join isn't necessary if the goal is to count pairs per country 

SELECT SUM(count * count)
FROM (
  SELECT country, COUNT(*) AS count
  FROM people_big
  GROUP BY country
) t;

This reduces complexity from the O(n^2) to O(n) -> wayyy faster
It uses aggregation instead of multiplying rows -> so it reduces usage of memory and CPU

#### Indexing
If joins and/or grouping is not avoidable indexing speeds it up by a lot. (This doesn't fix issues alone though.)

Example: 
CREATE INDEX idx_people_country ON people_big(country);

#### Pre-Aggregation / Views
If this is necessary and is frequently needed we can compute a view to begin with. Then it will only need to be doen once, queries become easier and it's better for quick analysis.

Like this: 
CREATE MATERIALIZED VIEW country_counts AS
SELECT country, COUNT(*) AS count
FROM people_big
GROUP BY country;

### Scalability Improvements
#### Seperate OLTP & OLAP Workloads
*Problem*
- OLTP DB is handling an analytical query -> slow transactions
*Solution*
- Keep OLTP for the transactions only
- Move the analysis to something else like DB-Warehouse

### Overall Efficienccy Improvements
#### Data Modeling Improvements
maintaining a summary table for country and the population count would make this problem go away - could be seperate view

#### Query management
- Making sure that queries can't run longer then X amount of minutes
- Limit Join result sizes
- Educating users on how to make safe queries before letting them loose on the DB

## Exercise 3
**What the Spark code does:**  
- It creates a session with JDBC
- Loads the people_big table from PostgreSQL into the Spark dataframe 
- df_big.count triggers data loading
- makes temporary sql view for dataframe to allow queries
- Query A:
  - computes avg salary per department 
  - using groupby, agg, orderby and limit
  - uses grouped aggreagation with sorting and only showing top selection
- Query B: 
  - computes avg deparment salary per country & then avg by country
  - uses nested aggregation to do this
- Query C:
  - retrieves the top 10 highest salaries 
  - ordering by column & limit result
- Query D:
  - self-join on country & counts all resulting rows
  - desc in ex2 why isn'tt good way to do so
- Query D - the safe one:
  - uses groupyby, count and then sum of square to do the same
  - way faster (3 sec vs. 20 sec)

- **Architectural contrasts with PostgreSQL:**  
  - *Architecture*
    - Spark: distributed, cluster-based, in-memory processing
    - PostgreSQL: single node, can only handle parallel queries on multi-core CPUs
  - *Scalability*
    - Spark: horizontal scaling 
    - PostgreSQL: vertical scaling
  - *Parallelism*
    - Spark: parallel across partitions
    - PostgreSQL: limited by CPU cores
  - *Data model*
    - Spark: schema-on-read
    - PostgreSQL: schema-on-write

- **Advantages and limitations:**  
  - *Advantages*
    - Handles large dbs across mult. nodes
    - faster than disk-based for iterative workloads
    - supports SWL, dataframa api & ML pipelines
  - *Limitations*
    - requires cluster setup & configuration for production - more complex
    - for smaller tables local postgreSQL may be better
    - needs sufficient memory and CPU - resource-heavy

- **Relation to Exercise 2:**  
  - shows difference between bad query and good query in QUERY D case

## Exercise 4
The file can be found in /data/Ex04.ipynb