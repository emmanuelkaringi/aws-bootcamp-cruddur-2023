# Week 2 — Distributed Tracing

## What is distributed tracing?
Distributed tracing is a method of tracking application requests as they flow from frontend devices to backend services and databases.

Can also be defined as a method of profiling and monitoring software applications that are composed of multiple services or microservices.

### Benefits of of distributed tracing
- Increased productivity.
- Improved collaboration among teams.
- Flexible implementation.

### Types of distributed tracing
1. **Instrumentation-based tracing**: This type of tracing involves adding instrumentation code to the application codebase. This instrumentation code is responsible for generating trace data and propagating it through the application. This type of tracing is usually more comprehensive and detailed, but can also be more resource-intensive.

2. **Tracing-proxy-based tracing**: This type of tracing involves using a tracing proxy or sidecar to intercept network traffic between services and collect trace data. This proxy can be added to the network infrastructure without requiring any changes to the application code. This type of tracing is usually less detailed but requires less overhead.

### Tools used for distributed tracing
- AWS X-Ray
- Honeycomb
- Jaeger
- Zipkin
- Datadog
- New Relic

## What is distributed logging?
Distributed logging is a method of aggregating and analyzing logs from multiple sources across a distributed system.

### Benefits of of distributed logging
- Centralized logging
- Scalability
- Real-time analysis
- Correlation

### Types of distributed logging
- **Centralized logging**: This is the most basic type of distributed logging, where logs from different sources are sent to a central location for storage and analysis. This central location can be a dedicated log server or a cloud-based log management platform. Centralized logging makes it easier to search, filter, and analyze logs from different sources in one place.

- **Distributed logging with agents**: This type of distributed logging uses logging agents or daemons installed on each server or container to collect and forward logs to a central repository. The agents can be configured to filter, transform, and enrich the logs before sending them to the central repository. This approach provides more control over the log collection process and allows for more customization.

- **Log forwarding**: In this type of distributed logging, logs are forwarded directly from one component to another, without necessarily going through a central location. This approach can be useful for reducing latency and network overhead, but it can also make it harder to aggregate and analyze logs from different sources.

- **Log streaming**: This type of distributed logging involves streaming logs in real-time to a central repository or analysis platform. The logs are usually indexed and searchable in real-time, which allows for quick detection and response to issues.

- **Log aggregation**: In this type of distributed logging, logs are collected from multiple sources and aggregated into a single stream or index. This can be useful for correlating logs from different sources and identifying patterns and trends in system behavior.

### Tools used for distributed logging
- RollBar
- AWS CloudWatch
- Elastic Stack
- Fluentd
- Graylog
- Splunk
- Loggly

## Install Honeycomb

Add the following code on the backend-flask/requirements.text
```
opentelemetry-api 
opentelemetry-sdk 
opentelemetry-exporter-otlp-proto-http 
opentelemetry-instrumentation-flask 
opentelemetry-instrumentation-requests
```

Install the dependency.
```
pip install -r requirements.txt
```

Add the following on the app.py
```
# Honeycomb
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
```
```
# Honeycomb
# Initialize tracing and an exporter that can send data to Honeycomb
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)
```

```
# Honeycomb
# Initialize automatic instrumentation with Flask
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```

In the docker-compose.yml, add the following code for the env variables to backend-flask
```
OTEL_EXPORTER_OTLP_ENDPOINT: "https://api.honeycomb.io"
OTEL_EXPORTER_OTLP_HEADERS: "x-honeycomb-team=${HONEYCOMB_API_KEY}"
OTEL_SERVICE_NAME: "${HONEYCOMB_SERVICE_NAME}"
```

From honeycomb.io, grab the unique code and create the gitpod env var
```
gp env HONEYCOMB_API_KEY=""
```


To create span and attribute, add the following code on the home_activities.py
```
from opentelemetry import trace
tracer = trace.get_tracer("home.activities")
```

```
with tracer.start_as_current_span("home-activities-mock-data"):
    span = trace.get_current_span()
```
```
span.set_attribute("app.now", now.isoformat())
 ```
 at the end of the code, put the following
 ```
span.set_attribute("app.result_lenght", len(results))

 ```

