## Load Unstructured Data

- Create Stage object with connection to the location where file is stored
- Load raw data into separate table with one column of VARIANT Data type which can handle unstructured data including JSON files
- Analyse and Parse using Snowflake functions
- Since data can be hierarchical, Flatten the data and Load into final table

## JSON files
- JavaScript object notation
- JSON is a text format for storing and transporting data
- "Attribute name" : "Value"
- Arrays
- Nested objects to store data

## Creating stage and raw file
```sql
// First step: Load Raw JSON

// Create stage object
CREATE OR REPLACE stage MANAGE_DB.EXTERNAL_STAGES.JSONSTAGE
     url='s3://bucketsnowflake-jsondemo';

// Create file format with property TYPE JSON
CREATE OR REPLACE file format MANAGE_DB.FILE_FORMATS.JSONFORMAT
    TYPE = JSON;

// Create separate table to load raw data with one column of variant data type
CREATE OR REPLACE table OUR_FIRST_DB.PUBLIC.JSON_RAW (
    raw_file variant);

// Copy date into table using stage object and file format object
COPY INTO OUR_FIRST_DB.PUBLIC.JSON_RAW
    FROM @MANAGE_DB.EXTERNAL_STAGES.JSONSTAGE
    file_format= MANAGE_DB.FILE_FORMATS.JSONFORMAT
    files = ('HR_data.json');


SELECT * FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;
```

## Assignment 7 - Load Raw JSON

```sql
// Load Raw JSON

USE EXERCISE_DB;

// Create stage object
CREATE OR REPLACE stage JSONSTAGE
     url='s3://snowflake-assignments-mc/unstructureddata/';

LIST @JSONSTAGE // 1 json FILE jobskills.json


// Create file format with property TYPE JSON
CREATE OR REPLACE file format JSONFORMAT
    TYPE = JSON;

// Create separate table to load raw data with one column of variant data type
CREATE OR REPLACE table JSON_RAW (
    raw_file variant);

// Copy date into table using stage object and file format object
COPY INTO JSON_RAW
    FROM @JSONSTAGE
    file_format= JSONFORMAT


SELECT * FROM JSON_RAW;
```

## Parsing JSON

```sql
// Second step: Parse & Analyse Raw JSON

    // Create columns and parse JSON data to work with the data easily
   // Selecting attribute/column

// Query the city
// SELECT column : attribute city from table
SELECT RAW_FILE:city FROM OUR_FIRST_DB.PUBLIC.JSON_RAW

// SELECT column 1 : attribute first_name from table
SELECT $1:first_name FROM OUR_FIRST_DB.PUBLIC.JSON_RAW


   // Selecting attribute/column - formatted

// convert to another data type using ::
// SELECT column with attribute first name :: convert to string with alias as first_name from table
// converted to real string to remove double quotes
SELECT RAW_FILE:first_name::string as first_name  FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;

// SELECT column with attribute id :: convert to integer with alias id from table
SELECT RAW_FILE:id::int as id  FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;

// combined multiple columns
SELECT
    RAW_FILE:id::int as id,  
    RAW_FILE:first_name::STRING as first_name,
    RAW_FILE:last_name::STRING as last_name,
    RAW_FILE:gender::STRING as gender

FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;



   // Handling nested data
   // nested within one row and column
SELECT RAW_FILE:job as job  FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;
```

## Handling nested data

```sql
// Handling nested data
// nested within one row and column
SELECT RAW_FILE:job as job  FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;

// Example { "salary"32000, "title":"Financial Analyst"}

// SELECT parent attribute job. child attribute salary :: cast to data type integer with alias
SELECT
   RAW_FILE:job.salary::INT as salary
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;
// SALARY column with integer values


// combined columns
SELECT
 RAW_FILE:first_name::STRING as first_name,
 // parent.child attributes
 RAW_FILE:job.salary::INT as salary,
 // parent.child attributes
 RAW_FILE:job.title::STRING as title
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;


// Handling arrays

SELECT
 RAW_FILE:prev_company as prev_company
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;
// Prev_company column has [] empty array rows and ["Schiller-Mills", "Hessel-Smiltham"] multiple objects

// SELECT first value
// 0 for first element, 1 for second element
SELECT
 RAW_FILE:prev_company[1]::STRING as prev_company
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;

// Previous jobs have multiple elements / objects in the array
// This is a hierarchical structure
// Aggregate the data using ARRAY_SIZE function
// Aggregate the number of companies a person has worked for by counting elements in the array
SELECT
 ARRAY_SIZE(RAW_FILE:prev_company) as prev_company
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;
// 200 rows



// UNION ALL command by combining and expanding the rows
// First row will be 0 element of prev_company
// Second row will be 1 element of prev_company
// For every value, we get two rows
SELECT
 RAW_FILE:id::int as id,  
 RAW_FILE:first_name::STRING as first_name,
 RAW_FILE:prev_company[0]::STRING as prev_company
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW
UNION ALL
SELECT
 RAW_FILE:id::int as id,  
 RAW_FILE:first_name::STRING as first_name,
 RAW_FILE:prev_company[1]::STRING as prev_company
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW
ORDER BY id
// 400 rows

```

