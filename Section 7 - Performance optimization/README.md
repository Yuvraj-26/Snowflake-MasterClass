## Performance Optimization

Optimization of the performance of data warehouse

Goal: Make queries run faster, which save costs (compute power usage)

Traditional way:
- Add indexes, primary keys
- Create table partitions
- Analyse the query execution table plan
- Remove unnecessary full table scans

In Snowflake:
- Automatically managed micro-partitions - ensures performance when running queries

### Our job:
- Assign appropriate data types
- sizing virtual warehouses
- cluster keys for large tables

#### Dedicated virtual warehouses
- separated according to different workloads
- create a dedicated warehouse for certain user groups that are larger with different needs and queries

#### Scaling Up
- For known patterns of high work load, increase size of the data warehouse

#### Scaling Out
- Introducing multi cluster warehouses
- Dynamically, depending on the workload, scale our multi cluster warehouse out, and automatically create new warehouse clusters
- Dynamically for unknown patterns of work loads (different amount of users and workload)

#### Maximise Cache usage
- Automatic caching can be maximised

#### Cluster keys
- For large tables


## Create dedicated virtual warehouse
- Central data warehouse connected to different user groups such as BI team (ETL/ELT, Talend, Informatica), Data science team (python, SparkSQL), Marketing team (tableau)
- dedicated virtual warehouses are only available to the specific user groups
- dedicated virtual warehouses to different workloads and different tasks
- Each dedicated warehouse can be allocated varied compute / processing power


Identify and Classify
- Identify and classify groups of workloads/users
- dedicated server for department across the organisation and location. BI Team, Data Science Team, Marketing department

Create dedicated virtual warehouses
- For every class of workload and assign users

Considerations
- Not too many VW
  - too fine granularity or creating tens of VW for small user groups means VW will be started and suspend often and not utilised, even with auto-resume or auto-suspend options, there will be time where VW will not be used incurring costs
  - must avoid underutilization
- Refine classifications
  - work patterns can change so refined classifications based on these times

## Implement Dedicated Virtual Warehouse

```sql
//  Create virtual warehouse for data scientist & DBA

// Identify user groups - Data Scientists and DB Administrators

// Data Scientists
CREATE WAREHOUSE DS_WH
WITH WAREHOUSE_SIZE = 'SMALL'
WAREHOUSE_TYPE = 'STANDARD'
AUTO_SUSPEND = 300
AUTO_RESUME = TRUE
MIN_CLUSTER_COUNT = 1
MAX_CLUSTER_COUNT = 1
SCALING_POLICY = 'STANDARD';

// DBA
CREATE WAREHOUSE DBA_WH
WITH WAREHOUSE_SIZE = 'XSMALL'
WAREHOUSE_TYPE = 'STANDARD'
AUTO_SUSPEND = 300
AUTO_RESUME = TRUE
MIN_CLUSTER_COUNT = 1
MAX_CLUSTER_COUNT = 1
SCALING_POLICY = 'STANDARD';



// Create role for Data Scientists & DBAs
// Using ACCOUNTADMIN role

CREATE ROLE DATA_SCIENTIST;
GRANT USAGE ON WAREHOUSE DS_WH TO ROLE DATA_SCIENTIST;

CREATE ROLE DBA;
GRANT USAGE ON WAREHOUSE DBA_WH TO ROLE DBA;


// Setting up users with roles

// Data Scientists
CREATE USER DS1 PASSWORD = 'DS1' LOGIN_NAME = 'DS1' DEFAULT_ROLE='DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH'  MUST_CHANGE_PASSWORD = FALSE;
CREATE USER DS2 PASSWORD = 'DS2' LOGIN_NAME = 'DS2' DEFAULT_ROLE='DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH'  MUST_CHANGE_PASSWORD = FALSE;
CREATE USER DS3 PASSWORD = 'DS3' LOGIN_NAME = 'DS3' DEFAULT_ROLE='DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH'  MUST_CHANGE_PASSWORD = FALSE;

// Grant role to user
GRANT ROLE DATA_SCIENTIST TO USER DS1;
GRANT ROLE DATA_SCIENTIST TO USER DS2;
GRANT ROLE DATA_SCIENTIST TO USER DS3;

// DBAs
CREATE USER DBA1 PASSWORD = 'DBA1' LOGIN_NAME = 'DBA1' DEFAULT_ROLE='DBA' DEFAULT_WAREHOUSE = 'DBA_WH'  MUST_CHANGE_PASSWORD = FALSE;
CREATE USER DBA2 PASSWORD = 'DBA2' LOGIN_NAME = 'DBA2' DEFAULT_ROLE='DBA' DEFAULT_WAREHOUSE = 'DBA_WH'  MUST_CHANGE_PASSWORD = FALSE;

// Grant role to user
GRANT ROLE DBA TO USER DBA1;
GRANT ROLE DBA TO USER DBA2;

// Log in using credentials
// User will have DATA_SCIENTIST role and PUBLIC role
// Only has access to DS_WH

// Drop objects again
// Drop users, roles and warehouses

DROP USER DBA1;
DROP USER DBA2;

DROP USER DS1;
DROP USER DS2;
DROP USER DS3;

DROP ROLE DATA_SCIENTIST;
DROP ROLE DBA;

DROP WAREHOUSE DS_WH;
DROP WAREHOUSE DBA_WH;
```

