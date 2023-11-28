## Loading Data

Snowflake provides the following main solutions for data loading. The best solution may depend upon the volume of data to load and the frequency of loading.

### Bulk Loading

This option enables loading batches of data from files already available in cloud storage, or copying (i.e. staging) data files from a local machine to an internal (i.e. Snowflake) cloud storage location before loading the data into tables using the COPY command.

- Most frequent method
- Uses compute power of warehouses
- Loading from stages
- COPY command
- Transformations possible

### Bulk Loading
This option is designed to load small volumes of data (i.e. micro-batches) and incrementally make them available for analysis. Snowpipe loads data within minutes after files are added to a stage and submitted for ingestion. This ensures users have the latest results, as soon as the raw data is available.

- Designed to load small volumes of Data that need to be up-to-date immediately
- Automatically once they are added to stages
- Latest results for analysis
- Snowpipe (Server-less feature)

### Stages
Snowflake Stages are locations where data files are stored (staged) for loading and unloading data. They are used to move data from one place to another, and the locations for the stages could be internal or external to the Snowflake environment.

- Not to be confused with data warehouse Stages or staging area
- Database object that contains the location of data files where data can be loaded from

#### External stage
- External cloud provider
  - S3 Bucket
  - GCP
  - Microsoft Azure Blob Storage
- Database object created in Schema
- CREATE STAGE (URL, access settings)
- Additional costs may apply if region/platform differs. Snowflake US East, Platform AWS, so if the platform we want to create the stage from and load data from is different, charges may apply for data transfer.

#### Internal stage
- Local storage maintained by Snowflake (on-site server)


### Creating Stage

```sql
// Database to manage stage objects, fileformats etc.

// MANAGE_DB manages and holds objects created to ensure all important administrative objects are organised in a central data
CREATE OR REPLACE DATABASE MANAGE_DB;

CREATE OR REPLACE SCHEMA external_stages;

// Creating external stage

// CREATE OR REPLACE STAGE aws_stage if context selected from UI
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.aws_stage
    // Accessible URL of external stage
    url='s3://bucketsnowflakes3'
    // Not best practice to include credentials when creating the satge object, instead use a storage integration object, which is more secure and easier to administrate the credentials.
    credentials=(aws_key_id='ABCD_DUMMY_ID' aws_secret_key='1234abcd_key');

// Description of external stage

DESC STAGE MANAGE_DB.external_stages.aws_stage;

// Alter external stage   

ALTER STAGE aws_stage
    SET credentials=(aws_key_id='XYZ_DUMMY_ID' aws_secret_key='987xyz');

// Publicly accessible staging area    

CREATE OR REPLACE STAGE MANAGE_DB.external_stages.aws_stage
    url='s3://bucketsnowflakes3';

// List files in stage using @

LIST @aws_stage;

//Load data using from external stage into Snowflake using copy command

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    FROM @aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*';
```

### COPY command to Load Data

```sql
// Creating ORDERS table
// Use the fully qualified table name with database name, schema, and table name
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT INT,
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));

// Check table creation
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS;

// First copy command
// Specify where we want to copy data into

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    // Select MANAGE_DB and EXTERNAL_STAGES Context or must use fully qualified stage object as below.
    FROM @aws_stage
    file_format = (type = csv field_delimiter=',' skip_header=1);




// Copy command with fully qualified stage object

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    FROM @MANAGE_DB.external_stages.aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1);

// Error: Number of columns in file (11) does not match that of the corresponding table (6), use file format option error_on_column_count_mismatch=false to ignore this error
// There are 3 csv files in the external stage and Snowflake attempts to load all columns from 3 files into the table


// List files contained in stage shows 3 csv files

LIST @MANAGE_DB.external_stages.aws_stage;    




// Copy command with specified file(s)

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    FROM @MANAGE_DB.external_stages.aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1)
    files = ('OrderDetails.csv');
// Status is now LOADED, and 1500 rows loaded

// Check table data loaded
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS;

// Copy command with pattern for file names
// Use .* for wildcard, so every file that contains Order and is ending with csv will be loaded

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    FROM @MANAGE_DB.external_stages.aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*';

// Copy executed with 0 files processed. Snowflake recognises based on the metadata that the file has been already loaded and will not be loaded again in general by default. This can be modified
// Create and Replace the table again, and Load the data using the COPY INTO command with pattern
// Status is LOADED and 1500 rows are loaded

```

