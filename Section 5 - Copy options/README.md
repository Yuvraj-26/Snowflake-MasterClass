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




```sql
```
```sql
```
