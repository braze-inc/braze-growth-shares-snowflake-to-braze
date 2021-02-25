# Braze Snowflake Connector
## Summary
The Braze Snowflake Connector is a python3 script using the [Snowflake python connector](https://docs.snowflake.net/manuals/user-guide/python-connector.html) that pulls data from Snowflake and sends it to Braze.

The connector works by pulling data from Snowflake tables of `attributes` or `events`, then transforms the data, and posts the data to Braze via the [Rest API Endpoint](https://www.braze.com/docs/developer_guide/rest_api/user_data/#user-track-request). Company settings are kept in a configuration table, and [Snowflake Seq](https://docs.snowflake.net/manuals/user-guide/querying-sequences.html) are used as an auto incrementing identifier to keep track of what was processed.

The process will be run in batches based on the `braze_maxrecords` size. Do not set the value higher than what's [supported by the api](https://www.braze.com/docs/developer_guide/rest_api/basics/#api-limits). We highly recommend testing this script against the size of your tables (i.e, the number of users, event volume, and attribute volume).

  * **`braze_id` or `external_id` is required to be in the tables.**
  * **An auto-incrementing id is required to be in the tables.**
  * **While running, `last_response` will be set to a `processing` state to avoid repetitive runs.**
  * **For `events` table, `name` and `event` are both required fields.**
  * **Data can be added to the table via streaming or manually.**
  * **API is chunked based on maximum api limit ie `75`.**
  * **Max Columns/Properties ie `attributes` are limited to the api limit ie `75`.**
  * **`attributes` and `events` names are lowercase prior to sending to braze.**
  * **Uses a `rsaFile` or `rsaPrivateKey` for authentication.**


## Config Table
The config table will contain the configuration settings per company and table name. It contains the following information:

| Column Name | Type | Description |
|------------|------------|------------|
| company_name | VARCHAR | name of company |
| table_name | VARCHAR | name of table to query |
| table_type | VARCHAR | type of data in table ie "attributes" or "events" default "attributes" |
| braze_id_type | VARCHAR | braze api identifier. possible values "braze_id" or "external_id" |
| braze_id_column | VARCHAR | column which contains the "braze_id". |
| braze_index | VARCHAR | column which contains the index/table sequence id. Optional if "process_full_table" is true|
| braze_endpoint | VARCHAR | rest url of endpoint to submit data  |
| braze_apikey | VARCHAR | api key for endpoint |
| braze_maxrecords | NUMBER | maximum lines per record. default 75 |
| braze_maxcolumns | NUMBER | maximum attributes or events per record. default 75 |
| last_rownum | NUMBER | number of last row the api was ran at. |
| created_date | TIMESTAMP_TZ | date of when config was created |
| lastrun_date | TIMESTAMP_TZ | date of when report was last ran |
| enabled | BOOLEAN | if config setting is valid and ready to be run |
| last_message | VARCHAR | message of when process was last ran |
| last_response | VARCHAR | response of when process was last ran |
| process_full_table | BOOLEAN | set to true if entire table should be run every time. **See note below** |

**For `process_full_table`, this will reprocess the entire table every time. Please be aware of any performance issues, and data point usage when enabled.
An updated table of only the data to be process should be generated prior to running the schedule. Or a view may be used which automatically references only the necessary data.**

### Config Table Creation
```
create or replace TABLE BZ_CONNECTOR_CONFIG (
  company_name VARCHAR NOT NULL comment 'name of company',
  table_name VARCHAR NOT NULL comment 'name of table to query',
  table_type VARCHAR NOT NULL comment 'type of data in table ie "attributes" or "events" default "attributes"',
  braze_id_type VARCHAR NOT NULL DEFAULT 'braze_id' comment 'braze api identifier. possible values "braze_id" or "external_id"',
  braze_id_column VARCHAR NOT NULL DEFAULT 'braze_id' comment 'column which contains the "braze_id".',
  braze_index VARCHAR NOT NULL DEFAULT 'row_id' comment 'column which contains the index/table id.',
  braze_endpoint VARCHAR NOT NULL comment 'rest url of endpoint to submit data ',
  braze_apikey VARCHAR NOT NULL comment 'api key for endpoint',
  braze_maxrecords NUMBER DEFAULT 75 comment 'maximum lines per record. default 75',
  braze_maxcolumns NUMBER DEFAULT 75 comment 'maximum attributes or events per record. default 75',
  last_rownum NUMBER DEFAULT 0 comment 'number of last row the api was ran at.',
  created_date TIMESTAMP_TZ  DEFAULT CURRENT_TIMESTAMP() comment 'date of when config was created',
  lastrun_date TIMESTAMP_TZ  DEFAULT CURRENT_TIMESTAMP() comment 'date of when report was last ran',
  enabled BOOLEAN DEFAULT TRUE comment 'if config setting is valid',
  last_message VARCHAR comment 'message of when process was last ran',
  last_response VARCHAR comment 'response of when process was last ran',
  process_full_table BOOLEAN default false comment 'set to true to process entire table every time'
);
```

### Config Example:
|FIELD| EXAMPLE |
|-------|------- |
|COMPANY_NAME | Braze |
|TABLE_NAME | TEST_ATTRIBUTES |
|TABLE_TYPE | attributes |
|BRAZE_ID_TYPE | external_id |
|BRAZE_ID_COLUMN | braze_id |
|BRAZE_INDEX | row_id |
|BRAZE_ENDPOINT | https://www.braze.com/ |
|BRAZE_APIKEY | API_KEY |
|BRAZE_MAXRECORDS | 75 |
|BRAZE_MAXCOLUMNS | 75 |
|LAST_ROWNUM | 327 |
|CREATED_DATE | 2019-03-11 20:16:33.506 +0000 |
|LASTRUN_DATE | 2019-03-12 16:56:59.388 +0000 |
|ENABLED | TRUE |
|LAST_MESSAGE | {'attributes_processed': 45, 'message': 'success'} |
|LAST_RESPONSE | success |
|PROCESS_FULL_TABLE | false |

## Log Table
The log table will log the events of each api batch if `LogResponse` is set to `True`.
**Note potential latency and additional table writes for each batch. Set the `LogResponse` setting to `False` if this is an issue.**
The log table contains the following information:

| Column Name | Type | Description |
|------------|------------|------------|
| company_name | VARCHAR | name of company |
| table_name | VARCHAR | name of table to query |
| table_type | VARCHAR | type of data in table ie attributes or events |
| row_startnum | NUMBER | starting row number when process starts |
| row_endnum | NUMBER | ending row number when process ends |
| message | VARCHAR | message from response |
| response | VARCHAR | response from server |
| start_time | TIMESTAMP_TZ | when process started |
| end_time | TIMESTAMP_TZ | when process finished |
| number_updated | NUMBER | count of how many records was updated by this api call |

### Log Table Creation
```
create or replace TABLE BZ_CONNECTOR_LOG (
  company_name VARCHAR NOT NULL comment 'name of company',
  table_name VARCHAR NOT NULL comment 'name of table to query',
  table_type VARCHAR NOT NULL comment 'type of data in table ie attributes or events',
  row_startnum NUMBER  DEFAULT 0 comment 'starting row number when process starts',
  row_endnum NUMBER  DEFAULT 0 comment 'ending row number when process ends',
  message VARCHAR comment 'message from response',
  response VARCHAR comment 'response from server',
  start_time TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP() comment 'when process started',
  end_time TIMESTAMP_TZ comment 'when process finished',
  number_updated NUMBER DEFAULT 0 comment 'count of how many records was updated by this api call'
);

```

### Log Example:
| COMPANY_NAME | TABLE_NAME | TABLE_TYPE | ROW_STARTNUM | ROW_ENDNUM | MESSAGE | RESPONSE | START_TIME | END_TIME | NUMBER_UPDATED |
|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|
| Braze | TEST_ATTRIBUTES | attributes | 5 | 79 | success | {'attributes_processed': 75, 'message': 'success'} | 2019-03-11 16:17:38.979 +0000 | 2019-03-11 16:17:39.110 +0000 | 75 |
| Braze | TEST_ATTRIBUTES | attributes | 80 | 154 | success | {'attributes_processed': 75, 'message': 'success'} | 2019-03-11 16:17:39.862 +0000 | 2019-03-11 16:17:40.170 +0000 | 75 |


## Adding/Modifying Companies Instructions
Adding or Modifying companies within the configuration can be done via standard insert or update statements.
Disabling a company can be done by setting `enabled` to `false`.

```
insert into BZ_CONNECTOR_CONFIG (company_name, table_name, table_type, braze_id_type, braze_id_column, braze_index, braze_endpoint, braze_apikey, braze_maxrecords, braze_maxcolumns, last_rownum, enabled) values
('Braze','TEST_ATTRIBUTES','attributes','external_id','braze_id','row_id','https://wwww.braze.com/','API_KEY',75,75,0,true),
('Braze','TEST_EVENTS','events','external_id','braze_id','row_id','https://www.braze.com/','API_KEY',75,75,0,true);
```

# Script Setup
Script settings are saved in the `config.cfg` file.
The config file starts with the `[Snowflake]` section with the following info:

| SETTING | DESCRIPTION |
|--------|--------|
| sfUser  | Snowflake user name |
| sfPwd  | Base64 encoded password |
| sfAccount | Snowflake account |
| sfRegion | Snowflake region ie US-EAST-1 |
| sfDebug  | Boolean default False, debug state for logging |
| databaseName  | Snowflake database name |
| warehouseName  | Snowflake warehouse name |
| schemaName  | Snowflake schema name |
| configTable  | configuration table name |
| eventColPrefix  | Default `bce_`, custom event prefix for events column|
| eventColPostfix  | Default `_value`,  custom event postvalue for events value|
| rsaPrivateKey | full private ssh key OR |
| rsaFile | private ssh file |
| logResponse | Boolean default False, set if all individual response should be logged to log table. |
| logTable  | log table name |
| trackEndPoint  | braze user track endpoint |
| logName  | log file name, optional otherwise output to `stdout` |
| logSize  | Number default 5, max log file size in MB |


### API User
API user and role will need to be created if necessarily within Snowflake account settings.

### Setting up API Authentication
SSL is use to connect to Snowflake. Instructions and documentation can be [found here](https://docs.Snowflake.net/manuals/user-guide/snowsql-start.html#using-key-pair-authentication).

```
$ openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8
```

Phasephrase will be Base64 encoded for `sfPwd` config setting.

```
$ openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

Set the RSA public key from `rsa_key.pub` to user profile within Snowflake:
```
alter user [username] set rsa_public_key='MIIBIjANBgkqh...';
```

Reference `rsa_key.p8` as `rsaFile` or the content as `rsaPrivateKey`.

  * **`rsaFile` or `rsaPrivateKey` config variables is required.**


## Script Config Example
`config.cfg` file example:

```
[snowflake]
sfUser = API_USER                                     # Snowflake user name
sfPwd = BASE64_PWD                                    # [Base64 encoded password](https://www.base64encode.org/)
sfAccount= AAA1234                                    # Snowflake account
sfRegion= US-EAST-1                                   # Snowflake region ie US-EAST-1
sfDebug = True                                        # Boolean, debug state for logging, optional
databaseName = BRAZE_DB                               # Snowflake database name
warehouseName = BRAZE_WH                              # Snowflake warehouse name
schemaName = BRAZE                                    # Snowflake schema name
configTable = BZ_CONFIG                               # configuration table name
eventColPrefix = bce_                                 # custom event prefix for events column
eventColPostfix = _value                              # custom event postfix for events value
rsaPrivateKey = -----BEGIN ENCRYPTED PRIVATE KEY----- # rsa private key OR
  IW4hANOBVTN0ljhwHjIw/ktsYRrKPTo3iAENn
  fmWx0bdRckBOWHsWf44zllxShwvUYFKRTSgSW
  5RE4Ue19Is7YxJ2HPbVN7d3GDb6JUF+T1FawQ
  PBQ2UqnxIiC5iPTv09RiYLuG6s1coVtY8lwbY
  N2H6kKVbU7Qc61TgFOsWHQqmmPDZJWH+BEke9
  hSWZps2zdMvxJ6bkgP7tlZcViutWmq5zvoQCT
  75xXTfqXkVrfhkvbag==
  -----END ENCRYPTED PRIVATE KEY-----
rsaFile = ./rsa_key.p8                                # rsa file location
logResponse = True                                    # log response for each batch
logTable = BZ_LOG                                     # log table name
trackEndPoint = /users/track                          # braze user track endpoint

logName = ./snowflake_braze_connector.log             # log file name, optional otherwise output to stdout
logSize = 5                                           # Number, max log file size in MB, optional
```

## Usage/Command line
To sync the data, run the python3 bash command with script name and company name:

```
  python3 snowflake_braze.py CompanyName
```

For companys with spaces, quote the company name:

```
  python3 snowflake_braze.py "Company Name"
```

Automated process can be set to run the sync multiple times throughout the day.


### Installing requirements
To run locally, the dependencies need to be installed:

```
  pip3 install -r requirements.txt
```

This can also be done via a virtual python environment to avoid dependency issues.

## Data Tables
The client will need to setup the feed from their DB to the changes DB. This can be done manually or via [Snowflake Streams and Tasks](https://docs.snowflake.net/manuals/LIMITEDACCESS/data-load-elt.html). Tables can be shared with Brazed via [Data Sharing](https://docs.snowflake.net/manuals/user-guide/data-sharing-intro.html).

### Attributes Table Setup
  * **Before creating the `attributes` table, ensure a [Snowflake Sequence](https://docs.snowflake.net/manuals/user-guide/querying-sequences.html) is created.**
  * **Attribute names will be lowercase prior to sending to Braze**
  * **For standard [Braze `attributes`](https://www.braze.com/docs/developer_guide/rest_api/user_data/#user-attributes-example-request), make sure the name matches.**

Create a table with attributes as necessary and include the sequence.

EXAMPLE:
```
create or replace sequence test_attribute_seq;
create or replace TABLE TEST_ATTRIBUTES (
  row_id number  DEFAULT test_attribute_seq.nextval comment 'row id. required unless process_full_table is true, but can be named differently based on config info.',
  braze_id VARCHAR NOT NULL  comment 'column with user id for Braze. required but can be named differently based on config info.',
  attr_1_string VARCHAR,
  attr_2_number NUMBER,
  attr_3_boolean BOOLEAN,
  attr_4_decimal DECIMAL,
  attr_5_date DATE,
  attr_6_timestamp TIMESTAMP,
  attr_7_variant VARIANT,
  attr_8_float FLOAT,
  att_9_time TIME
);
```

### Events Table Setup
  * **Before creating the `events` table, ensure a [Snowflake Sequence](https://docs.snowflake.net/manuals/user-guide/querying-sequences.html) is created.**
  * **Ensure there's a field with `name` and `time`.**
  * **For custom events properties, use `eventColPrefix` prefix and `eventColPostfix` postfix to map the custom property name to value.**
    * For example, if the event property name is `order_type`, `eventColPrefix` = `bce_` and `eventColPostfix` = `_value`:
     * The table will have a column called `bce_order_type` = `Order Item` and `bce_order_type_value` = `TV`.
     * The process will set a custom event properties of `Order Item` = `TV` for the custom event.
  * **Other fields will be send as is.**
  * **Event name will be lowercase prior to sending to Braze**

Create a table with events as necessary and include the sequence.

EXAMPLE:
```
create or replace sequence test_event_seq;
create or replace TABLE TEST_EVENTS (
  row_id number DEFAULT test_event_seq.nextval comment 'row id. required unless process_full_table is true, but can be named differently based on config info.',
  braze_id VARCHAR NOT NULL comment 'column with user id for Braze. required but can be named differently based on config info.',
  name VARCHAR NOT NULL comment 'event name. required',
  time TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP() comment 'event time. required',
// For custom event properties, use eventColPrefix and bce_order_type_value to map the property name to value
  bce_order_type varchar(50) default null,
  bce_order_type_value varchar(50) default null
  ...
  [eventColPrefix]PropertyName .... DEFAULT NULL,
  [eventColPrefix]PropertyName[eventColPostfix] .... DEFAULT NULL,
);
```

## Heroku
Snowflake Connector has been tested with [Heroku](https://www.heroku.com/) and the [Scheduler](https://elements.heroku.com/addons/scheduler) addon.

## Sample Heroku Instructions
```
git clone git@github.com:Appboy/Growth_Snowflake_Connector.git
cd Growth_Snowflake_Connector
heroku create [customid]-snowflake-connector

git push heroku master
```

### Set Heroku Environment variables:
Set all config variables within the Heroku app via the Dashboard (`Settings -> Config Vars`) or [command line](https://devcenter.heroku.com/articles/config-vars).

Install [Heroku Scheduler](https://elements.heroku.com/addons/scheduler) addon under `Resources`.

  * **Overall performance is approximately 100records/sec.**
    * **If overall records average per call is over 300k/hour, then alternative options should be explore for reliability purposes.**

### Set Schedule
Set a daily or hourly schedule the command can run.
```
python3 snowflake_braze.py [companyname]
```

# Braze Growth Shares
Growth Sharess are open source resources created by the Braze Growth Department.