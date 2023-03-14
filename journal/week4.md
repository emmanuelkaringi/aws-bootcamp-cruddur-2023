# Week 4 — Postgres and RDS

## Provision RDS Instance
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

## Connect to psql via the psql client cli tool.

psql -Upostgres --host localhost

Common PSQL commands:
```
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
```
CREATE database cruddur;
```
```
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

We'll create an new folder called bin to hold all our bash scripts.

We'll create a new bash script bin/db-connect

`
#! /usr/bin/bash

psql $CONNECTION_URL
`

We'll make it executable:

chmod u+x bin/db-connect
To execute the script:

./bin/db-connect

## Shell script to drop the database
./bin/db-drop

#! /usr/bin/bash

NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<<"$CONNECTION_URL")
psql $NO_DB_CONNECTION_URL -c "DROP database cruddur;"

We'll make it executable:

chmod u+x bin/db-drop

## Shell script to create the database
./bin/db-create

#! /usr/bin/bash

echo "db-drop"

NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<<"$CONNECTION_URL")
psql $NO_DB_CONNECTION_URL -c "DROP database cruddur;"

## Shell script to load the schema
bin/db-schema-load

#! /usr/bin/bash

schema_path="$(realpath .)/db/schema.sql"

echo $schema_path

NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<<"$CONNECTION_URL")
psql $NO_DB_CONNECTION_URL cruddur < $schema_path