### Assignment 3 - Create a Stage and Load Data

```sql
//Create database
CREATE OR REPLACE DATABASE EXERCISE_DB;

//Creating the table / Meta data
//Database Name, Schema PUBLIC, Table Name
CREATE OR REPLACE TABLE "EXERCISE_DB"."PUBLIC"."CUSTOMERS" (
  "customer_ID" INT,
  "first_name" VARCHAR,
  "last_name" VARCHAR,
  "email" VARCHAR,
  "age" INT,
  "city" VARCHAR);

//Check that table is empy
USE EXERCISE_DB

SELECT * FROM CUSTOMERS;

//Create stage object
CREATE OR REPLACE STAGE EXERCISE_DB.public.aws_stage
    url='s3://snowflake-assignments-mc/loadingdata';

//Description of external stage
DESC STAGE EXERCISE_DB.PUBLIC.AWS_STAGE;

//List files in stage
// 2 csv files in stage
LIST @EXERCISE_DB.public.aws_stage;

// Load the data
COPY INTO EXERCISE_DB.PUBLIC.CUSTOMERS
    FROM @aws_stage
    file_format= (type = csv field_delimiter=';' skip_header=1)
// File1: customers2.csv LOADED status 750 rows loaded
// File2: customer3.csv LOADED status 850 rows loaded
```

### Transforming Data

```sql
// Transforming using the SELECT statement
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    // stage name is given an alias s
    // alias displays that column 1 is coming from external stage s, for example
    // use the dollar symbol to refer to the column number
    // column 1 is Order ID, column 2 is Amount
    FROM (select s.$1, s.$2 from @MANAGE_DB.external_stages.aws_stage s)
    file_format= (type = csv field_delimiter=',' skip_header=1)
    files=('OrderDetails.csv');



// Example 1 - Table

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
    ORDER_ID VARCHAR(30),
    AMOUNT INT
    )


SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX;

// Example 2 - Table    

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
    ORDER_ID VARCHAR(30),
    AMOUNT INT,
    PROFIT INT,
    PROFITABLE_FLAG VARCHAR(30)

    )


// Example 2 - Copy Command using a SQL function (subset of functions available) - Transformation
// Supported SQL functions for COPY transformations listed in the Snowflake documentation

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    FROM (select
            s.$1,
            s.$2,
            s.$3,
            // Transform data type using CAST
            // CAST column 3 from external stage s as int and comparison to 0
            // CASE to flag as profitable or not profitable
            CASE WHEN CAST(s.$3 as int) < 0 THEN 'not profitable' ELSE 'profitable' END
          from @MANAGE_DB.external_stages.aws_stage s)
    file_format= (type = csv field_delimiter=',' skip_header=1)
    files=('OrderDetails.csv');


SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
// Profitable flag displays flag for profit as expected


// Example 3 - Table

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
    ORDER_ID VARCHAR(30),
    AMOUNT INT,
    PROFIT INT,
    CATEGORY_SUBSTRING VARCHAR(5)

    )


// Example 3 - Copy Command using a SQL function (subset of functions available)

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    // Use select statement
    FROM (select
            s.$1,
            s.$2,
            s.$3,
            // Use a substring instead of complete category name
            // use substring function, refer to column 5 from external stage s
            // start from character 1 and length of string is 5, to extract first 5 characters of category string
            substring(s.$5,1,5)
          from @MANAGE_DB.external_stages.aws_stage s)
    file_format= (type = csv field_delimiter=',' skip_header=1)
    files=('OrderDetails.csv');


SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
// CATEGORY_SUBSTRING column is now substring of 5 characters
```

### More Transformations

