---
title: Cloud Challenge Day 2 - Using AWS CloudFront
date: 2022-10-21 12:00 -800
categories: [homelab]
tags: [cloud-challenge]
---

## The Technology
AWS Cloudfront allows caching at edge locations, which provides a users who are accessing our S3 bucket website a smoothly experience. They do not need to travel across the globe to grab the data, and instead reach out to an edge location to grab the data.

If an edge location doesn't have the data cached, it'll use Amazon's network to fetch the data, and cache it.

## AWS CloudFront

The goal is to ensure our S3 bucket website can be accessed through cloudfront and respond to HTTPS. 

On cloudfront, we first create a distribution and select our S3 bucket as the origin domain. The origin access control settings is the option to use to allow our S3 website access only through cloudfront. This will create

* An origin access control 
* A bucket policy (If an existing bucket policy exists, it needs to be updated with the created policy)

Under Viewer, there is a viewer protocol policy that can be selected to redirect HTTP to HTTPS. This is the option I will use to force HTTPS connections to my S3 bucket website endpoint.

Lastly, under settings, the default root object (an option field) is specified to be my html document in my S3 bucket. This ensures if a user goes to cloudfront to access the website, index.html will render to the page.

The remaining settings were left at defaults. 

## Terraform

To automate cloudfront settings, I looked at the cloudfront distribution resource. 

I set viewer_protocol_policy to redirect-to-https to get the https behavior I am looking for on my website. I alsoi set the default_root_object to index.html to ensure that the html document gets served to anyone accessing my website through cloudfront.


```tf
resource "aws_cloudfront_origin_access_control" "cloudfront-s3" {
  name                              = "cloudfront-s3-hosting-access-control"
  description                       = "Default Policy"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}


resource "aws_cloudfront_distribution" "s3_website_distribution" {
  depends_on = [aws_cloudfront_origin_access_control.cloudfront-s3, aws_s3_object.index]
  origin {
    domain_name              = aws_s3_bucket.website.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.cloudfront-s3.id
    origin_id                = "myS3Origin"

  }

  enabled = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "myS3Origin"

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
  }

  restrictions {
    geo_restriction {
      restriction_type = "whitelist" #none if no restriction
      locations        = ["US"]
    }
  }
  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

Although this did setup my cloudfront distribution, I was still unable to access my website through the cloudfront domain url. 

The last step to make it work involved setting the bucket permissions to only serve the bucket contents on the cloudfront url. I added a 'depends_on' condition so this resource only gets applied after the aws_cloudfront_distribution resource is finished. 

```tf
resource "aws_s3_bucket_policy" "site" {
  bucket = aws_s3_bucket.website.id
  depends_on = [
    aws_cloudfront_distribution.s3_website_distribution
  ]

  policy = jsonencode({
        "Version": "2008-10-17",
        "Id": "PolicyForCloudFrontPrivateContent",
        "Statement": [
            {
                "Sid": "AllowCloudFrontServicePrincipal",
                "Effect": "Allow",
                "Principal": {
                    "Service": "cloudfront.amazonaws.com"
                },
                "Action": "s3:GetObject",
                "Resource": "${aws_s3_bucket.website.arn}/*",
                "Condition": {
                    "StringEquals": {
                      "AWS:SourceArn": "${aws_cloudfront_distribution.s3_website_distribution.arn}"
                    }
                }
            }
        ]
  })
}
```

The website is accessible through HTTPS afterwards