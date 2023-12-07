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


## Restoring Data



```sql
// Setting up table

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

// Use-case: Update data (by mistake)


UPDATE OUR_FIRST_DB.public.test
SET LAST_NAME = 'Tyson';


UPDATE OUR_FIRST_DB.public.test
SET JOB = 'Data Analyst';

SELECT * FROM OUR_FIRST_DB.public.test; // all last names and all job updated

// Use time travel query ID method
SELECT * FROM OUR_FIRST_DB.public.test before (statement => '01b0d5aa-0604-db2d-0002-4843000360ae')
// job has been reverted



// // // Bad method

// recreate table using select time travel statement
// recreate table using query from second update mistake
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test as
SELECT * FROM OUR_FIRST_DB.public.test before (statement => '01b0d5aa-0604-db2d-0002-4843000360ae')
// result is jobs reverted but name not yet reverted as expected

SELECT * FROM OUR_FIRST_DB.public.test

// now use time travel to recreate table using query from first update mistake
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test as
SELECT * FROM OUR_FIRST_DB.public.test before (statement => '01b0d5aa-0604-db2e-0002-4843000310e2')

// Time travel data is not available for table TEST. The requested time is either beyond the allowed time travel period or before the object creation time.
// Recreating the table means we have dropped the table meaning the metadata stored to use time travel does not work due to being a new table with new table id


// // // Good method

// create a backup restore table
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test_backup as
SELECT * FROM OUR_FIRST_DB.public.test before (statement => '01b0d5be-0604-dc51-0002-484300039022') // run select statement first to test

// truncate initial table - still have time travel available
TRUNCATE OUR_FIRST_DB.public.test

// insert data backup restore table into initial table, so initial table is not dropped or recreated to avoid losing time travel capabilities
// this means we have the same table id, same metadata for time travel history
INSERT INTO OUR_FIRST_DB.public.test
SELECT * FROM OUR_FIRST_DB.public.test_backup

// Now we can time travel even further back in time as initial table time travel capabilities is kept
// use second mistake update query id
// 01b0d5be-0604-db2e-0002-484300031106

// create a backup restore table
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.test_backup as
SELECT * FROM OUR_FIRST_DB.public.test before (statement => '01b0d5be-0604-db2e-0002-484300031106') // run select statement first to test

// truncate initial table - still have time travel available
TRUNCATE OUR_FIRST_DB.public.test

// insert data backup restore table into initial table
INSERT INTO OUR_FIRST_DB.public.test
SELECT * FROM OUR_FIRST_DB.public.test_backup

// good method, time travel capabilities maintained, and table restored
// both updates restored
SELECT * FROM OUR_FIRST_DB.public.test

```

## UNDROP

Restores the specified object to the system.
- Not all DROP commands have a corresponding UNDROP
- UNDROP relies on the Snowflake Time Travel feature
- An object can be restored only if the object was deleted within the Data Retention Period. The default value is 24 hours.

