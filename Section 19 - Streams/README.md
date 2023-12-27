## Streams

- Data from multiple sources (HR data, Sales data) stored in Data Sources table
- ETL process extracts data from the Data Sources table and loads it to data warehouse
- Delta load where additional records or rows in the Data Source table, only the additional rows, deleted rows, or updated rows are updated to data warehouse table
- Capture changes in source table and make updates or changes in the target date warehouse table

Definition of Streams:
- Object that records (DML-) changes made to a table
- DML- Data manipulation language changes such as insert, update, delete
- The process of capturing these changes is called change data capture (DTC)

How Streams work:
- Stream object created on top of one specific table and all changes including DELETE, INSERT, UPDATE are captured
- Stream object contains rows that have been updated, inserted
- Stream object contains same columns as the table it is created on top of but contains 3 additional columns:
  - METADATA$ACTION
  - METADATA$UPDATE
  - METADATA$ROW_ID
- Values visible in the stream object are not stored in the stream object but are retrieving them from the original table meaning no additional costs except for minimal storage costs for METADATA columns. Therefore no additional storage costs for the data stored in the original table.
- Changes captured in the stream object that have been made in the original table, are then INSERT into the target table
- Once INSERT done, table is removed from the stream object. This is the purpose of the stream object in facilitating the ETL process


## INSERT operation


```sql

-------------------- Stream example: INSERT ------------------------
CREATE OR REPLACE TRANSIENT DATABASE STREAMS_DB;

-- Create example table - source table where we want to create a stream object and capture any changes
create or replace table sales_raw_staging(
  id varchar,
  product varchar,
  price varchar,
  amount varchar,
  // store_id refers to store table which contains additional data about the store where product is sold
  store_id varchar);

-- insert values
insert into sales_raw_staging
    values
        (1,'Banana',1.99,1,1),
        (2,'Lemon',0.99,1,1),
        (3,'Apple',1.79,1,2),
        (4,'Orange Juice',1.89,1,2),
        (5,'Cereals',5.98,2,1);  


create or replace table store_table(
  store_id number,
  location varchar,
  employees number);

// insert stores and employees
INSERT INTO STORE_TABLE VALUES(1,'Chicago',33);
INSERT INTO STORE_TABLE VALUES(2,'London',12);

// create final / target table
create or replace table sales_final_table(
  id int,
  product varchar,
  price number,
  amount int,
  store_id int,
  location varchar,
  employees int);

 -- Insert into final table
INSERT INTO sales_final_table
    SELECT
    SA.id,
    SA.product,
    SA.price,
    SA.amount,
    ST.STORE_ID,
    ST.LOCATION,
    ST.EMPLOYEES
    FROM SALES_RAW_STAGING SA
    // join to store table on store ID
    JOIN STORE_TABLE ST ON ST.STORE_ID=SA.STORE_ID ;



-- Create a stream object
// on sales_raw table
create or replace stream sales_stream on table sales_raw_staging;


SHOW STREAMS;

DESC STREAM sales_stream;

-- Get changes on data using stream (INSERTS)
// no changes made on raw table so stream is empty as expected
select * from sales_stream;

// raw table contains 5 rows
select * from sales_raw_staging;



-- insert values into raw table
insert into sales_raw_staging  
    values
        (6,'Mango',1.99,1,2),
        (7,'Garlic',0.99,1,1);
        // 2 rows inserted

-- Get changes on data using stream (INSERTS)
select * from sales_stream; // 2 rows of data where insert changes have been made
// Contains ID, Product, Price, Amount, Store_ID columns from raw table
// And 3 additional columns


// METADATA$ACTION - indicates the DML operation (INSERT, DELETE) recorded. - value is INSERT indicates INSERT operation of rows
// METADATA$ISUPDATE - Indicates whether the operation was part of an UPDATE statement - value is False indicated it is not an UPDATE
// METADATA$ROW_ID - Specifies the unique and immutable ID for the row, which can be used to track changes to specific rows over time.

select * from sales_raw_staging; // 7 rows

// final table still has 5 rows as expected and not yet updated
// Use stream object to insert changes into final table
select * from sales_final_table;        


-- Consume stream object
INSERT INTO sales_final_table
    SELECT
    SA.id,
    SA.product,
    SA.price,
    SA.amount,
    ST.STORE_ID,
    ST.LOCATION,
    ST.EMPLOYEES
    // from sales_stream object with alias SA
    FROM SALES_STREAM SA
    JOIN STORE_TABLE ST ON ST.STORE_ID=SA.STORE_ID ;
    // 2 rows inserted


-- Get changes on data using stream (INSERTS)
// once consumed with insert, stream object is emptied
select * from sales_stream;




-- insert values
insert into sales_raw_staging  
    values
        (8,'Paprika',4.99,1,2),
        (9,'Tomato',3.99,1,2);

-- stream now contains newly inserted values
select * from sales_stream;


 -- Consume stream object
INSERT INTO sales_final_table
    SELECT
    SA.id,
    SA.product,
    SA.price,
    SA.amount,
    ST.STORE_ID,
    ST.LOCATION,
    ST.EMPLOYEES
    FROM SALES_STREAM SA
    JOIN STORE_TABLE ST ON ST.STORE_ID=SA.STORE_ID ;

-- stream object table emptied
select * from sales_stream;

// final table / target table contains 9 rows
SELECT * FROM SALES_FINAL_TABLE;        

// raw table contains 9 rows
SELECT * FROM SALES_RAW_STAGING;     

// stream has been consumed meaning stream is now empty
SELECT * FROM SALES_STREAM;
```

