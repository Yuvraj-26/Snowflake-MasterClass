## Data Sampling

- SAMPLE / TABLESAMPLE
Returns a subset of rows sampled randomly from the specified table. The following sampling methods are supported:
  - Sample a fraction of a table, with a specified probability for including a given row. The number of rows returned depends on the size of the table and the requested probability. A seed can be specified to make the sampling deterministic.
  - Sample a fixed, specified number of rows. The exact number of specified rows is returned unless the table contains fewer rows.

## Why data sampling?
- Extremely large database exist in snowflake, if we want to develop on a large dataset, executing queries and views consume time and compute resources
- Increasing the compute resources increases cost, therefore the idea is to use a random sample to develop. This allows for testing of views on the data and data analysis for example.
- Use cases: Query development, data analysis
- Faster and more cost efficient queries (less compute resources)


## Methods of sampling
- ROW or BERNOULLI method
  - Every row is chosen with percentage p
  - More randomness
  - Use when dealing with Smaller tables
- BLOCK or SYSTEM method
  - Every block (micro partitions) is chosen with percentage p
  - more efficient and effective processing
  - Use when dealing with Large tables

## Sampling Data
```sql
// create transient database
// Snowflake supports creating transient tables that persist until explicitly dropped and are available to all users with the appropriate privileges. Transient tables are similar to permanent tables with the key difference that they do not have a Fail-safe period. As a result, transient tables are specifically designed for transitory data that needs to be maintained beyond each session (in contrast to temporary tables), but does not need the same level of data protection and recovery provided by permanent tables.
CREATE OR REPLACE TRANSIENT DATABASE SAMPLING_DB;

// snowflake sampling db address sample has 32.5 million rows (large table 600mb)

// select takes 26 seconds
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF10TCL.CUSTOMER_ADDRESS

// select using sample row takes 2 seconds
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF10TCL.CUSTOMER_ADDRESS
// ROW method percentage that every row is included
// 1% if row is included so 1% of data will be included
// SEED allows reproducing of result, so another developer will get exactly the same sample result
SAMPLE ROW (1) SEED(27);

// Create view for sample called address_sample
CREATE OR REPLACE VIEW ADDRESS_SAMPLE
AS
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF10TCL.CUSTOMER_ADDRESS
SAMPLE ROW (10) SEED(27); // use 10%

SELECT * FROM ADDRESS_SAMPLE;


// for ca_location_type
// calculate percentage for single, condo, appartment, null
SELECT CA_LOCATION_TYPE, COUNT(*)/3254250*100 // count divided by rows*100 to get percentage // add 0 as 10 times results
FROM ADDRESS_SAMPLE
GROUP BY CA_LOCATION_TYPE;

// Value of Null is 2.99%
// Now increase sample ROW percentage to 2
// Value of Null is 2.98%
// This shows there is sufficient accuracy to estimate on sampled data depending on the use case

// To improve processing time, use SYSTEM method
// Used for extremely large sized tables as it provides better performance
// Sampling from micro partition
// 1% whether micro partition will be included
// not 1 percent exactly due to use of larger blocks
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF10TCL.CUSTOMER_ADDRESS
SAMPLE SYSTEM (1) SEED(23);

// increase sample to 10%
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCDS_SF10TCL.CUSTOMER_ADDRESS
SAMPLE SYSTEM (10) SEED(23);

```

## Assignment 13 - Data Sampling

```sql
// Data sampling

// 1. Sample 5% from the table "SNOWFLAKE_SAMPLE_DATA"."TPCDS_SF10TCL"."CUSTOMER_ADDRESS"
// Use the ROW method and seed(2) to get a reproducible result.
// Store the result in the DEMO_DB in a table called CUSTOMER_ADDRESS_SAMPLE_5.

-- First sample --
USE DEMO_DB;
CREATE TABLE CUSTOMER_ADDRESS_SAMPLE_5 AS
SELECT * FROM "SNOWFLAKE_SAMPLE_DATA"."TPCDS_SF10TCL"."CUSTOMER_ADDRESS" SAMPLE ROW(5) SEED(2);


// 2. Sample 1% from the table "SNOWFLAKE_SAMPLE_DATA"."TPCDS_SF10TCL"."CUSTOMER"
// Use the SYSTEM method and seed(2) to get a reproducible result.
// Store the result in the DEMO_DB in a table called CUSTOMER_SAMPLE_1.

-- Second sample --
USE DEMO_DB;
CREATE TABLE CUSTOMER_SAMPLE_1 AS
SELECT * FROM "SNOWFLAKE_SAMPLE_DATA"."TPCDS_SF10TCL"."CUSTOMER" SAMPLE SYSTEM(1) SEED(2);

SELECT * FROM CUSTOMER_SAMPLE_1;

// How many rows are in the second created table CUSTOMER_SAMPLE_1?
//  1,376,518 rows
```