```sql

// Setting up table

// create a stage and specify URL and file format object
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.time_travel_stage
URL = 's3://data-snowflake-fundamentals/time-travel/'
file_format = MANAGE_DB.file_formats.csv_file;

// create customers table
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.customers (
id int,
first_name string,
last_name string,
email string,
gender string,
Job string,
Phone string);

// copy into customers from external stage
COPY INTO OUR_FIRST_DB.public.customers
from @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv');
// 1000 rows LOADED

SELECT * FROM OUR_FIRST_DB.public.customers;


// UNDROP command - Tables

DROP TABLE OUR_FIRST_DB.public.customers;

SELECT * FROM OUR_FIRST_DB.public.customers; // Object does not exist

UNDROP TABLE OUR_FIRST_DB.public.customers;


// UNDROP command - Schemas

DROP SCHEMA OUR_FIRST_DB.public;

SELECT * FROM OUR_FIRST_DB.public.customers; // schema does not exist

UNDROP SCHEMA OUR_FIRST_DB.public;


// UNDROP command - Database

DROP DATABASE OUR_FIRST_DB;

SELECT * FROM OUR_FIRST_DB.public.customers; // database does not exist

UNDROP DATABASE OUR_FIRST_DB;





// Restore replaced table
// using UNDROP command


UPDATE OUR_FIRST_DB.public.customers
SET LAST_NAME = 'Tyson';


UPDATE OUR_FIRST_DB.public.customers
SET JOB = 'Data Analyst';

SELECT * FROM OUR_FIRST_DB.public.customers; // last name and job updated by mistake



// // // Undroping a with a name that already exists

// undo second mistake using update job query id
// using bad method, we have replaced the table mening time travel no longer works
CREATE OR REPLACE TABLE OUR_FIRST_DB.public.customers as
SELECT * FROM OUR_FIRST_DB.public.customers before (statement => '01b0d5cf-0604-db2e-0002-484300031136')
// Time travel data is not available for table CUSTOMERS. The requested time is either beyond the allowed time travel period or before the object creation time.
// Table has been dropped meaning cannot use time travel

SELECT * FROM OUR_FIRST_DB.public.customers

// Attempt to undo first mistake using update last name query id
SELECT * FROM OUR_FIRST_DB.public.customers before (statement => '01b0d5cf-0604-dbe1-0002-484300032142') // Does not work as Time travel data is not available for newly created table

// Undrop customer table dropped in create and replace command
UNDROP table OUR_FIRST_DB.public.customers;
// Error: Object 'CUSTOMERS' already exists.

// Rename Customers table
ALTER TABLE OUR_FIRST_DB.public.customers
RENAME TO OUR_FIRST_DB.public.customers_wrong;

UNDROP table OUR_FIRST_DB.public.customers; // Table Customers successfully restored

// Time travel feature works again as we have undropped replaced table
SELECT * FROM OUR_FIRST_DB.public.customers before (statement => '01b0d5cf-0604-dbe1-0002-484300032142')

// Restore replaced table or undrop/restore a table that already exists by renaming the table and undrop the table

DESC table OUR_FIRST_DB.public.customers
```


## Assignment 10 - Undrop

```sql
-- Create database and schema
CREATE OR REPLACE DATABASE TIMETRAVEL_EXERCISE;
CREATE OR REPLACE SCHEMA TIMETRAVEL_EXERCISE.COMPANY_X;

USE DATABASE TIMETRAVEL_EXERCISE;
USE SCHEMA COMPANY_X;

-- Create customers table
CREATE OR REPLACE TABLE CUSTOMER AS
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.CUSTOMER
LIMIT 500;

SELECT * FROM TIMETRAVEL_EXERCISE.COMPANY_X.CUSTOMER;


-- Drop the schema
DROP SCHEMA TIMETRAVEL_EXERCISE.COMPANY_X;

SELECT * FROM TIMETRAVEL_EXERCISE.COMPANY_X.CUSTOMER;  // schema does not exist

// undrop schema

UNDROP SCHEMA TIMETRAVEL_EXERCISE.COMPANY_X;

SELECT * FROM TIMETRAVEL_EXERCISE.COMPANY_X.CUSTOMER;  // schema undrop successfully, table is displayed

```

## Retention time
When data in a table is modified, including deletion of data or dropping an object containing data, Snowflake preserves the state of the data before the update. The data retention period specifies the number of days for which this historical data is preserved and, therefore, Time Travel operations (SELECT, CREATE … CLONE, UNDROP) can be performed on the data.

Note: A retention period of 0 days for an object effectively disables Time Travel for the object.

For Snowflake Standard Edition, the retention period can be set to 0 (or unset back to the default of 1 day) at the account and object level (i.e. databases, schemas, and tables).
- Time travel up to 1 day

For Snowflake Enterprise Edition (and higher):

For transient databases, schemas, and tables, the retention period can be set to 0 (or unset back to the default of 1 day). The same is also true for temporary tables.

For permanent databases, schemas, and tables, the retention period can be set to any value from 0 up to 90 days.
- time travel up to 90 days

## Retention period
- DEFAULT = 1
- Retention period property of a table set to 1 means we can travel back for 24 hours for this table