## UPDATE operation

```sql


-- ******* UPDATE 1 ********

// raw table contains 9 rows
SELECT * FROM SALES_RAW_STAGING;     

// no updates since last stream consume / insert meaning stream is empty
SELECT * FROM SALES_STREAM;

// update in raw table
UPDATE SALES_RAW_STAGING
SET PRODUCT ='Potato' WHERE PRODUCT = 'Banana'; // 1 rows update, 0 multi-joined rows updated

// Stream contains 2 rows
// Stream has captured METADATA$ACTION INSERT for Potato and METADATA$ACTION DELETE for Banana
// METADATA$ISUPDATE is true for Banana as the DELETE has been used in an UPDATE
// METADATA$ISUPDATE is true for Potato as the INSERT is related to the UPDATE

SELECT * FROM SALES_STREAM;


// MERGE
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
using SALES_STREAM S                -- Stream that has captured the changes
 on  f.id = s.id                  -- Update columns on where id column values are equal
when matched                        -- matched verifies that only update based on metadata values
  and S.METADATA$ACTION ='INSERT'
  and S.METADATA$ISUPDATE ='TRUE'        -- Indicates the record has been updated
  then update
  set f.product = s.product,
      f.price = s.price,
      f.amount= s.amount,
      f.store_id=s.store_id;

// Potato has been updated for the correct row
SELECT * FROM SALES_FINAL_TABLE;

SELECT * FROM SALES_RAW_STAGING;     

// stream is now empty
SELECT * FROM SALES_STREAM;

-- ******* UPDATE 2 ********

UPDATE SALES_RAW_STAGING
SET PRODUCT ='Green apple' WHERE PRODUCT = 'Apple';

// Apple has been updated in the raw table
SELECT * FROM SALES_RAW_STAGING;

// target table not updated yet
SELECT * FROM SALES_FINAL_TABLE;

// stream contains 2 rows
// ID 3 Product Green Apple METADATA$ACTION INSERT METADATA$ISUPDATE TRUE
// ID 3 Product Apple METADATA$ACTION DELETE METADATA$ISUPDATE TRUE
SELECT * FROM SALES_STREAM;

// MERGE to process change in final table
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
using SALES_STREAM S                -- Stream that has captured the changes
 on  f.id = s.id                 
when matched
  and S.METADATA$ACTION ='INSERT'
  and S.METADATA$ISUPDATE ='TRUE'        -- Indicates the record has been updated
  then update
  set f.product = s.product,
      f.price = s.price,
      f.amount= s.amount,
      f.store_id=s.store_id;

// final table updated to Green apple
SELECT * FROM SALES_FINAL_TABLE;

// verify raw table
SELECT * FROM SALES_RAW_STAGING;     

// verify stream is empty after merge
SELECT * FROM SALES_STREAM;
```

