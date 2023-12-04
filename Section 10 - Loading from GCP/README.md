## Loading Data from GCP

## Create a bucket in GCP
- Select region us (multiple regions in United States)
- create a Google Cloud bucket for storing CSV file and one one for storing JSON file
- Upload CSV and JSON files to relevant bucket


## Set up connection from snowflake to GC bucket using Integration Object


```sql
USE DATABASE DEMO_DB;
USE ROLE ACCOUNTADMIN;

-- create integration object that contains the access information
CREATE STORAGE INTEGRATION gcp_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = GCS
  ENABLED = TRUE
  // One or more buckets, subfolders, or files
  STORAGE_ALLOWED_LOCATIONS = ('gcs://bucket/path', 'gcs://bucket/path2');

-- Describe integration object to provide access
DESC STORAGE integration gcp_integration;

```

- Use STORAGE_GCP_SERVICE_ACCOUNT property and copy link
- In GCP Cloud Storage Buckets
- Select checkbox of both buckets
- Permissions
- Grant access to 2 resources
- Add principal: Paste in STORAGE_GCP_SERVICE_ACCOUNT property
- Assign role: Cloud Storage - Storage Admin
- Save


## Create Stage

```sql
-- create file format for CSV
create or replace file format demo_db.public.fileformat_gcp
    TYPE = CSV
    FIELD_DELIMITER = ','
    SKIP_HEADER = 0; // 0 to view column headers initially, set to 1 when loading data into tables

-- create stage object
create or replace stage demo_db.public.stage_gcp
    STORAGE_INTEGRATION = gcp_integration
    URL = 'gcs://snowflakebucketgcp'
    FILE_FORMAT = fileformat_gcp;

LIST @demo_db.public.stage_gcp; // 1 csv file size 21687
```

## Query and Load Data
- For CSV files, mention the column number and query from file


```sql
---- Query files & Load data ----

--query files
SELECT
$1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,
$12,$13,$14,$15,$16,$17,$18,$19,$20
FROM @demo_db.public.stage_gcp;

// create table with column names and transformed data types
create or replace table happiness (
    country_name varchar,
    regional_indicator varchar,
    ladder_score number(4,3),
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

// change file format skip header to 1 before loading

// load data from stage using @
COPY INTO HAPPINESS
FROM @demo_db.public.stage_gcp;
// status LOADED 149 rows loaded

// final table
SELECT * FROM HAPPINESS;

```

- We have set up the integration object with provided read access and write access.
- For both buckets, permissions set to grant access to principal service user with assigned storage admin.
- As Service user has read access and write access, we can Unload files. Write data from our table to the bucket (requires sufficient privileges - we have set up integration object with admin role )

## Unload data
- Put data from table and create a file and place file into external stage
- Requires service user of integration object to be set up with storage admin permissions to write to buckets


```sql
------- Unload data -----
USE ROLE ACCOUNTADMIN;
USE DATABASE DEMO_DB;


-- create integration object that contains the access information
CREATE or REPLACE STORAGE INTEGRATION gcp_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = GCS
  ENABLED = TRUE
  STORAGE_ALLOWED_LOCATIONS = ('gcs://snowflakebucketgcp', 'gcs://snowflakebucketgcpjson');



-- create file format
create or replace file format demo_db.public.fileformat_gcp
    TYPE = CSV
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1;

-- create stage object
// create a file location that does not yet exist
create or replace stage demo_db.public.stage_gcp
    STORAGE_INTEGRATION = gcp_integration
    // specify a path in bucket that does not exist, csv_happiness will be new folder in bucket
    URL = 'gcs://snowflakebucketgcp/csv_happiness'
    FILE_FORMAT = fileformat_gcp
    // specify a compression
    -- compression = gzip | auto
    // default is auto meaning compression algorithm method will be determined automatically or can specify compression method such as gzip

    ;

// can alter the locations that is allowed for integration object by specifying different buckets or paths
ALTER STORAGE INTEGRATION gcp_integration
SET  storage_allowed_locations=('gcs://snowflakebucketgcp', 'gcs://snowflakebucketgcpjson')

SELECT * FROM HAPPINESS; // view data in HAPPINESS table which is the data we want to unload to new URL

// unload data into stage from table
COPY INTO @stage_gcp
FROM
HAPPINESS;
// rows unloaded 149 input_bytes 21052 output_bytes 8030
```

- Validate in GCP
- csv_happiness folder created in bucket
- gzip compressed data file is stored in folder to be used or downloaded
