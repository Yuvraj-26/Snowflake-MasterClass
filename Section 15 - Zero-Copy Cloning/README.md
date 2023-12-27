## Zero-Copy Cloning

- Create copies of a database, a schema, a table, and other objects
- Traditional databases require copying of database structure such as metadata, primary keys etc.
- Snowflake uses clone command to copy structure, data, metadata of databases

## Zero-Copy Cloning
- Clone command is a metadata operation. Copied table is available in the database but only references the metadata of the original table and data storage
- Clone complete table without additional storage costs (if table is updated, additional cost are only related to the data updated as the metadata management system is very efficient)
- Quick and cost efficient way of cloning objects
- Cloned object is independent from original table
  - Modifications will only affect the cloned table as it is an independent object in our snowflake account
- Easy to copy all meta data and improved storage management
- Used for creating backups for development purposes (includes creating a clone test environment for testing or development)
- Cloning works with time travel
  - Can create clone table (copy of source table) at a certain timestamp using time travel  TIMESTAMP feature
- For databases, schemas, and tables, a clone does not contribute to the overall data storage for the object until operations are performed on the clone that modify existing data or add new data.



## Additional rules
- Any structure of the object and meta data is inherited
  - clustering keys, comments, etc
- Data storage objects (permanent and transient) can be cloned - no temporary objects. The following objects can be cloned:
  - databases
  - Schemas
  - Tables
- Configuration objects can be cloned:
  - Stages
  - File formats
  - Tasks



## Cloning tables
```sql
// Cloning

SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS;

CREATE TABLE OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE
CLONE OUR_FIRST_DB.PUBLIC.CUSTOMERS;


// Validate the data
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE;


// Update cloned table
// set lastname to null in clone
UPDATE OUR_FIRST_DB.public.CUSTOMERS_CLONE
SET LAST_NAME = NULL;

// original table unaffected
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS ;

// clone table has NULL last_name column
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE;



// Cloning a temporary table is not possible
CREATE OR REPLACE TEMPORARY TABLE OUR_FIRST_DB.PUBLIC.TEMP_TABLE(
  id int);

// cloning temporary table to newly created permanent (default) table is not possible
CREATE TABLE OUR_FIRST_DB.PUBLIC.TABLE_COPY
CLONE OUR_FIRST_DB.PUBLIC.TEMP_TABLE;
// 002119 (0A000): SQL compilation error: Temp table cannot be cloned to a permanent table; clone to a transient table instead.

// cloning temproary table to newly created temporary table is possible
CREATE TEMPORARY TABLE OUR_FIRST_DB.PUBLIC.TABLE_COPY
CLONE OUR_FIRST_DB.PUBLIC.TEMP_TABLE;

SELECT * FROM OUR_FIRST_DB.PUBLIC.TABLE_COPY;

```


## Cloning schemas and databases  


```sql

// Cloning Schema

// create transient schema
CREATE TRANSIENT SCHEMA OUR_FIRST_DB.COPIED_SCHEMA
// from source schema
CLONE OUR_FIRST_DB.PUBLIC;

// cloned / copied schema has all the data from original schema
SELECT * FROM COPIED_SCHEMA.CUSTOMERS;

// copy schema across databases
// clone stage objects
CREATE TRANSIENT SCHEMA OUR_FIRST_DB.EXTERNAL_STAGES_COPIED
CLONE MANAGE_DB.EXTERNAL_STAGES;



// Cloning Database
// create transient copy of OUR_FIRST_DB
// Copy DB has all stages, objects, schemas, structure in our clone
CREATE TRANSIENT DATABASE OUR_FIRST_DB_COPY
CLONE OUR_FIRST_DB;

DROP DATABASE OUR_FIRST_DB_COPY;
DROP SCHEMA OUR_FIRST_DB.EXTERNAL_STAGES_COPIED;
DROP SCHEMA OUR_FIRST_DB.COPIED_SCHEMA;



```
## Cloning with time travel