## Assignment 8: Parsing and handling array




```sql
// Load Raw JSON

USE EXERCISE_DB;

// Create stage object
CREATE OR REPLACE stage JSONSTAGE
     url='s3://snowflake-assignments-mc/unstructureddata/';

LIST @JSONSTAGE // 1 json FILE jobskills.json


// Create file format with property TYPE JSON
CREATE OR REPLACE file format JSONFORMAT
    TYPE = JSON;

// Create separate table to load raw data with one column of variant data type
CREATE OR REPLACE table JSON_RAW (
    raw_file variant);

// Copy date into table using stage object and file format object
COPY INTO JSON_RAW
    FROM @JSONSTAGE
    file_format= JSONFORMAT


SELECT * FROM JSON_RAW;

// Parse & Analyse Raw JSON

    // Create columns and parse JSON data to work with the data easily
   // Selecting attribute/column

// Query the first_name
// SELECT column RAW_FILE : attribute first_name from table
SELECT RAW_FILE:first_name FROM EXERCISE_DB.PUBLIC.JSON_RAW
// Query the first_name
// SELECT column 1 : attribute first_name from table
SELECT $1:first_name FROM EXERCISE_DB.PUBLIC.JSON_RAW


   // Selecting attribute/column - formatted

// convert to another data type using ::
// SELECT column with attribute first name :: convert to string with alias as first_name from table
// converted to real string to remove double quotes
SELECT RAW_FILE:first_name::string as first_name  FROM EXERCISE_DB.PUBLIC.JSON_RAW;

// SELECT column with attribute last_name :: convert to string with alias last_name from table
SELECT RAW_FILE:last_name::string as last_name  FROM EXERCISE_DB.PUBLIC.JSON_RAW;

// SELECT column with attribute skills :: convert to integer with alias last_name from table
SELECT RAW_FILE:skills::string as skills  FROM EXERCISE_DB.PUBLIC.JSON_RAW;


   // Handling arrays
   // nested within one row and column
SELECT RAW_FILE:Skills as Skills  FROM EXERCISE_DB.PUBLIC.JSON_RAW;


// Handling arrays

SELECT
    RAW_FILE:Skills as Skills
FROM EXERCISE_DB.PUBLIC.JSON_RAW;
// skills column has arrays with multiple objects

// SELECT first value
// 0 for first element, 1 for second element
SELECT
    RAW_FILE:Skills[0]::STRING as skills
FROM EXERCISE_DB.PUBLIC.JSON_RAW;

SELECT
    RAW_FILE:Skills[1]::STRING as skills
FROM EXERCISE_DB.PUBLIC.JSON_RAW;


// combined SELECT
SELECT
$1:first_name::STRING,
$1:last_name::STRING,
$1:Skills[0]::STRING,
$1:Skills[1]::STRING
FROM JSON_RAW;

// The skills column contains an array.
// Query the first two values in the skills attribute for every record in a separate column:
// Copy data in table
// Create a table and insert the data for these 4 columns in that table.

CREATE OR REPLACE TABLE SKILLS AS
SELECT
$1:first_name::STRING as first_name,
$1:last_name::STRING as last_name,
$1:Skills[0]::STRING as Skill_1,
$1:Skills[1]::STRING as Skill_2
FROM JSON_RAW;

// Query from table
SELECT * FROM SKILLS

// Query: What is the first skill of the person with first_name 'Florina'?
// Answer: Hatha Yoga
SELECT * FROM SKILLS
WHERE FIRST_NAME='Florina';
```

## Dealing  with hierarchy

```sql
```



```sql
```
