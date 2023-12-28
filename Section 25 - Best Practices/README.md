## Best Practices

## Virtual warehouses
- Enable Auto-Suspend - set AUTO_SUSPEND time in SQL or interface when creating virtual warehouse to minimise costs and auto suspend after a certain amount of time
- Enable Auto_Resume - enables users to query the database and automatically resume the compute resources
- Set appropriate timeouts
  - AUTO_SUSPEND property
    - ETL/Data Loading (Immediately - AUTO_SUSPEND = 0)
    - BI / SELECT queries (10 min) to make use of warm cache meaning queries can be executed faster and user experience will be better
    - DevOps / Data Science (5 min) as we have more unique queries, warm cache is not as important. Queries are not similar like BI/Select queries so we usually cannot benefit from warm cache.
- Choose appropriate size of the virtual warehouse
  - complex queries that take a lot of time, set a larger virtual warehouse size and decrease in future
  - large virtual warehouse process queries a lot faster and cost increase is not too significant


## Table design
- Appropriate table type
  - Staging tables (database or schema for loading raw data) - Transient
  - Productive tables - Permanent
  - Development tables (queries and development work, decrease costs for time travel and fail safe for development that makes a lot of changes) - Transient
- Appropriate data type
  - VARCHAR vs VARCHAR(n) - do not need to specify length n unless we have specific length for validation and finding errors in the data
- Set cluster keys only if necessary (Snowflake manages micro=partitions automatically) Cluster keys are necessary for:
  - Large table
  - Most query time for table scan (table can benefit from cluster key)
  - Dimensions (query or filter against dimensions that are a region and are not in the right order of data loading). For example if table was loaded based on transaction data and data is more focused on regions and we often filter by region, it can make sense to introduce cluster key on this dimension column.

## Monitoring
- Monitor usage in terms of storage and compute resoruces (virtual warehouses)
- Interface shows storage usage and credit usages
- Use queries to analyse specific usage

```sql
-- Views to monitor Snowflake usage

-- Table Storage
-- displays time travel and fail safe bytes
-- use groupby databases

SELECT * FROM "SNOWFLAKE"."ACCOUNT_USAGE"."TABLE_STORAGE_METRICS";

-- Query history view
-- How much is queried in databases
SELECT * FROM "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY";

-- compare data in database and how often it is queried
SELECT
DATABASE_NAME,
COUNT(*) AS NUMBER_OF_QUERIES,
SUM(CREDITS_USED_CLOUD_SERVICES)
FROM "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
GROUP BY DATABASE_NAME;

-- Warehouse metering history view
-- Usage of credits by warehouses
SELECT * FROM "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY";

-- Usage of credits by warehouses // Grouped by day
SELECT
DATE(START_TIME),
SUM(CREDITS_USED)
FROM "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
GROUP BY DATE(START_TIME);

-- Usage of credits by warehouses // Grouped by warehouse
SELECT
WAREHOUSE_NAME,
SUM(CREDITS_USED)
FROM "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
GROUP BY WAREHOUSE_NAME;

-- Usage of credits by warehouses // Grouped by warehouse & day
-- find outliers of warehouses, decrease size or find warehouses not using best Practices such as auto-suspend
SELECT
DATE(START_TIME),
WAREHOUSE_NAME,
SUM(CREDITS_USED)
FROM "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
GROUP BY WAREHOUSE_NAME,DATE(START_TIME);
```

## Retention Time
- Time travel consumes data storage, therefore is a cost factor. Not all data in tables need continuos data protection cycle of time travel and fail safe
- Staging database - 0 days (transient tables as we can easily extract raw data from the source and no transformations are performed in staging area - no time travel storage consumption needed)
- Production - 4 - 8 days (1 day minimum
- Large high-churn tables - 0 days (transient) For example, table of 20GB that is updated 20 times per day, time travel with 1 day retention time will be 400GB and Fail Safe for last 7 days will be 2.8TB
