## Snowpipe
- Enables automatically loading once a file appears in a bucket
- If data such as transactional or event data needs to be available immediately for analysis
- Snowpipe uses server-less features instead of warehouses, meaning compute resources are managed by Snowflake
- Snowpipe is not intended for batch loading but for continuos loading. For batch loading we should just use the COPY command.

## Snowpipe Scenario
- Given a bucket, we can copy data manually into a table in Snowflake DB
- Snowpipe automatically detects new file in S3 bucket
- Event S3 notification triggers Snowpipe
- Snowpipe server-less loads the data into table, once a S3 notification is received.

## Setting Up Snowpipe
- Create Stage
  - connection and location to the external stage that we want to copy data from
- Test COPY COMMAND
  - to ensure it works
- Create Snowpipe
  - create pipe as object with COPY COMMAND definition
- S3 Event Notification
  - to trigger snowpipe

## Creating pipe and configuring pipe and notifications
- Create table
- Create file format object
- Create a stage object with integration object s3_int and file format object
- Create schema to store and manage pipes
- Test COPY COMMAND into created table from external Stage
- Create or replace pipe
- Describe pipe and get the notification_channel value to set up the event notification trigger in S3 management console
  - Navigate to S3 bucket
  - Properties
  - Create event notification with event name and prefix to limit the notifications to objects with key starting with specified characters
  - Set prefix to csv/snowpipe
  - Specify event type as All object create events
  - Select Destination as SQS queue
  - Enter SQS queue ARN and paste notification_channel value
- Test snowpipe


```sql
// employee_data_1.csv file
// contains details about employee

// Create table first
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.employees (
  id INT,
  first_name STRING,
  last_name STRING,
  email STRING,
  location STRING,
  department STRING
  )


// Create file format object
CREATE OR REPLACE file format MANAGE_DB.file_formats.csv_fileformat
    type = csv
    field_delimiter = ','
    skip_header = 1
    // handle null values, string nulls converted to database nulls
    null_if = ('NULL','null')
    // empty fields converted to database nulls
    empty_field_as_null = TRUE;


 // Create stage object with integration object & file format object
CREATE OR REPLACE stage MANAGE_DB.external_stages.csv_folder
    // snowpipe folder is sub folder of csv folder already in s3 bucket
    URL = 's3://snowflakes3bucket/csv/snowpipe'
    // s3_int integration object created for external provider S3 with Storage AWS role ARN and Storage allowed locations
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = MANAGE_DB.file_formats.csv_fileformat
// stage area CSV_FOLDER created

 // Create stage object with integration object & file format object
LIST @MANAGE_DB.external_stages.csv_folder  // 1 file employee_data_1.csv


// Create schema to keep things organized to store and manage pipes
CREATE OR REPLACE SCHEMA MANAGE_DB.pipes

// Define pipe
// create pipe with fully qualified name
CREATE OR REPLACE pipe MANAGE_DB.pipes.employee_pipe
auto_ingest = TRUE
AS
COPY INTO OUR_FIRST_DB.PUBLIC.employees // test copy command, first copy data into employee table created using external stage - status LOADED 100 rows loaded
FROM @MANAGE_DB.external_stages.csv_folder  

// Describe pipe object
DESC pipe employee_pipe
// pipe name, database name, schema name, definition is TESTED Copy Command using created table
// use notification_channel value

SELECT * FROM OUR_FIRST_DB.PUBLIC.employees  // 100 rows
// 200 rows after running select query shows snowpipe works

```

## Handling Errors
- Handling errors with error in copy command
- Edit file format field_delimiter to | gives error when running copy command
- Upload a new file 3 from AWS management console with column delimiter , and trigger the event to test snowpipe
- Upload a new file 4 from AWS management console


