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

aws logs create-log-group --log-group-name cruddur
aws logs put-retention-policy --log-group-name cruddur --retention-in-days 1

## Create ECS Cluster
aws ecs create-cluster \
--cluster-name cruddur \
--service-connect-defaults namespace=cruddur

## Gaining Access to ECS Fargate Container

Login to ECR

aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com"

## Create ECR repo and push image

### For Base-image python

aws ecr create-repository \
  --repository-name cruddur-python \
  --image-tag-mutability MUTABLE

Set URL

export ECR_PYTHON_URL="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/cruddur-python"
echo $ECR_PYTHON_URL

Pull Image

docker pull python:3.10-slim-buster

Tag Image

docker tag python:3.10-slim-buster $ECR_PYTHON_URL:3.10-slim-buster

Push Image

docker push $ECR_PYTHON_URL:3.10-slim-buster

For Flask

In your flask dockerfile update the from to instead of using DockerHub's python image you use your own:

`FROM 257344229169.dkr.ecr.us-east-1.amazonaws.com/cruddur-python:3.10-slim-buster`

Create Repo

aws ecr create-repository \
  --repository-name backend-flask \
  --image-tag-mutability MUTABLE

Set URL

export ECR_BACKEND_FLASK_URL="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/backend-flask"
echo $ECR_BACKEND_FLASK_URL

Build Image - Ensure to cd into `backend-flask`

docker build -t backend-flask .

Tag Image

docker tag backend-flask:latest $ECR_BACKEND_FLASK_URL:latest

Push Image

docker push $ECR_BACKEND_FLASK_URL:latest

For Frontend React

Create Repo

aws ecr create-repository \
  --repository-name frontend-react-js \
  --image-tag-mutability MUTABLE

Set URL

export ECR_FRONTEND_REACT_URL="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/frontend-react-js"
echo $ECR_FRONTEND_REACT_URL

Build Image

docker build \
--build-arg REACT_APP_BACKEND_URL="https://4567-$GITPOD_WORKSPACE_ID.$GITPOD_WORKSPACE_CLUSTER_HOST" \
--build-arg REACT_APP_AWS_PROJECT_REGION="$AWS_DEFAULT_REGION" \
--build-arg REACT_APP_AWS_COGNITO_REGION="$AWS_DEFAULT_REGION" \
--build-arg REACT_APP_AWS_USER_POOLS_ID="ca-central-1_CQ4wDfnwc" \
--build-arg REACT_APP_CLIENT_ID="5b6ro31g97urk767adrbrdj1g5" \
-t frontend-react-js \
-f Dockerfile.prod \
.

Tag Image

docker tag frontend-react-js:latest $ECR_FRONTEND_REACT_URL:latest

Push Image

docker push $ECR_FRONTEND_REACT_URL:latest

If you want to run and test it

docker run --rm -p 3000:3000 -it frontend-react-js 