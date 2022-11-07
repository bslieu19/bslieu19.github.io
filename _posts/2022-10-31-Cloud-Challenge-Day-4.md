---
title: Cloud Challenge Day 4 - API Gateway and Lambda
date: 2022-10-31 12:00 -800
categories: [homelab]
tags: [cloud-challenge]
---

## Lambda Functions

Lambda functions are a serverless offering that allows us to respond to events. A lambda function will be used to execute a script that would grab information about the visitors to our website. 

Separation between clients and the backend is not possible just using lambda functions. This separation is important to make the website easier to troubleshoot and more control over how frontend elements interact with the backend. For this, we will use the API Gateway.

The lambda function creation was done using Node.js and the code to fetch the data was pasted into the script.

## API Gateway

The API Gateway sits between the lambda functions and the website. This provides the separation we are looking for. 

Through the API Gateway, the lambda function can be integrated to a specific path.

The gateway needs a method defined to determine if it should be using GET, POST, DELETE, etc, and a lambda function attached to it. It also needs a stage to be created so it knows what the path the application needs to go to invoke the lambda function. 

## Terraform

There were multiple depdencies to automate the deployment of both these resources. One important thing to note is in order to upload lambda function code, the file must be in zip format and lambda functions and API Gateway need a proper IAM role in order to talk to each other. 

For API Gateway:

```tf
resource "aws_api_gateway_rest_api" "example" {
  name        = "example"
  description = "This is my API for s3 website"
  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

resource "aws_api_gateway_deployment" "example" {
  depends_on = [
    aws_api_gateway_method.example,
    aws_api_gateway_integration.example
  ]
  rest_api_id = aws_api_gateway_rest_api.example.id

  triggers = {
    redeployment = sha1(jsonencode(aws_api_gateway_rest_api.example.body))
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_api_gateway_resource" "example" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  parent_id   = aws_api_gateway_rest_api.example.root_resource_id
  path_part   = "example"
}

#create a method 

resource "aws_api_gateway_method" "example" {
  rest_api_id   = aws_api_gateway_rest_api.example.id
  resource_id   = aws_api_gateway_resource.example.id
  http_method   = "ANY"
  authorization = "NONE"
}

#Create Integration

resource "aws_api_gateway_integration" "example" {
  rest_api_id             = aws_api_gateway_rest_api.example.id
  resource_id             = aws_api_gateway_resource.example.id
  http_method             = aws_api_gateway_method.example.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.example.invoke_arn
}

# Lambda

variable "AWS_DEFAULT_REGION" {
  type = string
}

variable "AWS_ACCOUNT_ID" {
  type = string
}

resource "aws_lambda_permission" "example" {
  statement_id  = "AllowExecutionFromAPIGateway"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.function_name
  principal     = "apigateway.amazonaws.com"

  # More: http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-control-access-using-iam-policies-to-invoke-api.html
  source_arn = "arn:aws:execute-api:${var.AWS_DEFAULT_REGION}:${var.AWS_ACCOUNT_ID}:${aws_api_gateway_rest_api.example.id}/*/${aws_api_gateway_method.example.http_method}${aws_api_gateway_resource.example.path}"
}

#Create a stage

resource "aws_api_gateway_stage" "example" {
  depends_on = [
    aws_api_gateway_rest_api.example,
    aws_api_gateway_deployment.example
  ]
  deployment_id = aws_api_gateway_deployment.example.id
  rest_api_id   = aws_api_gateway_rest_api.example.id
  stage_name    = "test"
}
```

For AWS Lambda Function:

```tf
resource "aws_cloudwatch_log_group" "example" {
  name = "/aws/lambda/example"
}
resource "aws_iam_role" "example" {
  name = "example"
  depends_on = [
    aws_cloudwatch_log_group.example
  ]

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      },
    ]
  })
}

resource "aws_iam_role_policy" "example" {
  name = "example"
  role = aws_iam_role.example.id

  policy = jsonencode(
    {
      "Version" = "2012-10-17"
      "Statement" = [
        {
          "Effect"   = "Allow"
          "Action"   = "logs:CreateLogGroup",
          "Resource" = "arn:aws:logs:region:example:*"
        },
        {
          "Effect" : "Allow",
          "Action" : [
            "logs:CreateLogStream",
            "logs:PutLogEvents"
          ],
          "Resource" : [
            "${aws_cloudwatch_log_group.example.arn}:*"
          ]
        },
        {
          "Action" : [
            "application-autoscaling:DeleteScalingPolicy",
            "application-autoscaling:DeregisterScalableTarget",
            "application-autoscaling:DescribeScalableTargets",
            "application-autoscaling:DescribeScalingActivities",
            "application-autoscaling:DescribeScalingPolicies",
            "application-autoscaling:PutScalingPolicy",
            "application-autoscaling:RegisterScalableTarget",
            "cloudwatch:DeleteAlarms",
            "cloudwatch:DescribeAlarmHistory",
            "cloudwatch:DescribeAlarms",
            "cloudwatch:DescribeAlarmsForMetric",
            "cloudwatch:GetMetricStatistics",
            "cloudwatch:ListMetrics",
            "cloudwatch:PutMetricAlarm",
            "cloudwatch:GetMetricData",
            "datapipeline:ActivatePipeline",
            "datapipeline:CreatePipeline",
            "datapipeline:DeletePipeline",
            "datapipeline:DescribeObjects",
            "datapipeline:DescribePipelines",
            "datapipeline:GetPipelineDefinition",
            "datapipeline:ListPipelines",
            "datapipeline:PutPipelineDefinition",
            "datapipeline:QueryObjects",
            "ec2:DescribeVpcs",
            "ec2:DescribeSubnets",
            "ec2:DescribeSecurityGroups",
            "iam:GetRole",
            "iam:ListRoles",
            "kms:DescribeKey",
            "kms:ListAliases",
            "sns:CreateTopic",
            "sns:DeleteTopic",
            "sns:ListSubscriptions",
            "sns:ListSubscriptionsByTopic",
            "sns:ListTopics",
            "sns:Subscribe",
            "sns:Unsubscribe",
            "sns:SetTopicAttributes",
            "lambda:CreateFunction",
            "lambda:ListFunctions",
            "lambda:ListEventSourceMappings",
            "lambda:CreateEventSourceMapping",
            "lambda:DeleteEventSourceMapping",
            "lambda:GetFunctionConfiguration",
            "lambda:DeleteFunction",
            "resource-groups:ListGroups",
            "resource-groups:ListGroupResources",
            "resource-groups:GetGroup",
            "resource-groups:GetGroupQuery",
            "resource-groups:DeleteGroup",
            "resource-groups:CreateGroup",
            "tag:GetResources",
            "kinesis:ListStreams",
            "kinesis:DescribeStream",
            "kinesis:DescribeStreamSummary"
          ],
          "Effect" : "Allow",
          "Resource" : "*"
        },
        {
          "Action" : "cloudwatch:GetInsightRuleReport",
          "Effect" : "Allow",
          "Resource" : "arn:aws:cloudwatch:*:*:insight-rule/DynamoDBContributorInsights*"
        },
        {
          "Action" : [
            "iam:PassRole"
          ],
          "Effect" : "Allow",
          "Resource" : "*",
          "Condition" : {
            "StringLike" : {
              "iam:PassedToService" : [
                "application-autoscaling.amazonaws.com",
                "application-autoscaling.amazonaws.com.cn"
              ]
            }
          }
        },
        {
          "Effect" : "Allow",
          "Action" : [
            "iam:CreateServiceLinkedRole"
          ],
          "Resource" : "*",
          "Condition" : {
            "StringEquals" : {
              "iam:AWSServiceName" : [
                "replication.dynamodb.amazonaws.com",
                "dax.amazonaws.com",
                "dynamodb.application-autoscaling.amazonaws.com",
                "contributorinsights.dynamodb.amazonaws.com",
                "kinesisreplication.dynamodb.amazonaws.com"
              ]
            }
          }
        }
      ]
    }
  )
}

resource "aws_lambda_function" "example" {
  #name = "example"
  filename      = "index.zip"
  function_name = "example"
  runtime       = "nodejs16.x"
  handler       = "index.handler"
  role          = aws_iam_role.example.arn
}
```

## Problems 

Whenever terraform destroys and reapplies our resources, the api gateway will have a new url that our index.js file will need to reach out. This means our index.js file needs to be automatically created with the content of this new url.

Terraform has a local file function that can be used to solve this problem. This also means our s3 bucket index.js upload will also need to depend on another terraform resource. 

I also need to change the javascript contents entirely because we are switching from using local storage to grabbing data using the API gateway and lambda functions. This part did require some learning on javascript promises. 

```js
resource "local_file" "example" {
  depends_on = [
    aws_api_gateway_resource.example,
    aws_api_gateway_stage.example
  ]
  filename = "index.js"
  content  = <<-EOF
        url = "${aws_api_gateway_stage.example.invoke_url}/${aws_api_gateway_resource.example.path_part}"

        fetch(url)
            .then((response) => visits = response.json())
            .then((data) => {
                console.log(data)
                console.log(data.Item)
                console.log(data.Item.visitors)
                visitor_display= document.getElementById("page-view")
                visitor_display.innerHTML=data.Item.visitors.toString()
            })
    EOF
}
```