## Scaling Up/Down
- Changing the size of the virtual warehouse depending on different work loads in different periods
- Use cases
  - ETL runs at certain times (for example between 4pm and 8pm), increase the size of the VW
  - Special business event with more workload (more complex queries), increase the size of the VW
- Note: Common scenario is increased query complexity - NOT more users (then scaling out would be better)

Configure warehouse and change size in interface, or using code:

```sql
ALTER WAREHOUSE COMPUTE_WH
SET WAREHOUSE_SIZE = 'XSMALL';

```

## Scaling out

Scaling Up: increasing the size of virtual warehouses - more complex query

Scaling Out: Using additional warehouses / Multi-Cluster warehouses - more concurrent users / queries


- handling performance related to large number of concurrent clusters
- automation of the process if you have fluctuating number of users

Multi-Cluster - automated process, if we have more queries than that can be processed by a single cluster, automatically new instances of clusters can be created. Auto-Scaling

Considerations:
- If you use at least Enterprise Edition, all warehouses should be Multi-cluster
- Minimum: Default should be 1
- Maximum: Can be very high (minimal difference in cost / credit consumption between one VW that takes 3 hours to process or 3 VW that need one hour each to process)



## Scaling Out Simulation

```sql

// Using table WEBSITE as an example
// CROSS JOIN generates query that simulates queries across worksheets
// Execute query in several worksheets

SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF100TCL.WEB_SITE T1
CROSS JOIN SNOWFLAKE_SAMPLE_DATA.TPCDS_SF100TCL.WEB_SITE T2
CROSS JOIN SNOWFLAKE_SAMPLE_DATA.TPCDS_SF100TCL.WEB_SITE T3
CROSS JOIN (SELECT TOP 57 * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF100TCL.WEB_SITE)  T4


// COMPUTE_WH shows 2 active clusters out of 3, automatically started, to handle parallel query processing of several worksheets
```

## Caching

- Automatic process to speed up the queries
- If a query is executed twice, results are cached and can be re-used
- Results are cached for 24 hours or until underlying data has changed (Snowflake understands metadata to determine when cache cannot be used and query has to be re-executed )

Our job:
- Ensure that similar queries go on the same warehouses
- Example: Team of Data scientists run similar queries, so they should all use the same warehouse to maximise benefits of caching

## Maximising Caching

