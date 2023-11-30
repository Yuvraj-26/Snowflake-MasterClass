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
- For a combination of array with nested objects such as the spoken_languages parent attribute, which has array of two child objects - language and level
  - combine 3 elements using UNION ALL
  - better method is using table then flatten function to automatically flatten hierarchies. DO this by joining flatten function with parent attribute with original raw table, specifying the child objects using the value keyword and aliases.

```sql
// Combination of an array with nested objects
// parent is spoken_languages
// array that consists of two child objects - language and level

SELECT
    RAW_FILE:spoken_languages as spoken_languages
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;

SELECT * FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;


// aggregation of data using array_size to see how many languages spoken
SELECT
     array_size(RAW_FILE:spoken_languages) as spoken_languages
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW

// SELECT first name as string and array_size of spoken languages
SELECT
     RAW_FILE:first_name::STRING as first_name,
     array_size(RAW_FILE:spoken_languages) as spoken_languages
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW


// calling element 0 - first object of array
SELECT
    RAW_FILE:spoken_languages[0] as First_language
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;


// combine first name with column with nested object (language and level)
SELECT
    RAW_FILE:first_name::STRING as first_name,
    // element 0 of spoken_languages is nested object
    RAW_FILE:spoken_languages[0] as First_language
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW;

// combine first_name, first_language, level columns
SELECT
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[0].language::STRING as First_language,
    // query element 0 of level - spoken languages (parent).level(child)
    RAW_FILE:spoken_languages[0].level::STRING as Level_spoken
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW



// UNION ALL method
// For each person, we have 3 rows with the id, First_name, First_language, and level_spoken
SELECT
    RAW_FILE:id::int as id,
    RAW_FILE:first_name::STRING as First_name,
    // language with child attribute language
    RAW_FILE:spoken_languages[0].language::STRING as First_language,
    // language with child attribute level
    RAW_FILE:spoken_languages[0].level::STRING as Level_spoken
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW
UNION ALL
SELECT
    RAW_FILE:id::int as id,
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[1].language::STRING as First_language,
    RAW_FILE:spoken_languages[1].level::STRING as Level_spoken
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW
UNION ALL
SELECT
    RAW_FILE:id::int as id,
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[2].language::STRING as First_language,
    RAW_FILE:spoken_languages[2].level::STRING as Level_spoken
FROM OUR_FIRST_DB.PUBLIC.JSON_RAW
ORDER BY ID
// With UNION ALL method of 3 elements, it takes longer to code and as we do not know how many elements are within each nested object (how many languages each person speaks), we have 3 rows for each person, some containing null elements, for instance if a person speaks only 1 or 2 languages.


// combine 3 elements using table(flatten) function
// no null values and automatic function to flatten hierarchies
// useful for hierarchical and unstructured JSON data
select
      RAW_FILE:first_name::STRING as First_name,
      // for child objects we use value keyword
      // alias f shows two columns come from value f
    f.value:language::STRING as First_language,
   f.value:level::STRING as Level_spoken
// join original raw table with table(flatten) function
from OUR_FIRST_DB.PUBLIC.JSON_RAW,
// table (flatten) function with parent column for parent attribute used for nesting which is language // column is RAW_FILE and alias is f
table(flatten(RAW_FILE:spoken_languages)) f;
```

## Insert final data

```sql
// Option 1: CREATE TABLE AS
// Copy data while we created the table

// Create Languages table AS select statement
CREATE OR REPLACE TABLE Languages AS
select
      RAW_FILE:first_name::STRING as First_name,
    f.value:language::STRING as First_language,
   f.value:level::STRING as Level_spoken
from OUR_FIRST_DB.PUBLIC.JSON_RAW, table(flatten(RAW_FILE:spoken_languages)) f;

SELECT * FROM Languages;

truncate table languages;

// Option 2: INSERT INTO
// Create the table first using CREATE table statement
// Create columns with just metadata with no data in it
// Then insert the data into the table using the INSERT INTO statement

INSERT INTO Languages
select
      RAW_FILE:first_name::STRING as First_name,
    f.value:language::STRING as First_language,
   f.value:level::STRING as Level_spoken
from OUR_FIRST_DB.PUBLIC.JSON_RAW, table(flatten(RAW_FILE:spoken_languages)) f;
// 396 rows inserted into table


SELECT * FROM Languages;

//Summary
// 1, created a stage
// 2. loaded the raw JSON file into separate table with one single variant column
// 3. parsed and flattened the data to handle arrays, nested data, and hierarchical data
// 4. stored the results in a final table

```

