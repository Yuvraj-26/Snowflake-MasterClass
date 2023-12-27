## Materialized Views

A materialized view is a pre-computed data set derived from a query specification (the SELECT in the view definition) and stored for later use. Because the data is pre-computed, querying a materialized view is faster than executing a query against the base table of the view. This performance difference can be significant when a query is run frequently or is sufficiently complex. As a result, materialized views can speed up expensive aggregation, projection, and selection operations, especially those that run frequently and that run on large data sets.

Materialized views are designed to improve query performance for workloads composed of common, repeated query patterns. However, materializing intermediate results incurs additional costs. As such, before creating any materialized views, you should consider whether the costs are offset by the savings from re-using these results frequently enough.

- Scenario: We have a view that is queried frequently and that it takes a long time to be processed
  - result is bad user experience and more compute consumption due to the cost and time intensive query
- We can create a materialized view to solve this problem
- Use any SELECT statement to create this MV
- Result will be stored in a separate table and this will be updated automatically based on the base table. Combines the speed of a table with a SELECT statement with the updated data from a query expected in a view.
- View is automatically updated by an automated update services managed by Snowflake

## Using Materialized Views


```sql
-- Remove caching just to have a fair test -- Part 1

ALTER SESSION SET USE_CACHED_RESULT=FALSE; -- disable global caching
ALTER warehouse compute_wh suspend;
ALTER warehouse compute_wh resume;



-- Prepare table
CREATE OR REPLACE TRANSIENT DATABASE ORDERS;

CREATE OR REPLACE SCHEMA TPCH_SF100;

// 4.4GB 150 million rows
CREATE OR REPLACE TABLE TPCH_SF100.ORDERS AS
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF100.ORDERS;

SELECT * FROM ORDERS LIMIT 100;



-- Example statement view --
// Example statement we want to query frequently
// Query duration is 8.4s
// Many users querying the exact result often can increase compute costs and can create a bad user experience
SELECT
YEAR(O_ORDERDATE) AS YEAR,
MAX(O_COMMENT) AS MAX_COMMENT,
MIN(O_COMMENT) AS MIN_COMMENT,
MAX(O_CLERK) AS MAX_CLERK,
MIN(O_CLERK) AS MIN_CLERK
FROM ORDERS.TPCH_SF100.ORDERS
GROUP BY YEAR(O_ORDERDATE)
ORDER BY YEAR(O_ORDERDATE);
// returns 7 rows




-- Create materialized view
CREATE OR REPLACE MATERIALIZED VIEW ORDERS_MV
AS
SELECT
YEAR(O_ORDERDATE) AS YEAR,
MAX(O_COMMENT) AS MAX_COMMENT,
MIN(O_COMMENT) AS MIN_COMMENT,
MAX(O_CLERK) AS MAX_CLERK,
MIN(O_CLERK) AS MIN_CLERK
FROM ORDERS.TPCH_SF100.ORDERS
GROUP BY YEAR(O_ORDERDATE);


SHOW MATERIALIZED VIEWS;
// refreshed_on shows timestamp of last refresh
// The REFRESHED_ON and COMPACTED_ON columns show the timestamp of the last DML operation on the base table that was processed by the refresh and compaction operations, respectively.
// behind_by 0s indicates the amount of time that the updates to the materialized view are behind the updates to the base table
// is_secure is false indicating it is not a secure view

-- Query view
// query duration is only 333ms
// Improved performance than querying without materialized view
SELECT * FROM ORDERS_MV
ORDER BY YEAR;



-- UPDATE or DELETE values
// updates in the base table where data originated should be reflected in the materialized view, just like normal SELECT
UPDATE ORDERS
SET O_CLERK='Clerk#99900000'
WHERE O_ORDERDATE='1992-01-01';


//---------------------------------------------------
   -- Test updated data --
-- Example statement view --
SELECT
YEAR(O_ORDERDATE) AS YEAR,
MAX(O_COMMENT) AS MAX_COMMENT,
MIN(O_COMMENT) AS MIN_COMMENT,
MAX(O_CLERK) AS MAX_CLERK,
MIN(O_CLERK) AS MIN_CLERK
FROM ORDERS.TPCH_SF100.ORDERS
GROUP BY YEAR(O_ORDERDATE)
ORDER BY YEAR(O_ORDERDATE);



-- Query view
SELECT * FROM ORDERS_MV
ORDER BY YEAR;


SHOW MATERIALIZED VIEWS;

//---------------------------------------------------

-- Remove caching just to have a fair test -- Part 2

ALTER SESSION SET USE_CACHED_RESULT=FALSE; -- disable global caching
ALTER warehouse compute_wh suspend;
ALTER warehouse compute_wh resume;



-- Prepare table
CREATE OR REPLACE TRANSIENT DATABASE ORDERS;

CREATE OR REPLACE SCHEMA TPCH_SF100;

CREATE OR REPLACE TABLE TPCH_SF100.ORDERS AS
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF100.ORDERS;

SELECT * FROM ORDERS LIMIT 100;



-- Example statement view --
SELECT
YEAR(O_ORDERDATE) AS YEAR,
MAX(O_COMMENT) AS MAX_COMMENT,
MIN(O_COMMENT) AS MIN_COMMENT,
MAX(O_CLERK) AS MAX_CLERK,
MIN(O_CLERK) AS MIN_CLERK
FROM ORDERS.TPCH_SF100.ORDERS
GROUP BY YEAR(O_ORDERDATE)
ORDER BY YEAR(O_ORDERDATE);




-- Create materialized view
CREATE OR REPLACE MATERIALIZED VIEW ORDERS_MV
AS
SELECT
YEAR(O_ORDERDATE) AS YEAR,
MAX(O_COMMENT) AS MAX_COMMENT,
MIN(O_COMMENT) AS MIN_COMMENT,
MAX(O_CLERK) AS MAX_CLERK,
MIN(O_CLERK) AS MIN_CLERK
FROM ORDERS.TPCH_SF100.ORDERS
GROUP BY YEAR(O_ORDERDATE);


SHOW MATERIALIZED VIEWS;

-- Query view
SELECT * FROM ORDERS_MV
ORDER BY YEAR;


// update base table which should be reflected in materialized view

-- UPDATE or DELETE values
UPDATE ORDERS
SET O_CLERK='Clerk#99900000'
WHERE O_ORDERDATE='1992-01-01';



// Updated data reflected in SELECT statement
// Query duration is 8 seconds
   -- Test updated data --
-- Example statement view --
SELECT
YEAR(O_ORDERDATE) AS YEAR,
MAX(O_COMMENT) AS MAX_COMMENT,
MIN(O_COMMENT) AS MIN_COMMENT,
MAX(O_CLERK) AS MAX_CLERK,
MIN(O_CLERK) AS MIN_CLERK
FROM ORDERS.TPCH_SF100.ORDERS
GROUP BY YEAR(O_ORDERDATE)
ORDER BY YEAR(O_ORDERDATE);



-- Query view
// query duration is 673ms
// Changes reflected in materialized view
SELECT * FROM ORDERS_MV
ORDER BY YEAR;


SHOW MATERIALIZED VIEWS;
// behind_by 2m19s
// takes few minutes to refresh in materialized view
// refreshed_on and compacted_on updated and behind_by is 0s
// Execution of materialised view displays new values correctly and performance is very fast at 530ms



select * from table(information_schema.materialized_view_refresh_history());
```


