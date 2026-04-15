+++
date = '2026-04-15T00:00:00-07:00'
draft = false
title = 'How to Deploy a Hugo Static Site with S3, CloudFront, and Route 53'
tags = ['hugo', 'aws', 's3', 'cloudfront', 'route53', 'devops']
+++

Deploying a Hugo static site on AWS gives you a fast, scalable, and cost-effective setup. In this guide, we'll walk through hosting your site on **Amazon S3**, distributing it globally via **CloudFront**, and pointing a custom domain using **Route 53**.

## Prerequisites

Before starting, make sure you have:

- A built Hugo site (`hugo` command generates the `public/` folder)
- An AWS account with appropriate IAM permissions
- A domain name registered in Route 53 (or transferred there)
- The [AWS CLI](https://aws.amazon.com/cli/) installed and configured

---

## Step 1: Build Your Hugo Site

Run the Hugo build command to generate your static files:

```bash
hugo --minify
```

This outputs your site into the `public/` directory.

---

## Step 2: Create and Configure an S3 Bucket

### Create the bucket

```bash
aws s3 mb s3://your-domain.com --region us-east-1
```

> **Note:** For CloudFront to serve your site correctly, the bucket does **not** need to be configured for static website hosting. We'll use CloudFront as the entry point and keep the bucket private.

### Block all public access

Keep the bucket private — CloudFront will access it via an Origin Access Control (OAC):

```bash
aws s3api put-public-access-block \
  --bucket your-domain.com \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### Upload your site

```bash
aws s3 sync public/ s3://your-domain.com --delete
```

The `--delete` flag removes files from S3 that no longer exist locally, keeping your bucket in sync.

---

## Step 3: Request an SSL Certificate via ACM

CloudFront requires a certificate in the **us-east-1** region.

```bash
aws acm request-certificate \
  --domain-name your-domain.com \
  --subject-alternative-names "*.your-domain.com" \
  --validation-method DNS \
  --region us-east-1
```

After requesting, go to the **AWS Certificate Manager** console, find your certificate, and add the DNS validation CNAME records to Route 53. ACM can do this automatically if your domain is in Route 53 — click **"Create records in Route 53"** in the console.

Wait a few minutes for the certificate status to show **Issued**.

---

## Step 4: Create a CloudFront Distribution

### Create an Origin Access Control (OAC)

OAC is the modern, recommended way for CloudFront to securely access a private S3 bucket.

In the **AWS Console → CloudFront → Origin access → Create control setting**:

- **Name:** `your-domain-oac`
- **Origin type:** S3
- **Signing behavior:** Sign requests

Or via CLI:

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "your-domain-oac",
    "Description": "OAC for your-domain.com",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

Note the returned `Id` — you'll need it when creating the distribution.

### Create the distribution

In the **CloudFront Console**, create a new distribution with these key settings:

| Setting | Value |
|---|---|
| **Origin domain** | Your S3 bucket's regional domain (e.g., `your-domain.com.s3.us-east-1.amazonaws.com`) |
| **Origin access** | Origin access control settings (OAC) — select the OAC you created |
| **Viewer protocol policy** | Redirect HTTP to HTTPS |
| **Allowed HTTP methods** | GET, HEAD |
| **Alternate domain names (CNAMEs)** | `your-domain.com`, `www.your-domain.com` |
| **Custom SSL certificate** | Select the ACM certificate you created |
| **Default root object** | `index.html` |

### Update the S3 bucket policy

After creating the distribution, CloudFront will prompt you to copy a bucket policy. It looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-domain.com/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

Apply it:

```bash
aws s3api put-bucket-policy \
  --bucket your-domain.com \
  --policy file://bucket-policy.json
```

---

## Step 5: Configure Route 53

### Create an A record (Alias) for your domain

1. Open **Route 53 → Hosted Zones** and select your domain.
2. Click **Create record**.
3. Set the following:

| Field | Value |
|---|---|
| **Record name** | *(leave blank for root domain)* |
| **Record type** | A |
| **Alias** | Yes |
| **Route traffic to** | Alias to CloudFront distribution |
| **Distribution** | Select your CloudFront distribution |

4. Repeat for `www` by setting **Record name** to `www`.

It typically takes a few minutes for DNS to propagate.

---

## Step 6: Handle Hugo's Clean URLs (Optional but Recommended)

Hugo generates pages like `/about/index.html`. By default, CloudFront won't serve these if you navigate to `/about/`. You can fix this with a **CloudFront Function**.

In the **CloudFront Console → Functions → Create function**:

```javascript
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // If URI ends with '/', append 'index.html'
  if (uri.endsWith('/')) {
    request.uri += 'index.html';
  }
  // If URI has no file extension, append '/index.html'
  else if (!uri.includes('.')) {
    request.uri += '/index.html';
  }

  return request;
}
```

Associate this function with your CloudFront distribution under **Viewer request** for the default cache behavior.

---

## Step 7: Automate Deployments (Optional)

Manually syncing files works, but automating with a simple script or CI/CD pipeline is better. Here's a minimal deploy script:

```bash
#!/bin/bash
set -e

BUCKET="your-domain.com"
DISTRIBUTION_ID="YOUR_CLOUDFRONT_DISTRIBUTION_ID"

echo "Building Hugo site..."
hugo --minify

echo "Syncing to S3..."
aws s3 sync public/ s3://$BUCKET --delete

echo "Invalidating CloudFront cache..."
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/*"

echo "Deployment complete!"
```

The CloudFront invalidation ensures visitors always get the latest version of your content.

---

## Summary

Here's a quick recap of what we set up:

- **S3** stores your static Hugo files, kept private
- **CloudFront** serves your content globally over HTTPS using an OAC to securely access S3
- **ACM** provides the free SSL/TLS certificate for your custom domain
- **Route 53** routes your custom domain to the CloudFront distribution via Alias records

This architecture is highly reliable, globally fast, and extremely cost-effective for static sites — you only pay for storage and bandwidth, with no servers to manage.

---

*A note on how this post came together: this guide was written with the help of [Claude AI](https://claude.ai). Throughout the process of setting this up myself, I leaned heavily on AI tools to troubleshoot each stage, research AWS services, and make sense of the documentation. If you're working through a similar setup, I'd highly recommend doing the same — it makes navigating the AWS ecosystem a lot less painful.*