## DELETE operation
```sql

USE STREAMS_DB;

-- ******* DELETE  ********        


SELECT * FROM SALES_FINAL_TABLE; // 9 rows

SELECT * FROM SALES_RAW_STAGING; // 9 rows

SELECT * FROM SALES_STREAM; // empty

// DELETE 1 row
DELETE FROM SALES_RAW_STAGING
WHERE PRODUCT = 'Lemon';

SELECT * FROM SALES_RAW_STAGING; // 8 rows as 1 row deleted

// stream object contains one row
// ID 2 Product Lemon METADATA$ACTION DELETE METADATA$ISUPDATE FALSE

SELECT * FROM SALES_STREAM; // 1 row

SELECT * FROM SALES_FINAL_TABLE; // 9 rows as changes have not been processed or consumed by stream object via merge



-- ******* Process stream  ********            

// MERGE     
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
using SALES_STREAM S                -- Stream that has captured the changes
   on  f.id = s.id                  -- match on id column and metadata matches then delete
when matched
    and S.METADATA$ACTION ='DELETE'
    and S.METADATA$ISUPDATE = 'FALSE'
    then delete   ;            
// 1 row deleted


SELECT * FROM SALES_RAW_STAGING; // 8 rows in raw table

SELECT * FROM SALES_STREAM; // empty as stream has been processed and consumed

SELECT * FROM SALES_FINAL_TABLE; // final table is now 8 rows as stream has processed the deletion of 1 row - now missing row with id 2
```

## Processing all data changes


