# Week 6 — Deploying Containers

## Test RDS Connecetion
Add this test script into bin/db so we can easily check our connection from our container.


#!/usr/bin/env python3

import psycopg
import os
import sys

connection_url = os.getenv("CONNECTION_URL")

conn = None
try:
  print('attempting connection')
  conn = psycopg.connect(connection_url)
  print("Connection successful!")
except psycopg.Error as e:
  print("Unable to connect to the database:", e)
finally:
  conn.close()

Make it executable chmod u+x bin/db/test

## Health Check For Flask App
We'll add the following endpoint for our flask app:

@app.route('/api/health-check')
def health_check():
  return {'success': True}, 200

We'll create a new bin script at bin/flask/health-check

```py
#!/usr/bin/env python3

import urllib.request

try:
  response = urllib.request.urlopen('http://localhost:4567/api/health-check')
  if response.getcode() == 200:
    print("[OK] Flask server is running")
    exit(0) # success
  else:
    print("[BAD] Flask server is not running")
    exit(1) # false
# This for some reason is not capturing the error....
#except ConnectionRefusedError as e:
# so we'll just catch on all even though this is a bad practice
except Exception as e:
  print(e)
  exit(1) # false
```
Make the file executable chmod u+x bin/flask/health-check

## Create CloudWatch Log Group

aws logs create-log-group --log-group-name "/cruddur/fargate-cluster"
aws logs put-retention-policy --log-group-name "/cruddur/fargate-cluster" --retention-in-days 1

## Create ECS Cluster
aws ecs create-cluster \
--cluster-name cruddur \
--service-connect-defaults namespace=cruddur

