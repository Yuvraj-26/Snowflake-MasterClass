## Types of Tables


## Permanent
- CREATE TABLE
- Time Travel Retention Period 0 - 90 day
- Fail Safe
- Results in additional storage costs due to data protection features
- Until dropped
- Permanent data (default table for productive data)



## Transient
- CREATE TRANSIENT TABLE
- Time Travel Retention Period 0 - 1 day
- No Fail Safe (Non-configurable)
- Useful for large tables where data protection is not a requirement
- Until dropped
- Only for data that does not need to be protected


## Temporary
- CREATE TEMPORARY TABLE
- Time Travel Retention Period 0 - 1 day
- No Fail Safe
- Exists Only In Session (other users cannot see table, and closing session deletes table and time travel and data is not recoverable)
- Non-permanent data (only within session and data deleted on session close)


Data types is important for Managing storage costs

## Table types notes
- Types apply for other database objects (database, schema, etc.)
- If we create a temporary schema, then tables created under the schema will be temporary table
- For temporary table no naming conflicts with permanent/transient table
  - If we create a temporary table employee on top of permanent table employee, only the temporary table will be visible and queries/data transformations will only affect temporary table
  - Other tables will be effectively hidden


## Permanent Tables and Databases

```sql
CREATE OR REPLACE DATABASE PDB;

CREATE OR REPLACE TABLE PDB.public.customers (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

CREATE OR REPLACE TABLE PDB.public.helper (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

// Stage and file format
CREATE OR REPLACE FILE FORMAT MANAGE_DB.file_formats.csv_file
    type = csv
    field_delimiter = ','
    skip_header = 1

CREATE OR REPLACE STAGE MANAGE_DB.external_stages.time_travel_stage
    URL = 's3://data-snowflake-fundamentals/time-travel/'
    file_format = MANAGE_DB.file_formats.csv_file;

LIST  @MANAGE_DB.external_stages.time_travel_stage;


// Copy data and insert in table
COPY INTO PDB.public.helper
FROM @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv');




SELECT * FROM PDB.public.helper;

INSERT INTO PDB.public.customers
SELECT
t1.ID
,t1.FIRST_NAME
,t1.LAST_NAME
,t1.EMAIL
,t1.GENDER
,t1.JOB
,t1.PHONE
 FROM PDB.public.helper t1
CROSS JOIN (SELECT * FROM PDB.public.helper) t2
CROSS JOIN (SELECT TOP 100 * FROM PDB.public.helper) t3;




// Show table and validate
SHOW TABLES;







// Permanent tables

USE OUR_FIRST_DB

CREATE OR REPLACE TABLE customers (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

CREATE OR REPLACE DATABASE PDB;

// show is_current database and show options which shows table type, rentention type (default 1)
SHOW DATABASES;

SHOW TABLES;



// View table metrics (takes a bit to appear)
// shows in_transient, active_bytes, time_travel_bytes, failsafe_bytes
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS


SELECT 	ID,
       	TABLE_NAME,
		TABLE_SCHEMA,
        TABLE_CATALOG,
		ACTIVE_BYTES / (1024*1024*1024) AS ACTIVE_STORAGE_USED_GB,
		TIME_TRAVEL_BYTES / (1024*1024*1024) AS TIME_TRAVEL_STORAGE_USED_GB,
		FAILSAFE_BYTES / (1024*1024*1024) AS FAILSAFE_STORAGE_USED_GB,
        IS_TRANSIENT,
        DELETED,
        TABLE_CREATED,
        TABLE_DROPPED,
        TABLE_ENTERED_FAILSAFE
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
--WHERE TABLE_CATALOG ='PDB'
// tables have been already dropped
// tables are permanent not transient so there is no active storage but fail safe storage exists
WHERE TABLE_DROPPED is not null
ORDER BY FAILSAFE_BYTES DESC;
```


## Transient Tables and Databases



```sql
CREATE OR REPLACE DATABASE TDB;

// create transient table
CREATE OR REPLACE TRANSIENT TABLE TDB.public.customers_transient (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

// insert data
INSERT INTO TDB.public.customers_transient
SELECT t1.* FROM OUR_FIRST_DB.public.customers t1
CROSS JOIN (SELECT * FROM OUR_FIRST_DB.public.customers) t2

SHOW TABLES;



// Query storage
// is transient is YES
// no fail safe
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS


SELECT 	ID,
       	TABLE_NAME,
		TABLE_SCHEMA,
        TABLE_CATALOG,
		ACTIVE_BYTES,
		TIME_TRAVEL_BYTES / (1024*1024*1024) AS TIME_TRAVEL_STORAGE_USED_GB,
		FAILSAFE_BYTES / (1024*1024*1024) AS FAILSAFE_STORAGE_USED_GB,
        IS_TRANSIENT,
        DELETED,
        TABLE_CREATED,
        TABLE_DROPPED,
        TABLE_ENTERED_FAILSAFE
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE TABLE_CATALOG ='TDB'
ORDER BY TABLE_CREATED DESC;

// Set retention time to 0
// if table dropped, no time travel and no fail safe area

// for all transient tables, data retention can only be 0 -1 days
ALTER TABLE TDB.public.customers_transient
SET DATA_RETENTION_TIME_IN_DAYS  = 0

// SET DATA_RETENTION_TIME_IN_DAYS  = 2
// invalid value [2] for parameter 'DATA_RETENTION_TIME_IN_DAYS'


DROP TABLE TDB.public.customers_transient;

// if data rentetion set to 0, cannot undrop table as it is a time travel feature
UNDROP TABLE TDB.public.customers_transient;

SHOW TABLES;


// Creating transient schema and then table
// table will also always be transient

CREATE OR REPLACE TRANSIENT SCHEMA TRANSIENT_SCHEMA;

SHOW SCHEMAS;

CREATE OR REPLACE TABLE TDB.TRANSIENT_SCHEMA.new_table (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);


ALTER TABLE TDB.TRANSIENT_SCHEMA.new_table
SET DATA_RETENTION_TIME_IN_DAYS  = 2
// invalid value [2] for parameter 'DATA_RETENTION_TIME_IN_DAYS'
// must be 0 or 1 for transient tables

SHOW TABLES;
// new_table is_transient property is yes, and no fail safe

```

## Temporary Tables and Databases


```sql
// use permanent database
USE DATABASE PDB;

// Create permanent table

CREATE OR REPLACE TABLE PDB.public.customers (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);


INSERT INTO PDB.public.customers
SELECT t1.* FROM OUR_FIRST_DB.public.customers t1



SELECT * FROM PDB.public.customers


// Create temporary table (with the same name)
// effectively hides permanent csutomers table as table name already exists
// within session, temporary table is active
CREATE OR REPLACE TEMPORARY TABLE PDB.public.customers (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

// navigating to new worksheet, using ACCOUNTADMIN, permanent table still exists


// Validate temporary table is the active table
SELECT * FROM PDB.public.customers;

// Permanent table not affected by any transformations or queries performed on temporary table

// Create second temporary table (with a new name)
CREATE OR REPLACE TEMPORARY TABLE PDB.public.temp_table (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);

// Insert data in the new table
INSERT INTO PDB.public.temp_table
SELECT * FROM PDB.public.customers

SELECT * FROM PDB.public.temp_table
// temp_table only visible within session/worksheet
// no fail safe
// time travel only possible for 0 and 1 days

SHOW TABLES;

```
