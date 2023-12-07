## Fail Safe

Separate and distinct from Time Travel, Fail-safe ensures historical data is protected in the event of a system failure or other event (e.g. a security breach).

Fail-safe provides a (non-configurable) 7-day period during which historical data may be recoverable by Snowflake. This period starts immediately after the Time Travel retention period ends

Fail-safe is a data recovery service that is provided on a best effort basis and is intended only for use when all other recovery options have been attempted.

Fail-safe is not provided as a means for accessing historical data after the Time Travel retention period has ended. It is for use only by Snowflake to recover data that may have been lost or damaged due to extreme operational failures.

## Fail Safe Overview

- Part of the data protection life cycle
- Protection of historical data in case of disaster
- Non configurable 7-day period for permanent tables
- Period starts automatically after Time travel period ends
- No user interaction and recoverable only by Snowflake (cannot be queried directly like time travel)
- Contributes to storage cost

## Fail Safe in relation to Snowflake Components
-  Current data storage - Access and query data
- Time travel
  - SELECT ... AT | BEFORE
  - UNDROP
  - 1 - 90 days retention time
  - 0 retention time effectively disables time travel
- Fail Safe
  - Restoring only by snowflake support
  - recovery beyond time travel
  - no user operations/queries
  - permanent: 7 days transient : 0 days

## Temporary tables
Snowflake supports creating temporary tables for storing non-permanent, transitory data (e.g. ETL data, session-specific data). Temporary tables only exist within the session in which they were created and persist only for the remainder of the session. As such, they are not visible to other users or sessions. Once the session ends, data stored in the table is purged completely from the system and, therefore, is not recoverable, either by the user who created the table or Snowflake.

## Transient tables

Snowflake supports creating transient tables that persist until explicitly dropped and are available to all users with the appropriate privileges. Transient tables are similar to permanent tables with the key difference that they do not have a Fail-safe period. As a result, transient tables are specifically designed for transitory data that needs to be maintained beyond each session (in contrast to temporary tables), but does not need the same level of data protection and recovery provided by permanent tables.

## Fail Safe Storage



```sql
// Fail Safe
// Using ACCOUNTADMIN role

// Using Snowflake UI, Admin, Usage Type as Storage
// View Data consumed and stored for each object
// View overview storage usage chart for STAGE, DATABASE, FAIL SAFE


// Query using the SNOWFLAKE database
// Storage usage on account level

// View storage bytes, stage bytes, failsafe bytes, hybrid table storage bytes
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE ORDER BY USAGE_DATE DESC;


// Storage usage on account level formatted
// convert to GB

SELECT 	USAGE_DATE,
		STORAGE_BYTES / (1024*1024*1024) AS STORAGE_GB,  
		STAGE_BYTES / (1024*1024*1024) AS STAGE_GB,
		FAILSAFE_BYTES / (1024*1024*1024) AS FAILSAFE_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE ORDER BY USAGE_DATE DESC;


// Storage usage on table level
// View storage based on tables and schemas

SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS;


// Storage usage on table level formatted

SELECT 	ID,
		TABLE_NAME,
		TABLE_SCHEMA,
		ACTIVE_BYTES / (1024*1024*1024) AS STORAGE_USED_GB,
		TIME_TRAVEL_BYTES / (1024*1024*1024) AS TIME_TRAVEL_STORAGE_USED_GB,
		FAILSAFE_BYTES / (1024*1024*1024) AS FAILSAFE_STORAGE_USED_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY FAILSAFE_STORAGE_USED_GB DESC;

```
