## Scheduling Tasks

- Task is an object that stores a SQL statement which is executed at a specific time or in a certain interval
- Tasks can be used to schedule SQL statements
- Only specify one specific SQL statements
- Stand-alone tasks and trees of tasks (dependencies where one task is executed after one task has been completed)

A task can execute any one of the following types of SQL code:
- Single SQL statement
- Call to a stored procedure
- Procedural logic using Snowflake Scripting

Tasks can be combined with table streams for continuous ELT workflows to process recently changed table rows. Streams ensure exactly once semantics for new or changed data in a table. Tasks can also be used independently to generate periodic reports by inserting or merging rows into a report table or perform other periodic work.

## Creating Task


```sql
// create transient table
CREATE OR REPLACE TRANSIENT DATABASE TASK_DB;

// Prepare table
CREATE OR REPLACE TABLE CUSTOMERS (
    // id auto incremenets from 1
    CUSTOMER_ID INT AUTOINCREMENT START = 1 INCREMENT =1,
    // name is default
    FIRST_NAME VARCHAR(40) DEFAULT 'JENNIFER' ,
    // create date
    CREATE_DATE DATE);


// Create task
CREATE OR REPLACE TASK CUSTOMER_INSERT
    // specify warehouse
    WAREHOUSE = COMPUTE_WH
    // schedule can be specified as the interval in minutes
    SCHEDULE = '1 MINUTE'
    AS
    // define the actual task
    // use 1 SQL statement
    // Insert into customer table, the create data using current timestamp
    INSERT INTO CUSTOMERS(CREATE_DATE) VALUES(CURRENT_TIMESTAMP);

// display task id , database, schema, schedule, state (Suspended by default), definition,
SHOW TASKS;

// Task starting and suspending
ALTER TASK CUSTOMER_INSERT RESUME; // state is started
ALTER TASK CUSTOMER_INSERT SUSPEND;

// defined task inserts row into table every minute as scheduled
SELECT * FROM CUSTOMERS;
```

## Using CRON

```sql
// Creating task method with schedule in minutes
CREATE OR REPLACE TASK CUSTOMER_INSERT
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '60 MINUTE'
    AS
    INSERT INTO CUSTOMERS(CREATE_DATE) VALUES(CURRENT_TIMESTAMP);


// Alternative method of scheduling
// Using CRON method
CREATE OR REPLACE TASK CUSTOMER_INSERT
    WAREHOUSE = COMPUTE_WH
    // specify time zone
    // USING CRON
    // SCHEDULE = 'USING CRON * *,* * * UTC'
    // 5 stars are represented using diagram below
    // 5L is the last Friday of the month
    SCHEDULE = 'USING CRON 0 7,10 * * 5L UTC'
    AS
    INSERT INTO CUSTOMERS(CREATE_DATE) VALUES(CURRENT_TIMESTAMP);


# __________ minute (0-59)
# | ________ hour (0-23)
# | | ______ day of month (1-31, or L)
# | | | ____ month (1-12, JAN-DEC)
# | | | | __ day of week (0-6, SUN-SAT, or L)
# | | | | |
# | | | | |
# * * * * *




// Every minute
SCHEDULE = 'USING CRON * * * * * UTC';


// Every day at 6am UTC timezone
SCHEDULE = 'USING CRON 0 6 * * * UTC';

// Every hour starting at 9 AM and ending at 5 PM on Sundays for America/LA timezone
SCHEDULE = 'USING CRON 0 9-17 * * SUN America/Los_Angeles';

// Every hour starting at 7AM and 10 AM, L for every last day of the month on Monday to Friday
SCHEDULE = 'USING CRON 0 7,10 L * MON-FRI UTC';


CREATE OR REPLACE TASK CUSTOMER_INSERT
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON 0 9,17 * * * UTC'
    AS
    INSERT INTO CUSTOMERS(CREATE_DATE) VALUES(CURRENT_TIMESTAMP);

```

## Trees of Tasks
- One root task that does not depend on any other tasks
- Once Parent task is completed, child tasks will be created (creating dependencies between tasks)
- One child task can have only one parent task
- One parent task can have multiple child tasks (up to 100 child tasks for one parent tasks)
- One complete tree of tasks can include up to 1000 tasks


## Creating trees of tasks