```sql



-- ******* Process UPDATE,INSERT & DELETE simultaneously  ********                



USE STREAMS_DB;

SELECT * FROM SALES_RAW_STAGING; // 8 rows ID 2 is missing
SELECT * FROM SALES_STREAM; // empty
SELECT * FROM SALES_FINAL_TABLE; // same level as raw source table


// insert a value into raw table - lemon row
INSERT INTO SALES_RAW_STAGING VALUES (2,'Lemon',0.99,1,1);

SELECT * FROM SALES_RAW_STAGING; // 9 rows ID 2 lemon is inserted
SELECT * FROM SALES_STREAM; // stream reflects the INSERT


// MERGE
// merge used to match all id in the stream with final table
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
// using the SELECT for values for location and employees from store_table
USING ( SELECT STRE.*,ST.location,ST.employees
 FROM SALES_STREAM STRE
 JOIN STORE_TABLE ST
 ON STRE.store_id = ST.store_id
) S
ON F.id=S.id
// when matched for metdata columns
when matched                        -- DELETE condition
and S.METADATA$ACTION ='DELETE'
and S.METADATA$ISUPDATE = 'FALSE'
then delete                   
when matched                        -- UPDATE condition
and S.METADATA$ACTION ='INSERT'
and S.METADATA$ISUPDATE  = 'TRUE'       
then update
// update values using the values from the streams
set f.product = s.product,
 f.price = s.price,
 f.amount= s.amount,
 f.store_id=s.store_id
when not matched
and S.METADATA$ACTION ='INSERT'
// insert the employees and locations from the join for INSERT
then insert
(id,product,price,store_id,amount,employees,location)
values
(s.id, s.product,s.price,s.store_id,s.amount,s.employees,s.location);

// Result of MERGE is 1 row inserted


SELECT * FROM SALES_RAW_STAGING;
SELECT * FROM SALES_STREAM; // now empty as stream is processed
SELECT * FROM SALES_FINAL_TABLE; // lemon row inserted with the location and employee values



// INSERT INTO SALES_RAW_STAGING VALUES (2,'Lemon',0.99,1,1);



// update lemon to lemonade
UPDATE SALES_RAW_STAGING
SET PRODUCT = 'Lemonade'
WHERE PRODUCT ='Lemon';

SELECT * FROM SALES_RAW_STAGING; // raw table shows update
SELECT * FROM SALES_STREAM; // contains 2 rows for INSERT of lemonade and DELETE of lemon
SELECT * FROM SALES_FINAL_TABLE; // unchanged

// MERGE
// merge used to match all id in the stream with final table
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
// using the SELECT for values for location and employees from store_table
USING ( SELECT STRE.*,ST.location,ST.employees
 FROM SALES_STREAM STRE
 JOIN STORE_TABLE ST
 ON STRE.store_id = ST.store_id
) S
ON F.id=S.id
// when matched for metadata columns
when matched                        -- DELETE condition
and S.METADATA$ACTION ='DELETE'
and S.METADATA$ISUPDATE = 'FALSE'
then delete                   
when matched                        -- UPDATE condition
and S.METADATA$ACTION ='INSERT'
and S.METADATA$ISUPDATE  = 'TRUE'       
then update
// update values using the values from the streams
set f.product = s.product,
 f.price = s.price,
 f.amount= s.amount,
 f.store_id=s.store_id
when not matched
and S.METADATA$ACTION ='INSERT'
// insert the employees and locations from the join for INSERT
then insert
(id,product,price,store_id,amount,employees,location)
values
(s.id, s.product,s.price,s.store_id,s.amount,s.employees,s.location);

// Result of MERGE is 1 row updated


SELECT * FROM SALES_RAW_STAGING;
SELECT * FROM SALES_STREAM; //  stream empty after merge
SELECT * FROM SALES_FINAL_TABLE; // lemon updated to lemonade in final table


// Delete lemonade row
// 1 row deleted
DELETE FROM SALES_RAW_STAGING
WHERE PRODUCT = 'Lemonade';   

// MERGE
// merge used to match all id in the stream with final table
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
// using the SELECT for values for location and employees from store_table
USING ( SELECT STRE.*,ST.location,ST.employees
 FROM SALES_STREAM STRE
 JOIN STORE_TABLE ST
 ON STRE.store_id = ST.store_id
) S
ON F.id=S.id
// when matched for metdata columns
when matched                        -- DELETE condition
and S.METADATA$ACTION ='DELETE'
and S.METADATA$ISUPDATE = 'FALSE'
then delete                   
when matched                        -- UPDATE condition
and S.METADATA$ACTION ='INSERT'
and S.METADATA$ISUPDATE  = 'TRUE'       
then update
// update values using the values from the streams
set f.product = s.product,
 f.price = s.price,
 f.amount= s.amount,
 f.store_id=s.store_id
when not matched
and S.METADATA$ACTION ='INSERT'
// insert the employees and locations from the join for INSERT
then insert
(id,product,price,store_id,amount,employees,location)
values
(s.id, s.product,s.price,s.store_id,s.amount,s.employees,s.location);

// Result of MERGE is 1 row deleted

SELECT * FROM SALES_RAW_STAGING; // 8 rows
SELECT * FROM SALES_STREAM; //  stream empty after merge
SELECT * FROM SALES_FINAL_TABLE; // 8 rows - row with ID 2 lemonade deleted in final table

--- Example 2 ---

// Using all 3 INSERT, UPDATE, DELETE STATEMENT in combination
INSERT INTO SALES_RAW_STAGING VALUES (10,'Lemon Juice',2.99,1,1);

UPDATE SALES_RAW_STAGING
SET PRICE = 3
WHERE PRODUCT ='Mango';

DELETE FROM SALES_RAW_STAGING
WHERE PRODUCT = 'Potato';    

// Stream shows 4 records
// 2 inserts and 2 deletes
SELECT * FROM SALES_STREAM;
// verify changes in raw table
// 8 rows lemon juice included and price of mango set to 3
SELECT * FROM SALES_RAW_STAGING;

// MERGE
// merge used to match all id in the stream with final table
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
// using the SELECT for values for location and employees from store_table
USING ( SELECT STRE.*,ST.location,ST.employees
 FROM SALES_STREAM STRE
 JOIN STORE_TABLE ST
 ON STRE.store_id = ST.store_id
) S
ON F.id=S.id
// when matched for metdata columns
when matched                        -- DELETE condition
and S.METADATA$ACTION ='DELETE'
and S.METADATA$ISUPDATE = 'FALSE'
then delete                   
when matched                        -- UPDATE condition
and S.METADATA$ACTION ='INSERT'
and S.METADATA$ISUPDATE  = 'TRUE'       
then update
// update values using the values from the streams
set f.product = s.product,
 f.price = s.price,
 f.amount= s.amount,
 f.store_id=s.store_id
when not matched
and S.METADATA$ACTION ='INSERT'
// insert the employees and locations from the join for INSERT
then insert
(id,product,price,store_id,amount,employees,location)
values
(s.id, s.product,s.price,s.store_id,s.amount,s.employees,s.location);

// Result of MERGE is 1 row inserted, 1 row updated, 1 row deleted

SELECT * FROM SALES_STREAM; //  stream empty after merge
SELECT * FROM SALES_FINAL_TABLE; // final table reflects all changes made in combination and have been processed at the same time

```

## Combine streams and tasks

