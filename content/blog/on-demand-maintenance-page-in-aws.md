---
title: "On Demand Maintenance Page in Aws using CloudFront and an ALB"
date: 2019-11-09T19:07:31Z
tags: [aws, cloudfront, terraform]
---

Sometimes your site needs a maintenance page to be enabled when you do work
behind the scenes that may require downtime.

Recently we had to find a simple way to turn this page on and off entirely
within the constraints of the products offered by AWS.

We found an effective solution using
[CloudFront](https://aws.amazon.com/cloudfront/), an [Application Load
Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
and an [S3 bucket](https://aws.amazon.com/s3/).

Code examples will be in [Terraform](https://www.terraform.io/).

## Application Load Balancer

Our setup requires an application load balancer. We can set it up using [Terraform](https://www.terraform.io/docs/providers/aws/r/lb.html):

```
resource "aws_alb" "example" {
  name            = "example"
  subnets         = [subnet-a, subnet-b, subnet-c]
  security_groups = [example-security-group]
}
```

Set up your target group that forwards traffic to your instances:

```
resource "aws_alb_target_group" "example" {
  name                 = "example"
  port                 = 80
  protocol             = "HTTP"
  vpc_id               = some_vpc_id

  health_check {
    path    = "/status"
    matcher = "200"
  }
}
```

Add a listener that will receive the traffic on your specified port:

```
resource "aws_alb_listener" "example" {
  load_balancer_arn = aws_alb.example.id
  port              = "443"
  protocol          = "HTTPS"
  certificate_arn   = data.aws_acm_certificate.example.arn

  default_action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/html"
      status_code  = "503"
    }
  }
}
```

The thing to note here is the default action of `fixed-response`. The `503` HTTP
status is the code for [scheduled maintenance](https://httpstatuses.com/503).

If we deployed this now, our target would never receive any traffic. Let's
create an additional listener rule to allow traffic through:

```
resource "aws_lb_listener_rule" "default_forward" {
  listener_arn = aws_alb_listener.example.arn

  action {
    type             = "forward"
    target_group_arn = aws_alb_target_group.example.id
  }

  condition {
    field  = "path-pattern"
    values = ["*"]
  }

  lifecycle {
    ignore_changes = [condition]
  }
}
```

The rule uses the `path-pattern` condition with a wildcard which ensures all
traffic will be forwarded.

You can now browse to the DNS address of the load balancer and get to your
application.

## CloudFront and error pages

So how do we serve a maintenance page? This is where CloudFront comes in.

The Terraform resources can be quite lengthy, so I will only provide snippets.

### Default CloudFront origin

To begin, let's set up a CloudFront distribution to forward traffic to our load
balancer by default. This requires the `origin` and `default_cache_behaviour` arguments.

Create an origin to point to your load balancer:

```
resource "aws_cloudfront_distribution" "example" {

  origin {
    domain_name = aws_alb.example.dns_name
    origin_id   = "web-frontend"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

...
}
```

Create a default cache behaviour to serve this (note the `origin_id` being used):

```
resource "aws_cloudfront_distribution" "example" {
...

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "web-frontend"
    forwarded_values {
      cookies {
        forward = "all"
      }
      headers                 = ["*"]
      query_string            = true
      query_string_cache_keys = []
    }
    viewer_protocol_policy = "allow-all"
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 0
  }

...
}

```

This example bypasses all caching, but you should set your TTL to whatever caching you require

### Setting up custom error pages using S3

We need to set up custom error pages in CloudFront. This means that CloudFront
will serve the provided error page rather than the default CloudFront error
page.

Fortunately CloudFront is able to serve content using S3 as an origin, which
means we need an S3 bucket with an HTML file to point it at.

```
resource "aws_s3_bucket" "example" {
  bucket = "my_error_pages"
}
```

Add the permissions required by CloudFront to access the bucket:

```
data "aws_iam_policy_document" "my_error_pages" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.example.arn}/*"]

    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.my_error_pages.iam_arn]
    }
  }

  statement {
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.example.arn]

    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.my_error_pages.iam_arn]
    }
  }
}

resource "aws_s3_bucket_policy" "my_error_pages" {
  bucket = aws_s3_bucket.example.id
  policy = data.aws_iam_policy_document.my_error_pages.json
}

resource "aws_cloudfront_origin_access_identity" "my_error_pages" {}
```

When the bucket is created, upload your maintenance page. Let's call it `503.html`.

```
aws s3 cp 503.html s3://my_error_pages/errors/503.html
```

The error page should be prefixed with a path like `errors/`. This is required
to allow us to specify a specific path to serve an error from.

### Configuring CloudFront to serve the error page in S3

Add a new `origin to the CloudFront resource:

```
resource "aws_cloudfront_distribution" "example" {
...

  origin {
    domain_name = aws_s3_bucket.example.bucket_regional_domain_name
    origin_id   = "error-pages"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.my_error_pages.cloudfront_access_identity_path
    }
  }

...
}
```

Below our `default_cache_behavior`, add an `ordered_cache_behaviour`:

```
resource "aws_cloudfront_distribution" "example" {
...

  ordered_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "error-pages"
    forwarded_values {
      cookies {
        forward = "none"
      }
      query_string = false
    }
    viewer_protocol_policy = "allow-all"
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 0
    path_pattern           = "/errors/*.html"
  }

...
}
```

Finally, configure your custom error response:

```
resource "aws_cloudfront_distribution" "example" {
...

  custom_error_response {
    error_code         = 503
    response_code      = 503
    response_page_path = "/errors/503.html"
  }

...
}
```

If CloudFront receives a 503 status from your load balancer, it should now serve
the 503.html page we have stored in our S3 bucket.

## Switching the maintenance page on and off

Everything is now setup so we can enable and disable the maintenance page using
an API call to AWS.

To do this we're going to amend the listener rule we set up on the ALB to only
forward traffic matching a specific condition, rather than forward all traffic
(as specified by our wildcard `path-pattern`).

I would recommend changing the condition to match a specific header. That way
we can disable the world from seeing inside our environment, but given we set a
header to a specific value, we can still see how things look, which is
extremely useful.

I am going to use the AWS Ruby SDK for this example.

It requires the rule arn for the wildcard rule we set up.

```
client = Aws::ElasticLoadBalancingV2::Client.new(region: 'eu-west-2')
client.modify_rule(
  conditions: [
    {
      field: 'http-header',
      http_header_config: {
        http_header_name: "WhatsThePassword",
        values: ["OpenSesame"]
      }
    }
  ],
  rule_arn: rule_arn
)
```

There is a short delay between the rule coming into effect - about 10 seconds -
so be aware of that when enabling the rule.

This will mean that unless you have a header that is `WhatsThePassword:
OpenSesame`, you will see your custom 503 response.

To revert that change:

```
client.modify_rule(
  conditions: [
    {
      field: 'path-pattern',
      values: ["*"]
    }
  ],
  rule_arn: rule_arn
)
```

Again, there is a short delay, but you should now be able to browse your
application again.

## Summary

With the flexibility of Application Load Balancer listener rules and features
of CloudFront, we now have a very easy way to set a maintenance page.

You can also set up all your other error pages in exactly the same way, and
also enable some caching to ensure availability in a disaster.

If you have any questions about this post, please feel free to [ask me on
Twitter](https://twitter.com/surminus).
