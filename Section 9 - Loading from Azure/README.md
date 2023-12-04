## Loading Data from Azure

## Create a storage account
- Create a new storage account
- Subscription with multiple resource groups
  - create new resource group
  - create storage account name with same region as snowflake account
  - redundancy is locally redundant storage

## Create a container and load files in container
- Create snowflakecsv and snowflakejson containers
- Upload csv and json files into created containers


## Create integration object
- Set up connection from snowflake to access containers in Azure using integration object containing access information to the containers
- Azure Tenant ID found in Microsoft Entra ID
- Storage Allowed locations (path to container or sub path of container (folders), or access to a certain file) Can specify multiple locations / containers
- Create integration object

```sql
USE DATABASE DEMO_DB;
-- create integration object that contains the access information
// create integration object with name
CREATE STORAGE INTEGRATION azure_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  ENABLED = TRUE
  // Azure tenant id found in Microsoft Entra ID Overview
  AZURE_TENANT_ID = ''
  // Storage Locations - path to container, sub path folder, or specific file
  // account name.
  // give access to both json and csv containers
  STORAGE_ALLOWED_LOCATIONS = ('azure://<storageaccountname>.blob.core.windows.net/<your-container/path/data', 'azure://storageaccountsnow.blob.core.windows.net/snowflakejson');


-- Describe integration object to provide access
DESC STORAGE integration azure_integration;

```

- Describe Storage integration object and click on AZURE_CONSENT_URL
- Permission requested: Grant permission to our integration object so integration object has access to Azure containers
- Consent and Accept



### Azure Storage Account - Add Role Assignment

- Navigate to Azure Storage Account and IAM
- Add role assignment
- Need to specify what integration object is allowed to do
- Search and Select Storage Blob Data Contributor - allows for read, write, delete access to Azure Storage blob containers and data
- Use Describe statement to describe integration object to provide access and copy AZURE_MULTI_TENANT_APP_NAME

```sql
DESC STORAGE integration azure_integration;
```
- Selected role is Storage Blob Data Contributor
- Assign access to User, group, or service principal
- Select Memebers AZURE_MULTI_TENANT_APP_NAME - snowflakepacint
- Review and assign


## Create stage object
- create file format object of type csv
- create stage object with storage integration object, URL of container location, file format object
- list files to verify

```sql
--- Create file format & stage objects ----

-- create file format
create or replace file format demo_db.public.fileformat_azure
    TYPE = CSV
    FIELD_DELIMITER = ','
    // set skip header option to 0 to see column names initially for querying
    SKIP_HEADER = 0;

-- create stage object
create or replace stage demo_db.public.stage_azure
    // use integration object for secure access information and permissions to query and work on contaienrs
    STORAGE_INTEGRATION = azure_integration
    // container location
    URL = 'azure://storageaccountsnow.blob.core.windows.net/snowflakecsv'
    // file format object
    FILE_FORMAT = fileformat_azure;


-- list files
LIST @demo_db.public.stage_azure; // 1 csv file size 21687
```
## Load CSV file

```sql
---- Query files & Load data ----

--query files in CSV by mentioning column number in snowflake
SELECT
$1,
$2,
$3,
$4,
$5,
$6,
$7,
$8,
$9,
$10,
$11,
$12,
$13,
$14,
$15,
$16,
$17,
$18,
$19,
$20
// from stage using @
FROM @demo_db.public.stage_azure;

// column names shown as skip header is set to 0

// create table with column name and change file format to skip header set to 1
create or replace table happiness (
    country_name varchar,
    regional_indicator varchar,
    ladder_score number(4,3), // number (4,3) where 4 is max precision of 4 digits and 3 is scale where max scale is 3 decimal palces
    standard_error number(4,3),
    upperwhisker number(4,3),
    lowerwhisker number(4,3),
    logged_gdp number(5,3),
    social_support number(4,3),
    healthy_life_expectancy number(5,3),
    freedom_to_make_life_choices number(4,3),
    generosity number(4,3),
    perceptions_of_corruption number(4,3),
    ladder_score_in_dystopia number(4,3),
    explained_by_log_gpd_per_capita number(4,3),
    explained_by_social_support number(4,3),
    explained_by_healthy_life_expectancy number(4,3),
    explained_by_freedom_to_make_life_choices number(4,3),
    explained_by_generosity number(4,3),
    explained_by_perceptions_of_corruption number(4,3),
    dystopia_residual number (4,3));

// copy data into table
COPY INTO HAPPINESS
FROM @demo_db.public.stage_azure; // status LOADED 149 rows loaded 0 errors seen

SELECT * FROM HAPPINESS;
```

