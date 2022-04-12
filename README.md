# dbt_fundamentals
## Useful links
* [Configure dbt on Snowflake with Docker](https://www.startdataengineering.com/post/cicd-dbt/)
* [How we configure Snowflake](https://blog.getdbt.com/how-we-configure-snowflake/)

# Setup/Get Started

There are two options for you to get started with developing in dbt on your local machine.
1. [Python virtual environment](##python-virtual-environment)
2. [Docker](##docker)

## Python virtual environment

Note that procedure will be different for Mac and Windows.
### Create Environment Mac

It's **very important that you use Python 3.6, 3.7 or 3.8** to create your environment. Python 3.9 is incompatible for M1 Mac as of Q2 2022.

To check what version you're running:
```{bash}
python3 --version
```

If you get 3.6, 3.7 or 3.8 you're good to continue. If you're running 3.9 you need to either downgrade to 3.8 or simply use the [Docker method](##docker) to run dbt.

Navigate to the root directory and run the following command to create the environment.
```{bash}
python3 -m venv dbt-env
```
You can read more about Python's virtual environments [here](https://docs.python-guide.org/dev/virtualenvs/). The basic principles is that an environment allows you to install everything you want for the versions you need , isolated from your local machine. To use it, it needs to be activated like this:
```{bash}
source dbt-env/bin/activate
```
To deactive run `deactivate` in the terminal. Run `source dbt-env/bin/activate` to get back to where you where:)

Now, we need to install dbt and the packages needed to connect it to our DWH (in this case Snowflake). **Before installing**, double check that you see `(dbt-env)` at the leftmost side of your terminal line. This inidicates that you are in the active environment we want to work in.

### Install dbt in your new environment
```{bash}
pip install dbt-core
pip install dbt-snowflake
```



### Create Environment Windows

If you don't have Python installed you can get Python 3.8 [HERE](https://python.org/downloads/release/python-380/). Choose the "Windows x86 executable installer".

To be continued...
## Docker

If you struggle to get Python and pip to work with dbt, Docker is the safest option. However, it comes at a price of being a bit slower and the commands are a bit less user friendly.

### Installation steps
1. Install [Docker Desktop](https://docs.docker.com/desktop/#download-and-install), choose Windows or Mac.
2. Run Docker Desktop on your machine. **Remember to keep Docker Desktop running in the background!**. Otherwise you can't execute docker-commands.

### Commands

```{bash}
docker run --rm -v $(pwd):/usr/app -v $(pwd):/root/.dbt fishtownanalytics/dbt:1.0.0 run  
docker run --rm -v $(pwd):/usr/app -v $(pwd):/root/.dbt fishtownanalytics/dbt:1.0.0 test
docker run --rm -v $(pwd):/usr/app -v $(pwd):/root/.dbt fishtownanalytics/dbt:1.0.0 docs generate
docker run --rm -ip 8080:8080 -v $(pwd):/usr/app -v $(pwd)/:/root/.dbt fishtownanalytics/dbt:1.0.0 docs serve
# Ctrl+C to exit
docker run --rm -ip 8080:8080 -v $(pwd):/usr/app -v $(pwd)/:/root/.dbt fishtownanalytics/dbt:1.0.0 clean
```
# Snowflake
1. Go to Snowflake's [signup page](https://signup.snowflake.com/). Input your name, email, company and country.
2. Choose "Enterprise" and AWS as cloud provider. You'll be given a 30 day free trial.
3. Open the confirmation email, choose a username and password. This will be your personal development credentials used to connect dbt to Snowflake

The trial project generates a unique Snowflake Account identifier that's hosted on AWS. You'll need this to complete the setup. The Account identifier is on the following format, `your-host.your-region`. Mine looked something like this `e*66***.eu-****-1`.

You access the project through this URL:  [https://your-host.your-region.snowflakecomputing.com/console](https://your-host.your-region.snowflakecomputing.com/console)

## Best practices Snowflake setup

For more detailed info see dbt's own [How we configure Snowflake](https://blog.getdbt.com/how-we-configure-snowflake/).
### The benefits of Snowflake compared to Redshift
* Unlike in Redshift, you can use the same connection to access separate logical databases and compute warehouses, all accessed via a single login.
* Also unlike Redshift, Snowflake allows traffic from all IP addresses by default. We highly recommend disabling this feature and explicitly whitelisting IP addresses by managing [Network Policies](https://docs.snowflake.com/en/user-guide/network-policies.html) in the online console.

### Two Databases
* `raw` - For raw data, obviously
* `analytics` - Tables and views used by analysts and reporting

### Three Warehouses
* `loading` - For ex. Fivetran will use this warehouse to load new data. Can put significant strain on DWH if not kept separate
* `transforming` - Where dbt performs all data transformations
* `reporting` - Where BI tools connect to run queries and report results to end users.

### Four Roles
* `public` - All users starts with this set of user permissions
* `loader` - Owns tables in the `raw` database and connects to `loading` DWH
* `transformer` - Query permissions in `raw` and owns tables in `analytics`
* `reporter` - Permissions on `analytics` **ONLY**

## Configure the Snowflake DWH
Log into Snowflake, open a worksheet and run the following commands as `ACCOUNTADMIN`. This creates the warehouses, databases, users and roles needed to follow best practices.

```{sql}
USE ROLE ACCOUNTADMIN; -- you need accountadmin for user creation, future grants

DROP USER IF EXISTS DBT_CLOUD;
DROP USER IF EXISTS DBT_CLOUD_DEV;
DROP ROLE IF EXISTS TRANSFORMER;
DROP ROLE IF EXISTS TRANSFORMER_DEV;
DROP DATABASE IF EXISTS PROD CASCADE;
DROP DATABASE IF EXISTS DEV CASCADE;
DROP WAREHOUSE IF EXISTS TRANSFORMING;
DROP WAREHOUSE IF EXISTS TRANSFORMING_DEV;

-- creating a warehouse
CREATE WAREHOUSE TRANSFORMING WITH WAREHOUSE_SIZE = 'XSMALL' WAREHOUSE_TYPE = 'STANDARD' AUTO_SUSPEND = 300 AUTO_RESUME = TRUE COMMENT = 'Warehouse to transform data';

-- creating database
CREATE DATABASE PROD COMMENT = 'production data base';

-- creating schemas
CREATE SCHEMA "PROD"."RAW" COMMENT = 'landing zone for raw data';
CREATE SCHEMA "PROD"."ANALYTICS" COMMENT = 'data layer for end user';

-- creating an access role
CREATE ROLE TRANSFORMER COMMENT = 'Role for dbt';

-- granting role permissions
GRANT USAGE,OPERATE ON WAREHOUSE TRANSFORMING TO ROLE TRANSFORMER;
GRANT USAGE,CREATE SCHEMA ON DATABASE PROD TO ROLE TRANSFORMER;
GRANT USAGE ON SCHEMA "PROD"."RAW" TO ROLE TRANSFORMER;
GRANT ALL ON SCHEMA "PROD"."ANALYTICS" TO ROLE TRANSFORMER;
GRANT SELECT ON ALL TABLES IN SCHEMA "PROD"."RAW" TO ROLE TRANSFORMER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA "PROD"."RAW" TO ROLE TRANSFORMER;

-- creating user and associating with role
CREATE USER DBT_CLOUD PASSWORD='abc123' DEFAULT_ROLE = TRANSFORMER MUST_CHANGE_PASSWORD = true;
GRANT ROLE TRANSFORMER TO USER DBT_CLOUD;

-----------------------------------------------------------------------------------------------
-- DEV
-- creating a warehouse
CREATE WAREHOUSE TRANSFORMING_DEV WITH WAREHOUSE_SIZE = 'XSMALL' WAREHOUSE_TYPE = 'STANDARD' AUTO_SUSPEND = 300 AUTO_RESUME = TRUE COMMENT = 'Dev warehouse to transform data';

-- cloning prod database (this clones schemas and tables as well)
CREATE DATABASE DEV CLONE PROD;

-- creating an access role
CREATE ROLE TRANSFORMER_DEV COMMENT = 'Dev role for dbt';

-- granting role permissions
GRANT USAGE,OPERATE ON WAREHOUSE TRANSFORMING_DEV TO ROLE TRANSFORMER_DEV;
GRANT USAGE,CREATE SCHEMA ON DATABASE DEV TO ROLE TRANSFORMER_DEV;
GRANT USAGE ON SCHEMA "DEV"."RAW" TO ROLE TRANSFORMER_DEV;
GRANT ALL ON SCHEMA "DEV"."ANALYTICS" TO ROLE TRANSFORMER_DEV;
GRANT SELECT ON ALL TABLES IN SCHEMA "DEV"."RAW" TO ROLE TRANSFORMER_DEV;
GRANT SELECT ON FUTURE TABLES IN SCHEMA "DEV"."RAW" TO ROLE TRANSFORMER_DEV;

-- creating user and associating with role
CREATE USER DBT_CLOUD_DEV PASSWORD='abc123' DEFAULT_ROLE = TRANSFORMER_DEV MUST_CHANGE_PASSWORD = true;
GRANT ROLE TRANSFORMER_DEV TO USER DBT_CLOUD_DEV;
```

To add a new developer to the team you can just create a new user and grant them the `TRANSFORMER_DEV` role permission in Snowflake.

### Logging in to PROD and DEV accounts
Paste the correct host and domain name to be promted by login screen
```
https://your-host.your-region.snowflakecomputing.com/console/login?disableDirectLogin=true
```

Access to PROD
```
username: DBT_CLOUD
password: abc123
```
Access to DEV
```
username: DBT_CLOUD_DEV
password: abc123
```