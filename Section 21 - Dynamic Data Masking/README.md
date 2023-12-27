## Dynamic Data Masking

Dynamic Data Masking is a Column-level Security feature that uses masking policies to selectively mask plain-text data in table and view columns at query time.

In Snowflake, masking policies are schema-level objects, which means a database and schema must exist in Snowflake before a masking policy can be applied to a column. Currently, Snowflake supports using Dynamic Data Masking on tables and views.

At query runtime, the masking policy is applied to the column at every location where the column appears. Depending on the masking policy conditions, the SQL execution context, and role hierarchy, Snowflake query operators may see the plain-text value, a partially masked value, or a fully masked value.

- Data masking is used for security purposes
- Sensitive or confidential data that needs protecting, return a masked result at query execution
- Full control of which role can view the raw data
- Column-level security
- Custom maskings such as first letter or format of masking of phone number

## Creating a masking policy

```sql
USE DEMO_DB;
USE ROLE ACCOUNTADMIN;


-- Prepare table --
// table contains sensitive customer data
create or replace table customers(
  id number,
  full_name varchar,
  email varchar,
  phone varchar,
  spent number,
  create_date DATE DEFAULT CURRENT_DATE);

-- insert values in table --
insert into customers (id, full_name, email,phone,spent)
values
  (1,'Lewiss MacDwyer','lmacdwyer0@un.org','262-665-9168',140),
  (2,'Ty Pettingall','tpettingall1@mayoclinic.com','734-987-7120',254),
  (3,'Marlee Spadazzi','mspadazzi2@txnews.com','867-946-3659',120),
  (4,'Heywood Tearney','htearney3@patch.com','563-853-8192',1230),
  (5,'Odilia Seti','oseti4@globo.com','730-451-8637',143),
  (6,'Meggie Washtell','mwashtell5@rediff.com','568-896-6138',600);

SELECT * FROM customers;

-- set up roles
CREATE OR REPLACE ROLE ANALYST_MASKED;
CREATE OR REPLACE ROLE ANALYST_FULL;


-- grant select on table to roles
GRANT SELECT ON TABLE DEMO_DB.PUBLIC.CUSTOMERS TO ROLE ANALYST_MASKED;
GRANT SELECT ON TABLE DEMO_DB.PUBLIC.CUSTOMERS TO ROLE ANALYST_FULL;

GRANT USAGE ON SCHEMA DEMO_DB.PUBLIC TO ROLE ANALYST_MASKED;
GRANT USAGE ON SCHEMA DEMO_DB.PUBLIC TO ROLE ANALYST_FULL;

-- grant warehouse access to roles
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE ANALYST_MASKED;
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE ANALYST_FULL;


-- assign roles to a user
GRANT ROLE ANALYST_MASKED TO USER <username>;
GRANT ROLE ANALYST_FULL TO USER <username>;



-- Set up masking policy

create or replace masking policy phone
    // definition for policy
    // data type of original column must be same as return data type (masking characters)
    as (val varchar) returns varchar ->
            case      
            // specify multiple current role, return val plain text
            when current_role() in ('ANALYST_FULL', 'ACCOUNTADMIN') then val
            // else return masking characters
            else '##-###-##'
            end;


-- Apply policy on a specific column
// Set policy on column phone
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN phone
SET MASKING POLICY PHONE;




-- Validating policies

// ANALYST FULL role can view original value for phone
USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

// ANALYST MASKED context role can see masked phone column values
USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

```

## Unset and replace policy

```sql

-- #### More examples  #####

USE ROLE ACCOUNTADMIN;

--- 1) Apply policy to multiple columns

-- Apply policy on a specific column
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
SET MASKING POLICY phone;

-- Apply policy on another specific column
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN phone
SET MASKING POLICY phone;


-- Validating policies
USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

// full_name column has phone dynamic data masking
USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;



--- 2) Replace or drop policy

// Unable to DROP or REPLACE policy unless unset first
// Policy PHONE cannot be dropped/replaced as it is associated with one or more entities.
DROP masking policy phone;

create or replace masking policy phone as (val varchar) returns varchar ->
            case
            when current_role() in ('ANALYST_FULL', 'ACCOUNTADMIN') then val
            else CONCAT(LEFT(val,2),'*******')
            end;

-- List and describe policies
// view body which displays the roles in which the policy is assigned to
DESC MASKING POLICY phone;
// view policies
SHOW MASKING POLICIES;

-- Show columns with applied policies
SELECT * FROM table(information_schema.policy_references(policy_name=>'phone'));
// query from policy_references table shows FULL_NAME AND PHONE policy (columns which policy phone is already applied on)

-- Remove policy before replacing/dropping
// UNSET on full_name column
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
UNSET MASKING POLICY;

ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN email
UNSET MASKING POLICY;

// UNSET on phone column
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN phone
UNSET MASKING POLICY;

// Drop policy after UNSET
DROP masking policy phone; // PHONE successfully dropped.


-- replace policy
create or replace masking policy names as (val varchar) returns varchar ->
            case
            when current_role() in ('ANALYST_FULL', 'ACCOUNTADMIN') then val
            // only first 2 characters visible in mask
            else CONCAT(LEFT(val,2),'*******')
            end;

-- apply policy
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
SET MASKING POLICY names;


-- Validating policies
USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;
```

## Alter an existing policy

