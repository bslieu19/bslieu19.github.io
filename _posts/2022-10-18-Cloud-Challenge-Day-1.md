---
title: Cloud Challenge Day 1 - HTML, CSS, and AWS S3
date: 2022-10-18 12:00 -800
categories: [homelab]
tags: [cloud-challenge]
---

## Purpose of the Cloud Challenge

The cloud challenge takes inspiration from the cloud resume challenge and will closely follow the goals defined on the website. This can be found at: https://cloudresumechallenge.dev/

What I would like to accomplish from this challenge is to learn a variety of technologies, and slowly learn to recreate some components of my homelab in the cloud using AWS. 

## HTML, CSS

IN HTML, I created a template for a section in the resume on the html document. The purpose is to reuse it on other sections as much as possible.  

```html
    <block id="class-name">
        <h3>Header</h3>
        <block id="project-name">
            <h4>
                Project Name 
                <block class="date"> Date </block> 
            </h4> 
            <ul>
                <li>Some Text</li>
                <li>Some Text2</li>
            </ul>
        </block>
    </block>
```

There are multiple blocks, each with a different class or id name. This was done in order to allow more freedom when apply CSS to the HTML document. 

Moving into CSS, I made the decision to center the text and move all the dates to the right of the document. I prefer to keep it simple, and additional CSS can always be applied in the future. 

## AWS

I created an AWS account to use the static website hosting option on an S3 bucket. 

Each bucket needs to have a unique name, and by default, denies all internet requests. 

The bucket needs to have the option "block public access" turned off, and a bucket policy created to define what can be done with the S3 bucket in json. As a test, I used the below bucket policy.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::bucket_name",
                "arn:aws:s3:::bucket_name/*"
            ]
        }
    ]
}
```

The static website hosting option needs to also be enabled, and the index.html document defined; error.html is an optional field.

After performing the steps, the website is accessible. 

## Terraform 

I want to minimize the amount I am spending in AWS, and decided to use Terraform to quickly provision and destroy my resources.

Terraform is capable of quickly provisioning my AWS resources and destroy them when needed. This stops the need to delete each resource individually.

First, I created an IAM user that only uses a token to log into AWS, and provided it into the AWS provider as specified on the Hashicorp website. 

```tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "region"
  access_key = "access_key"
  secret_key = "secret_key"
}

```

Using the documentation on Hashicorp's website, I did the following:

* Create an S3 Bucket using aws_s3_bucket resource
* Upload html and css using aws_s3_object resource
* Configure static website hosting using aws_s3_bucket_website_configuration resource
* Set the bucket acl to public-read using aws_s3_bucket_acl resource
* Create ab S3 Bucket policy using aws_s3_bucket_policy resource

```tf
resource "aws_s3_bucket" "website" {
  bucket = "my-bucket"
}

resource "aws_s3_object" "html" {
    bucket = aws_s3_bucket.website.id
    key = "index.html"
    source = "../index.html"
    content_type = "text/html"
}

resource "aws_s3_object" "css" {
    bucket = aws_s3_bucket.website.id
    key = "index.css"
    source = "../index.css"
    content_type = "text/css"
}

resource "aws_s3_bucket_website_configuration" "site" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

resource "aws_s3_bucket_acl" "site" {
  bucket = aws_s3_bucket.website.id

  acl = "public-read"
}

resource "aws_s3_bucket_policy" "site" {
  bucket = aws_s3_bucket.website.id
  depends_on = [
    aws_cloudfront_distribution.s3_website_distribution
  ]

  policy = jsonencode({
        "Version": "2022-10-17",
        "Id": "PolicyForCloudFrontPrivateContent",
        "Statement": [
            {
                "Sid": "AllowCloudFrontServicePrincipal",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "${aws_s3_bucket.website.arn}/*",
            }
        ]
})
}

```

After creating these resources:

```shell
terraform init
terraform plan
terraform apply 
```

The website is accessible. 

To destroy all resources:
```shell
terraform destroy 
```

## Troubleshooting

After applying the terraform code, the website was not serving html, but rather is downloading the html document instead. To resolve this, I added content_type = "text/html" to the aws_s3_object containing the html document based on stackoverflow: https://stackoverflow.com/questions/18296875/amazon-s3-downloads-index-html-instead-of-serving

```tf
resource "aws_s3_object" "html" {
    bucket = aws_s3_bucket.site.id
    key = "index.html"
    source = "../index.html"
    content_type = "text/html"
}
```