## Querying PARQUET Data
Apache Parquet is an open source, column-oriented data file format designed for efficient data storage and retrieval. It provides efficient data compression and encoding schemes with enhanced performance to handle complex data in bulk
- File format PARQUET of the Apache Hadoop Ecosystem
- Efficient data compressing
- Column oriented meaning values of each table column are stored next to each other
- Open source and self-describing
- File compression is act of making a file smaller


```sql
    // Create file format and stage object

CREATE OR REPLACE FILE FORMAT MANAGE_DB.FILE_FORMATS.PARQUET_FORMAT
    TYPE = 'parquet';

// Create stage object that contains URL to S3 bucket
CREATE OR REPLACE STAGE MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE
    url = 's3://snowflakeparquetdemo'   
    FILE_FORMAT = MANAGE_DB.FILE_FORMATS.PARQUET_FORMAT;
    // set fileformat property to our file format object
    // Must specify file format in external stage unless CSV file format used. CSV is the default file format and is not necessary



    // Preview the data

LIST  @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;  // 1 file - daily_sales_items.parquet

// Query data directly from external stage to look at the data and columns


SELECT * FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;



// File format in Queries

// if we do not specify the file format or specify the file format object, and run SELECT *, we get Error: SELECT with no columns

CREATE OR REPLACE STAGE MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE
    url = 's3://snowflakeparquetdemo'  

SELECT * FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;


// if we have not specified file format in stage object
// we can specify in the select statement
// use syntax with single quotes

SELECT *
FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE
(file_format => 'MANAGE_DB.FILE_FORMATS.PARQUET_FORMAT')

// Quotes can be omitted in case of the current namespace
// single quotes mandatory unless we specify the context, including the database and schema where FILE_FORMATS is stored
USE MANAGE_DB.FILE_FORMATS;

// set context using USE and run without quotes

SELECT *
FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE
(file_format => MANAGE_DB.FILE_FORMATS.PARQUET_FORMAT)

// Recreate the stage that includes file format object
CREATE OR REPLACE STAGE MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE
    url = 's3://snowflakeparquetdemo'   
    FILE_FORMAT = MANAGE_DB.FILE_FORMATS.PARQUET_FORMAT;


// metadata of file or data we are working with
/*
{
  "__index_level_0__": 7,
  "cat_id": "HOBBIES",
  "d": 489,
  "date": 1338422400000000,
  "dept_id": "HOBBIES_1",
  "id": "HOBBIES_1_008_CA_1_evaluation",
  "item_id": "HOBBIES_1_008",
  "state_id": "CA",
  "store_id": "CA_1",
  "value": 12
}
*/


    // Syntax for Querying unstructured data
// Create a SELECT statement with one column with all the column names
// $1 stands for first column
// : refers to the object
// use quotes if we have spaces in the name
SELECT
$1:__index_level_0__,
$1:cat_id,
$1:date,
$1:"__index_level_0__",
$1:"cat_id",
$1:"d",
$1:"date",
$1:"dept_id",
$1:"id",
$1:"item_id",
$1:"state_id",
$1:"store_id",
$1:"value"
FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;


    // Date conversion
    // Date column is 1338422400000000
    // saved as integers

SELECT 1; // returns value 1

// use SQL DATE function
SELECT DATE(60*60*24); // returns 1970-01-02
SELECT DATE(365*60*60*24); // returns 1971-01-01 - next year



    // Querying with conversions and aliases

SELECT
$1:__index_level_0__::int as index_level,
$1:cat_id::VARCHAR(50) as category,
// DATE function
//   convert data into integer format using ::
DATE($1:date::int ) as Date,
$1:"dept_id"::VARCHAR(50) as Dept_ID,
$1:"id"::VARCHAR(50) as ID,
$1:"item_id"::VARCHAR(50) as Item_ID,
$1:"state_id"::VARCHAR(50) as State_ID,
$1:"store_id"::VARCHAR(50) as Store_ID,
$1:"value"::int as value
FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;

```