```sql
------- Automatate the updates using tasks --


// create a task to combine streams and tasks
CREATE OR REPLACE TASK all_data_changes
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '1 MINUTE'
    // task is only executed if condition if true
    // SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') shows FALSE as sales_stream object is currently has no data, when stream contains data, condition is true and task will be executed
    WHEN SYSTEM$STREAM_HAS_DATA('SALES_STREAM')
    // AS is the statement the task will execute - combining all the processing of data changes such as INSERT, UPDATE, AND DELETE
    AS
merge into SALES_FINAL_TABLE F      -- Target table to merge changes from source table
USING ( SELECT STRE.*,ST.location,ST.employees
        FROM SALES_STREAM STRE
        JOIN STORE_TABLE ST
        ON STRE.store_id = ST.store_id
       ) S
ON F.id=S.id
when matched                        -- DELETE condition
    and S.METADATA$ACTION ='DELETE'
    and S.METADATA$ISUPDATE = 'FALSE'
    then delete                   
when matched                        -- UPDATE condition
    and S.METADATA$ACTION ='INSERT'
    and S.METADATA$ISUPDATE  = 'TRUE'       
    then update
    set f.product = s.product,
        f.price = s.price,
        f.amount= s.amount,
        f.store_id=s.store_id
when not matched
    and S.METADATA$ACTION ='INSERT'
    then insert
    (id,product,price,store_id,amount,employees,location)
    values
    (s.id, s.product,s.price,s.store_id,s.amount,s.employees,s.location);

// Start the task
ALTER TASK all_data_changes RESUME;
// task ALL_DATA_CHANGES started
SHOW TASKS;

// Change data

// INSERT 3 values
INSERT INTO SALES_RAW_STAGING VALUES (11,'Milk',1.99,1,2);
INSERT INTO SALES_RAW_STAGING VALUES (12,'Chocolate',4.49,1,2);
INSERT INTO SALES_RAW_STAGING VALUES (13,'Cheese',3.89,1,1);

// UPDATE product
UPDATE SALES_RAW_STAGING
SET PRODUCT = 'Chocolate bar'
WHERE PRODUCT ='Chocolate';

// DELETE product
DELETE FROM SALES_RAW_STAGING
WHERE PRODUCT = 'Mango';    


// Verify results
SELECT * FROM SALES_RAW_STAGING; // changes immediately reflected in raw table

SELECT * FROM SALES_STREAM; // stream shows changes with METADATA$ columns
// stream cleared as soon as stream data changes have been automatically processed

// Final table contains all the data changes
SELECT * FROM SALES_FINAL_TABLE;



// Verify the history
// sate of each query for ALL_DATA_CHANGES tasks show Succeeded, Skipped or Scheduled
select *
from table(information_schema.task_history())
order by name asc,scheduled_time desc;

```

## Types of streams

- Append-only stream
  - Specifies whether this is an append-only stream. Append-only streams track row inserts only.
  - This type of stream improves query performance over standard streams and is very useful for extract, load, transform (ELT) and similar scenarios that depend exclusively on row inserts.
  - A standard stream joins the deleted and inserted rows in the change set to determine which rows were deleted and which were updated. An append-only stream returns the appended rows only and therefore can be much more performant than a standard stream. For example, the source table can be truncated immediately after the rows in an append-only stream are consumed, and the record deletions do not contribute to the overhead the next time the stream is queried or consumed.


```sql
------- Append-only type ------
USE STREAMS_DB;
// view properties of SALES_STREAM
// property mode is set to DEFAULT type
SHOW STREAMS;

// STANDARD TYPE captures INSERT, UPDATE, DELETE changes
// APPEND_ONLY TYPE captures INSERT changes only

SELECT * FROM SALES_RAW_STAGING; // 10 rows

-- Create stream with default
CREATE OR REPLACE STREAM SALES_STREAM_DEFAULT
ON TABLE SALES_RAW_STAGING;

-- Create stream with append-only
// mode is APPEND_ONLY
CREATE OR REPLACE STREAM SALES_STREAM_APPEND
ON TABLE SALES_RAW_STAGING
// use append_only property set to TRUE
APPEND_ONLY = TRUE;

-- View streams
SHOW STREAMS;


-- Insert values
INSERT INTO SALES_RAW_STAGING VALUES (14,'Honey',4.99,1,1);
INSERT INTO SALES_RAW_STAGING VALUES (15,'Coffee',4.89,1,2);
INSERT INTO SALES_RAW_STAGING VALUES (15,'Coffee',4.89,1,2);




SELECT * FROM SALES_STREAM_APPEND; // 3 INSERT rows in stream
SELECT * FROM SALES_STREAM_DEFAULT; // 3 INSERT rows in stream

-- Delete values
SELECT * FROM SALES_RAW_STAGING;

// DELEET row where ID is 7
DELETE FROM SALES_RAW_STAGING WHERE ID=7;


SELECT * FROM SALES_STREAM_APPEND; // 3 INSERT rows in stream
SELECT * FROM SALES_STREAM_DEFAULT; // 4 rows in stream as DELETE captured

-- 2 options to consume stream (make stream empty)
-- Use INSERT statement or Use Create or Replace statement
-- CREATE OR REPLACE [...] AS SELECT * FROM STREAM_NAME
-- Consume stream via "CREATE TABLE ... AS"
CREATE OR REPLACE TEMPORARY TABLE PRODUCT_TABLE
AS SELECT * FROM SALES_STREAM_DEFAULT;
CREATE OR REPLACE TEMPORARY TABLE PRODUCT_TABLE
AS SELECT * FROM SALES_STREAM_APPEND;

// Both streams consumed and empty
SELECT * FROM SALES_STREAM_APPEND;
SELECT * FROM SALES_STREAM_DEFAULT;


-- Update
// 2 rows updated
UPDATE SALES_RAW_STAGING
SET PRODUCT = 'Coffee 200g'
WHERE PRODUCT ='Coffee';

// 0 rows in APPEND_ONLY stream as no changes captured because Append-only streams track row inserts only
SELECT * FROM SALES_STREAM_APPEND;
// 4 rows in DEFAULT stream as INSERT and DELETE captured
SELECT * FROM SALES_STREAM_DEFAULT;
```
## Change clause
- The CHANGES clause enables querying the change tracking metadata for a table or view within a specified interval of time without having to create a stream with an explicit transactional offset. Multiple queries can retrieve the change tracking metadata between different transactional start and endpoints.
- Note: Change tracking must be enabled on the source table or the source view and its underlying tables

