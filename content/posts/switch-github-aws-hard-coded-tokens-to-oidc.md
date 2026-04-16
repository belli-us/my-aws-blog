+++
date = '2026-04-15T00:00:00-07:00'
draft = false
title = 'How to Switch GitHub-AWS Integrations from Hard-Coded Tokens to OIDC'
tags = ['github-actions', 'aws', 'oidc', 'iam', 'security', 'devops']
+++

If your GitHub Actions workflow talks to AWS using `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, you are relying on long-lived credentials stored in GitHub secrets. That works, but it creates more secret management overhead than you need. A better approach is to use **OpenID Connect (OIDC)** so GitHub can request short-lived AWS credentials at runtime.

In this guide, I'll walk through how to migrate a typical GitHub-to-AWS deployment from hard-coded tokens to OIDC. The pattern is especially useful for static site deployments, where GitHub Actions builds the site, uploads it to **Amazon S3**, and then invalidates **CloudFront**.

## Why move to OIDC?

With OIDC:

- You stop storing long-lived AWS access keys in GitHub
- AWS issues short-lived credentials only when a workflow runs
- You can restrict access to a specific repository, branch, or environment
- Rotating and cleaning up credentials becomes much simpler

In other words, OIDC reduces both risk and maintenance.

---

## What the old setup usually looks like

A secrets-based GitHub Actions step often looks like this:

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

This pattern depends on an IAM user with long-lived access keys. Those keys need to be created, stored, rotated, and eventually revoked.

With OIDC, that step changes to a role assumption flow instead.

---

## How OIDC works between GitHub and AWS

At a high level, the flow is:

1. GitHub Actions requests an OIDC token for the running workflow.
2. AWS IAM trusts GitHub's OIDC provider at `https://token.actions.githubusercontent.com`.
3. AWS checks the token's audience and subject claims.
4. If the token matches your trust policy, AWS STS returns short-lived credentials for an IAM role.
5. Your workflow uses those temporary credentials to deploy.

GitHub documents the AWS-specific OIDC setup here:

- [GitHub Docs: Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws?apiVersion=2022-11-28)
- [AWS IAM Docs: Create an OpenID Connect identity provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)

---

## Prerequisites

Before you switch, make sure you have:

- Access to the AWS account that owns your deployment resources
- Permission to create or update IAM identity providers, roles, and policies
- Permission to edit your GitHub Actions workflow
- A clear idea of which branch or environment should be allowed to deploy

For this walkthrough, I'll assume:

- Your workflow deploys from the `main` branch
- Your workflow uploads to an S3 bucket
- Your workflow optionally creates a CloudFront invalidation afterward

---

## Step 1: Add GitHub as an OIDC provider in AWS IAM

In the AWS Console:

1. Open **IAM**
2. Go to **Identity providers**
3. Choose **Add provider**
4. Select **OpenID Connect**
5. Set the provider URL to:

```text
https://token.actions.githubusercontent.com
```

6. Add this audience:

```text
sts.amazonaws.com
```

7. Save the provider

That tells AWS to trust GitHub as a token issuer for workloads that match your IAM role trust policies.

---

## Step 2: Create an IAM role that GitHub Actions can assume

Next, create a role for your deploy workflow. The most important part is the **trust policy**.

Here is a tight trust policy that only allows deployments from the `main` branch of one repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_ORG/YOUR_REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Replace:

- `123456789012` with your AWS account ID
- `YOUR_GITHUB_ORG` with your GitHub user or organization
- `YOUR_REPO` with your repository name

If you deploy through a GitHub **environment** like `production`, GitHub's docs note that the `sub` claim changes shape. In that case, use:

```text
repo:YOUR_GITHUB_ORG/YOUR_REPO:environment:production
```

That is a great option if you want environment protection rules and approvals layered on top of OIDC.

---

## Step 3: Attach a least-privilege permissions policy

The role also needs permission to do the actual deployment work.

For a Hugo static site deployment to S3 plus a CloudFront invalidation, a policy like this is a good starting point:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListAndLocateBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME"
    },
    {
      "Sid": "ManageSiteFiles",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    },
    {
      "Sid": "InvalidateCloudFront",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::123456789012:distribution/YOUR_DISTRIBUTION_ID"
    }
  ]
}
```

If your workflow only uploads to S3 and does not invalidate CloudFront, you can drop the final statement.

---

## Step 4: Update your GitHub Actions workflow

Now update the workflow itself.

There are two important changes:

1. Add the `id-token: write` permission
2. Replace static AWS secrets with `role-to-assume`

Here is a full example based on a typical Hugo deploy pipeline:

```yaml
name: Deploy Hugo Site to AWS

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build Hugo Site
        run: hugo --minify

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: us-east-1
          role-to-assume: arn:aws:iam::123456789012:role/github-hugo-deploy-role

      - name: Deploy to S3
        run: aws s3 sync public/ s3://YOUR_BUCKET_NAME --delete

      - name: Invalidate CloudFront
        run: aws cloudfront create-invalidation --distribution-id YOUR_DISTRIBUTION_ID --paths "/*"
```

The nice part is that the rest of your workflow usually stays the same. You still build the site, sync to S3, and invalidate CloudFront. The only real authentication change is replacing stored secrets with a role assumption.

> **Tip:** For production workflows, pin third-party actions to a full commit SHA when possible instead of relying only on a moving version tag.

---

## Step 5: Remove the old GitHub secrets

Once the OIDC-based workflow succeeds, clean up the old credentials:

1. Delete `AWS_ACCESS_KEY_ID` from your repository or organization secrets
2. Delete `AWS_SECRET_ACCESS_KEY` from your repository or organization secrets
3. Deactivate and remove the old IAM user access keys in AWS
4. If the IAM user existed only for GitHub Actions, consider deleting the user entirely

This is the step that actually reduces your long-lived credential footprint.

---

## Step 6: Verify the new flow

After pushing the workflow change, watch the next Actions run closely.

Things to verify:

- The workflow reaches the AWS credentials step without reading GitHub AWS secrets
- The S3 sync succeeds
- The CloudFront invalidation succeeds
- AWS CloudTrail shows the role being assumed with web identity

If you want an extra sanity check, you can temporarily add:

```yaml
- name: Verify caller identity
  run: aws sts get-caller-identity
```

That confirms the workflow is running under the expected IAM role.

---

## Common gotchas

### `Not authorized to perform sts:AssumeRoleWithWebIdentity`

This usually means one of three things:

- The `sub` claim in your trust policy does not match the repository, branch, or environment actually running the workflow
- The OIDC provider URL or audience is wrong
- The workflow is missing `id-token: write`

### Branch deployments fail after switching to environments

If you begin using GitHub environments, update the trust policy to match the environment-based `sub` format instead of the branch-based one.

### The OIDC provider already exists

That is fine. You only need one GitHub OIDC provider per AWS account. Reuse it and create additional IAM roles as needed.

### Pull requests from forks

Be careful here. Do not broaden your trust policy unless you truly want forked or preview workflows to assume AWS roles.

---

## Summary

Switching from hard-coded AWS tokens to OIDC is one of the highest-value security improvements you can make in a GitHub-to-AWS deployment pipeline.

You get:

- Short-lived credentials instead of long-lived secrets
- Tighter repository and branch scoping
- Less secret rotation overhead
- A cleaner and more auditable deployment setup

For a Hugo site deployed through GitHub Actions, the migration is usually straightforward: create the OIDC provider, create a tightly scoped IAM role, update the workflow to request an ID token, and remove the old AWS secrets once everything passes.

If you are still deploying to AWS from GitHub using hard-coded credentials, this is a very worthwhile cleanup.