## Maintenance Costs


```sql
HOW MATERIALIZED VIEWS;


// Changing data in a base  underlying table of materialized view, data will be automatically updated in the materialized view
// This is managed by snowflake, virtual warehouse is not used, credits are charged for this additional service
// Use ACCOUNTADMIN, View ACCOUNT and then USAGE
// MATERIALIZED_VIEW_MAINTENANCE Cost

// view refresh history and associated cost / credits used
select * from table(information_schema.materialized_view_refresh_history());
```

## When to use materialized views

- View would take a long time to be processed and is used frequently
- Underlying data is changed not frequently and on a rather irregular basis

If the data is updated on a very regular basis...
- Using task & streams could be a better alternative
- Example: we have an underlying table where a statement on that table that takes a long time to execute/obtain a result that we want to store in a view, create a TASK with MERGE statement to schedule task on a regular basis at a specific time (once a week for example). The data will then be merged into the target table. We can also combine this with stream object on top of source table to track changes and use TASK to automatically merge data into target table as required.
- This is an alternative method that is more cost effective and we have more control over
- Summary:
  - Don't use materialize view if data changes are very frequent and the underlying table has large amounts of data
  - Keep maintenance cost in mind
  - Consider leveraging tasks ($ streams ) instead to build a solution for creating a view as an alternative if a materialized view would have very high costs.

## Limitations
- Only available for Enterprise edition or higher
- Joins (including self-joins) are not supported - if SELECT statement for materialized statement contains join, it cannot be implemented in MV
- Limited amount of aggregation functions
- Cannot Use:
  - UDFs (User defined functions)
  - HAVING clauses
  - ORDER BY clause
  - LIMIT clause


## Assignment 14



```sql
/* Create a materialized view called PARTS in the database DEMO_DB from the following statement:

SELECT
AVG(PS_SUPPLYCOST) as PS_SUPPLYCOST_AVG,
AVG(PS_AVAILQTY) as PS_AVAILQTY_AVG,
MAX(PS_COMMENT) as PS_COMMENT_MAX
FROM"SNOWFLAKE_SAMPLE_DATA"."TPCH_SF100"."PARTSUPP"

Execute the SELECT before creating the materialized view and note down the time until the query is executed.

Questions for this assignment
How long did the SELECT statement take initially?
// 8 seconds

How long did the execution of the materialized view take?
// Less than 1 second
**/

USE ROLE ACCOUNTADMIN;
USE DEMO_DB;

-- Example statement view --
SELECT
AVG(PS_SUPPLYCOST) as PS_SUPPLYCOST_AVG,
AVG(PS_AVAILQTY) as PS_AVAILQTY_AVG,
MAX(PS_COMMENT) as PS_COMMENT_MAX
FROM"SNOWFLAKE_SAMPLE_DATA"."TPCH_SF100"."PARTSUPP"

// query duration: 8.1 seconds


-- Create materialized view
CREATE OR REPLACE MATERIALIZED VIEW PARTS
AS
SELECT
AVG(PS_SUPPLYCOST) as PS_SUPPLYCOST_AVG,
AVG(PS_AVAILQTY) as PS_AVAILQTY_AVG,
MAX(PS_COMMENT) as PS_COMMENT_MAX
FROM"SNOWFLAKE_SAMPLE_DATA"."TPCH_SF100"."PARTSUPP";

// behind_by 0s
SHOW MATERIALIZED VIEWS;

-- MV Query view
// query duration is 212 ms
SELECT * FROM PARTS

```