## Load JSON file

```sql
--- Load JSON ----

// create file format object for JSON
create or replace file format demo_db.public.fileformat_azure_json
    TYPE = JSON;




// create stage object for JSON
// we have allowed location of JSON file and CSV file in integration object already using STORAGE_ALLOWED_LOCATIONS
create or replace stage demo_db.public.stage_azure
    STORAGE_INTEGRATION = azure_integration
    URL = 'azure://storageaccountsnowflake3.blob.core.windows.net/snowflakejson'
    FILE_FORMAT = fileformat_azure_json;

LIST  @demo_db.public.stage_azure; // 1 JSON file size 38694
-- For JSON file, we can quickly and directly query from the file
-- Query from stage  
SELECT * FROM @demo_db.public.stage_azure;  // result shows one column of data type variant


-- Query one attribute/column
SELECT $1:"Car Model" FROM @demo_db.public.stage_azure;

-- Convert data type  using ::
SELECT $1:"Car Model"::STRING FROM @demo_db.public.stage_azure;

-- Query all attributes  
SELECT
$1:"Car Model"::STRING,
$1:"Car Model Year"::INT,
$1:"car make"::STRING,
$1:"first_name"::STRING,
$1:"last_name"::STRING
FROM @demo_db.public.stage_azure;   

-- Query all attributes and use aliases
SELECT
$1:"Car Model"::STRING as car_model,
$1:"Car Model Year"::INT as car_model_year,
$1:"car make"::STRING as "car make", // if alias column name has space, use quotes
$1:"first_name"::STRING as first_name,
$1:"last_name"::STRING as last_name
FROM @demo_db.public.stage_azure;     

// create table
Create or replace table car_owner (
    car_model varchar,
    car_model_year int,
    car_make varchar,
    first_name varchar,
    last_name varchar)

// copy into to load data directly from stage
// stage has data type variant
// use select statement to transform columns as needed
COPY INTO car_owner
FROM
(SELECT
$1:"Car Model"::STRING as car_model,
$1:"Car Model Year"::INT as car_model_year,
$1:"car make"::STRING as "car make",
$1:"first_name"::STRING as first_name,
$1:"last_name"::STRING as last_name
FROM @demo_db.public.stage_azure);
// status LOADED 324 rows loaded

SELECT * FROM CAR_OWNER;


-- Alternative: Using a raw file table step
// it is best practice to introduce an intermediary step
truncate table car_owner;
select * from car_owner;

// create raw table with variant data type for more control to trace errors easily
create or replace table car_owner_raw (
  raw variant);

// copy rows of data of variant type directly into raw file table from stage
COPY INTO car_owner_raw
FROM @demo_db.public.stage_azure; // status LOADED

SELECT * FROM car_owner_raw; // result shows variant data type, one column storage


INSERT INTO car_owner  
(SELECT
$1:"Car Model"::STRING as car_model,
$1:"Car Model Year"::INT as car_model_year,
$1:"car make"::STRING as car_make,
$1:"first_name"::STRING as first_name,
$1:"last_name"::STRING as last_name
FROM car_owner_raw)  
// 324 rows inserted

// final table
select * from car_owner;
```
