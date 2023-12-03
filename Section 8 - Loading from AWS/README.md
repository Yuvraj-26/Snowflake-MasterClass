## Loading Data from AWS

## Create S3 Bucket
- Select same region as hosted snowflake account - us-east-1
- data transfer is free if snowflake account and hosted S3 bucket is the same
- do not incur data transfer costs

### Configure the connection to Snowflake and set policy for snowflake to access the bucket
- Uplaod files to S3 bucket in csv and json folders

### Creating Policy using IAM in AWS
- An IAM role is an IAM identity that you can create in your account that has specific permissions.
- IAM role is a secure way to grant permissions between different entities that we trust such as AWS account to Snowflake account
- Create IAM role
- Trusted entity type is AWS account (This account)
- Require external ID (Best practice - This ID will be snowflake ID)
- Select permission such as AmazonS3FullAccess
- Access role summary - trust relationships show trusted entity
- Edit trust policy JSON file

### Creating Integration Object


```sql
// Create storage integration object

// create storage integration called s3_int
create or replace storage integration s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  // enable integration object
  ENABLED = TRUE
  // storage AWS role arn from IAM AWS
  STORAGE_AWS_ROLE_ARN = ''
  // allow access to multiple buckets or paths to subfolders
  STORAGE_ALLOWED_LOCATIONS = ('s3://<your-bucket-name>/<your-path>/', 's3://<your-bucket-name>/<your-path>/')
   COMMENT = 'This an optional comment'
   // initially, SQL access control error: insufficient privileges to operate on account, select ACCOUNTADMIN role

// Successfully created integration object
// See storage integration properties to fetch external_id so we can update it in S3

DESC integration s3_int;

// copy STORAGE_AWS_IAM_USER_ARN and paste in AWS trust polcy JSON "AWS":
// copy STORAGE_AWS_EXTERNAL_ID and paste in AWS trust polcy "sts:ExternalID":
```

### Loading from S3
- Use integration with stage object to load data from S3 buckets
- create storage integration object with access information to stage
- Attach storage integration object to stage object
- use COPY INTO command to load data from s3 into created table


```sql
// Create table first
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.movie_titles (
  show_id STRING,
  type STRING,
  title STRING,
  director STRING,
  cast STRING,
  country STRING,
  date_added STRING,
  release_year STRING,
  rating STRING,
  duration STRING,
  listed_in STRING,
  description STRING )



// Create file format object
CREATE OR REPLACE file format MANAGE_DB.file_formats.csv_fileformat
    type = csv
    field_delimiter = ','
    skip_header = 1
    // if string value as NULL or null, convert to database null
    null_if = ('NULL','null')
    // if empty field (no value) also convert to null
    empty_field_as_null = TRUE;


 // Create stage object with integration object & file format object
 // create or replace stage with full qualified stage name
CREATE OR REPLACE stage MANAGE_DB.external_stages.csv_folder
    // s3 url and path to folder
    URL = 's3://<your-bucket-name>/<your-path>/'
    // use created storage integration object and attach to stage
    STORAGE_INTEGRATION = s3_int
    // use file format object and attach to stage
    FILE_FORMAT = MANAGE_DB.file_formats.csv_fileformat



// Use Copy command       
COPY INTO OUR_FIRST_DB.PUBLIC.movie_titles
    FROM @MANAGE_DB.external_stages.csv_folder
    // Error: number of columns in file 26 does not match that of corresponding table 12
    // commas interpreted as column delimiters but we want commas in data so enclose using text qualifiers " " (double quote)



// Create file format object
// adjust file format object
CREATE OR REPLACE file format MANAGE_DB.file_formats.csv_fileformat
    type = csv
    field_delimiter = ','
    skip_header = 1
    null_if = ('NULL','null')
    empty_field_as_null = TRUE    
    // some values can be enclosed by a text qualifier
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'   

// Use Copy command   
COPY INTO OUR_FIRST_DB.PUBLIC.movie_titles
    FROM @MANAGE_DB.external_stages.csv_folder
    // LOADED 7787 rows

SELECT * FROM OUR_FIRST_DB.PUBLIC.movie_titles
```

### Handling json