```sql
//Example 3 - Table

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
    ORDER_ID VARCHAR(30),
    AMOUNT INT,
    PROFIT INT,
    PROFITABLE_FLAG VARCHAR(30)

    )



//Example 4 - Using subset of columns
// Created table from Example 3 has 4 columns
// We do not need to write data in all coliumns available in the table, in this example, we only use 2 columns meaning the other 2 columns remain empty

// Use brackets after table name specifying ORDER_ID, PROFIT columns required
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX (ORDER_ID,PROFIT)
    // Copying data into specified two columns
    FROM (select
            s.$1,
            s.$3
          from @MANAGE_DB.external_stages.aws_stage s)
    file_format= (type = csv field_delimiter=',' skip_header=1)
    files=('OrderDetails.csv');

SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX;
// Table created has four columns but only ORDER_ID and PROFIT have data available, whereas other two columns are empty showing NULL



//Example 5 - Table Auto increment

CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
    // default is autoincrement start 1 increment by 1 for every new row of the table
    ORDER_ID number autoincrement start 1 increment 1,
    AMOUNT INT,
    PROFIT INT,
    PROFITABLE_FLAG VARCHAR(30)

    )



//Example 5 - Auto increment ID
// ORDER_ID is currently B-25601, so we require an auto increment ID column

COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX (PROFIT,AMOUNT)
    // Select statemnet to copy data into two specified colummns
    FROM (select
            s.$2,
            s.$3
          from @MANAGE_DB.external_stages.aws_stage s)
    file_format= (type = csv field_delimiter=',' skip_header=1)
    files=('OrderDetails.csv');


SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX;
// ORDER_ID column is now auto incremented starting at 1
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX WHERE ORDER_ID > 15;
// Filter on ORDER_ID using WHERE statement which is useful for when copying specific data into our databases


DROP TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX
```

### Copy Option: ON_ERROR

