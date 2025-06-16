# 🌐 Custom Domain Setup for 43kae.xyz

This document outlines how the custom domain **43kae.xyz** was configured and linked to an AWS S3 static site using AWS services and CLI tools.

---

## 🔧 Prerequisites

- Domain purchased via Namecheap: `43kae.xyz`
- AWS account with S3 and Route 53 permissions
- AWS CLI configured
- Portfolio project ready and synced with S3

---

## 🛠 Step 1: Create Route 53 Hosted Zone

1. Open AWS Console → Route 53 → Hosted zones
2. Click **Create hosted zone**
3. Enter domain name: `43kae.xyz`
4. Type: **Public hosted zone**
5. Copy the 4 **NS** records

---

## 🔁 Step 2: Update DNS on Namecheap

1. Go to your domain dashboard on Namecheap
2. Click **Manage** next to `43kae.xyz`
3. Under **Nameservers**, select **Custom DNS**
4. Paste the 4 NS values from AWS Route 53

⏳ Allow 5–30 mins for propagation

---

## 🪣 Step 3: Create and Configure S3 Buckets

### Main bucket: `43kae.xyz`

```bash
aws s3api create-bucket \
  --bucket 43kae.xyz \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1
```

#### 🔓 Disable Public Access Block

```bash
aws s3api put-public-access-block \
  --bucket 43kae.xyz \
  --public-access-block-configuration 'BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false'
```

Enable static hosting:

```bash
aws s3 website s3://43kae.xyz/ \
  --index-document index.html \
  --error-document index.html
```

Bucket policy (`s3-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::43kae.xyz/*"
    }
  ]
}
```

Apply policy:

```bash
aws s3api put-bucket-policy \
  --bucket 43kae.xyz \
  --policy file://s3-policy.json
```

Upload project files:

```bash
aws s3 sync ./cloud-portfolio-s3/ s3://43kae.xyz --delete
```

---

### Redirect bucket: `www.43kae.xyz`

```bash
aws s3api create-bucket \
  --bucket www.43kae.xyz \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1
```

#### 🔓 Disable Public Access Block

```bash
aws s3api put-public-access-block \
  --bucket www.43kae.xyz \
  --public-access-block-configuration 'BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false'
```

Redirect config (`www-redirect.json`):

```json
{
  "RedirectAllRequestsTo": {
    "HostName": "43kae.xyz",
    "Protocol": "http"
  }
}
```

Apply redirect:

```bash
aws s3api put-bucket-website \
  --bucket www.43kae.xyz \
  --website-configuration file://www-redirect.json
```

Apply public policy:

```bash
aws s3api put-bucket-policy \
  --bucket www.43kae.xyz \
  --policy file://s3-policy.json
```

---

## 🌍 Step 4: Add Route 53 DNS Records

### Root domain (43kae.xyz):

- **Type**: A
- **Alias**: Yes
- **Alias target**: S3 website endpoint for `43kae.xyz`

### www subdomain ([www.43kae.xyz](http://www.43kae.xyz)):

- **Type**: A
- **Alias**: Yes
- **Alias target**: S3 website endpoint for `www.43kae.xyz`

⏳ Allow propagation and test:

- [http://43kae.xyz](http://43kae.xyz)
- [http://www.43kae.xyz](http://www.43kae.xyz) → should redirect to root

---

## 🔐 Step 5: Optional — HTTPS with CloudFront + ACM

### 1. Request Certificate with ACM

```bash
aws acm request-certificate \
  --domain-name 43kae.xyz \
  --subject-alternative-names www.43kae.xyz \
  --validation-method DNS \
  --region us-east-1
```

Copy the certificate ARN and note the DNS validation records. Create these records in Route 53 to validate the domain.

### 2. Create CloudFront Distribution

Create a CloudFront distribution with:

- **Origin Domain**: `43kae.xyz.s3-website-us-east-1.amazonaws.com`
- **Origin Type**: `Custom Origin`
- **Viewer Protocol Policy**: Redirect HTTP to HTTPS
- **Aliases**: `43kae.xyz`, `www.43kae.xyz`
- **SSL Certificate**: Use the validated ACM certificate

Once deployed, update Route 53 alias records to point to the CloudFront domain name.

---

## ✅ Success!

Your static site is now fully live on a custom domain. Optionally, use HTTPS via CloudFront and ACM for security and trust.

---

## 📁 Repository Notes

- This setup is part of the `cloud-portfolio-s3` project
- All commands are tested with AWS CLI v2
- Repo includes site files, policies, and this setup guide

