## Access Management

## Access Control

- Who can access and perform operations on objects in Snowflake
- Two aspects of access control combined:
  - Discretionary Access Control (DAC)
    - Each object has an owner who can grant access to that object
  - Role-based Access Control (RBAC)
    - Access privileges are assigned to roles, which are in turn assigned to users
- Example: Role 1 Creates (Owns) Table allowing granting privileges to Roles 2 and 3 where the Roles have Users assigned to them which inherit the privileges.
- GRANT < privileged > ON < object > TO < role >
- GRANT < role > TO < user >


## Securable Objects
-  Account objects
  - User
  - Role
  - Database
    - Schema
      - Table
      - View
      - Stage
      - Integration
      - Other Schema objects
  - Warehouse
  - Other Account objects

  Note:
  - Every object owned by a single role (multiple users)
  - Owner (role) has all privileges per default

## Snowflake Roles
- Snowflake roles have hierarchy for inherited privileges
- ACCOUNTADMIN, SECURITYADMIN, USERADMIN, SYSADMIN, PUBLIC

## Key concepts
- USER - people or systems
- ROLE - entity to which privileges are granted (role hierarchy)
- PRIVILEGE - level of access to an object (SELECT, DROP, CREATE, etc.)
- SECURABLE OBJECT - Objects to which privileges can be granted (Database, Table, Warehouse, etc.)

## Roles overview
- 5 system defined Roles:
  - Top level is ACCOUNT ADMIN, combines the roles of SECURITYADMIN, AND SYSADMIN
  - Below the SECURITYADMIN is the USERADMIN
  - Lowest level role is PUBLIC (default for every user)
  - Can create Custom roles and its recommended to assign them using with SYSADMIN and create hierarchy of Roles and inherited privileges


## Snowflake roles
| ACCOUNTADMIN    | SECURITYADMIN     | SYSADMIN     | USERADMIN     | PUBLIC     |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| SYSADMIN and SECURITYADMIN     | USERADMIN role is granted to the SECURITYADMIN     | Create warehouses and databases (and more objects)     | Dedicated to user and role management only     |   Automatically granted to every user   |
| top level role in the system   | Can manage users and roles     | Recommend that all custom roles are assigned     | Can create users and roles     |  Can create own objects like every other role (available to every other user/role)      |
| should be granted only to a limited number of users     | Can manage any object grant globally     |      |   |    |


## ACCOUNTADMIN
- Top level role
- manage and view all objects
- all configurations on account level
- account operations (creates reader account)
- first user will have this role assigned
- initial setup and managing account level objects
- Best practices
  - Very controlled assignment strongly recommended
  - Multi-factor authentication
  - At least two users should be assigned to the role
  - Avoid creating objects with that role unless you have to


## ACCOUNTADMIN  in practice
- Account admin tab
- Billing and Usage
- Reader Account
- Multi-Factor authentication
- Create other users (including additional user for ACCOUNTADMIN role)


```sql

--- User 1 ---
CREATE USER maria PASSWORD = '123'
DEFAULT_ROLE = ACCOUNTADMIN
MUST_CHANGE_PASSWORD = TRUE;

GRANT ROLE ACCOUNTADMIN TO USER maria;


--- User 2 ---
CREATE USER frank PASSWORD = '123'
DEFAULT_ROLE = SECURITYADMIN
MUST_CHANGE_PASSWORD = TRUE;

GRANT ROLE SECURITYADMIN TO USER frank;


--- User 3 ---
CREATE USER adam PASSWORD = '123'
DEFAULT_ROLE = SYSADMIN
MUST_CHANGE_PASSWORD = TRUE;

GRANT ROLE SYSADMIN TO USER adam;
```

## SECURITYADMIN
- Account admin tab (limited features)
- Create and manage users and roles
- Grant and revoke privileges to roles

## SECURITYADMIN in practice
- Create Sales Admin Role that inherits privileges from the created Sales Role, assigned to the SYSADMIN
- Create a HR Admin Role that inherits privileges from the HR Role but is not connected to the SYSADMIN

Log in with user FRANK:


```sql
// IN Frank account default is SECURITYADMIN
// Cannot view billing but can view users and role

-- SECURITYADMIN role --
--  Create and Manage Roles & Users --


-- Create Sales Roles & Users for SALES--

create role sales_admin;
create role sales_users;

-- Create hierarchy
grant role sales_users to role sales_admin;

-- As per best practice assign roles to SYSADMIN
grant role sales_admin to role SYSADMIN;


-- create sales user
CREATE USER simon_sales PASSWORD = '123' DEFAULT_ROLE =  sales_users
MUST_CHANGE_PASSWORD = TRUE;
GRANT ROLE sales_users TO USER simon_sales;

-- create user for sales administration
CREATE USER olivia_sales_admin PASSWORD = '123' DEFAULT_ROLE =  sales_admin
MUST_CHANGE_PASSWORD = TRUE;
GRANT ROLE sales_admin TO USER  olivia_sales_admin;

-----------------------------------

-- Create Sales Roles & Users for HR--

create role hr_admin;
create role hr_users;

-- Create hierarchy
grant role hr_users to role hr_admin;

-- This time we will not assign roles to SYSADMIN (against best practice)
-- grant role hr_admin to role SYSADMIN;


-- create hr user
CREATE USER oliver_hr PASSWORD = '123' DEFAULT_ROLE =  hr_users
MUST_CHANGE_PASSWORD = TRUE;
GRANT ROLE hr_users TO USER oliver_hr;

-- create user for sales administration
CREATE USER mike_hr_admin PASSWORD = '123' DEFAULT_ROLE =  hr_admin
MUST_CHANGE_PASSWORD = TRUE;
GRANT ROLE hr_admin TO USER mike_hr_admin;
```