```sql
// Create new stage
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.aws_stage_errorex
   url='s3://bucketsnowflakes4'

// List files in stage
LIST @MANAGE_DB.external_stages.aws_stage_errorex;


// Create example table
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
   ORDER_ID VARCHAR(30),
   AMOUNT INT,
   PROFIT INT,
   QUANTITY INT,
   CATEGORY VARCHAR(30),
   SUBCATEGORY VARCHAR(30));

// Demonstrating error message
// Error: Numeric value 'one thousand' is not recognized
// If you would like to continue loading when an error is encountered, use other values such as 'SKIP_FILE' or 'CONTINUE' for the ON_ERROR option
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv');


// Validating table is empty    
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX    


 // Error handling using the ON_ERROR option
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv')
   ON_ERROR = 'CONTINUE';
   // Status PARTIALLY_LOADED
   // 1498 rows loaded of 1500 rows parsed
   // errors_seen 2 and first_error shown

 // Validating results and truncating table
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
SELECT COUNT(*) FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX

// Removes all data from the table
TRUNCATE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX;

// Error handling using the ON_ERROR option = ABORT_STATEMENT (default)
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv','OrderDetails_error2.csv')
   ON_ERROR = 'ABORT_STATEMENT';
   // Error: Numeric value 'one thousand' is not recognized
   // No loading occurs / no data row will be copied
   // if one error occurs in the entire copy statement, then no loading occurs

 // Validating results and truncating table
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
SELECT COUNT(*) FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX

TRUNCATE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX;

// Error handling using the ON_ERROR option = SKIP_FILE
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv','OrderDetails_error2.csv')
   ON_ERROR = 'SKIP_FILE';
   // Exact file where error occured will be skipped, but next files will be loaded if no error exists
   // File error.csv LOAD_FAILED 0 rows loaded
   // File error2.csv LOADED 285 rows loaded


 // Validating results and truncating table
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
SELECT COUNT(*) FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX

TRUNCATE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX;    


// Error handling using the ON_ERROR option = SKIP_FILE_<number>
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv','OrderDetails_error2.csv')
   ON_ERROR = 'SKIP_FILE_2';
   // 2 is the error limit
   // if less than the error limit of 2, file will be loaded, same as CONTINUE ON_ERROR statement
   // if error limit is reached, 2 or 3 errors, the file will be skipped
   // error limit reached, error limit in error.csv has been reached as 2 erros exists.
   // error2.csv file will be still loaded
   // File error.csv LOAD_FAILED error_limit 2
   // File error2.csv LOADED error_limit 2


 // Validating results and truncating table
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
SELECT COUNT(*) FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX

TRUNCATE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX;    


// Error handling using the ON_ERROR option = SKIP_FILE_<number>
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv','OrderDetails_error2.csv')
   ON_ERROR = 'SKIP_FILE_3';
   // 3 is the error limit
   // File has 2 errors so error_limit of 3 is not reached, both files will be loaded
   // File error.csv PARTIALLY_LOADED error_limit 3, errors_seen 2
   // File error2.csv LOADED error_limit 3, errors_seen 2


 // Validating results and truncating table
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX
SELECT COUNT(*) FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX

TRUNCATE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX;    


// Error handling using the ON_ERROR option = SKIP_FILE_<number>
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv','OrderDetails_error2.csv')
   ON_ERROR = 'SKIP_FILE_0.1%'; //SKIP_FILE_3% //SKIP_FILE_0.5%
   // Error limit as a dynamic value (percentage of total amount of rows)

// Examples:
// SKIP_FILE_3% means error_limit is 3% of 1500 which is 45 rows
// Error limit not reached by 2 existing errors
// error.csv PARTIALLY_LOADED
// error2.csv LOADED
// Both Files loaded normally

// SKIP_FILE_0.1% means error_limit is 0,1% of 1500 which is 1.5
// Error limit of errors.csv file reached as 2 existing errors
// error.csv LOAD_FAILED
// error2.csv LOADED
// One File loaded


SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS_EX


CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
   ORDER_ID VARCHAR(30),
   AMOUNT INT,
   PROFIT INT,
   QUANTITY INT,
   CATEGORY VARCHAR(30),
   SUBCATEGORY VARCHAR(30));


COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
   FROM @MANAGE_DB.external_stages.aws_stage_errorex
   file_format= (type = csv field_delimiter=',' skip_header=1)
   files = ('OrderDetails_error.csv','OrderDetails_error2.csv')
   ON_ERROR = SKIP_FILE_3
   SIZE_LIMIT = 30;
   // error2.csv LOADED only

   // Number (> 0) that specifies the maximum size (in bytes) of data to be loaded for a given COPY statement. When the threshold is exceeded, the COPY operation discontinues loading files. This option is commonly used to load a common group of files using multiple COPY statements. For each statement, the data load continues until the specified SIZE_LIMIT is exceeded, before moving on to the next statement. Note that at least one file is loaded regardless of the value specified for SIZE_LIMIT unless there is no file to be loaded.
   ```
