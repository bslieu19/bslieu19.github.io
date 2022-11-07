---
title: Cloud Challenge Day 3 - Javascript
date: 2022-10-26 12:00 -800
categories: [homelab]
tags: [cloud-challenge]
---

## Javascript 

The goal of javascript is to display a visitor counter on the top of the page. 

On Firefox and Google Chrome, there is local storage that can be used to increment the visitor count on the page. By going to developer tools, the visitor count can be obtained. 

![local-storage](/assets/images/2022-10-26/developer-tools.png)

```javascript
const f = document.getElementById("visitor_count")
var visitCountor = localStorage.getItem("visitors")

if (visitCount) {
    visitCount = Number(visitorCountor) + 1
    localStorage.setItem("visitors", visitorCount)
} else {
    visitCount = 1
    localStorage.setItem("visitors", visitorCount)
}
f.innerHTML = visitCountor
```

One main problem is that this storage is "local storage." When opening the webpage in incongito, the counter is reseted back to zero. This brings in databases, specifically DynamoDB. 

## Terraform 

Upload the index.js document to S3

```tf
resource "aws_s3_object" "css" {
    bucket = aws_s3_bucket.website.id
    key = "index.css"
    source = "../index.css"
    content_type = "text/css"
}
```