## Install Cloudwatch


Add to the requirements.txt
```
watchtower
```

Install the dependency.
```
pip install -r requirements.txt
```

Add the following code on the app.py on our backend-flask
```
# Cloudwatch
import watchtower
import logging
from time import strftime
```

```
# Configuring Logger to Use CloudWatch
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur')
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
LOGGER.info("test log")
```

```
@app.after_request
def after_request(response):
    timestamp = strftime('[%Y-%b-%d %H:%M]')
    LOGGER.error('%s %s %s %s %s %s', timestamp, request.remote_addr, request.method, request.scheme, request.full_path, response.status)
    return response
```

Add code to the requirements.text on the backend-flask folder
```
opentelemetry-instrumentation-requests
```

In the docker-compose.yml, add the following code for the env variables

```
AWS_DEFAULT_REGION: "${AWS_DEFAULT_REGION}"
AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
```

## Install Xray
Add the env var for the AWS region using the following command
```
export AWS_REGION="us-east-1"
gp env AWS_REGION="us-east-1"
```

In the backend-flask requirements.text, insert the following
```
aws-xray-sdk
```
Install the dependency.
```
pip install -r requirements.txt
```

Insert the following code inside the app.py

```
# Xray
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
```
```
# Xray
xray_url = os.getenv("AWS_XRAY_URL")
xray_recorder.configure(service='backend-flask', dynamic_naming=xray_url)
```

Create xray.json inside the folder aws/json/ and insert the following code

```
{
    "SamplingRule": {
        "RuleName": "Cruddur",
        "ResourceARN": "*",
        "Priority": 9000,
        "FixedRate": 0.1,
        "ReservoirSize": 5,
        "ServiceName": "backend-flask",
        "ServiceType": "*",
        "Host": "*",
        "HTTPMethod": "*",
        "URLPath": "*",
        "Version": 1
    }
  }
```

Run this command in the terminal to create the group on xray from the backend-flask folder
```
aws xray create-group \
   --group-name "Cruddur" \
   --filter-expression "service(\"backend-flask")"
```

From the main folder type the following command to create the sample rule
```
aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json
```

Add this line for the dockercompose.yml
```
AWS_XRAY_URL: "*4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}*"
AWS_XRAY_DAEMON_ADDRESS: "xray-daemon:2000"
```
Create the daemon necessary for Xray
```
xray-daemon:
    image: "amazon/aws-xray-daemon"
    environment:
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
      AWS_REGION: "eu-west-2"
    command:
      - "xray -o -b xray-daemon:2000"
    ports:
      - 2000:2000/udp
```

```
simple_processor = SimpleSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(simple_processor)
```

```
# xray
XRayMiddleware(app, xray_recorder)
```

## Install Rollbar
Create a new project on Rollbar

Add this code on requirements.text
```
blinker
rollbar
```

Install the dependency.
```
pip install -r requirements.txt
```

Create the env var.
```
export ROLLBAR_ACCESS_TOKEN=""
gp env ROLLBAR_ACCESS_TOKEN=""
```

Add to backend-flask for docker-compose.yml
```
ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"
```

Insert the following code to the backend-flask/app.py
```
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception
```
```
# Rollbar
rollbar_access_token = os.getenv('ROLLBAR_ACCESS_TOKEN')
@app.before_first_request
def init_rollbar():
    """init rollbar module"""
    rollbar.init(
        # access token
        rollbar_access_token,
        # environment name
        'production',
        # server root directory, makes tracebacks prettier
        root=os.path.dirname(os.path.realpath(__file__)),
        # flask already sets up logging
        allow_logging_basic_config=False)

    # send exceptions from `app` to rollbar, using flask's signal system.
    got_request_exception.connect(rollbar.contrib.flask.report_exception, app)
```
```
@app.route('/rollbar/test')
def rollbar_test():
    rollbar.report_message('Hello World!', 'warning')
    return "Hello World!"
```