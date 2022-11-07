---
title: Cloud Challenge Day 5 - DynamoDB and AWS SDK For Lambda
date: 2022-11-06 12:00 -800
categories: [homelab]
tags: [cloud-challenge]
---

## DynamoDB

DynamoDB is a serverless NoSQL database offering that will be used to keep track of the website visitor count. It features a partiion key that will be used as the table's primary key and allows access to the other items in the table.

We will be storing the number of visitors into a dynamodb table.

![dynamodb-table](/assets/images/2022-11-06/DynamoDB.png)

## Lambda Function

To talk to dynamodb, the lambda function needs to communicate using the aws sdk. This will establish a connection to dynamodb. The lambda function has been modified to work with our dynamodb resource.

One important thing to note is the "Access-Control-Allow-Origin." By default, my website in S3 is assumed to not have the permissions needed to fetch data in the dynamodb table because they are in two different resources. 

```js
const AWS = require("aws-sdk");

const dynamo = new AWS.DynamoDB.DocumentClient();
let body, update_dynamo

exports.handler = async (event) => {
    // TODO implement
    let body;
    let statusCode = 200;
    const headers = {
        "Content-Type": "application/json",
        "Access-Control-Allow-Headers" : "Content-Type",
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "OPTIONS,POST,GET"
     };
    
    try {
        body = await dynamo.get({ 
            TableName: "example",
            Key: {
                website: "example"
            }
        }).promise()

        update_dynamo = await dynamo.put({
            TableName: "example",
            Item: {
              website: "example",
              visitors: body.Item.visitors + 1
            }
        }).promise()
        
    } 
    catch (err) {
        statusCode = 400;
        body = err.message;
    }
    finally {
        body = JSON.stringify(body);
    }
    
    
    return {
        statusCode,
        body,
        headers
    };
}
```

## Terraform

Using terraform, we create the dynamodb table and specify an item on the table. We also updated the zip file with our new node.js script

```tf
resource "aws_dynamodb_table" "example" {
  #billing_mode     = "PAY_PER_REQUEST"
  hash_key         = "website"
  name             = "example"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  read_capacity  = 1
  write_capacity = 1

  attribute {
    name = "website"
    type = "S"
  }

  tags = {
    Architect = "Eleanor"
    Zone      = "SW"
  }
}

resource "aws_dynamodb_table_item" "example" {
  depends_on = [
    aws_dynamodb_table.example
  ]
  table_name = aws_dynamodb_table.example.name
  hash_key   = aws_dynamodb_table.example.hash_key

  item = <<ITEM
  {
    "website": {"S": "example"},
    "visitors": {"N": "1"}
  }
  ITEM
}
```