```sql
// Show tables shows retention time of table objects
// default is 1
SHOW TABLES like '%CUSTOMERS%';

// Method 1: If table already exists, set retention time property to another value
ALTER TABLE OUR_FIRST_DB.PUBLIC.CUSTOMERS
SET DATA_RETENTION_TIME_IN_DAYS = 2;

// retention_time is now set to 2
SHOW TABLES like '%CUSTOMERS%';


// Method 2: On table creation, set the retention time property

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ret_example (
id int,
first_name string,
last_name string,
email string,
gender string,
Job string,
Phone string)
DATA_RETENTION_TIME_IN_DAYS = 3;

// retention_time is now set to 3
SHOW TABLES like '%EX%';

// A retention period of 0 days for an object effectively disables Time Travel for the object.
ALTER TABLE OUR_FIRST_DB.PUBLIC.ret_example
SET DATA_RETENTION_TIME_IN_DAYS = 0;

// UNDROP is a time travel feature
DROP TABLE OUR_FIRST_DB.PUBLIC.ret_example;
// Table RET_EXAMPLE did not exist or was purged.
// UNDROP cannot be used if retention time is 0
UNDROP TABLE OUR_FIRST_DB.PUBLIC.ret_example;

```

## Time travel cost

Storage fees are incurred for maintaining historical data during both the Time Travel and Fail-safe periods.

Storage fees are incurred for maintaining historical data during both the Time Travel and Fail-safe periods.

The fees are calculated for each 24-hour period (i.e. 1 day) from the time the data changed. The number of days historical data is maintained is based on the table type and the Time Travel retention period for the table.

Temporary and Transient Tables

To help manage the storage costs associated with Time Travel and Fail-safe, Snowflake provides two table types, temporary and transient, which do not incur the same fees as standard (i.e. permanent) tables:

- Transient tables can have a Time Travel retention period of either 0 or 1 day.
- Temporary tables can also have a Time Travel retention period of 0 or 1 day; however, this retention period ends as soon as the table is dropped or the session in which the table was created ends.
- Transient and temporary tables have no Fail-safe period.





```sql
// Show tables shows retention time of table objects
// default is 1
SHOW TABLES like '%CUSTOMERS%';

// Method 1: If table already exists, set retention time property to another value
ALTER TABLE OUR_FIRST_DB.PUBLIC.CUSTOMERS
SET DATA_RETENTION_TIME_IN_DAYS = 2;

// retention_time is now set to 2
SHOW TABLES like '%CUSTOMERS%';


// Method 2: On table creation, set the retention time property

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ret_example (
id int,
first_name string,
last_name string,
email string,
gender string,
Job string,
Phone string)
DATA_RETENTION_TIME_IN_DAYS = 3;

// retention_time is now set to 3
SHOW TABLES like '%EX%';

// A retention period of 0 days for an object effectively disables Time Travel for the object.
ALTER TABLE OUR_FIRST_DB.PUBLIC.ret_example
SET DATA_RETENTION_TIME_IN_DAYS = 0;

// UNDROP is a time travel feature
DROP TABLE OUR_FIRST_DB.PUBLIC.ret_example;
// Table RET_EXAMPLE did not exist or was purged.
// Undrop cannot be used if retention time is 0
UNDROP TABLE OUR_FIRST_DB.PUBLIC.ret_example;



// Time travel cost
// More time travel retention means more storage required to account for time travel capabilities

// query storage usage ordered by usage data
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE ORDER BY USAGE_DATE DESC;

// query for time travel bytes
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS;

// Query time travel storage
// convert bytes to gigabytes
// shows table, schema, database, storage used, and time travel storage used
// 0 time travel storage costs as not changes within retention time period
SELECT 	ID,
		TABLE_NAME,
		TABLE_SCHEMA,
        TABLE_CATALOG,
		ACTIVE_BYTES / (1024*1024*1024) AS STORAGE_USED_GB,
		TIME_TRAVEL_BYTES / (1024*1024*1024) AS TIME_TRAVEL_STORAGE_USED_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY STORAGE_USED_GB DESC,TIME_TRAVEL_STORAGE_USED_GB DESC;

// Query used for monitoring the cost and storage used for storage total and time travel features

```
