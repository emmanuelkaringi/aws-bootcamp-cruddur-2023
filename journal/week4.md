# Week 4 — Postgres and RDS

## Postgres
Postgres is a popular open-source relational database management system (RDBMS).

## Amazon Relational Database Service (RDS)
Amazon Relational Database Service (Amazon RDS) is a collection of managed services that makes it simple to set up, operate, and scale databases in the cloud. 

## Provision RDS Instance
```bash
aws rds create-db-instance \
  --db-instance-identifier cruddur-db-instance \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version  14.6 \
  --master-username root \
  --master-user-password huEE33z2Qvl383 \
  --allocated-storage 20 \
  --availability-zone ca-central-1a \
  --backup-retention-period 0 \
  --port 5432 \
  --no-multi-az \
  --db-name cruddur \
  --storage-type gp2 \
  --publicly-accessible \
  --storage-encrypted \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --no-deletion-protection
```

## Connect to psql via the psql client cli tool.
```bash
psql -Upostgres --host localhost
```

### Common PSQL commands:
```sql
\x on -- expanded display when looking at data
\q -- Quit PSQL
\l -- List all databases
\c database_name -- Connect to a specific database
\dt -- List all tables in the current database
\d table_name -- Describe a specific table
\du -- List all users and their roles
\dn -- List all schemas in the current database
CREATE DATABASE database_name; -- Create a new database
DROP DATABASE database_name; -- Delete a database
CREATE TABLE table_name (column1 datatype1, column2 datatype2, ...); -- Create a new table
DROP TABLE table_name; -- Delete a table
SELECT column1, column2, ... FROM table_name WHERE condition; -- Select data from a table
INSERT INTO table_name (column1, column2, ...) VALUES (value1, value2, ...); -- Insert data into a table
UPDATE table_name SET column1 = value1, column2 = value2, ... WHERE condition; -- Update data in a table
DELETE FROM table_name WHERE condition; -- Delete data from a table
```

## Create (and dropping) a database
```sql
CREATE database cruddur;
```
```sql
DROP database cruddur;
```

## Import Script
Create a new SQL file called `schema.sql` in backend-flask/db

The command to import: quit first `\q` before running this. cd into `backend-flask`.

```
psql cruddur < db/schema.sql -h localhost -U postgres
```

## Add UUID Extension
We are going to have Postgres generate out UUIDs. Add the code/line to the sql file.

```
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

## Shell Script to Connect to DB

For things we commonly need to do we can create a new directory called bin

We'll create a new folder called bin to hold all our bash scripts.

We'll create a new bash script bin/db-connect

```bash
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-connect"
printf "${CYAN}==== ${LABEL}${NO_COLOR}\n"

if [ "$1" = "prod" ]; then
  echo "Running in production mode"
  URL=$PROD_CONNECTION_URL
else
  URL=$CONNECTION_URL
fi

psql $URL
```

We'll make it executable:

` chmod u+x bin/db-connect ` 

To execute the script:

`./bin/db-connect`

## Shell script to drop the database
`./bin/db-drop`

```bash
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-drop"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<<"$CONNECTION_URL")
psql $NO_DB_CONNECTION_URL -c "drop database cruddur;"
```

We'll make it executable:

`chmod u+x bin/db-drop`

## Shell script to create the database
`./bin/db-create`

```bash
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-create"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<<"$CONNECTION_URL")
psql $NO_DB_CONNECTION_URL -c "create database cruddur;"
```

## Shell script to load the schema
`./bin/db-schema-load`

```bash
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-schema-load"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

# Get the absolute path of this script
ABS_PATH=$(readlink -f "$0")
# Get the parent of the parent path of this script (back-end folder)
PARENT_DIR="$(dirname "$(dirname "$ABS_PATH")")"

schema_path="$PARENT_DIR/db/schema.sql"


if [ "$1" = "prod" ]; then
  echo "---------  Running in production mode  ---------"
  URL=$PROD_CONNECTION_URL
else
  echo "---------  Running in local mode  --------"
  URL=$CONNECTION_URL