### File Format object
```sql

// Specifying file_format in Copy command
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    FROM @MANAGE_DB.external_stages.aws_stage_errorex
    file_format = (type = csv field_delimiter=',' skip_header=1)
    files = ('OrderDetails_error.csv')
    ON_ERROR = 'SKIP_FILE_3';

// Similar to defining the stage as an object
// We define file format as an object, allowing re-use of the fileformat objects in COPY commands

// Creating table
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.ORDERS_EX (
    ORDER_ID VARCHAR(30),
    AMOUNT INT,
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));    

// Creating schema to keep things organized in MANAGED_DB database
// Store all file_formats in a schema
CREATE OR REPLACE SCHEMA MANAGE_DB.file_formats;

// Creating file format object
// key word file format with fully qualified name | database.schema.my_file_format name
CREATE OR REPLACE file format MANAGE_DB.file_formats.my_file_format;

// See properties of file format object
DESC file format MANAGE_DB.file_formats.my_file_format;



// Using file format object in Copy command       
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    FROM @MANAGE_DB.external_stages.aws_stage_errorex
    // specify file_format using FORMAT_NAME with fully qualified name
    file_format= (FORMAT_NAME=MANAGE_DB.file_formats.my_file_format)
    files = ('OrderDetails_error.csv')
    ON_ERROR = 'SKIP_FILE_3';
    // LOAD_FAILED error_limt 3 reached


// Altering file format object
ALTER file format MANAGE_DB.file_formats.my_file_format
    SET SKIP_HEADER = 1;
// PARTIALLY_LOADED normally as errors_seen is now 2 as expected

// Defining properties on creation of file format object   
CREATE OR REPLACE file format MANAGE_DB.file_formats.my_file_format
    TYPE=JSON,
    TIME_FORMAT=AUTO;    

// See properties of file format object    
DESC file format MANAGE_DB.file_formats.my_file_format;   
// TYPE property_value is now JSON

// Using file format object in Copy command       
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    FROM @MANAGE_DB.external_stages.aws_stage_errorex
    file_format= (FORMAT_NAME=MANAGE_DB.file_formats.my_file_format)
    files = ('OrderDetails_error.csv')
    ON_ERROR = 'SKIP_FILE_3';
    // SQL compilation error as we are working with CSV and not JSON
    // For CSV type, there are different file format options attached to the file format object than in the JSON file format object, so its not possible to alter the type to another type

// Therefore we see Error: File format type cannot be changed
// Altering the type of a file format is not possible
ALTER file format MANAGE_DB.file_formats.my_file_format
SET TYPE = CSV;


// Recreate file format (default = CSV)
CREATE OR REPLACE file format MANAGE_DB.file_formats.my_file_format
// Type property_value is CSV


// See properties of file format object    
DESC file format MANAGE_DB.file_formats.my_file_format;   



// Truncate table
TRUNCATE table OUR_FIRST_DB.PUBLIC.ORDERS_EX;


// We can override certain properties of an existing file format object
// file_format= (FORMAT_NAME=MANAGE_DB.file_formats.my_file_format)
// file format has SKIP_HEADER property_value of 0 (default)


// Overwriting properties of file format object      
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS_EX
    FROM  @MANAGE_DB.external_stages.aws_stage_errorex
    // add additonal file format options, we would like to use in this single COPY command, therefore skip_header is now true
    // good for using a certain file format object with altered file format options
    file_format = (FORMAT_NAME= MANAGE_DB.file_formats.my_file_format  field_delimiter = ',' skip_header=1 )
    files = ('OrderDetails_error.csv')
    ON_ERROR = 'SKIP_FILE_3';

// We can define these properties in the STAGE object instead of the File format object
// Properties such as TYPE, field delimiter, skip header, are also defined in the STAGE object
// Excel file shows that all file format object properties are available in the stage object.
// Best practice to use a seperate file format object to ovveride the properties of the stage object
// STAGE Object has additional STAGE_COPY_OPTIONS such as ON_ERROR, SIZE_LIMIT, PURGE...

DESC STAGE MANAGE_DB.external_stages.aws_stage_errorex;
```

### Assignment 4 - Create file format object and use copy option

```sql
//Create database
CREATE OR REPLACE DATABASE EXERCISE_DB;

//Creating the table / Meta data
//Database Name, Schema PUBLIC, Table Name
CREATE OR REPLACE TABLE "EXERCISE_DB"."PUBLIC"."CUSTOMERS" (
  "customer_ID" INT,
  "first_name" VARCHAR,
  "last_name" VARCHAR,
  "email" VARCHAR,
  "age" INT,
  "city" VARCHAR);

SELECT * FROM CUSTOMERS;

//Create stage object
CREATE OR REPLACE STAGE EXERCISE_DB.public.aws_stage
    url='s3://snowflake-assignments-mc/fileformat';

//Description of external stage
DESC STAGE EXERCISE_DB.public.aws_stage

//List files in stage
// 1 csv file in stage
LIST @EXERCISE_DB.public.aws_stage;

// Creating file format object
CREATE OR REPLACE FILE FORMAT EXERCISE_DB.public.aws_fileformat
TYPE = CSV
FIELD_DELIMITER='|'
SKIP_HEADER=1;

// See properties of file format object
DESC file format EXERCISE_DB.public.aws_fileformat;


// Load the data     
COPY INTO EXERCISE_DB.PUBLIC.CUSTOMERS
    FROM @aws_stage
      file_format= (FORMAT_NAME=EXERCISE_DB.public.aws_fileformat)

// Alternative
COPY INTO EXERCISE_DB.PUBLIC.CUSTOMERS
    FROM @aws_stage
      file_format= EXERCISE_DB.public.aws_fileformat
```
