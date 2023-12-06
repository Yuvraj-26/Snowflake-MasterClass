## Time Travel

Snowflake Time Travel enables accessing historical data (i.e. data that has been changed or deleted) at any point within a defined period. It serves as a powerful tool for performing the following tasks:
- Restoring data-related objects (tables, schemas, and databases) that might have been accidentally or intentionally deleted.
- Duplicating and backing up data from key points in the past.
- Analysing data usage/manipulation over specified periods of time.


## Using Time travel
- Enterprise edition or higher - up to 90 days time travel
- Standard edition - up to 1 day time travel

```sql
// Setting up table

CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string)


// create file format
CREATE OR REPLACE FILE FORMAT MANAGE_DB.file_formats.csv_file
    type = csv
    field_delimiter = ','
    skip_header = 1

// create stage
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.time_travel_stage
    URL = 's3://data-snowflake-fundamentals/time-travel/'
    file_format = MANAGE_DB.file_formats.csv_file;



LIST @MANAGE_DB.external_stages.time_travel_stage // 1 csv file customer.csv


// copy file data into table
COPY INTO OUR_FIRST_DB.public.test
from @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv')
// 100 rows LOADED


SELECT * FROM OUR_FIRST_DB.public.test

// Use-case: Update data (by mistake)
// no where clause to filter certain rows

UPDATE OUR_FIRST_DB.public.test
SET FIRST_NAME = 'Joyen' // 1000 rows updated

SELECT * FROM OUR_FIRST_DB.public.test



// // // Using time travel: Method 1 - 2 minutes back
SELECT * FROM OUR_FIRST_DB.public.test at (OFFSET => -60*1.5)








// // // Using time travel: Method 2 - before timestamp
// use timestamp with string converted to timestamp
SELECT * FROM OUR_FIRST_DB.public.test before (timestamp => '2021-04-15 17:47:50.581'::timestamp)


-- Setting up table
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

COPY INTO OUR_FIRST_DB.public.test
from @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv');


SELECT * FROM OUR_FIRST_DB.public.test;


2021-04-17 08:16:24.259
2023-12-07 18:56:56.186 +0000

-- Setting up UTC time for convenience


ALTER SESSION SET TIMEZONE ='UTC'
SELECT DATEADD(DAY, 1, CURRENT_TIMESTAMP)


// update all job to data scientists
UPDATE OUR_FIRST_DB.public.test
SET Job = 'Data Scientist'


SELECT * FROM OUR_FIRST_DB.public.test;

// before timestamp method
SELECT * FROM OUR_FIRST_DB.public.test before (timestamp => '2023-12-07 18:56:56.186'::timestamp)








// // // Using time travel: Method 3 - before Query ID

// Preparing table
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Phone string,
  Job string)

COPY INTO OUR_FIRST_DB.public.test
from @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv')


SELECT * FROM OUR_FIRST_DB.public.test


// Altering table (by mistake)
UPDATE OUR_FIRST_DB.public.test
SET EMAIL = null



SELECT * FROM OUR_FIRST_DB.public.test

// time travel before a query has been executed
// use query history to find query id for the update statement
SELECT * FROM OUR_FIRST_DB.public.test before (statement => '01b0d013-0604-db2c-0002-484300034072')
// table displayed with original emails

```

## Assignment 9 - Time Travel

```sql
-- Switch to role of accountadmin --

USE ROLE ACCOUNTDMIN;
USE DATABASE DEMO_DB;
USE WAREHOUSE COMPUTE_WH;

CREATE OR REPLACE TABLE DEMO_DB.PUBLIC.PART
AS
SELECT * FROM "SNOWFLAKE_SAMPLE_DATA"."TPCH_SF1"."PART";

SELECT * FROM PART
ORDER BY P_MFGR DESC;

-- Update the table
-- Update the Manufacturer#5


UPDATE DEMO_DB.PUBLIC.PART
SET P_MFGR='Manufacturer#CompanyX'
WHERE P_MFGR='Manufacturer#5'; -- 40135 rows updated

----> Note down query id for the update here: 01b0d023-0604-db2b-0002-48430002e0ea

SELECT * FROM PART
ORDER BY P_MFGR DESC;

-- Travel back using the offset until we get the result

// // // Using time travel: Method 3 minutes back
SELECT * FROM PART at (OFFSET => -60*3.0)
ORDER BY P_MFGR DESC;


-- Travel back using the query id to get the result before the update

// // // Using time travel: Method 3 - before Query ID
SELECT * FROM PART before (statement => '01b0d023-0604-db2b-0002-48430002e0ea')


-- Solution
-- Step 3.1: Travel back using the offset until you get the result of before the update

SELECT * FROM PART at (OFFSET => -60*1.5) ORDER BY P_MFGR DESC;

-- Step 3.2: Travel back using the query id to get the result before the update

SELECT * FROM PART before (statement => 'your-query-id');

```


```sql

```


```sql

```