fi

psql $URL cruddur < $schema_path
```

## Make prints nicer
We we can make prints for our shell scripts coloured so we can see what we're doing:

https://stackoverflow.com/questions/5947742/how-to-change-the-output-color-of-echo-in-linux

```bash
CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-schema-load"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"
```

## Create our tables
https://www.postgresql.org/docs/current/sql-createtable.html

In the `schema.sql` file:

```sql
DROP TABLE IF EXISTS public.users;
DROP TABLE IF EXISTS public.activities;
```

```sql
CREATE TABLE public.users (
  uuid UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  display_name text,
  handle text
  cognito_user_id text,
  created_at TIMESTAMP default current_timestamp NOT NULL
);

```

```sql
CREATE TABLE public.activities (
  uuid UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  message text NOT NULL,
  replies_count integer DEFAULT 0,
  reposts_count integer DEFAULT 0,
  likes_count integer DEFAULT 0,
  reply_to_activity_uuid integer,
  expires_at TIMESTAMP,
  created_at TIMESTAMP default current_timestamp NOT NULL
);
```

## Create a seed file
In `bin` folder, create a new file `db-seed`

`chmod u+x bin/db-seed`

`./bin/db-seed`

```bash
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-seed"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

# Get the absolute path of this script
ABS_PATH=$(readlink -f "$0")
# Get the parent of the parent path of this script (back-end folder)
PARENT_DIR="$(dirname "$(dirname "$ABS_PATH")")"

seed_path="$PARENT_DIR/db/seed.sql"

if [ "$1" = "prod" ]; then
  echo "Running in production mode"
  URL=$PROD_CONNECTION_URL
else
  URL=$CONNECTION_URL
fi

psql $URL cruddur < $seed_path
```

In the `db` folder, create `seed.sql`

```sql
-- this file was manually created
INSERT INTO public.users (display_name, handle, cognito_user_id)
VALUES
  ('Andrew Brown', 'andrewbrown' ,'MOCK'),
  ('Andrew Bayko', 'bayko' ,'MOCK');

INSERT INTO public.activities (user_uuid, message, expires_at)
VALUES
  (
    (SELECT uuid from public.users WHERE users.handle = 'andrewbrown' LIMIT 1),
    'This was imported as seed data!',
    current_timestamp + interval '10 day'
  )
```

## See what connections we are using

Create a new file ib bin `db-session`

```bash
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-session"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

if [ "$1" = "prod" ]; then
  echo "Running in production mode"
  URL=$PROD_CONNECTION_URL
else
  URL=$CONNECTION_URL
fi

NO_DB_URL=$(sed 's/\/cruddur//g' <<<"$URL")
psql $NO_DB_URL -c "select pid as process_id, \
       usename as user,  \
       datname as db, \
       client_addr, \
       application_name as app,\
       state \
from pg_stat_activity;"
```

## Easily setup (reset) everything for our database
Create new file in bin `db-setup`

```bash
#! /usr/bin/bash
-e # stop excuting this script if it fails at any script of below so don't continue.

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-setup"
printf "${CYAN}==== ${LABEL}${NO_COLOR}\n"

# Get the absolute path of this script
ABS_PATH="$(readlink -f "$0")"
# Get the parent path of this script (bin folder)
bin_path="$(dirname "$ABS_PATH")"

source "$bin_path/db-drop"
source "$bin_path/db-create"
source "$bin_path/db-schema-load"
source "$bin_path/db-seed"
```

## Install Postgres Client
We need to set the env var for our backend-flask application:
```
  backend-flask:
    environment:
      CONNECTION_URL: "${CONNECTION_URL}"
https://www.psycopg.org/psycopg3/
```

We'll add the following to our requirments.txt

```
psycopg[binary]
psycopg[pool]
```

```bash
pip install -r requirements.txt
```

## DB Object and Connection Pool
`lib/db.py`

```py
from psycopg_pool import ConnectionPool
import os

def query_wrap_object(template):
  sql = f"""
  (SELECT COALESCE(row_to_json(object_row),'{{}}'::json) FROM (
  {template}
  ) object_row);
  """
  return sql