## SYSADMIN
- Create and manage objects
- Create and manage warehouses, databases, tables, etc.
- Custom roles should be assigned to the SYSADMIN role as the parent
- Then this role also has the ability to grant privileges on warehouses, databases, and other objects to the custom roles

## SYSADMIN in practice
- Create a virtual warehouse and assign it to the custom roles
- Create a database and table and assign it to the custom roles



```sql
// / IN Adam account default is SYSADMIN  
// SALES_ADMIN and SALES_USERS roles visible from this account as these roles are attached to the SYSADMIN

-- SYSADMIN --

-- Create a warehouse of size X-SMALL
create warehouse public_wh with
warehouse_size='X-SMALL'
auto_suspend=300
auto_resume= true;

-- grant usage to role public
grant usage on warehouse public_wh
to role public;

-- create a database accessible to everyone
create database common_db;
grant usage on database common_db to role public;

-- create sales database for sales
create database sales_database;
// grant ownership of object to sales_admin
// transfer ownership to sales_admin
grant ownership on database sales_database to role sales_admin;
grant ownership on schema sales_database.public to role sales_admin;

SHOW DATABASES; // owner of sales_database is SALES_ADMIN but SYSADMIN has all inherited privileges so can manage objects


-- create database for hr
created database hr_db;
// grant ownership to hr_admin
// Error: Database 'HR_DB does not exist or not authorized'
grant ownership on database hr_db to role hr_admin;
grant ownership on schema hr_db.public to role hr_admin;
// we have transferred the ownership to the role hr_admin which does not inherit privileges of the hr_admin role
// cannot drop database or see the database in SYSADMIN account
drop database hr_db;
// demonstrates that custom roles should be assigned to SYSADMIN so the SYSADMIN can manage privileges and ownership of objects

```

## Custom roles
- Customize roles to our needs and create own hierarchies that reflect the needs of our organisation (roles for business lines / departments)
- Custom roles are usually created by SECURITYADMIN and then assigned to SYSADMIN (which can manage all of the objects that are created by the custom roles )
- Should be leading up to the SYSADMIN role


```sql
// IN SYSADMIN role account
// SALES_ADMIN and SALES_USERS roles visible

// USE role SALES_ADMIN to administrate objects for the sales team
USE ROLE SALES_ADMIN;
USE SALES_DATABASE;

-- Create table --
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

SHOW TABLES; // ownership is SALES_ADMIN

-- query from table --
SELECT* FROM CUSTOMERS;
USE ROLE SALES_USERS;

-- grant usage to role
USE ROLE SALES_ADMIN;

// setup privileges to allow sale users to query from this table and use the database and schema
GRANT USAGE ON DATABASE SALES_DATABASE TO ROLE SALES_USERS;
// grant access to schema
GRANT USAGE ON SCHEMA SALES_DATABASE.PUBLIC TO ROLE SALES_USERS;
// grant select on table usage
GRANT SELECT ON TABLE SALES_DATABASE.PUBLIC.CUSTOMERS TO ROLE SALES_USERS;


-- Validate privileges --
USE ROLE SALES_USERS;
SELECT* FROM CUSTOMERS;
// insufficient privileges to drop or delete
DROP TABLE CUSTOMERS;
DELETE FROM CUSTOMERS;
SHOW TABLES;

-- grant DROP on table
// use sales admin to grant delete to sale users role
USE ROLE SALES_ADMIN;
GRANT DELETE ON TABLE SALES_DATABASE.PUBLIC.CUSTOMERS TO ROLE SALES_USERS;

// using role sales users
USE ROLE SALES_USERS;
SELECT* FROM CUSTOMERS;
// delete now works as role has delete privileges
DROP TABLE CUSTOMERS;
DELETE FROM CUSTOMERS;
SHOW TABLES;


USE ROLE SALES_ADMIN;

```

## USERADMIN
- Create users and roles (dedicated to User and Role management)
- Not for granting privileges (only the one that is own)
- No global grant privileges

## USERADMIN in practice



```sql
-- USERADMIN --

// Solve the problem of the hr users roles that have object created but are not connected to the SYSADMIN

// USERADMIN has create user privileges and can create roles

// Create user with defauly role ACCOUNTADMIN
--- User 4 ---
CREATE USER ben PASSWORD = '123'
DEFAULT_ROLE = ACCOUNTADMIN
MUST_CHANGE_PASSWORD = TRUE;

// Grant not executed: insufficient privileges
GRANT ROLE HR_ADMIN TO USER ben;


// Using SECURITYADMIN
// Grant successful
GRANT ROLE HR_ADMIN TO USER ben;

// Using SECURITYADMIN
// grant role hr_admin to role SYSADMIN to fix hr problem that is not connected to SYSADMIN
GRANT ROLE HR_ADMIN TO ROLE SYSADMIN;

SHOW ROLES;
// ROLE HR_ADMIN has owner SECURITYADMIN, therefore USER_ADMIN cannot grant roles or privileges
// USE SECURITYADMIN to perform this task

```

## PUBLIC
- Least privileged role (bottom of hierarchy)
- Every user is automatically assigned to this role
- Can own object (role can ben given access to a database, and therefore can create and own objects)
- These objects then visible and available to everyone as PUBLIC role is assigned to every user by default
- Use case: If we want a Table or database accessible by everyone, then grant this privilege to the role PUBLIC 
