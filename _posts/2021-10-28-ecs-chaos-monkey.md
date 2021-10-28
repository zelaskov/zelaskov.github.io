---
title: "ecs chaos monkey"
date: 2021-10-28
---

i always wanted to create chaos monkey to see how our system behaves if something suddenly goes down. i decided to create simple functions in python using boto3 and deploy it using [serverless](https://www.serverless.com/) framework.

---

let's start with lambda itself

**handler.py**

```
import boto3
import random
import json


cluster="some_cluster"
region="some_region"
ecs = boto3.client("ecs", region_name=region)


def list_services():
    response = ecs.list_services(
        cluster=cluster,
        )
    services = response["serviceArns"]
    return services

def delete_random_service(event, context):
    response = ecs.delete_service(
        cluster=cluster,
        service=random.choice(list_services())
    )
    print("chaos monkey randomly removed one of the services in " + json.dumps(response['service']['clusterArn']) + " and that service was a " + json.dumps(response['service']['serviceName'], indent=4, sort_keys=True, default=str))

```
* we import bodo3 as we are going to use it to go through services in ECS and we are going to use it to delete one of the services
* we import random as we are going to use it to choose random service from our ECS cluster
* we import json to make a response from AWS a little bit cleaner

i defined some variables that we are going to reuse in our functions - our ECS **cluster**, our **region** and we are assigning *boto3.client* to **ecs*.

our first function **list_services** simply goes through our **cluster** and returns *a lot of information* about services. we use **list_services** method from boto3 to achieve this. we don't really want to get all information about services, we only want their ARNs -> *response['serviceArns']*. we assign those ARNs to **services** and we return it.

our second function **delete_random_service** is using boto3 **delete_service** method. we are passing our ECS cluster and we use *random.choice()* to select our service from previous function. we print out the information which service chaos monkey removed. (not really proud of this, but it is only for test purposes)


**serverless.yml**

```
provider:
  name: aws
  runtime: python3.8
  stage: alpha
  lambdaHashingVersion: 20201221
  region: "some_region"
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - ecs:ListServices
        - ecs:DeleteService
      Resource: "*"


functions:
  function:
    handler: handler.delete_random_service
    events:
      - schedule:
          rate: rate(10 minutes)
          enabled: true
```

simple serverless template - we are selecting python 3.8 as a runtime, we pass our region. the most important thing is **iamRoleStatements**. i allowed *ecs:ListServices* and *ecs:DeleteService* on all resources as we are declaring what cluster we are going to use in our lambda function, so there is no way that we will remove services from production cluster for example. but you can pass cluster ARN if you want to.

i also added some cronjob to run function every 10 minutes, just for test purposes.

we type *serverless deploy*

```
Serverless: Stack update finished...
Service Information
service: ecs-chaos-monkey
stage: alpha
region: some_region
stack: ecs-chaos-monkey-alpha
resources: 8
api keys:
  None
endpoints:
functions:
  function: ecs-chaos-monkey-alpha-function
layers:
  None
```
we can check out CloudFormation console to see if our stack was created.

![cloudformation screenshot](/assets/2021-10-28/2021-10-28-[0].png)

in the resource tab of cloudformation stack we can go to our *loggroup* and *lambda function*

we can manually trigger our function from lambda console or wait 10 minutes to see if lambda will trigger automatically.

i waited and went into our CloudWatch log group and there it is!
```
chaos monkey randomly removed one of the services in "arn:aws:ecs:$some_region:$account_id:cluster/some_cluster" and that service was a "xyz"
```

you can also checkout ECS console if service was removed just to be sure.

if you still got any issues with setting it up, you can check out my [repo](https://github.com/zelaskov/ecs-chaos-monkey).
