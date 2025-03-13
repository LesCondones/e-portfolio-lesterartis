# ePortfolio Hosted on AWS S3 and CloudFront

This repository contains the files for my ePortfolio, which is hosted on **AWS S3** with **CloudFront** for secure, fast, and scalable delivery.

## Hosting Overview

The ePortfolio is hosted using **Amazon S3** with static website hosting and **CloudFront** for enhanced security and performance. This setup provides:

- **Static website hosting** on S3 for cost-effective storage
- **HTTPS support** for secure connections via CloudFront
- **Faster global content delivery** via CloudFront's CDN
- **AWS Free Tier** eligibility for budget-friendly hosting

## Prerequisites

- An AWS account
- AWS CLI installed and configured on your local machine
- Your portfolio code in this GitHub repository

## Setup Instructions

If you'd like to replicate this setup or host your own static website using AWS, follow these steps:

1. **Create an S3 Bucket:**
   ```bash
   aws s3api create-bucket --bucket your-portfolio-name --region us-east-1 --create-bucket-configuration LocationConstraint=us-east-1
   ```

2. **Enable Public Access:**
   ```bash
   aws s3api put-public-access-block --bucket your-portfolio-name --public-access-block-configuration 'BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false'
   ```

3. **Upload Files:**
   ```bash
   aws s3 sync /path/to/your/project s3://your-portfolio-name --acl public-read
   ```

4. **Enable Static Website Hosting:**
   ```bash
   aws s3 website s3://your-portfolio-name/ --index-document index.html --error-document error.html
   ```

5. **Set Bucket Policy for Public Read Access:**
   ```bash
   aws s3api put-bucket-policy --bucket your-portfolio-name --policy '{
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "PublicReadGetObject",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::your-portfolio-name/*"
       }
     ]
   }'
   ```

## Detailed Setup Instructions

### Option 1: S3 Static Website Hosting (Simplest)

#### 1. Create an S3 Bucket

```bash
# Create the bucket (replace 'your-portfolio-name' with a unique bucket name)
aws s3api create-bucket --bucket your-portfolio-name --region us-east-1
```

For regions other than us-east-1:
```bash
aws s3api create-bucket --bucket your-portfolio-name --region your-region --create-bucket-configuration LocationConstraint=your-region
```

#### 2. Enable Static Website Hosting

```bash
# Create website.json configuration file
cat > website.json << 'EOF'
{
    "IndexDocument": {
        "Suffix": "index.html"
    },
    "ErrorDocument": {
        "Key": "error.html"
    }
}
EOF

# Apply configuration
aws s3api put-bucket-website --bucket your-portfolio-name --website-configuration file://website.json
```

#### 3. Set Public Access

```bash
# Create bucket-policy.json
cat > bucket-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-portfolio-name/*"
        }
    ]
}
EOF

# Apply policy (replace 'your-portfolio-name' with your bucket name)
aws s3api put-bucket-policy --bucket your-portfolio-name --policy file://bucket-policy.json

# Disable block public access settings
aws s3api put-public-access-block --bucket your-portfolio-name --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

#### 4. Deploy the Portfolio

```bash
# From your local repository directory
aws s3 sync . s3://your-portfolio-name --exclude ".git/*" --exclude ".github/*" --exclude "README.md"
```

#### 5. Access Your Website

Your website will be available at:
```
http://your-portfolio-name.s3-website-your-region.amazonaws.com
```

### Option 2: CloudFront with S3 (Enhanced Performance & Security)

CloudFront provides global content delivery, HTTPS, and better security. It's also available in the AWS Free Tier.

#### 1. Create S3 Bucket (Private)

```bash
# Create the bucket
aws s3api create-bucket --bucket your-portfolio-name --region us-east-1
```

#### 2. Upload Content

```bash
# From your local repository
aws s3 sync . s3://your-portfolio-name --exclude ".git/*" --exclude ".github/*" --exclude "README.md"
```

#### 3. Create CloudFront Origin Access Identity (OAI)

```bash
# Create OAI
aws cloudfront create-cloud-front-origin-access-identity --cloud-front-origin-access-identity-config CallerReference=portfolio-oai,Comment=PortfolioOAI
```
Note the ID from the response.

#### 4. Update Bucket Policy for CloudFront

```bash
# Create cloudfront-policy.json (replace values)
cat > cloudfront-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CloudFrontGetObject",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity YOUR_OAI_ID"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-portfolio-name/*"
        }
    ]
}
EOF

# Apply policy
aws s3api put-bucket-policy --bucket your-portfolio-name --policy file://cloudfront-policy.json
```

#### 5. Create CloudFront Distribution

```bash
# Create distribution-config.json
cat > distribution-config.json << 'EOF'
{
    "CallerReference": "portfolio-distribution",
    "DefaultRootObject": "index.html",
    "Origins": {
        "Quantity": 1,
        "Items": [
            {
                "Id": "S3-your-portfolio-name",
                "DomainName": "your-portfolio-name.s3.amazonaws.com",
                "S3OriginConfig": {
                    "OriginAccessIdentity": "origin-access-identity/cloudfront/YOUR_OAI_ID"
                }
            }
        ]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "S3-your-portfolio-name",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {
            "Quantity": 2,
            "Items": ["GET", "HEAD"],
            "CachedMethods": {
                "Quantity": 2,
                "Items": ["GET", "HEAD"]
            }
        },
        "ForwardedValues": {
            "QueryString": false,
            "Cookies": {
                "Forward": "none"
            }
        },
        "MinTTL": 0,
        "DefaultTTL": 86400,
        "MaxTTL": 31536000
    },
    "Enabled": true,
    "Comment": "Portfolio Website Distribution"
}
EOF

# Create distribution
aws cloudfront create-distribution --distribution-config file://distribution-config.json
```

#### 6. Access Your Website

Once the distribution is deployed (which can take 15-30 minutes), your website will be available at:
```
https://[distribution-id].cloudfront.net
```

## Benefits of CloudFront Setup

For enhanced performance and security, the S3 bucket configured with **Amazon CloudFront** provides:

- **HTTPS support** for secure connections
- **Faster global content delivery** via CloudFront's CDN
- **Privacy**: Only CloudFront has access to the S3 bucket, making the bucket itself private
- **Improved security posture** with private S3 assets

## Updating Your Portfolio

When you make changes to your repository, update your AWS-hosted version:

```bash
# Pull latest changes
git pull origin main

# Sync to S3
aws s3 sync . s3://your-portfolio-name --exclude ".git/*" --exclude ".github/*" --exclude "README.md"

# If using CloudFront, create an invalidation to clear the cache
aws cloudfront create-invalidation --distribution-id YOUR_DISTRIBUTION_ID --paths "/*"
```

## AWS Free Tier Limits

- **S3**: 5GB storage, 20,000 GET requests, 2,000 PUT requests
- **CloudFront**: 1TB data transfer, 10 million requests per month (first 12 months)

For a typical portfolio site, these limits are usually sufficient.

## Security Note

When using S3 website hosting directly (Option 1), your bucket must allow public read access to function as a website. This means anyone can view the files you upload.

CloudFront with OAI (Option 2) provides better security as your S3 bucket remains private while still serving your website.
