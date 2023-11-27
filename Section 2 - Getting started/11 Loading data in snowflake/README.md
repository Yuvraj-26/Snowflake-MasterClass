## Setting up Warehouse

Minimum role SYSADMIN needed (use ACCOUNTADMIN)
```sql
USE ROLE SYSADMIN;

-- Create Warehouse
CREATE WAREHOUSE EXERCISE_WH
WAREHOUSE_SIZE = XSMALL
AUTO_SUSPEND = 600 -- automatically suspend the virtual warehouse after 10 minutes of not being used
AUTO_RESUME = TRUE
COMMENT = 'This is a virtual warehouse of size X-SMALL that can be used to process queries.';

-- Drop Warehouse
DROP WAREHOUSE EXERCISE_WH;
```

## Assignment 1: Creating a Virtual Data Warehouse

```sql
CREATE OR REPLACE WAREHOUSE COMPUTE_WAREHOUSE
WITH
  WAREHOUSE_SIZE = XSMALL,
  MAX_CLUSTER_COUNT = 3,
  AUTO_SUSPEND = 300,
  AUTO_RESUME = TRUE,
  INITIALLY_SUSPENDED = TRUE,
  COMMENT = 'This is our second warehouse';
```

## Scaling Policy

### Multi-Clustering

Multi-cluster warehouses enable you to scale compute resources to manage your user and query concurrency needs as they change, such as during peak and off hours.

By default, a virtual warehouse consists of a single cluster of compute resources available to the warehouse for executing queries. As queries are submitted to a warehouse, the warehouse allocates resources to each query and begins executing the queries. If sufficient resources are not available to execute all the queries submitted to the warehouse, Snowflake queues the additional queries until the necessary resources become available.

With multi-cluster warehouses, Snowflake supports allocating, either statically or dynamically, additional clusters to make a larger pool of compute resources available. A multi-cluster warehouse is defined by specifying the following properties:

- Maximum number of clusters, greater than 1 (up to 10).
- Minimum number of clusters, equal to or less than the maximum (up to 10).

Automatically depending on the workload, dynamically shut down or start additional clusters, allow redistributing of queries that cannot be processed by a single warehouses

Additionally, multi-cluster warehouses support all the same properties and actions as single-cluster warehouses, including:

- Specifying a warehouse size.
- Resizing a warehouse at any time.
- Auto-suspending a running warehouse due to inactivity; note that this does not apply to individual clusters, but rather the entire multi-cluster warehouse.
- Auto-resuming a suspended warehouse when new queries are submitted.

Use case for multi-clustering is for more workload and more queries. This can mean more users at certain times.
Use case not well suited for multi-clustering is if more complex queries, in this case increase size of the existing warehouse from size S to M for example.

### Auto-Scaling for Multi-Cluster
- If there are more queries than can be processed by a single warehouse, a queue of additional queries exists
- Multi-clusters automatically start new clusters
- Auto-Scaling: When to start an additional cluster?

### Scaling Policy
- Standard - Favours starting additional warehouses
- Economy - Favours conserving credits rather than starting additional warehouses

### Policy

| Policy               |Description                                                                                                                             | Cluster Starts                                                                                                                                      | Cluster Shuts Down                                                                                                                                        |
|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Standard (default)   | Prevents/minimizes queuing by favouring starting additional clusters over conserving credits.                                            | The first cluster starts immediately when a query is queued or one more query than running clusters can handle is detected. Successive clusters start 20 seconds after the prior one. | After 2-3 consecutive successful checks (at 1-minute intervals), load redistribution occurs without spinning up the cluster again.                       |
| Economy              | Conserves credits by favouring keeping running clusters fully-loaded rather than starting additional clusters, resulting in potential query queueing. Result: May result in queries being queued and taking longer to complete. | Starts only if the system estimates enough query load for at least 6 minutes. After 5-6 consecutive successful checks (at 1-minute intervals), load redistribution occurs without spinning up the cluster again.|After 5 to 6 consecutive successful checks (performed at 1 minute intervals), which determine whether the load on the least-loaded cluster could be redistributed to the other clusters without spinning up the cluster again.|

### Loading Data in Snowflake from External Source (S3 Bucket)


```sql
//Rename database & creating the table + metadata
ALTER DATABASE FIRST_DB RENAME TO OUR_FIRST_DB;



//Creating the table / Meta data
CREATE TABLE "OUR_FIRST_DB"."PUBLIC"."LOAN_PAYMENT" (
  "Loan_ID" STRING,
  "loan_status" STRING,
  "Principal" STRING,
  "terms" STRING,
  "effective_date" STRING,
  "due_date" STRING,
  "paid_off_time" STRING,
  "past_due_days" STRING,
  "age" STRING,
  "education" STRING,
  "Gender" STRING);

//Check that table is empty
USE DATABASE OUR_FIRST_DB;

SELECT * FROM LOAN_PAYMENT;

//Loading the data from S3 bucket

COPY INTO LOAN_PAYMENT
    FROM s3://bucketsnowflakes3/Loan_payments_data.csv
    file_format = (type = csv
                   field_delimiter = ','
                   skip_header=1);

//Validate
 SELECT * FROM LOAN_PAYMENT;
 ```

 ## Assignment 2: Load data

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

//Check that table is empty
USE EXERCISE_DB

//Check empty table
SELECT * FROM CUSTOMERS;

//Loading the data from S3 bucket
COPY INTO CUSTOMERS
    FROM s3://snowflake-assignments-mc/gettingstarted/customers.csv
    file_format = (type = csv
                   // Data type: CSV - delimited by ',' (comma)
                   field_delimiter = ','
                   // Header is in the first line
                   skip_header=1);

//Validate
SELECT * FROM CUSTOMERS;
```