```sql
// SELECT AVERAGE from SNOWFLAKE SAMPLE DATA.SCHEMA.CUSTOMER TABLE
SELECT AVG(C_BIRTH_YEAR) FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF100TCL.CUSTOMER

// View query duration (compilation and execution time) - 3 seconds
// Select query ID and view query execution plan
// Query profile shows TableScan, aggregate, then result
// Run the query again - 1 second
// Query duration is less, query is run quicker
// Query profile shows Query Result Reuse
// Result used from cache


// Setting up an additional user
CREATE ROLE DATA_SCIENTIST;
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE DATA_SCIENTIST;

CREATE USER DS1 PASSWORD = 'DS1' LOGIN_NAME = 'DS1' DEFAULT_ROLE='DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH'  MUST_CHANGE_PASSWORD = FALSE;
GRANT ROLE DATA_SCIENTIST TO USER DS1;

// Log in with new user
// User has access to SNOWFLAKE SAMPLE DATA DB
// Running the same query - 1 second
// Query profile shows Query Result Reuse
// Result used from cache, showing benefits of caching for users with similar queries
```

## Clustering in Snowflake
Improves performance

### Cluster keys
- Snowflake creates cluster key on columns to create subset of rows to locate the data in micro-partitions
- For large tables this improves the scan efficiency in our queries
- Snowflake automatically maintains these cluster keys
- In general Snowflake produces well-clustered tables
- Cluster keys are not always ideal and can change over time
- Manually customize these cluster keys

When to cluster?
- Clustering is not for all tables
- Mainly very large tables of multiple terabytes can benefit

How to cluster?
- Columns that are used most frequently in WHERE-clauses (often data columns for event tables)
- If you typically use filters on two columns then the table can also benefit from two cluster keys
- Columns that are frequently used in joins
- Large enough number of distinct values to enable effective grouping. Small enough number of distinct values to allow effective grouping.

## Clustering in Snowflake Practical


```sql
// Publicly accessible staging area    

CREATE OR REPLACE STAGE MANAGE_DB.external_stages.aws_stage
    url='s3://bucketsnowflakes3';

// List files in stage

LIST @MANAGE_DB.external_stages.aws_stage; // 3 csv files

//Load data using copy command

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    FROM @MANAGE_DB.external_stages.aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*OrderDetails.*';


// Create table

CREATE OR REPLACE TABLE ORDERS_CACHING (
ORDER_ID	VARCHAR(30)
,AMOUNT	NUMBER(38,0)
,PROFIT	NUMBER(38,0)
,QUANTITY	NUMBER(38,0)
,CATEGORY	VARCHAR(30)
,SUBCATEGORY	VARCHAR(30)
,DATE DATE)    



INSERT INTO ORDERS_CACHING
SELECT
t1.ORDER_ID
,t1.AMOUNT
,t1.PROFIT
,t1.QUANTITY
,t1.CATEGORY
,t1.SUBCATEGORY
,DATE(UNIFORM(1500000000,1700000000,(RANDOM())))
FROM ORDERS t1
CROSS JOIN (SELECT * FROM ORDERS) t2
CROSS JOIN (SELECT TOP 100 * FROM ORDERS) t3
// 225000000 rows inserted


// Query Performance before Cluster Key

SELECT * FROM ORDERS_CACHING  WHERE DATE = '2020-06-09'#
// Query duration 911ms
// 60% of total execution time is used by TableScan
// 59 out of 59 partitions scanned
// Scanned 100% 1.1GB

// Improve performance using Cluster Key

// Adding Cluster Key & Compare the result

ALTER TABLE ORDERS_CACHING CLUSTER BY ( DATE )

SELECT * FROM ORDERS_CACHING  WHERE DATE = '2020-01-05'
// Initial query run
// Still 60% of total execution time is used by TableScan as
// Snowflake takes time to implement cluster key

// 45 mins later
// Query run is faster
// 3 partitions scanned of 74 total
// 0 % of total execution time is used by TableScan



// Not ideal clustering & adding a different Cluster Key using function

// Filtering using expression on month
// Query duration 5.1 seconds - not ideal
SELECT * FROM ORDERS_CACHING  WHERE MONTH(DATE)=11

// ALTER by using month expression as cluster key
// Once Snowflake has implemented clustering, performance is improved
ALTER TABLE ORDERS_CACHING CLUSTER BY ( MONTH(DATE) )
```