def query_wrap_array(template):
  sql = f"""
  (SELECT COALESCE(array_to_json(array_agg(row_to_json(array_row))),'[]'::json) FROM (
  {template}
  ) array_row);
  """
  return sql

connection_url = os.getenv("CONNECTION_URL")
pool = ConnectionPool(connection_url)
```

In our home activities we'll replace our mock endpoint with real api call:

```py
from lib.db import pool, query_wrap_array

      sql = query_wrap_array("""
      SELECT
        activities.uuid,
        users.display_name,
        users.handle,
        activities.message,
        activities.replies_count,
        activities.reposts_count,
        activities.likes_count,
        activities.reply_to_activity_uuid,
        activities.expires_at,
        activities.created_at
      FROM public.activities
      LEFT JOIN public.users ON users.uuid = activities.user_uuid
      ORDER BY activities.created_at DESC
      """)
      print(sql)
      with pool.connection() as conn:
        with conn.cursor() as cur:
          cur.execute(sql)
          # this will return a tuple
          # the first field being the data
          json = cur.fetchone()
      return json[0]
```

## Establish Connection to RDS database
In the terminal:
```bash
echo $PROD_CONNECTION_URL
psql $CONNECTION_URL
psql $PROD_CONNECTION_URL
```

In order to connect to the RDS instance we need to provide our Gitpod IP and whitelist for inbound traffic on port 5432.

`GITPOD_IP=$(curl ifconfig.me)`

We'll create an inbound rule for Postgres (5432) and provide the GITPOD ID.

We'll get the security group rule id so we can easily modify it in the future from the terminal here in Gitpod.

Whenever we need to update our security groups we can do this for access.
```bash
export DB_SG_ID=""
gp env DB_SG_ID=""


export DB_SG_RULE_ID=""
gp env DB_SG_RULE_ID=""
```
```bash
aws ec2 modify-security-group-rules \
    --group-id $DB_SG_ID \
    --security-group-rules "SecurityGroupRuleId=$DB_SG_RULE_ID,SecurityGroupRule={Description=GITPOD,IpProtocol=tcp,FromPort=5432,ToPort=5432,CidrIpv4=$GITPOD_IP/32}"
```

## Setup Cognito post confirmation lambda

Create the handler function
Create lambda in same vpc as rds instance Python 3.8

Add a layer for psycopg2 with one of the below methods for development or production

The function
```py
import json
import psycopg2

def lambda_handler(event, context):
    user = event['request']['userAttributes']
    try:
        conn = psycopg2.connect(
            host=(os.getenv('PG_HOSTNAME')),
            database=(os.getenv('PG_DATABASE')),
            user=(os.getenv('PG_USERNAME')),
            password=(os.getenv('PG_SECRET'))
        )
        cur = conn.cursor()
        cur.execute("INSERT INTO users (display_name, handle, cognito_user_id) VALUES(%s, %s, %s)", (user['name'], user['email'], user['sub']))
        conn.commit() 

    except (Exception, psycopg2.DatabaseError) as error:
        print(error)
        
    finally:
        if conn is not None:
            cur.close()
            conn.close()
            print('Database connection closed.')

    return event
```

Development
https://github.com/AbhimanyuHK/aws-psycopg2

This is a custom compiled psycopg2 C library for Python. Due to AWS Lambda missing the required PostgreSQL libraries in the AMI image, we needed to compile psycopg2 with the PostgreSQL libpq.so library statically linked libpq library instead of the default dynamic link.

EASIEST METHOD

Some precompiled versions of this layer are available publicly on AWS freely to add to your function by ARN reference.

https://github.com/jetbridge/psycopg2-lambda-layer

Just go to Layers + in the function console and add a reference for your region


ENV variables needed for the lambda environment.

```
PG_HOSTNAME='cruddur-db-instance.czz1cuvepklc.ca-central-1.rds.amazonaws.com'
PG_DATABASE='cruddur'
PG_USERNAME='root'
PG_PASSWORD=''
```