```sql
// use task database
USE TASK_DB;

SHOW TASKS;

// display customer table with previous customer_insert task
SELECT * FROM CUSTOMERS;

// Prepare a second table
CREATE OR REPLACE TABLE CUSTOMERS2 (
    CUSTOMER_ID INT,
    FIRST_NAME VARCHAR(40),
    CREATE_DATE DATE);


// Suspend parent task
ALTER TASK CUSTOMER_INSERT SUSPEND;

// Create a child task
CREATE OR REPLACE TASK CUSTOMER_INSERT2
    WAREHOUSE = COMPUTE_WH
    // after parent task
    AFTER CUSTOMER_INSERT
    AS
    // task definition is to insert all from customers into customers2
    INSERT INTO CUSTOMERS2 SELECT * FROM CUSTOMERS;


// Prepare a third table
CREATE OR REPLACE TABLE CUSTOMERS3 (
    CUSTOMER_ID INT,
    FIRST_NAME VARCHAR(40),
    CREATE_DATE DATE,
    INSERT_DATE DATE DEFAULT DATE(CURRENT_TIMESTAMP));


// Create a child task
CREATE OR REPLACE TASK CUSTOMER_INSERT3
    WAREHOUSE = COMPUTE_WH
    // after task 2
    AFTER CUSTOMER_INSERT2
    AS
    // insert all from customer2 into customer3 table
    INSERT INTO CUSTOMERS3 (CUSTOMER_ID,FIRST_NAME,CREATE_DATE) SELECT * FROM CUSTOMERS2;


SHOW TASKS; // 3 tasks and have owner ACCOUNTADMIN privileges
// privileges from owner inherited to the tasks

ALTER TASK CUSTOMER_INSERT
SET SCHEDULE = '1 MINUTE';

// initial parent task has schedule altered to 1 minute
// view predecessor tasks
SHOW TASKS;

// Resume tasks (first root task)
ALTER TASK CUSTOMER_INSERT RESUME; // resume parent task finally
ALTER TASK CUSTOMER_INSERT2 RESUME; // resume child task 2 next
ALTER TASK CUSTOMER_INSERT3 RESUME; // resume child task 3 first

// CUSTOMER TABLE had 29 rows inserted initially
// Parent task was executed after schedule of 1 minute, inserting one row to the CUSTOMER TABLE, resulting in 30 rows

SELECT * FROM CUSTOMERS2; // 30 rows inserted into CUSTOMER2 with create date

SELECT * FROM CUSTOMERS3; // 30 rows inserted with CUSTOMER3 with create data and insert date

// Suspend tasks again
ALTER TASK CUSTOMER_INSERT SUSPEND;
ALTER TASK CUSTOMER_INSERT2 SUSPEND;
ALTER TASK CUSTOMER_INSERT3 SUSPEND;

```

## Calling a stored procedure


```sql
// Create a stored procedure

// set context
USE TASK_DB;

SELECT * FROM CUSTOMERS; // 48 rows in CUSTOMERS Table


// Create PROCEDURE, specify name, and use 1 variable which is CREATE_DATE (this variable can be passed to the stored procedure)
CREATE OR REPLACE PROCEDURE CUSTOMERS_INSERT_PROCEDURE (CREATE_DATE varchar)
    // returns string with constraint Not Null
    RETURNS STRING NOT NULL
    // language is JavaScript
    LANGUAGE JAVASCRIPT
    // statement
    // variable is insert  into column create data of customers table
    // ;1 refers to the variable (argument passing)
    // sqlText is to specify the SQL INSERT command will be executed
    // refers to the variable sql_command that stores the SQL command to be used
    // return successfully executed
    AS
        $$
        var sql_command = 'INSERT INTO CUSTOMERS(CREATE_DATE) VALUES(:1);'
        snowflake.execute(
            {
            sqlText: sql_command,
            binds: [CREATE_DATE]
            });
        return "Successfully executed.";
        $$;


// calling a stored procedure in a task
CREATE OR REPLACE TASK CUSTOMER_TASK_PROCEDURE
WAREHOUSE = COMPUTE_WH
SCHEDULE = '1 MINUTE'
// call stored procedure and pass the value we are passing
AS CALL  CUSTOMERS_INSERT_PROCEDURE (CURRENT_TIMESTAMP);


SHOW TASKS;

ALTER TASK CUSTOMER_TASK_PROCEDURE RESUME;

// After 1 minute, 49 rows
// this demonstrates that a stored procedure can be called within a task
SELECT * FROM CUSTOMERS;

ALTER TASK CUSTOMER_TASK_PROCEDURE SUSPEND;
```

## Task History and Error handling



