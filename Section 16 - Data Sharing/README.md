## Data Sharing

Secure Data Sharing lets you share selected objects in a database in your account with other Snowflake accounts. You can share the following Snowflake database objects:

- External tables
- Dynamic tables
- Secure views
- Secure materialized views
- Secure UDFs
- Tables

## Overview
- Usually this can be a complicated processes due to copying of data and extracting and updated changes in database
- In snowflake, data sharing without actual copy of the data & update
- Shared data can be consumed by the own compute resources
- Non snowflake users can also access through a reader account
- Example: Account 1 Producer shares to Account 2 consumer (read-only and uses own compute resources)
- Create a reader account in snowflake instance to share data to non snowflake users

## Using data sharing

```sql
CREATE OR REPLACE DATABASE DATA_S;


CREATE OR REPLACE STAGE aws_stage
    url='s3://bucketsnowflakes3';

// List files in stage
LIST @aws_stage; // 3 CSV files

// Create table to share to another account
CREATE OR REPLACE TABLE ORDERS (
ORDER_ID	VARCHAR(30)
,AMOUNT	NUMBER(38,0)
,PROFIT	NUMBER(38,0)
,QUANTITY	NUMBER(38,0)
,CATEGORY	VARCHAR(30)
,SUBCATEGORY	VARCHAR(30))   


// Load data using copy command
// 1500 rows LOADED
COPY INTO ORDERS
    FROM @MANAGE_DB.external_stages.aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*OrderDetails.*';

// Validate data is loaded
SELECT * FROM ORDERS;




// Create a share object
CREATE OR REPLACE SHARE ORDERS_SHARE;

---- Setup Grants ----

// Grant usage on database
// to share object
GRANT USAGE ON DATABASE DATA_S TO SHARE ORDERS_SHARE;

// Grant usage on schema
GRANT USAGE ON SCHEMA DATA_S.PUBLIC TO SHARE ORDERS_SHARE;

// Grant SELECT on table

GRANT SELECT ON TABLE DATA_S.PUBLIC.ORDERS TO SHARE ORDERS_SHARE;

// Validate Grants
// table shows created grant time stamp and privilege, granted from and granted to etc
SHOW GRANTS TO SHARE ORDERS_SHARE;


---- Add Consumer Account ----
ALTER SHARE ORDERS_SHARE ADD ACCOUNT=<consumer-account-id>;

```

```sql
// In consumer account

// Show all shares (consumer and producers)
SHOW SHARES;

// See details on shares
DESC SHARE <consumer-account-id>.ORDERS_SHARE;

// Create database in consumer account using the share
CREATE DATABASE DATA_S FROM SHARE <consumer-account-id>.ORDERS_SHARE;

// Validate table access and view shared data in consumer account
SELECT * FROM DATA_S.PUBLIC.ORDERS

```

## Create share through interface
- Shares tab
- Switch role to ACCOUNTADMIN
- View and configure INBOUND AND OUTBOUND shares
- Create a secure share and add database objects to it
- Select database, tables from schema to share
- Add comment to share object
- Share is created
- Add Consumers with either Reader (non-snowflake user) or Full account type
- Manage and edit all shares within interface (including dropping share or revoking / granting access)

## Sharing with non-snowflake users
- Create independent reader account within our snowflake account
- Compute resources are paid by our snowflake account creating the reader accounts.
- If we share data with a non-snowflake user using a reader account, a dedicated virtual warehouse of the provider account has to be set up
- Reader accounts (formerly known as “read-only accounts”) provide a quick, easy, and cost-effective way to share data without requiring the consumer to become a Snowflake customer.
Each reader account belongs to the provider account that created it. As a provider, you use shares to share databases with reader accounts; however, a reader account can only consume data from the provider account that created it

### In producer snowflake account
- New Reader account
  - Independent instance with own url and own compute resources
- Share data
  - Share database and table
### In producer snowflake account
- Create users
  - As administrator create users and roles (multiple log ins meaning multiple users can access reader account and consume data)
- Create database
  - in reader account create database from share


