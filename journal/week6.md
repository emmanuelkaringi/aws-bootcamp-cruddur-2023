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

## Create Parameters
aws ssm put-parameter --type "SecureString" --name "/cruddur/backend-flask/AWS_ACCESS_KEY_ID" --value $AWS_ACCESS_KEY_ID
aws ssm put-parameter --type "SecureString" --name "/cruddur/backend-flask/AWS_SECRET_ACCESS_KEY" --value $AWS_SECRET_ACCESS_KEY
aws ssm put-parameter --type "SecureString" --name "/cruddur/backend-flask/CONNECTION_URL" --value $PROD_CONNECTION_URL
aws ssm put-parameter --type "SecureString" --name "/cruddur/backend-flask/ROLLBAR_ACCESS_TOKEN" --value $ROLLBAR_ACCESS_TOKEN
aws ssm put-parameter --type "SecureString" --name "/cruddur/backend-flask/OTEL_EXPORTER_OTLP_HEADERS" --value "x-honeycomb-team=$HONEYCOMB_API_KEY"

## create an execution role & policy

`AWS/policies/service-assume-role-execution-policy.json`

{
    "Version":"2012-10-17",
    "Statement":[{
        "Action":["sts:AssumeRole"],
        "Effect":"Allow",
        "Principal":{
            "Service":["ecs-tasks.amazonaws.com"]
    }
  }]
}

aws iam create-role \    
--role-name CruddurServiceExecutionRole  \   
--assume-role-policy-document file://AWS/policies/service-assume-role-execution-policy.json

`AWS/policies/service-execution-policy.json`

{
    "Version":"2012-10-17",
    "Statement":[{
        "Action":["ssm:GetParameters", "ssm:GetParameter"],
        "Effect":"Allow",
        "Resource": "arn:aws:ssm:us-east-1:<placeholder>:parameter/cruddur/backend-flask/*"
  }]
}

aws iam put-role-policy \
  --policy-name CruddurServiceExecutionPolicy \
  --role-name CruddurServiceExecutionRole \
  --policy-document file://AWS/policies/service-execution-policy.json

## Create TaskRole
aws iam create-role \
    --role-name CruddurTaskRole \
    --assume-role-policy-document "{
  \"Version\":\"2012-10-17\",
  \"Statement\":[{
    \"Action\":[\"sts:AssumeRole\"],
    \"Effect\":\"Allow\",
    \"Principal\":{
      \"Service\":[\"ecs-tasks.amazonaws.com\"]
    }
  }]
}"

aws iam put-role-policy \
  --policy-name SSMAccessPolicy \
  --role-name CruddurTaskRole \
  --policy-document "{
  \"Version\":\"2012-10-17\",
  \"Statement\":[{
    \"Action\":[
      \"ssmmessages:CreateControlChannel\",
      \"ssmmessages:CreateDataChannel\",
      \"ssmmessages:OpenControlChannel\",
      \"ssmmessages:OpenDataChannel\"
    ],
    \"Effect\":\"Allow\",
    \"Resource\":\"*\"
  }]
}
"

aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess --role-name CruddurTaskRole
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --role-name CruddurTaskRole


Create Json file
Create a new folder called aws/task-defintions and place the following files in there:

backend-flask.json

{
    "family": "backend-flask",
    "executionRoleArn": "arn:aws:iam::AWS_ACCOUNT_ID:role/CruddurServiceExecutionRole",
    "taskRoleArn": "arn:aws:iam::AWS_ACCOUNT_ID:role/CruddurTaskRole",
    "networkMode": "awsvpc",
    "containerDefinitions": [
      {
        "name": "backend-flask",
        "image": "BACKEND_FLASK_IMAGE_URL",
        "cpu": 256,
        "memory": 512,
        "essential": true,
        "portMappings": [
          {
            "name": "backend-flask",
            "containerPort": 4567,
            "protocol": "tcp",
            "appProtocol": "http"
          }
        ],
        "logConfiguration": {
          "logDriver": "awslogs",
          "options": {
              "awslogs-group": "cruddur",
              "awslogs-region": "us-east-1",
              "awslogs-stream-prefix": "backend-flask"
          }
        },
        "environment": [
          {"name": "OTEL_SERVICE_NAME", "value": "backend-flask"},
          {"name": "OTEL_EXPORTER_OTLP_ENDPOINT", "value": "https://api.honeycomb.io"},
          {"name": "AWS_COGNITO_USER_POOL_ID", "value": ""},
          {"name": "AWS_COGNITO_USER_POOL_CLIENT_ID", "value": ""},
          {"name": "FRONTEND_URL", "value": ""},
          {"name": "BACKEND_URL", "value": ""},
          {"name": "AWS_DEFAULT_REGION", "value": ""}
        ],
        "secrets": [
          {"name": "AWS_ACCESS_KEY_ID"    , "valueFrom": "arn:aws:ssm:AWS_REGION:AWS_ACCOUNT_ID:parameter/cruddur/backend-flask/AWS_ACCESS_KEY_ID"},
          {"name": "AWS_SECRET_ACCESS_KEY", "valueFrom": "arn:aws:ssm:AWS_REGION:AWS_ACCOUNT_ID:parameter/cruddur/backend-flask/AWS_SECRET_ACCESS_KEY"},
          {"name": "CONNECTION_URL"       , "valueFrom": "arn:aws:ssm:AWS_REGION:AWS_ACCOUNT_ID:parameter/cruddur/backend-flask/CONNECTION_URL" },
          {"name": "ROLLBAR_ACCESS_TOKEN" , "valueFrom": "arn:aws:ssm:AWS_REGION:AWS_ACCOUNT_ID:parameter/cruddur/backend-flask/ROLLBAR_ACCESS_TOKEN" },
          {"name": "OTEL_EXPORTER_OTLP_HEADERS" , "valueFrom": "arn:aws:ssm:AWS_REGION:AWS_ACCOUNT_ID:parameter/cruddur/backend-flask/OTEL_EXPORTER_OTLP_HEADERS" }
          
        ]
      }
    ]
}

frontend-react.json

{
    "family": "frontend-react-js",
    "executionRoleArn": "arn:aws:iam::AWS_ACCOUNT_ID:role/CruddurServiceExecutionRole",
    "taskRoleArn": "arn:aws:iam::AWS_ACCOUNT_ID:role/CruddurTaskRole",
    "networkMode": "awsvpc",
    "containerDefinitions": [
      {
        "name": "frontend-react-js",
        "image": "BACKEND_FLASK_IMAGE_URL",
        "cpu": 256,
        "memory": 256,
        "essential": true,
        "portMappings": [
          {
            "name": "frontend-react-js",
            "containerPort": 3000,
            "protocol": "tcp", 
            "appProtocol": "http"
          }
        ],
  
        "logConfiguration": {
          "logDriver": "awslogs",
          "options": {
              "awslogs-group": "cruddur",
              "awslogs-region": "us-east-1",
              "awslogs-stream-prefix": "frontend-react"
          }
        }
      }
    ]
}
Register Task Defintion
aws ecs register-task-definition --cli-input-json file://aws/task-definitions/backend-flask.json
aws ecs register-task-definition --cli-input-json file://aws/task-definitions/frontend-react-js.json

## For Frontend React

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