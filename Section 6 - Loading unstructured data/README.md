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
```

```sql
```

```sql
```