```sql


-- Alter existing policies
// Alter instead of recreating as recreating means having to apply policy on all required columns again
// Changing the definition of policy for all columns using the ALTER command

USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

USE ROLE ACCOUNTADMIN;


// alter command using policy name
alter masking policy phone set body ->
case        
 when current_role() in ('ANALYST_FULL', 'ACCOUNTADMIN') then val
 else '**-**-**'
 end;


  ALTER TABLE CUSTOMERS MODIFY COLUMN email UNSET MASKING POLICY;

```

## Real-life examples

```sql
-- ### More examples - 1 - ###

USE ROLE ACCOUNTADMIN;

// Apply a policy on emails column but only the domain should be unmasked
create or replace masking policy emails as (val varchar) returns varchar ->
case
  when current_role() in ('ANALYST_FULL') then val
  when current_role() in ('ANALYST_MASKED') then regexp_replace(val,'.+\@','*****@') -- leave email domain unmasked
  else '********'
end;


-- apply policy
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN email
SET MASKING POLICY emails;


-- Validating policies
// email visible in plain text
USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

// email masked with only domain showing as per definition in policy
USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

USE ROLE ACCOUNTADMIN;


### More examples - 2 - ###

// create a masking policy sha2 to make names anonymous but we want to know if a name is repeated elsewhere in the table
create or replace masking policy sha2 as (val varchar) returns varchar ->
case
  when current_role() in ('ANALYST_FULL') then val
  // use sha2 hash algorithm function
  else sha2(val) -- return hash of the column value
end;



// unset policy for full name column first
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
UNSET MASKING POLICY;

-- apply policy
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
SET MASKING POLICY sha2;



-- Validating policies
// full_name visible in plain text
USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

// full_name masked using sha2 hashing, names not identifiable but same object or person will have the same sha2 hash
// method to anonymise the data and hide personal details
USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

USE ROLE ACCOUNTADMIN;


### More examples - 3 - ###

// create a policy to mask the data
// val date and return data must be the same data type
create or replace masking policy dates as (val date) returns date ->
case
  when current_role() in ('ANALYST_FULL') then val
  else date_from_parts(0001, 01, 01)::date -- returns 0001-01-01 00:00:00.000
end;


-- Apply policy on a specific column in ACCOUNTADMIN role
ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN create_date
SET MASKING POLICY dates;


-- Validating policies
// create_date visible in plain text
USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;
// create_date masked to 0001-01-01 as in policy definition
USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;
```

## Assignment 15 - Data Masking

```sql
// 1. Prepare the table and two roles to test the masking policies (you can use the statement below)

USE DEMO_DB;
USE ROLE ACCOUNTADMIN;

-- Prepare table --
create or replace table customers(
  id number,
  full_name varchar,
  email varchar,
  phone varchar,
  spent number,
  create_date DATE DEFAULT CURRENT_DATE);


-- insert values in table --
insert into customers (id, full_name, email,phone,spent)
values
  (1,'Lewiss MacDwyer','lmacdwyer0@un.org','262-665-9168',140),
  (2,'Ty Pettingall','tpettingall1@mayoclinic.com','734-987-7120',254),
  (3,'Marlee Spadazzi','mspadazzi2@txnews.com','867-946-3659',120),
  (4,'Heywood Tearney','htearney3@patch.com','563-853-8192',1230),
  (5,'Odilia Seti','oseti4@globo.com','730-451-8637',143),
  (6,'Meggie Washtell','mwashtell5@rediff.com','568-896-6138',600);


-- set up roles
CREATE OR REPLACE ROLE ANALYST_MASKED;
CREATE OR REPLACE ROLE ANALYST_FULL;

-- grant select on table to roles
GRANT SELECT ON TABLE DEMO_DB.PUBLIC.CUSTOMERS TO ROLE ANALYST_MASKED;
GRANT SELECT ON TABLE DEMO_DB.PUBLIC.CUSTOMERS TO ROLE ANALYST_FULL;

GRANT USAGE ON SCHEMA DEMO_DB.PUBLIC TO ROLE ANALYST_MASKED;
GRANT USAGE ON SCHEMA DEMO_DB.PUBLIC TO ROLE ANALYST_FULL;

-- grant warehouse access to roles
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE ANALYST_MASKED;
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE ANALYST_FULL;

-- assign roles to a user
GRANT ROLE ANALYST_MASKED TO USER <YOUR-USER-NAME>;
GRANT ROLE ANALYST_FULL TO USER <YOUR-USER-NAME>;


// 2. Create masking policy called name that is showing '***' instead of the original varchar value except the role analyst_full is used in this case show the original value.

create or replace masking policy name
    as (val varchar) returns varchar ->
            case        
            when current_role() in ('ANALYST_FULL') then val
            else '***'
            end;


// 3. Apply the masking policy on the column full_name

ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
SET MASKING POLICY name;



// 4. Validate the result using the role analyst_masked and analyst_full

USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

// 5 Unset policies

// Validate policies

USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

USE ROLE ACCOUNTADMIN;

ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
UNSET MASKING POLICY;


// 6. Alter the policy so that the last two characters are shown and before that only '***' (example: ***er)
alter masking policy name set body ->
            case
            when current_role() in ('ANALYST_FULL') then val
            else CONCAT('***',RIGHT(val,2))
            end;


// 7. Apply the policy again on the column full name and validate the policy

ALTER TABLE IF EXISTS CUSTOMERS MODIFY COLUMN full_name
SET MASKING POLICY name;

// Validate policies

USE ROLE ANALYST_FULL;
SELECT * FROM CUSTOMERS;

USE ROLE ANALYST_MASKED;
SELECT * FROM CUSTOMERS;

```
