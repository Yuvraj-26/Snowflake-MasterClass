## Visualization - Power BI and Tableau


- Roles can be managed to the needs and hierarchy in the organisation
- Connecting tools to Snowflake

## Connecting Power BI & Snowflake

- Download Power BI
- Power BI Desktop
- Get data ... More
- Search Snowflake
- Connect 
- Specify Server (Snowflake Instance)
- Specify warehouse (authenticate with a user that has access to the warehouse - PUBLIC_WH for example)
- Data connectivity mode (DirectQuery from database or Import data from Snowflake and data is processed locally on machine - not directly using the warehouse but can refresh to update most recent data, import not suitable for very large files)
- Enter Snowflake credentials
- Select Database, Schema and Tables required (use preview)
- Transform data or Load data
- Select Fields and Visualizations

## Connecting Tableau & Snowflake

- Download Tableau
- Connect to a Server
- Select Snowflake
- Insert credentials and server information (username and password)
- Select warehouse
- Select database
- Select schema
- Drag and drop tables
- Edit relationships using keys
- View data by directly connecting to data, speed depends on warehouse size in snowflake
- Export data to be processed in Tableau engine
- Select Fields and Visualizations