```sql
// Cloning

SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS;

CREATE TABLE OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE
CLONE OUR_FIRST_DB.PUBLIC.CUSTOMERS;


// Validate the data
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE;


// Update cloned table
// set lastname to null in clone
UPDATE OUR_FIRST_DB.public.CUSTOMERS_CLONE
SET LAST_NAME = NULL;

// original table unaffected
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS ;

// clone table has NULL last_name column
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE;



// Cloning a temporary table is not possible
CREATE OR REPLACE TEMPORARY TABLE OUR_FIRST_DB.PUBLIC.TEMP_TABLE(
  id int);

// cloning temporary table to newly created permanent (default) table is not possible
CREATE TABLE OUR_FIRST_DB.PUBLIC.TABLE_COPY
CLONE OUR_FIRST_DB.PUBLIC.TEMP_TABLE;
// 002119 (0A000): SQL compilation error: Temp table cannot be cloned to a permanent table; clone to a transient table instead.

// cloning temproary table to newly created temporary table is possible
CREATE TEMPORARY TABLE OUR_FIRST_DB.PUBLIC.TABLE_COPY
CLONE OUR_FIRST_DB.PUBLIC.TEMP_TABLE;

SELECT * FROM OUR_FIRST_DB.PUBLIC.TABLE_COPY;



// Cloning Schema

// create transient schema
CREATE TRANSIENT SCHEMA OUR_FIRST_DB.COPIED_SCHEMA
// from source schema
CLONE OUR_FIRST_DB.PUBLIC;

// cloned / copied schema has all the data from original schema
SELECT * FROM COPIED_SCHEMA.CUSTOMERS;

// copy schema across databases
// clone stage objects
CREATE TRANSIENT SCHEMA OUR_FIRST_DB.EXTERNAL_STAGES_COPIED
CLONE MANAGE_DB.EXTERNAL_STAGES;



// Cloning Database
// create transient copy of OUR_FIRST_DB
// Copy DB has all stages, objects, schemas, structure in our clone
CREATE TRANSIENT DATABASE OUR_FIRST_DB_COPY
CLONE OUR_FIRST_DB;

DROP DATABASE OUR_FIRST_DB_COPY;
DROP SCHEMA OUR_FIRST_DB.EXTERNAL_STAGES_COPIED;
DROP SCHEMA OUR_FIRST_DB.COPIED_SCHEMA;










// Cloning using time travel

// Setting up table

CREATE OR REPLACE TABLE OUR_FIRST_DB.public.time_travel (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);



CREATE OR REPLACE FILE FORMAT MANAGE_DB.file_formats.csv_file
    type = csv
    field_delimiter = ','
    skip_header = 1;

CREATE OR REPLACE STAGE MANAGE_DB.external_stages.time_travel_stage
    URL = 's3://data-snowflake-fundamentals/time-travel/'
    file_format = MANAGE_DB.file_formats.csv_file;



LIST @MANAGE_DB.external_stages.time_travel_stage; // customers.csv files


// copy csv file into time_travel table
// 1000 rows LOADED
COPY INTO OUR_FIRST_DB.public.time_travel
from @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv');


SELECT * FROM OUR_FIRST_DB.public.time_travel



// Update data
// set first name to Frank for all 1000 rows
UPDATE OUR_FIRST_DB.public.time_travel
SET FIRST_NAME = 'Frank'

SELECT * FROM OUR_FIRST_DB.public.time_travel




// Using time travel
// view data before update qeury executed
SELECT * FROM OUR_FIRST_DB.public.time_travel at (OFFSET => -60*1)



// Using time travel with clone feature
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.time_travel_clone
CLONE OUR_FIRST_DB.public.time_travel at (OFFSET => -60*1.5)

SELECT * FROM OUR_FIRST_DB.PUBLIC.time_travel_clone


// Update data again in clone
// we can make a clone of our clone also

UPDATE OUR_FIRST_DB.public.time_travel_clone
SET JOB = 'Snowflake Data Engineer'

// query id: 01b10110-0604-df32-0002-48430003f0da

// Using time travel: Method 2 - before Query
SELECT * FROM OUR_FIRST_DB.public.time_travel_clone before (statement => '01b10110-0604-df32-0002-48430003f0da')

// create a clone of a clone
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.time_travel_clone_of_clone
CLONE OUR_FIRST_DB.public.time_travel_clone before (statement => '01b10110-0604-df32-0002-48430003f0da')

// verify clone table is table before update query executed
SELECT * FROM OUR_FIRST_DB.public.time_travel_clone_of_clone

```
## Swapping tables

- Use-case: Development table into production table
- Similar to cloning as it involves swapping the meta data
- Swap renames two tables in a single transaction.
- ALTER TABLE
- SWAP WITH


## Assignment 11 - Zero-Copy cloning



```sql
USE ROLE ACCOUNTDMIN;
USE DATABASE DEMO_DB;
USE WAREHOUSE COMPUTE_WH;

CREATE OR REPLACE TABLE DEMO_DB.PUBLIC.SUPPLIER
AS
SELECT * FROM "SNOWFLAKE_SAMPLE_DATA"."TPCH_SF1"."SUPPLIER";


// Cloning

SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER;

// create clone from supplier table (source)
CREATE OR REPLACE TABLE DEMO_DB.PUBLIC.SUPPLIER_CLONE
CLONE DEMO_DB.PUBLIC.SUPPLIER;


// Validate the data
SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER_CLONE;


// Update the clone table and copy the query id
UPDATE SUPPLIER_CLONE
SET S_PHONE='###';

--> Query ID: 01b10125-0604-df2e-0002-48430003a0b6

// original table unaffected
SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER;

// clone table phone numbers updated for all rows
SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER_CLONE;


// Create another clone form the updated clone using the time travel feature to clone before the update query was executed

// create a clone of a clone
CREATE OR REPLACE TABLE DEMO_DB.PUBLIC.SUPPLIER_TIME_TRAVEL_CLONE_OF_CLONE
CLONE DEMO_DB.PUBLIC.SUPPLIER_CLONE before (statement => '01b10125-0604-df2e-0002-48430003a0b6')

// verify clone table is table before update query executed
// Phone numbers restored to correct numbers
SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER_TIME_TRAVEL_CLONE_OF_CLONE

// if we delete the source table, clone still exists since it is an independent object.
DROP TABLE DEMO_DB.PUBLIC.SUPPLIER;

UNDROP TABLE DEMO_DB.PUBLIC.SUPPLIER;

SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER_TIME_TRAVEL_CLONE_OF_CLONE // clone exists if source or previous clone is deleted
SELECT * FROM DEMO_DB.PUBLIC.SUPPLIER; // Object 'DEMO_DB.PUBLIC.SUPPLIER' does not exist or not authorized.

```