## Loading PARQUET Data

```sql
// Now that we have learned that we can query data directly from the external stage, we can do this for PARQUET data
// Use colon : to refer to the according objects
// Transform the data types of columns:

SELECT
$1:__index_level_0__::int as index_level,
$1:cat_id::VARCHAR(50) as category,
// DATE function
//   convert data into integer format using ::
DATE($1:date::int ) as Date,
$1:"dept_id"::VARCHAR(50) as Dept_ID,
$1:"id"::VARCHAR(50) as ID,
$1:"item_id"::VARCHAR(50) as Item_ID,
$1:"state_id"::VARCHAR(50) as State_ID,
$1:"store_id"::VARCHAR(50) as Store_ID,
$1:"value"::int as value
FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;

    // Adding metadata

SELECT
$1:__index_level_0__::int as index_level,
$1:cat_id::VARCHAR(50) as category,
DATE($1:date::int ) as Date,
$1:"dept_id"::VARCHAR(50) as Dept_ID,
$1:"id"::VARCHAR(50) as ID,
$1:"item_id"::VARCHAR(50) as Item_ID,
$1:"state_id"::VARCHAR(50) as State_ID,
$1:"store_id"::VARCHAR(50) as Store_ID,
$1:"value"::int as value,
// query the FILENAME
METADATA$FILENAME as FILENAME,
// query the FILE_ROW_NUMBER with timestamp of when data was loaded
METADATA$FILE_ROW_NUMBER as ROWNUMBER,
// SQL TO_TIMESTAMP NTZ (NO TIME ZONE) Function
TO_TIMESTAMP_NTZ(current_timestamp) as LOAD_DATE
FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE;

SELECT current_timestamp // Timezone inlcude -0800

SELECT TO_TIMESTAMP_NTZ(current_timestamp) // Timezone not included



   // Create destination table

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.PARQUET_DATA (
    ROW_NUMBER int,
    index_level int,
    cat_id VARCHAR(50),
    date date,
    dept_id VARCHAR(50),
    id VARCHAR(50),
    item_id VARCHAR(50),
    state_id VARCHAR(50),
    store_id VARCHAR(50),
    value int,
    Load_date timestamp default TO_TIMESTAMP_NTZ(current_timestamp))


   // Load the parquet data
   // Load the data into the table using COPY INTO and SELECT statement

COPY INTO OUR_FIRST_DB.PUBLIC.PARQUET_DATA
    FROM (SELECT
            METADATA$FILE_ROW_NUMBER,
            $1:__index_level_0__::int,
            $1:cat_id::VARCHAR(50),
            DATE($1:date::int ),
            $1:"dept_id"::VARCHAR(50),
            $1:"id"::VARCHAR(50),
            $1:"item_id"::VARCHAR(50),
            $1:"state_id"::VARCHAR(50),
            $1:"store_id"::VARCHAR(50),
            $1:"value"::int,
            TO_TIMESTAMP_NTZ(current_timestamp)
        FROM @MANAGE_DB.EXTERNAL_STAGES.PARQUETSTAGE);
        // status LOADED rows parsed 2038050 rows loaded 2038050

SELECT * FROM OUR_FIRST_DB.PUBLIC.PARQUET_DATA;

// Note:
// We have not created a separate table where we copied the raw file and the parsed and copied the data into final
// but this is best practice for tracking errors and provides more control of data    
```
