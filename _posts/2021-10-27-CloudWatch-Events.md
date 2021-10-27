---
title: "monitor cloudformation api calls with cloudwatch events"
date: 2021-10-27
---

recently, i run into a issue that one of my colleagues removed CloudFormation stack because he thought that it is no longer used,and long story short, it caused some issues for us.
i started to wonder if there is a way to somehow monitor CloudFormation stacks. and then i remembered that CloudTrail stores all AWS API events, CloudFormation ones as well.

so basically i wanted any kind of notification if somebody deletes CloudFormation stack, preferably a mail. turns out, it is super easy to do using AWS SNS and AWS CloudWatch Event rules.

---

i created a whole config in terraform, but it could be done in CloudFormation as well.

**our event rule**

```
resource "aws_cloudwatch_event_rule" "event-rule" {
  name          = "cloudwatch-cloudformation-event"
  description   = "Capture CFN DeleteStack event"
  event_pattern = <<PATTERN
{
  "source": [
    "aws.cloudformation"
  ],
  "detail-type": [
    "AWS API Call via CloudTrail"
  ],
  "detail": {
    "eventSource": [
      "cloudformation.amazonaws.com"
    ],
    "eventName": [
      "DeleteStack"
    ]
  }
}
PATTERN
}

resource "aws_cloudwatch_event_target" "sns" {
  rule      = aws_cloudwatch_event_rule.event-rule.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.sns-topic.arn

  input_transformer {
    input_paths = {"stack":"$.detail.requestParameters.stackName","sourceIP":"$.detail.sourceIPAddress","eventTime":"$.detail.eventTime","userType":"$.detail.userIdentity.type","event":"$.detail.eventName","region":"$.detail.awsRegion","userName":"$.detail.userIdentity.userName"}
    input_template = <<TEMPLATE
    "Stack <stack> is processing event <event>. User that triggered event: <userType> <userName> at region <region>, IP <sourceIP> at <eventTime>."
    TEMPLATE
  }
}
```
first thing, rule itself. the most important part is the event patern - we are looking for AWS API CloudFormation calls, more precisely, we are looking for **DeleteStack** event. if it's needed we can also extend it to any other events that we are interested in.
then, we are creating a target for our CW Event rule which is SNS topic that we gonna create later. we also gonna use a **input_transformer** to transform a JSON that AWS will send us. we don't need all information that AWS will send us, just the stack name, sourceIP, eventTime, event, region and userType and userName.

**our sns topic**

```
resource "aws_sns_topic" "sns-topic" {
  name = "cloudformation-stack-delete"
}

resource "aws_sns_topic_policy" "default" {
  arn    = aws_sns_topic.sns-topic.arn
  policy = data.aws_iam_policy_document.sns-topic-policy.json
}

resource "aws_sns_topic_subscription" "sns-mail-subscription" {
  topic_arn = aws_sns_topic.sns-topic.arn
  protocol  = "email"
  endpoint  = "marcin.zelasko@icloud.com"
}

data "aws_iam_policy_document" "sns-topic-policy" {
  statement {
    effect  = "Allow"
    actions = ["SNS:Publish"]

    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }

    resources = [aws_sns_topic.sns-topic.arn]
  }
}
```

we are creating SNS topic with a really basic policyDocument that allows us to publish messages. then we are adding a subscription for that with **email** protocol and endpoint is our email. and that's basically it, we are running *terraform apply*

if it was successful we should get an email to confirm our subscription to SNS topic.
```
You have chosen to subscribe to the topic: 
arn:aws:sns:$REGION:$ACCID:cloudformation-stack-delete

To confirm this subscription, click or visit the link below (If this was in error no action is necessary): 
Confirm subscription
```
we click yes, and to try it out, i'll remove my test CFN stack.
```
"Stack arn:aws:cloudformation:$REGION:$ACCID:stack/testCFNstack/3d71x780-3azx-11eb-a455-12baxvce1294 is processing event DeleteStack. User that triggered event: IAMUser marcin.zelasko at region $REGION, IP $SOMEIP."
```
we got an email, with the only information that we need. (ofc, i changed some information to not put sensitive data here)

if you still got any issues with setting it up, you can check out my [repo](https://github.com/zelaskov/cloudwatch-cloudformation-sns).