```sql
// Handling errors


// Create file format object
CREATE OR REPLACE file format MANAGE_DB.file_formats.csv_fileformat
    type = csv
    field_delimiter = ',' // correct delimiter
    //field_delimiter = '|' // create an error as field delimiter for file is ',' and running copy command gives error as number of columns in file (1) does not match table (0), as commas will not be interpreted as column delimiters
    skip_header = 1
    null_if = ('NULL','null')
    empty_field_as_null = TRUE;


SELECT * FROM OUR_FIRST_DB.PUBLIC.employees  // number of columns in file (1) error

// alter pipe refresh command shows employee_data_2.csv and employee_data_3.csv status as SENT
// event has been notified
ALTER PIPE employee_pipe refresh


// SELECT shows file 3 did not work due to wrong column delimiter, still 200 rows loaded
SELECT * FROM OUR_FIRST_DB.PUBLIC.employees

// Validate pipe is actually working
// execution state is running and pending file count is 0
SELECT SYSTEM$PIPE_STATUS('employee_pipe')

// Snowpipe error message
SELECT * FROM TABLE(VALIDATE_PIPE_LOAD(
    PIPE_NAME => 'MANAGE_DB.pipes.employee_pipe',
    START_TIME => DATEADD(HOUR,-2,CURRENT_TIMESTAMP()))) // date or time we want to get error messages - last 2 hours

// 1 error message Remote file 'https://snowflakes3bucket.s3.us-east-1.amazonaws.com/csv/snowpipe//employee_data_3.csv' was not found. There are several potential causes. The file might not exist. The required credentials may be missing or invalid. If you are running a copy command, please make sure files are not deleted when they are being loaded or files are not being loaded into two different tables concurrently with auto purge option.


// COPY command history from table to see error massage
// use copy history from table for better error messages
SELECT * FROM TABLE (INFORMATION_SCHEMA.COPY_HISTORY(
   table_name  =>  'OUR_FIRST_DB.PUBLIC.EMPLOYEES',
   START_TIME =>DATEADD(HOUR,-2,CURRENT_TIMESTAMP())))

// for file 3, first_error_message created by copy command initiated by snowpipe
// Number of columns in file (1) does not match that of the corresponding table (6), use file format option error_on_column_count_mismatch=false to ignore this error


// Upload file 4 csv using AWS management console
// Select statement shows 200 rows still loaded
// ALTER pipe refresh command shows file 4 csv SENT
// Copy from COPY HISTORY shows error message as expected

// Change file format back to correct delimiter
// snowpipe will work again
// upload files 3 and 4 csv to S3 bucket
// rows increased showing snowpipe worked

```

## Manage Pipes


```sql
-- Manage pipes --

// Overview of existing pipes and their properties

DESC pipe MANAGE_DB.pipes.employee_pipe;

// list pipes
SHOW PIPES;

// limit pipes using like filter with wildcards - case insensitive
SHOW PIPES like '%employee%'

SHOW PIPES in database MANAGE_DB

SHOW PIPES in schema MANAGE_DB.pipes

SHOW PIPES like '%employee%' in Database MANAGE_DB



-- Changing pipe (alter stage or file format) --

// Preparation table first
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.employees2 (
  id INT,
  first_name STRING,
  last_name STRING,
  email STRING,
  location STRING,
  department STRING
  )

// Aim: Alter copy into definition to load data automatically, once file appears in bucket, into new table

// Pause pipe
ALTER PIPE MANAGE_DB.pipes.employee_pipe SET PIPE_EXECUTION_PAUSED = true


// Verify pipe is paused and has pendingFileCount 0
SELECT SYSTEM$PIPE_STATUS('MANAGE_DB.pipes.employee_pipe')
// execution state is PAUSED, and pending file count is 0

 // Recreate the pipe to change the COPY statement in the definition
 // Recreate using create or replace with altered COPY definition with new employees2 table
CREATE OR REPLACE pipe MANAGE_DB.pipes.employee_pipe
auto_ingest = TRUE
AS
COPY INTO OUR_FIRST_DB.PUBLIC.employees2
FROM @MANAGE_DB.external_stages.csv_folder  

// view a list of all of the files received by the pipe
// metadata for the pipe is still available even though it has been recreated
// files 1 to 4 csv received files is still displayed

ALTER PIPE  MANAGE_DB.pipes.employee_pipe refresh

// List files in stage - 4 files in stage
LIST @MANAGE_DB.external_stages.csv_folder  

// table is empty
SELECT * FROM OUR_FIRST_DB.PUBLIC.employees2

 // Reload files manually that where already in the bucket
COPY INTO OUR_FIRST_DB.PUBLIC.employees2
FROM @MANAGE_DB.external_stages.csv_folder  // status 4 files LOADED


// Resume pipe
ALTER PIPE MANAGE_DB.pipes.employee_pipe SET PIPE_EXECUTION_PAUSED = false

// Verify pipe is running again
SELECT SYSTEM$PIPE_STATUS('MANAGE_DB.pipes.employee_pipe')
```
