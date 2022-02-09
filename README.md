# AWS S3 CloudFront CI/CD GitHub Actions

## Create GitHub Identity Role (Trust Relationships / Entities)

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::[your-aws-account-number]:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:[your-github-account-name]/[your-s3-bucketname]:*"
                }
            }
        }
    ]
}
```

## AWS Role Permissions Policy

#### Attach To GitHub Role Created Above

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ReplicateObject",
                "s3:PutObject",
                "s3:PutBucketWebsite",
                "s3:DeleteObjectVersion",
                "s3:RestoreObject",
                "s3:DeleteObject",
                "cloudfront:CreateInvalidation",
                "s3:PutBucketVersioning",
                "s3:ReplicateDelete"
            ],
            "Resource": [
                "arn:aws:s3:::[your-s3-bucket-name]/*",
                "arn:aws:s3:::[your-s3-bucket-name]",
                "arn:aws:cloudfront::[your-user-account-number]:distribution/[your-cloudfront-distribution-id]"
            ]
        }
    ]
}
```

## S3 Bucket Policy

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::[your-s3-bucketname]/*"
        },
        {
            "Sid": "GitHubRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "[your-arn-from-the-role-you-created-for-github-actions]"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::[your-s3-bucketname]"
        }
    ]
}
```

## GitHub Actions YAML

```YAML
name: s3 cloudfront ci/cd deployment

on:
  workflow_dispatch:
  push:
    branches:
      - 'whatever-branches-you-want' # the branch we want this to run on
      - 'another-branch-you-want'

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # required to use OIDC authentication
      contents: read # required to checkout the code from the repo

    steps:
      - name: Checkout repo
        uses: actions/checkout@v2

      - name: Node setup
        uses: actions/setup-node@v1
        with:
          node-version: 12

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{secrets.AWS_ROLE_TO_ASSUME}}
          role-duration-seconds: 900 # the ttl of the session, in seconds.
          aws-region: ap-southeast-2 # use your region here.

      - name: npm install
        run: npm i

      - name: copy files to s3
        run: aws s3 sync ./client s3://${{secrets.AWS_S3_BUCKET}} # .client is the client folder directory

      - name: Invalidate cache
        run: |
          aws cloudfront  create-invalidation --distribution-id ${{secrets.AWS_CLOUDFRONT_DIST}} --paths "/*"
```

## Helpful Resources:

- https://benoitboure.com/securely-access-your-aws-resources-from-github-actions
- https://github.com/aws-actions/configure-aws-credentials#assuming-a-role
- https://github.com/brodeynewman/simple-react-to-s3/blob/develop/.github/workflows/dev.yml
- https://gist.github.com/stevekinney/6ab02582829f039b6a14c973923909f8