```sql
// Check history and overview of tasks to handle errors

SHOW TASKS;
// 4 tasks created CUSTOMER_INSERT, CUSTOMER_INSERT2, CUSTOMER_INSERT3, CUSTOMER_TASK_PROCEDURE
// View database, owner, schedule, predecessors, and state (started or suspended)

// select context
USE DEMO_DB;

// Use the table function "TASK_HISTORY()"
select *
  from table(information_schema.task_history())
  order by scheduled_time desc;

// displays last 100 records of tasks
// succeeded and scheduled tasks
// displays query start
// error message  for failed state is useful for fixing store procedure statements or task definitions



// See results for a specific Task in a given time
select *
from table(information_schema.task_history(
    // specify scheduled time range using dateadd function, set -4 hours from the current_timestamp
    // displays all results in the last 4 hours limited to the latest 5 only showing results for task name CUSTOMER_INSERT2
    scheduled_time_range_start=>dateadd('hour',-4,current_timestamp()),
    result_limit => 5,
    task_name=>'CUSTOMER_INSERT2'));

// displays task executions for INSERT2
// Query text, state succeeded, no error message or code, schedule time, query start time, next schedule time, completed time, attempt number




// See results for a given time period
select *
  from table(information_schema.task_history(
    scheduled_time_range_start=>to_timestamp_ltz('2023-12-24 11:28:32.776 -0700'),
    // default end date will be current timestamp if not specified
    scheduled_time_range_end=>to_timestamp_ltz('2023-12-26 22:44:13.776 -0700')));  

// find current timestamp
SELECT TO_TIMESTAMP_LTZ(CURRENT_TIMESTAMP)  
```

## Tasks with condition

```sql
// Task Option WHEN

USE TASK_DB

// Prepare table
CREATE OR REPLACE TABLE CUSTOMERS (
    CUSTOMER_ID INT AUTOINCREMENT START = 1 INCREMENT = 1,
    FIRST_NAME VARCHAR(40) DEFAULT 'JENNIFER',
    CREATE_DATE DATE);

// Crete task
CREATE OR REPLACE TASK CUSTOMER_INSERT
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '1 MINUTE'
    // WHEN option, select condition of expression that is evaluated to a boolean true or false
    // if evaluated to false, task will not be executed
    // 1 does not equal 2 so evaluated to false, insert will not be executed
    WHEN 1=2
    AS
    INSERT INTO CUSTOMERS(CREATE_DATE, FIRST_NAME) VALUES(CURRENT_TIMESTAMP, 'MIKE');


// Create both tasks and resume
// task starting and suspending
ALTER TASK CUSTOMER_INSERT RESUME;
ALTER TASK CUSTOMER_INSERT2 RESUME;

ALTER TASK CUSTOMER_INSERT SUSPEND;
ALTER TASK CUSTOMER_INSERT2 SUSPEND;
// Crete task
CREATE OR REPLACE TASK CUSTOMER_INSERT2
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '1 MINUTE'
    // WHEN option, select condition of expression that is evaluated to a boolean true or false
    // if evaluated to true, task will be executed
    // 1 does equal 1 so evaluated to true, insert will be executed
    WHEN 1=1
    AS
    INSERT INTO CUSTOMERS(CREATE_DATE, FIRST_NAME) VALUES(CURRENT_TIMESTAMP, 'DEPIKA');



// After 1 minute schedule time
// Results show record with first name DEPIKA inserted and MIKE not inserted as expected by the result of the boolean statements
SELECT * FROM CUSTOMERS;


// Use the table function "TASK_HISTORY()"
// displays last 100 records of tasks
select *
  from table(information_schema.task_history())
  order by scheduled_time desc;

// History shows Condition_text
// 1=1 Succeeded
// 1=2 Skipped error code 0040003 error message Conditional expression for task evaluated to false.


// Currently can only use the specific function in WHEN statement called
// SYSTEM$GET_PREDECESSOR_RETURN_VALUE, SYSTEM$STREAM_HAS_DATA

// Crete task
CREATE OR REPLACE TASK CUSTOMER_INSERT2
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '1 MINUTE'
    // condition gives error
    // Invalid expression for task condition expression. Only some data type conversions and the following functions are allowed: [SYSTEM$GET_PREDECESSOR_RETURN_VALUE, SYSTEM$STREAM_HAS_DATA].
    // system stream has data function can be used
    WHEN CURRENT_TIMESTAMP LIKE '20%'
    AS
    INSERT INTO CUSTOMERS(CREATE_DATE, FIRST_NAME) VALUES(CURRENT_TIMESTAMP, 'DEPIKA');


```