```sql
// Handling JSON files

// Create file format object
CREATE OR REPLACE file format MANAGE_DB.file_formats.json_fileformat
    type = JSON

// Create stage object with integration object & file format object
// create or replace stage with full qualified stage name
CREATE OR REPLACE stage MANAGE_DB.external_stages.json_folder
    // s3 url and path to json folder
    URL = 's3://<your-bucket-name>/<your-path>/'  
    // use created storage integration object and attach to stage
    STORAGE_INTEGRATION = s3_int
    // use json file format object and attach to stage
    FILE_FORMAT = MANAGE_DB.file_formats.json_fileformat



// Taming the JSON file

// First query from S3 Bucket   
// View structure of JSON file, Data type Variant
SELECT * FROM @MANAGE_DB.external_stages.json_folder



// Introduce columns
// use objects in json files by selecting column $1 variant column
// use colon to navigate to the object and use as column in snowflake
SELECT
$1:asin,
$1:helpful,
$1:overall,
$1:reviewText,
$1:reviewTime,
$1:reviewerID,
$1:reviewTime,
$1:reviewerName,
$1:summary,
$1:unixReviewTime
FROM @MANAGE_DB.external_stages.json_folder

// Format columns & use DATE function
SELECT
// quotes converted to string using ::
$1:asin::STRING as ASIN,
$1:helpful as helpful,
$1:overall as overall,
$1:reviewText::STRING as reviewtext,
$1:reviewTime::STRING,
$1:reviewerID::STRING,
$1:reviewTime::STRING,
$1:reviewerName::STRING,
$1:summary::STRING,
// unixreviewtime is time as an integer
// Convert review time to int and
// handle using DATE function which converts int into date format
DATE($1:unixReviewTime::int) as Revewtime
FROM @MANAGE_DB.external_stages.json_folder

// Format columns & handle custom date
// reviewtime column stored as custom data as string
// cannot run data functions or query on data function format
SELECT
$1:asin::STRING as ASIN,
$1:helpful as helpful,
$1:overall as overall,
$1:reviewText::STRING as reviewtext,
DATE_FROM_PARTS( <year>, <month>, <day> )
$1:reviewTime::STRING,
$1:reviewerID::STRING,
$1:reviewTime::STRING,
$1:reviewerName::STRING,
$1:summary::STRING,
DATE($1:unixReviewTime::int) as Revewtime
FROM @MANAGE_DB.external_stages.json_folder

// Use DATE_FROM_PARTS and see another difficulty
SELECT
$1:asin::STRING as ASIN,
$1:helpful as helpful,
$1:overall as overall,
$1:reviewText::STRING as reviewtext,
// DATE FROM PARTS FUNCTION DATE_FROM_PARTS( <year>, <month>, <day> )
// convert to string, use 4 characters from RIGHT (year)
// use first 2 characters from LEFT (month)
// use SUBSTRING function to extract (day)
DATE_FROM_PARTS( RIGHT($1:reviewTime::STRING,4), LEFT($1:reviewTime::STRING,2), SUBSTRING($1:reviewTime::STRING,4,2) ),
$1:reviewerID::STRING,
$1:reviewTime::STRING,
$1:reviewerName::STRING,
$1:summary::STRING,
DATE($1:unixReviewTime::int) as unixRevewtime
FROM @MANAGE_DB.external_stages.json_folder
// Error: Numeric value "6," is not recognised - 6 and comma extracted from substring as data is not clean as only one didgit used for day


// Use DATE_FROM_PARTS and handle the case difficulty
SELECT
$1:asin::STRING as ASIN,
$1:helpful as helpful,
$1:overall as overall,
$1:reviewText::STRING as reviewtext,
DATE_FROM_PARTS(
  RIGHT($1:reviewTime::STRING,4),
  LEFT($1:reviewTime::STRING,2),
  // CASE to detect comma or not
  // IF extracted value is a comma, then use a different case
  CASE WHEN SUBSTRING($1:reviewTime::STRING,5,1)=','
        // THEN extract correct values for comma ELSE extract 2 digit values as normal (dates 10, 21)
        THEN SUBSTRING($1:reviewTime::STRING,4,1) ELSE SUBSTRING($1:reviewTime::STRING,4,2) END),
$1:reviewerID::STRING,
$1:reviewTime::STRING,
$1:reviewerName::STRING,
$1:summary::STRING,
DATE($1:unixReviewTime::int) as UnixRevewtime
FROM @MANAGE_DB.external_stages.json_folder
// DATE_FROM_PARTS successful


// Create destination table
CREATE OR REPLACE TABLE OUR_FIRST_DB.PUBLIC.reviews (
asin STRING,
helpful STRING,
overall STRING,
reviewtext STRING,
reviewtime DATE,
reviewerid STRING,
reviewername STRING,
summary STRING,
unixreviewtime DATE
)

// Copy transformed data into destination table // LOADED 10261 rows, error limit 1, errors seen 0
COPY INTO OUR_FIRST_DB.PUBLIC.reviews
    FROM (SELECT
$1:asin::STRING as ASIN,
$1:helpful as helpful,
$1:overall as overall,
$1:reviewText::STRING as reviewtext,
DATE_FROM_PARTS(
  RIGHT($1:reviewTime::STRING,4),
  LEFT($1:reviewTime::STRING,2),
  CASE WHEN SUBSTRING($1:reviewTime::STRING,5,1)=','
        THEN SUBSTRING($1:reviewTime::STRING,4,1) ELSE SUBSTRING($1:reviewTime::STRING,4,2) END),
$1:reviewerID::STRING,
$1:reviewerName::STRING,
$1:summary::STRING,
DATE($1:unixReviewTime::int) Revewtime
FROM @MANAGE_DB.external_stages.json_folder)


// Validate results
SELECT * FROM OUR_FIRST_DB.PUBLIC.reviews

```