```sql

-- Create Reader Account --
// for external company use case for company called tech_joy

CREATE MANAGED ACCOUNT tech_joy_account
// account admin can create users and act as ACCOUNTADMIN role
ADMIN_NAME = tech_joy_admin,
ADMIN_PASSWORD = 'set-PWD12345',
TYPE = READER;

// Make sure to have selected the role of accountadmin

// Show accounts
// view url for log in
SHOW MANAGED ACCOUNTS;


-- Share the data --
// using previously created share
ALTER SHARE ORDERS_SHARE
ADD ACCOUNT = JUB34756; // use reader locator id and share the data
//ADD ACCOUNT = <reader-account-id>;

// Sharing is not allowed from an account on BUSINESS CRITICAL edition to an account on a lower edition.


ALTER SHARE ORDERS_SHARE
ADD ACCOUNT =  JUB34756
// override this error by manuallying setting restriction to false
// best practice is to create another account with subset of data that is not sensitive or business account and then share for security
SHARE_RESTRICTIONS=false;



-- Create database from share --

// Copy reader account URL to log in to reader account (own independent instance)
// username tech_joy_admin
// password set-PWD12345
SHOW MANAGED ACCOUNTS;

// in consumer reader account (new tab)
// Show all shares (consumer & producers)
// for INBOUND share view owner account and name of share  
SHOW SHARES;

// See details on share using owner account and name of share
DESC SHARE AGJGJVW.XOB50846.ORDERS_SHARE;

// Create a database in consumer account using the share
// CREATE DATABASE DATA_SHARE_DB FROM SHARE <account_name_producer>.ORDERS_SHARE;
CREATE DATABASE DATA_SHARE_DB FROM SHARE AGJGJVW.XOB50846.ORDERS_SHARE;


// Validate table access (requires virtual warehouse in reader account)
SELECT * FROM  DATA_SHARE_DB.PUBLIC.ORDERS;


// Setup virtual warehouse as it is an independent account
// warehouse in reader account uses compute resources
CREATE WAREHOUSE READ_WH WITH
WAREHOUSE_SIZE='X-SMALL'
AUTO_SUSPEND = 180
AUTO_RESUME = TRUE
INITIALLY_SUSPENDED = TRUE;

// Validate table access and view shared data
SELECT * FROM  DATA_SHARE_DB.PUBLIC.ORDERS;
```




```sql
//--------------------------
// Log into tech_joy reader account in new tab
// using ACCOUNTADMIN role
// show all shares (consumer and producer)
SHOW SHARES;
//COMPLETE_SCHEMA_SHARE exists

// See details of share
// displays database schema and tables with date/time shared on
DESC SHARE AGJGJVW.XOB50846.COMEPLETE_SCHEMA_SHARE

// Create a database in consumer account using the share
// Create new database and use share fully qualified path
CREATE DATABASE OUR_FIRST_DB_SHARE FROM SHARE AGJGJVW.XOB50846.COMEPLETE_SCHEMA_SHARE;

// Validate table access and view data
SELECT * FROM OUR_FIRST_DB_SHARE.PUBLIC.ORDERS;
// Validate other tables
SELECT * FROM OUR_FIRST_DB_SHARE.PUBLIC.EMPLOYEES;


//--------------------------


// if we add a new table in the producer account, we must apply grants to the reader account to access the table

// if we update data of an existing table
// this will be represented in reader account where data is consumed
// Updating data is immediately reflected in reader account when select statement run
UPDATE OUR_FIRST_DB.PUBLIC.ORDERS
SET PROFIT=0 WHERE PROFIT < 0;

// Add new table
// creating a new table only visible in producer account
// consumer account cannot view tables unless grant usage on database and schema (this can be done by using GRANT USAGE commands on all tables above
CREATE TABLE OUR_FIRST_DB.PUBLIC.NEW_TABLE (ID int);

```



## Secure vs Normal view



```sql

-- Create database & table --
CREATE OR REPLACE DATABASE CUSTOMER_DB;

// create table that view is based on
CREATE OR REPLACE TABLE CUSTOMER_DB.public.customers (
   id int,
   first_name string,
  last_name string,
  email string,
  gender string,
  Job string,
  Phone string);


// Stage and file format
CREATE OR REPLACE FILE FORMAT MANAGE_DB.file_formats.csv_file
    type = csv
    field_delimiter = ','
    skip_header = 1;
// external stage pointing to public s3 bucket
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.time_travel_stage
    URL = 's3://data-snowflake-fundamentals/time-travel/'
    file_format = MANAGE_DB.file_formats.csv_file;

LIST  @MANAGE_DB.external_stages.time_travel_stage; // customers csv file


// Copy data and insert in table
COPY INTO CUSTOMER_DB.public.customers
FROM @MANAGE_DB.external_stages.time_travel_stage
files = ('customers.csv');

// verify data is LOADED
SELECT * FROM  CUSTOMER_DB.PUBLIC.CUSTOMERS;

// Only want to show a view of the data
// Select which columns to share where job is not data scientist

-- Create VIEW object --
CREATE OR REPLACE VIEW CUSTOMER_DB.PUBLIC.CUSTOMER_VIEW AS
SELECT
FIRST_NAME,
LAST_NAME,
EMAIL
FROM CUSTOMER_DB.PUBLIC.CUSTOMERS
WHERE JOB != 'DATA SCIENTIST';


-- Grant usage & SELECT --
GRANT USAGE ON DATABASE CUSTOMER_DB TO ROLE PUBLIC;
GRANT USAGE ON SCHEMA CUSTOMER_DB.PUBLIC TO ROLE PUBLIC;
GRANT SELECT ON TABLE CUSTOMER_DB.PUBLIC.CUSTOMERS TO ROLE PUBLIC;
// grant select on view to public role
GRANT SELECT ON VIEW CUSTOMER_DB.PUBLIC.CUSTOMER_VIEW TO ROLE PUBLIC;

// In a new session / worksheet, select PUBLIC role and SHOW view
// Definition of view shows SQL statement of object creation
// It shows information such as column job and data scientist which we may not want to share
SHOW VIEWS LIKE '%CUSTOMER%';



// Therefore secure view provides more data protection
-- Create SECURE VIEW --
CREATE OR REPLACE SECURE VIEW CUSTOMER_DB.PUBLIC.CUSTOMER_VIEW_SECURE AS
SELECT
FIRST_NAME,
LAST_NAME,
EMAIL
FROM CUSTOMER_DB.PUBLIC.CUSTOMERS
WHERE JOB != 'DATA SCIENTIST' ;

GRANT SELECT ON VIEW CUSTOMER_DB.PUBLIC.CUSTOMER_VIEW_SECURE TO ROLE PUBLIC;

// In a new session / worksheet
// text definition is no longer visible showing secure view creation
SHOW VIEWS LIKE '%CUSTOMER%';
```