```sql
----- Change clause ------

// Alternative method to track changes in a table without creating a stream object but instead using a change statement

----- Change clause ------

--- Create example db & table ---

USE STREAMS_DB;

// create new db
CREATE OR REPLACE DATABASE SALES_DB;

// create table we want to track the changes of
create or replace table sales_raw(
	id varchar,
	product varchar,
	price varchar,
	amount varchar,
	store_id varchar);

-- insert values
insert into sales_raw
	values
		(1, 'Eggs', 1.39, 1, 1),
		(2, 'Baking powder', 0.99, 1, 1),
		(3, 'Eggplants', 1.79, 1, 2),
		(4, 'Ice cream', 1.89, 1, 2),
		(5, 'Oats', 1.98, 2, 1);

// set change tracking property to True
ALTER TABLE sales_raw
SET CHANGE_TRACKING = TRUE;


SELECT * FROM SALES_RAW
// use changes statement with information value
// default type or append_only type
CHANGES(information => default)
// using time travel feature meaning time travel must be enabled (cannot use temporary tables or any tables which do not have time travel capability)
// can use offset or timestamp
AT (offset => -0.5*60);


SELECT CURRENT_TIMESTAMP;

-- Insert values
INSERT INTO SALES_RAW VALUES (6, 'Bread', 2.99, 1, 2);
INSERT INTO SALES_RAW VALUES (7, 'Onions', 2.89, 1, 2);
// 1 row inserted


SELECT * FROM SALES_RAW
CHANGES(information  => default)
AT (timestamp => 'your-timestamp'::timestamp_tz);

// view changes using timestamp time travel feature
SELECT * FROM SALES_RAW
CHANGES(information  => default)
AT (timestamp => '2023-12-27 01:00:42.659 -0800'::timestamp_tz);

UPDATE SALES_RAW
SET PRODUCT = 'Toast3' WHERE ID=6;


// information value

// Toast2 update captured
SELECT * FROM SALES_RAW
CHANGES(information  => default)
AT (timestamp => '2023-12-27 01:00:42.659 -0800'::timestamp_tz);

// Toast 3 update captured
// Note: per row only the latest information / change will be captured compared to the timestamp specified

// Using Append_only
// Toast update is not captured, only the INSERT statements are tracked
SELECT * FROM SALES_RAW
CHANGES(information  => append_only)
AT (timestamp => '2023-12-27 01:00:42.659 -0800'::timestamp_tz);


// When changes are consumed, change clause does not empty tables like stream object
// create a table from the change clause and query verifies this
// change clause provides a dynamic way to track changes without using streams
CREATE OR REPLACE TABLE PRODUCTS
AS
SELECT * FROM SALES_RAW
CHANGES(information  => append_only)
AT (timestamp => '2023-12-27 01:00:42.659 -0800'::timestamp_tz);

// verify products table is not empty after processing changes
SELECT * FROM PRODUCTS;

```
