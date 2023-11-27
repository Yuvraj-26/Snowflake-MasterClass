## Snowflake Architecture

## Data Warehouse
Database that is used for consolidating and integrating various data sources and use them for reporting and data analysis.

A data warehouse is a type of data management system that is designed to enable and support business intelligence (BI) activities, especially analytics. Data warehouses are solely intended to perform queries and analysis and often contain large amounts of historical data. The data within a data warehouse is usually derived from a wide range of sources such as application log files and transaction applications.

A data warehouse centralizes and consolidates large amounts of data from multiple sources. Its analytical capabilities allow organizations to derive valuable business insights from their data to improve decision-making. Over time, it builds a historical record that can be invaluable to data scientists and business analysts. Because of these capabilities, a data warehouse can be considered an organization’s “single source of truth.”

ETL - Extract data from data source, Transform the data (cleaning, quality checks, transformations), Load the Data

- Extract raw data from data sources
- Data integration (create relationships between data, clean, aggregate and transform data)
- Access layer (make data available to different solutions such as reporting, data science, applications that use the data warehouse as a source)

In snowflake:
- Raw data (staging area | internal or external stage)
- Data integration layer (Data transformation)
- Access layer

## Cloud Computing
Traditional Data Center - physical available on site, causes overheads in terms of maintaining infrastructure, security against physical and virtual breach, electricity for server cooling, software/hardware upgrades - managing virtual machines. Software-as-a-service SaaS moves software deployment and management to third-party cloud software service.

- SaaS means we are only responsible for the Application layer (Databases, tables) according to our data warehouse needs.
- Snowflake handles Software, Data, OS layers - by managing data storage, virtual warehouses, upgrades, metadata.
- Cloud provider (AWS, Azure, GCP) handles physical servers, virtual machines, and physical storage layers.

## Snowflake Editions
- Standard - introductory level
- Enterprise - for large scale enterprises
- Business Critical - high levels of data protection with extremely sensitive data
- Virtual Private - highest level of security with dedicated virtual servers and a completely separate Snowflake environment

## Snowflake Pricing
- Exchange currency $£ in Snowflake credits
- Decoupled the Compute from the storage
- Pay only what you needs
- Scalable amount of storage and compute resources at affordable cloud price
- Pricing depends on the region and platform
  - **Storage**
    - monthly storage fee
    - based on average storage used per monthly
    - cost calculated after hybrid columnar compression
    - cloud providers
  - **Compute**
    - charged for active warehouse per hour
    - depending on size of the warehouses
    - billed by second (minimum of 1 min)
    - charged in Snowflake credits

## Virtual Warehouse Sizes
Increasing size of the warehouse, consumption of credit per time increases but queries are processed quicker. Suitable virtual warehouse size must be selected based on complexity for queries and workload.

## Snowflake Storage Pricing
- On demand storage - pay only for what you used
- Capacity storage - pay only for defined capacity upfront
- For choosing between On demand Storage or Capacity Storage
  - Start with On Demands
  - Once you are sure about your usage use, choose Capacity Storage

## Monitor Usage
- Admin - ACCOUNTADMIN role
- Usage
  - Compute resources (warehouses) and select warehouse resource and view query usage, Cloud services only warehouse refers to compute costs related to authentication/metadata handling for cloud services, Storage is charged based monthly and can be viewed as data stored per per object, Data transfer means bringing data into and out of Snowflake
    - Not charged for incoming data - from AWS into Snowflake
    - Not charged for sharing data with another Snowflake account within the same region and same cloud provider, else charged for data transfer
- Fail-safe provides a (non-configurable) 7-day period during which historical data may be recoverable by Snowflake. Protects data in the event of system failure for example.

## Roles in Snowflake
- Top level is ACCOUNT ADMIN, combines the roles of SECURITYADMIN, AND SYSADMIN
- Below the SECURITYADMIN is the USERADMIN
- Lowest level role is PUBLIC
- Can create Custom roles and its recommended to assign them using with SYSADMIN and create hierarchy of Roles and inherited privileges 