## Sharing a secure view

With Secure Data Sharing, no actual data is copied or transferred between accounts. All sharing uses Snowflake’s services layer and metadata store. Shared data does not take up any storage in a consumer account and therefore does not contribute to the consumer’s monthly data storage charges. The only charges to consumers are for the compute resources (i.e. virtual warehouses) used to query the shared data.

```sql
SHOW SHARES;

// Create share object
CREATE OR REPLACE SHARE VIEW_SHARE;

// Grant usage on dabase & schema to created share object
GRANT USAGE ON DATABASE CUSTOMER_DB TO SHARE VIEW_SHARE;
GRANT USAGE ON SCHEMA CUSTOMER_DB.PUBLIC TO SHARE VIEW_SHARE;

// Grant select on view
// using normal view that we created previously
// A view can only be shared if it is created as a SECURE view, or marked SECURE using ALTER VIEW CUSTOMER_VIEW SET SECURE.
GRANT SELECT ON VIEW  CUSTOMER_DB.PUBLIC.CUSTOMER_VIEW TO SHARE VIEW_SHARE;
// As only secure view can be shared
// Grant select on secure view
GRANT SELECT ON VIEW  CUSTOMER_DB.PUBLIC.CUSTOMER_VIEW_SECURE TO SHARE VIEW_SHARE;


// Add account to share
ALTER SHARE VIEW_SHARE
ADD ACCOUNT=JUB34756;

// Log in to consumer account
// Tech_joy reader account to consume share

// using ACCOUNTADMIN role
// show all shares (consumer and producer)
SHOW SHARES;
// VIEW_SHARE exists

// See details of share
// diplays database schema and tables with date/time shared on
DESC SHARE AGJGJVW.XOB50846.VIEW_SHARE

// Create a database in consumer account using the share
// Create new database called view_share db from the share
CREATE DATABASE VIEW_SHARE_DB FROM SHARE AGJGJVW.XOB50846.VIEW_SHARE;
// database object shows datbase with secure view

// Validate table access and view data
SELECT * FROM VIEW_SHARE_DB.PUBLIC.CUSTOMER_VIEW_SECURE;
// Secure view only shows consumer account (reader account) specified columns first_name, last_name and email where no data where job column is data scientist rows are shared

```

## Assignment 12 - Data sharing

```sql
// Assume we want to create a share from the database CUSTOMER_DB and share the table CUSTOMERS.
// Database: CUSTOMER_DB
// Schema: PUBLIC
// Table: CUSTOMERS
// Provider account: asd163257
// Consumer account: sdw135733

// 1. How can we create a share object called CUSTOMER_SHARE and grant sufficient grant privileges (use sql commands)?

SHOW SHARES;

// Create share object
CREATE OR REPLACE SHARE CUSTOMER_SHARE;

// Grant usage on dabase & schema to created share object
GRANT USAGE ON DATABASE CUSTOMER_DB TO SHARE CUSTOMER_SHARE;
GRANT USAGE ON SCHEMA CUSTOMER_DB.PUBLIC TO SHARE CUSTOMER_SHARE;
GRANT SELECT ON TABLE CUSTOMER_DB.PUBLIC.CUSTOMERS TO SHARE CUSTOMER_SHARE;

// 2. How can we add the consumer account (sql commands)?

// Add account to share
ALTER SHARE CUSTOMER_SHARE ADD ACCOUNTS=sdw135733;
```
