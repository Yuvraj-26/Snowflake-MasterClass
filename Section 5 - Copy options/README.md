## Copy Options

## VALIDATION_MODE
Validate the data files instead of loading them

Useful for verifying or validating how the copy commands would work and if we would encounter errors

- RETURN_n_ROWS - Validates and returns the specified number of rows; fails at the first error encountered (RETURN_10_ROWS returns the first 10 rows if no error encountered, but not loaded into the table, if there are errors in first 10 rows, first error returned

- RETURN_ERRORS - Returns all errors in the Copy commands




```sql
---- VALIDATION_MODE ----
// Prepare database & table
CREATE OR REPLACE DATABASE COPY_DB;


CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/size/';

LIST @COPY_DB.PUBLIC.aws_stage_copy; // 2 csv files Orders.csv and Orders2.csv


 //Load data using copy command
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    VALIDATION_MODE = RETURN_ERRORS
    // Query returns no row results as no errors exists, data is suitable to be loaded

// Check
SELECT * FROM ORDERS;

COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
   VALIDATION_MODE = RETURN_5_ROWS
   // Query returns first five rows and as it is returned, no errors exist in the first 5 rows
   // Validation only, still no data is loaded into our Orders table
```

VALIDATION_MODE on new S3 bucket - return failed

```sql
---- VALIDATION_MODE ----
// Prepare database & table
CREATE OR REPLACE DATABASE COPY_DB;


CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/returnfailed/';

LIST @COPY_DB.PUBLIC.aws_stage_copy; // 4 csv files, 2 contain errors


 //Load data using copy command
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    VALIDATION_MODE = RETURN_ERRORS;
    // Query returns 4 rows where errors exists. note: no data is loaded into ORDERS table
    // Information shows the line, character, and category (conversion) of errors

// Check to see no data is loaded
SELECT * FROM ORDERS;

COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
   VALIDATION_MODE = RETURN_5_ROWS;
   // Query returns first five rows and as it is returned, no errors exist in the first 5 rows
   // Validation only, still no data is loaded into our Orders table

// Edit pattern to include files with errors
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*error.*'
   VALIDATION_MODE = RETURN_5_ROWS; // RETURN_1_ROWS returns the same error message as the error occurs in the first row
   // Query returns no longer returns rows
   // Returns the first occuring error message: Numeric value 'one thousand' is not recognized
   //   File 'returnfailed/Ord5erDetails_error.csv', line 2, character 14  Row 1, column "ORDERS"["PROFIT":3]
```
## Assignment 5 - Using the copy Options


```sql
// Prepare database & table
CREATE OR REPLACE DATABASE COPYOPTION_DB;


CREATE OR REPLACE TABLE  COPYOPTION_DB.PUBLIC.EMPLOYEES (
    CUSTOMER_ID INT,
    FIRST_NAME VARCHAR(50),
    LAST_NAME VARCHAR(50),
    EMAIL VARCHAR(50),
    AGE INT,
    DEPARTMENT VARCHAR(50));

// Prepare stage object
CREATE OR REPLACE STAGE COPYOPTION_DB.PUBLIC.aws_stage_copy
    url='s3://snowflake-assignments-mc/copyoptions/example1/';

LIST @COPYOPTION_DB.PUBLIC.aws_stage_copy; // 1 csv file, size 6965, employees.csv


// Creating file format object
CREATE OR REPLACE FILE FORMAT COPYOPTION_DB.PUBLIC.aws_fileformat
TYPE = CSV
FIELD_DELIMITER=','
SKIP_HEADER=1;

// See properties of file format object
DESC file format COPYOPTION_DB.PUBLIC.aws_fileformat;

// Use validation mode
// Use copy option to only validate if there are errros, and if yes what errors
COPY INTO COPYOPTION_DB.PUBLIC.EMPLOYEES
    FROM @aws_stage_copy
    //file_format= (type = csv field_delimiter=',' skip_header=1)
    // use file format object
    file_format= COPYOPTION_DB.PUBLIC.aws_fileformat
    VALIDATION_MODE = RETURN_ERRORS;
    // Query returns 1 rows where ERROR exists. note: no data is loaded into ORDERS table
    // Information shows the line 10 character 1, numeric value - is not recognized

// Check to see no data is loaded
SELECT * FROM EMPLOYEES;

// Using ON_ERROR
// Load the data anyway regardless of the error using the ON_ERROR option
COPY INTO COPYOPTION_DB.PUBLIC.EMPLOYEES
    FROM @aws_stage_copy
    //file_format= (type = csv field_delimiter=',' skip_header=1)
    file_format= COPYOPTION_DB.PUBLIC.aws_fileformat
    files = ('employees.csv')
    ON_ERROR = 'CONTINUE';
    // status PARTIALLY_LOADED 121 rows loaded of 122 rows parsed

// Check to see data is loaded
SELECT * FROM EMPLOYEES;
```

## Working with Rejected records
Advanced methods for working with error results from Copy commands and processing rows that have caused some errors.

```sql
---- VALIDATION_MODE ----
// Prepare database & table
CREATE OR REPLACE DATABASE COPY_DB;


CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/returnfailed/';

LIST @COPY_DB.PUBLIC.aws_stage_copy; // 4 csv files, 2 contain errors


---- Use files with errors ----
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/returnfailed/';

LIST @COPY_DB.PUBLIC.aws_stage_copy;    



COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    VALIDATION_MODE = RETURN_ERRORS



COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    VALIDATION_MODE = RETURN_1_rows




-------------- Working with error results -----------

---- 1) Saving rejected files after VALIDATION_MODE ----

CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));


COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    VALIDATION_MODE = RETURN_ERRORS;
    // 4 errors with rejected_record info


// Storing rejected /failed results in a table
CREATE OR REPLACE TABLE rejected AS
// select from the table function result_scan using the last query id function
// copied query id  01b0a2e8-0604-d7c0-0000-00024843559d
// select statement 4 displays reject records from the rejected_record column
select * from table(result_scan(last_query_id()));
select rejected_record from table(result_scan(last_query_id()));

// Create a table called REJECTED with the result
// Now we can inset additional records using INSERT loading dynamically from the previous result using last_query_id
INSERT INTO rejected
select rejected_record from table(result_scan(last_query_id()));

SELECT * FROM rejected;




---- 2) Saving rejected files without VALIDATION_MODE ----




// Validate previous copy command using the ON_ERROR=CONTINUE method
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    ON_ERROR=CONTINUE

// Validate previous copy commands and show errors that have occured using validate function
// validates the files loaded in a pasts ececution of the COPY INTO <table> command
// use the job_id and paste the query id or use _last for previous copy command
select * from table(validate(orders, job_id => '_last'));


---- 3) Working with rejected records ----

// Process these rejected errors

// We created a table called REJECTED
SELECT REJECTED_RECORD FROM rejected;
// Records stored in this table are of the data type Variant, which can contain variant values
// Example row 1: B-30601,1275,10,7-,Furniture, Bookcases

// Use the SPLIT_PART function with split delimiter as comma to apply on the column to split into multiple columns


CREATE OR REPLACE TABLE rejected_values as
// SELECT statement shows the columns have been split
// can now use the REPLACE function to replace 7- quantity to fix errors
SELECT
SPLIT_PART(rejected_record,',',1) as ORDER_ID,
SPLIT_PART(rejected_record,',',2) as AMOUNT,
SPLIT_PART(rejected_record,',',3) as PROFIT,
SPLIT_PART(rejected_record,',',4) as QUATNTITY,
SPLIT_PART(rejected_record,',',5) as CATEGORY,
SPLIT_PART(rejected_record,',',6) as SUBCATEGORY
FROM rejected;

// Now CREATE the table rejected values using SELECT statement

// Display the table
SELECT * FROM rejected_values;
```

## SIZE_LIMIT COPY OPTION
Specify maximum size (in bytes) of data loaded in that command (at least one file)
First file is always loaded regardless of the SIZE_LIMIT.


When threshold is exceeded, the COPY operation stops loading.

If after the first file is loaded, the SIZE_LIMIT is exceeded, the next file will not be loaded.

```sql


---- SIZE_LIMIT ----

// Prepare database & table
CREATE OR REPLACE DATABASE COPY_DB;

CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));


// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/size/';


// List files in stage
LIST @aws_stage_copy;
// Orders.csv has size of 54600 bytes
// Orders2.csv has size of 54598 bytes


//Load data using copy command
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    SIZE_LIMIT=20000; //Orders.csv first file LOADED

// Example: SIZE_LIMIT=20000
// Limit will be exceeded with the first file meaning first file will be loaded, and second will not
// Limit refers to combined size of files


//Load data using copy command
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    SIZE_LIMIT=60000; //Both files LOADED

// Example: SIZE_LIMIT=60000
// Limit is not exceeded with the first file meaning second file is loaded
```

## RETURN_FAILED_ONLY
Specifies whether to return only files that have failed to load in the statement result

```sql


---- RETURN_FAILED_ONLY ----



CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/returnfailed/';

LIST @COPY_DB.PUBLIC.aws_stage_copy // 4 csv files in S3 bucket


 //Load data using copy command
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    // shows files that have been loaded with errors
    RETURN_FAILED_ONLY = TRUE
    // For copy command with errors, it does not make sense to use RETURN_FAILED_ONLY = TRUE
    // as we recieve the first error. Numeric value 'one thousand' is not recognized



// Use RETURN_FAILED_ONLY copy option in combination with ON_ERROR=CONTINUE    
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    // continue to load on error meaning
    ON_ERROR =CONTINUE
    // shows some files that are loaded and some files that are partially loaded
    // therefore we can focus on the files that have errors when set to true
    RETURN_FAILED_ONLY = TRUE


// Default = FALSE

// replace table
CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// If RETURN_FAILED_ONLY option not included in the code, default is FALSE, as shown below
// This will display all files, including the files loaded without any errors
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    ON_ERROR =CONTINUE

```

## TRUNCATECOLUMNS
Specifies whether to truncate text strings that exceed the target column length

- TRUE = strings are automatically truncated to the target column length

- FALSE = COPY produces an error if a loaded string exceeds the target column length (DEFAULT = FALSE)


```sql
---- TRUNCATECOLUMNS ----



CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    // CATEGORY set to 10 character limit
    CATEGORY VARCHAR(10),
    SUBCATEGORY VARCHAR(30));

// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/size/';

LIST @COPY_DB.PUBLIC.aws_stage_copy


 //Load data using copy command
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    // User character length limit (10) exceeded by string 'Electronics'


// TRUNCATECOLUMNS - use up to character 10 and no character length limit error
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    TRUNCATECOLUMNS = true;

// SELECT statement shows Electronics has been truncated to 10 characters - Electronic
// COPY Option TRUNCATECOLUMN allows truncated lengths meaning we do not need to chnage the data type declared above
```

## FORCE
Specifies to load all files, regardless of whether they've been loaded previously and have not changed since they were loaded

Note that this option reloads files, potentially duplicating data in a table
DEFAULT = FALSE


```sql

---- FORCE ----



CREATE OR REPLACE TABLE  COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// Prepare stage object
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/size/';

LIST @COPY_DB.PUBLIC.aws_stage_copy // 2 csv files


 // Load data using copy command
 // Initial load shows 2 files status LOADED
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'

// Not possible to load file that have been loaded and data has not been modified
// Copy executed with 0 files processed, all files skipped as files not changed by default
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'


SELECT * FROM ORDERS;    


// Using the FORCE option, data will be loaded anyways

COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    FORCE = TRUE;

```

## Load History
Enables you to retrieve the history of data loaded into tables using the COPY INTO <table> commands

For every database such as COPY_DB, we see INFORMATION_SCHEMA with Views



```sql

-- Query load history within a database --

USE COPY_DB;

SELECT * FROM information_schema.load_history

// Query from the LOAD_HISTORY view, we can view the load history of the specific table
// information such as SCHEMA_NAME, TABLE_NAME, LAST_LOAD_TIME, STATUS, ROW_COUNT, FIRST_ERROR_MESSAGE
SELECT * FROM COPY_DB.INFORMATION_SCHEMA.LOAD_HISTORY



-- Query load history gloabally from SNOWFLAKE database --
// If previously replaced table, we can see all copies of tables
// SNOWFLAKE global database - ACCOUNT_USAGE - Views - LOAD_HISTORY

SELECT * FROM snowflake.account_usage.load_history
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.LOAD_HISTORY
// visible TABLE_ID for replaced and recreated versions of the table with the same name
// New TABLE_ID refers to a new table
// information displayed such as TABLE_ID, TABLE_NAME, SCHEMA_ID, FILE_NAME, LAST_LOAD_TIME, STATUS, ERRORS


// Filter on specific table & schema
SELECT * FROM snowflake.account_usage.load_history
    // filter on tables from PUBLIC schema and table name ORDERS
  where schema_name='PUBLIC' and
  table_name='ORDERS'

  // Useful is table has been created and replaced multiple times

// Filter on specific table & schema
SELECT * FROM snowflake.account_usage.load_history
  where schema_name='PUBLIC' and
  table_name='ORDERS' and
  // filter on error_count greater than 0 to see only tables with errors
  error_count > 0


// Filter on specific table & schema
SELECT * FROM snowflake.account_usage.load_history
// filter to see current date - 1 to see files loaded before today starting from yesterday
WHERE DATE(LAST_LOAD_TIME) <= DATEADD(days,-1,CURRENT_DATE)
```

## Assignment 6 - Using the copy options

```sql
// Create table
create or replace table EXERCISE_DB.PUBLIC.EMPLOYEES
(
  customer_id int,
  first_name varchar(50),
  last_name varchar(50),
  email varchar(50),
  age int,
  department varchar(50));

// Create stage object
CREATE OR REPLACE STAGE EXERCISE_DB.PUBLIC.aws_stage
    url='s3://snowflake-assignments-mc/copyoptions/example2';

LIST @EXERCISE_DB.PUBLIC.aws_stage // 1 csv emplyees_error.csv

// Creating file format object
CREATE OR REPLACE FILE FORMAT EXERCISE_DB.public.aws_fileformat
TYPE = CSV
FIELD_DELIMITER=','
SKIP_HEADER=1;

// See properties of file format object
DESC file format EXERCISE_DB.PUBLIC.aws_fileformat;

// Use the copy option to only validate if there are errors and if yes what errors
SELECT * FROM EMPLOYEES;

// Use VALIDATION_MODE
COPY INTO EXERCISE_DB.PUBLIC.EMPLOYEES
    FROM @aws_stage
        // use file format object
      file_format= EXERCISE_DB.public.aws_fileformat
      VALIDATION_MODE = RETURN_ERRORS;
// User character length limit (50) exceeded by string 'Edee Antoin Lothar Marcus Frank Alexander Gustav Aurelio'


// One value in the FIRST_NAME column has more than 50 characters, we assume the table column properties could not be changed. TRUNCATE COLUMNS Option use to load the record anyways and truncate the value after 50 characters


// use TRUNCATECOLUMNS - use up to character 50 to fix character length limit error
COPY INTO EXERCISE_DB.PUBLIC.EMPLOYEES
    FROM @aws_stage
      file_format= EXERCISE_DB.public.aws_fileformat
      TRUNCATECOLUMNS = TRUE;
    // status is now LOADED 62 rows loaded out of 62 rows parsed
    // error limit 1 errors seen 0

// SELECT statement shows name has been truncated to 50 characters
// COPY Option TRUNCATECOLUMN allows truncated lengths meaning we do not need to change the data type declared above
SELECT * FROM EMPLOYEES;
```
