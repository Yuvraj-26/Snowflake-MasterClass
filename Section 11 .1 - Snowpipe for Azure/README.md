## Snowpipe for Azure


## Steps
- Azure container is blob storage for file storage
- Once file dropped into container
- Trigger Event notification which is placed in Queue storage (stores messages and notifications)
- Notifications/messages are consumed by notification integration (object set up in snowflake)
- Server-less load of files
- Load to Snowflake DB

## Setting up Snowpipe
- Storage Integration
  - connection details to container
  - grant permissions
- Create stage
  - location to container
- Queue and Notification
  - to trigger Snowpipe
- Notification Integration object in Snowflake
  - notification can be received by Snowflake
  - grant permissions
- Create pipe
  - Create pipe as object with COPY COMMAND so files can be loaded automatically

## Create stage and storage integration
- Create a Storage account on Azure
- Create a Container to store csv files
- This is the container we want to create a stage and storage integration for in Snowflake:


```sql
// create new database
CREATE OR REPLACE DATABASE SNOWPIPE;

-- create integration object that contains the access information
CREATE OR REPLACE STORAGE INTEGRATION azure_snowpipe_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  ENABLED = TRUE
  // TENANT ID stored in azure active directory / microsoft Entra ID / properties
  AZURE_TENANT_ID =  '<your-tenant-id>'
  // In container, properties, copy URL
  STORAGE_ALLOWED_LOCATIONS = ( 'azure://snowpipeaccountstorage.blob.core.windows.net/snowpipecsv');

// Grant permissions using Describe command
// AZURE_CONSENT_URL copy into browser
// Accept permission for SnowflakePAC

// Grant read access to application
// Storage Account, Access control (IAM)
// Add role assignment
// Add Storage Blob Data Reader - Allows for read access to Azure blob storage containers and data
// Assign access to service principal and select members snowflakepacint (found in service principals in enterprise applications)
// Accept and review

-- Describe integration object to provide access
DESC STORAGE integration azure_snowpipe_integration;

---- Create file format & stage objects ----

-- create file format

create or replace file format snowpipe.public.fileformat_azure
    TYPE = CSV
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1;

-- create stage object
create or replace stage snowpipe.public.stage_azure
    STORAGE_INTEGRATION = azure_snowpipe_integration
    // Container location - allowed location from storage integration, also used in stage
    URL = 'azure://snowpipeaccountstorage.blob.core.windows.net/snowpipecsv'
    FILE_FORMAT = fileformat_azure;


-- list files to test access
LIST @snowpipe.public.stage_azure;

```

## Create Notification Integration


```sql
// Now set up queue in Azure Portal in Queues
// Create Queue called snowpipequeue

// Set up event notification in Events
// Create event subscription
// Specify name and system topic name (topic to which events will be published)
// Event type is Blob Created as we want to be notified when file added to container
// Endpoint is the Storage Queue and Select storage account and queue previously created

// Error: creation of System topic failed - subscription must be registered
// Fix: In subscriptions, resource providers, register Microft.EventGrid

// Create notification integration
// object that checks queue storage where event notifications are stored
CREATE OR REPLACE NOTIFICATION INTEGRATION snowpipe_event
  ENABLED = true
  TYPE = QUEUE
  NOTIFICATION_PROVIDER = AZURE_STORAGE_QUEUE
  // Location of queue storage
  // In Queues copy URL
  AZURE_STORAGE_QUEUE_PRIMARY_URI = 'https://snowpipeaccountstorage33.queue.core.windows.net/snowpipequeue'
  AZURE_TENANT_ID = '4416da96-a5ea-4e5e-964a-1167d2b6f5e0';


  -- Register Integration
  // Grant permissions using Describe command
  // AZURE_CONSENT_URL copy into browser
  // Accept permission for second snowflakepacint application

  // Accountstorage, Access Control (IAM)
  // Add role assignment Storage Queue Data Contributor which allows read, write, delete access to Azure queues and queue messages
  // Select member as second snowflakepacint application (notification integration)
  // Review and assign

  DESC notification integration snowpipe_event;
```

## Create Pipe and Load data

- In Azure, Upload csv file to container
- Query file on the stage
- Create destination table
- Test copy command works
- Recreate table and remove file from Container, and dequeue message in queue
- Create pipe using as to attach tested Copy command
-  Upload csv file to container and table will be automatically loaded using the pipe

```sql
--query file on the stage, shows stage and storage integration works
SELECT
$1,
$2,
$3,
$4,
$5,
$6,
$7,
$8,
$9,
$10,
$11,
$12,
$13,
$14,
$15,
$16,
$17,
$18,
$19,
$20
FROM @snowpipe.public.stage_azure;


-- create destination table
create or replace table snowpipe.public.happiness (
  country_name varchar,
  regional_indicator varchar,
  ladder_score number(4,3),
  standard_error number(4,3),
  upperwhisker number(4,3),
  lowerwhisker number(4,3),
  logged_gdp number(5,3),
  social_support number(4,3),
  healthy_life_expectancy number(5,3),
  freedom_to_make_life_choices number(4,3),
  generosity number(4,3),
  perceptions_of_corruption number(4,3),
  ladder_score_in_dystopia number(4,3),
  explained_by_log_gpd_per_capita number(4,3),
  explained_by_social_support number(4,3),
  explained_by_healthy_life_expectancy number(4,3),
  explained_by_freedom_to_make_life_choices number(4,3),
  explained_by_generosity number(4,3),
  explained_by_perceptions_of_corruption number(4,3),
  dystopia_residual number (4,3));


// Test Copy command in destination table first
COPY INTO HAPPINESS
FROM @snowpipe.public.stage_azure; // status LOADED 149 rows loaded

SELECT * FROM snowpipe.public.happiness; // COPY command has worked

// Now COPY Command is tested, delete data, recreate table, and delete file from Azure container
// Dequeue message 1 from Queue in Azure portal so queue has 0 messages, pipe can now be created
TRUNCATE TABLE snowpipe.public.happiness;


-- create pipe
create pipe azure_pipe
auto_ingest = true
// use notification integration object
integration = 'SNOWPIPE_EVENT'
as
// as statement to attach copy command
copy into snowpipe.public.happiness
from @snowpipe.public.stage_azure;

// Upload CSV file into Container

SELECT * FROM snowpipe.public.happiness; // table initially empty and now data loaded into table
// Viewing the queue shows its empty as message has been consumed and dequeued

// View status of pipe using name of pipe
// execution-state is running, pending file count is 0, last ingested file path and timestamp shown, error messages will be displayed if error exists
SELECT SYSTEM$PIPE_STATUS( 'AZURE_PIPE' );

```
