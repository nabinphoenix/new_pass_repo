# AWS Complete Setup Guide
### AWS CLI + AWS Toolkit + AWS SDK (Boto3) + CI/CD Pipeline with Elastic Beanstalk

---

## Table of Contents
1. [Install AWS CLI](#1-install-aws-cli)
2. [Install AWS Toolkit in VS Code](#2-install-aws-toolkit-in-vs-code)
3. [Configure AWS Credentials](#3-configure-aws-credentials)
4. [Verify Connection](#4-verify-connection)
5. [Install AWS SDK - Boto3 (Python)](#5-install-aws-sdk---boto3-python)
6. [Using Boto3 - Basic Examples](#6-using-boto3---basic-examples)
7. [Setup Elastic Beanstalk Environment](#7-setup-elastic-beanstalk-environment)
8. [Setup CI/CD Pipeline with GitHub Actions](#8-setup-cicd-pipeline-with-github-actions)
9. [Project Structure](#9-project-structure)
10. [Every New Lab Session Checklist](#10-every-new-lab-session-checklist)
11. [Common Errors and Fixes](#11-common-errors-and-fixes)

---

## 1. Install AWS CLI

### Step 1: Check if AWS CLI is already installed

Open **VS Code** → open **Terminal** (`Ctrl + `` ` ``) and run:

```cmd
aws --version
```

**If you see this → AWS CLI is already installed ✅:**
```
aws-cli/2.x.x Python/3.x.x Windows/11 exe/AMD64
```

**If you get an error → Install AWS CLI:**

Download the official AWS CLI v2 installer for Windows:

👉 **[Download AWS CLI v2](https://awscli.amazonaws.com/AWSCLIV2.msi)**

After downloading:
1. Double-click the `.msi` file
2. Click **Next → Next → Install**
3. Click **Finish**
4. **Restart VS Code**
5. Run `aws --version` again to verify

---

## 2. Install AWS Toolkit in VS Code

The AWS Toolkit extension gives you a graphical view of your AWS resources inside VS Code.

### Steps:
1. Open **VS Code**
2. Click the **Extensions icon** in the left sidebar (or press `Ctrl + Shift + X`)
3. Search for **"AWS Toolkit"**
4. Click **Install**
5. Reload VS Code when prompted

> 💡 **Note:** The AWS Toolkit also includes AWS CLI v1 automatically. However, it is recommended to use the **official AWS CLI v2** installed in Step 1.

---

## 3. Configure AWS Credentials

You need AWS credentials to connect your terminal to your AWS account. These come from your **AWS Learner Lab** or **AWS Account**.

### Step 1: Get your credentials

**From AWS Learner Lab:**
1. Go to your Cloud Labs portal
2. Click **"Start Lab"**
3. Click **"AWS Details"**
4. Copy the 3 credential values:

```
aws_access_key_id     = ASIAXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
aws_session_token     = IQoJb3Jp.... (very long token)
```

> ⚠️ **Important:** Keys starting with `ASIA` are **temporary** and expire after ~4 hours or when you end the lab session!

### Step 2: Configure AWS CLI

Run in terminal:

```cmd
aws configure
```

Enter your values:
```
AWS Access Key ID     : ASIAXXXXXXXXXXXXXXXXXXX
AWS Secret Access Key : XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Default region name  : us-east-1
Default output format: json
```

### Step 3: Set Session Token (Required for Learner Labs)

```cmd
aws configure set aws_session_token YOUR_SESSION_TOKEN_HERE
```

### Step 4: Verify credentials file

Your credentials are stored at:
```
C:\Users\YOUR_USERNAME\.aws\credentials
```

Check the file:
```cmd
type "C:\Users\YOUR_USERNAME\.aws\credentials"
```

It should show:
```ini
[default]
aws_access_key_id = ASIAXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
aws_session_token = IQoJb3Jp....
```

---

## 4. Verify Connection

Run this command to verify you are connected to AWS:

```cmd
aws sts get-caller-identity
```

**Successful response looks like:**
```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:user123=Your_Name",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/voclabs/user123=Your_Name"
}
```

✅ If you see your Account ID and Arn → **You are connected!**

❌ If you see `ExpiredTokenException` → Your session expired, get new credentials and repeat Step 3.

---

## 5. Install AWS SDK - Boto3 (Python)

Boto3 is the **AWS SDK for Python** — it lets you interact with AWS services using Python code instead of CLI commands.

### Step 1: Check Python is installed

```cmd
python --version
```

Should show: `Python 3.x.x`

### Step 2: Install Boto3

```cmd
pip install boto3
```

### Step 3: Verify installation

```cmd
pip show boto3
```

Should show version and location details.

### Step 4: Test import

```python
import boto3
```

> 💡 **Other AWS SDKs available:**
> - Java → [AWS SDK for Java](https://aws.amazon.com/sdk-for-java/)
> - JavaScript/Node.js → [AWS SDK for JavaScript](https://aws.amazon.com/sdk-for-javascript/)
> - .NET → [AWS SDK for .NET](https://aws.amazon.com/sdk-for-net/)
> - Go → [AWS SDK for Go](https://aws.amazon.com/sdk-for-go/)
>
> See the full list at: **[https://aws.amazon.com/tools/](https://aws.amazon.com/tools/)**

---

## 6. Using Boto3 - Basic Examples

Create a file called `boto3_test.py`:

### List S3 bucket contents:
```python
import boto3

s3 = boto3.client('s3', region_name='us-east-1')

# List files in a specific bucket
response = s3.list_objects_v2(Bucket='your-bucket-name')

print(f"Total files: {response['KeyCount']}")
for obj in response['Contents']:
    print(f"  - {obj['Key']}  ({obj['Size']} bytes)")
```

### Upload a file to S3:
```python
import boto3

s3 = boto3.client('s3', region_name='us-east-1')

# Upload a file
s3.upload_file('hello.txt', 'your-bucket-name', 'hello.txt')
print("File uploaded successfully!")
```

### List EC2 instances:
```python
import boto3

ec2 = boto3.client('ec2', region_name='us-east-1')

response = ec2.describe_instances()
instances = response['Reservations']

if len(instances) == 0:
    print("No EC2 instances found!")
else:
    for reservation in instances:
        for instance in reservation['Instances']:
            print(f"Instance ID : {instance['InstanceId']}")
            print(f"State       : {instance['State']['Name']}")
            print(f"Type        : {instance['InstanceType']}")
```

### Create EC2 instance:
```python
import boto3

ec2 = boto3.client('ec2', region_name='us-east-1')

response = ec2.run_instances(
    ImageId='ami-0c02fb55956c7d316',
    InstanceType='t2.micro',
    MinCount=1,
    MaxCount=1,
    TagSpecifications=[{
        'ResourceType': 'instance',
        'Tags': [{'Key': 'Name', 'Value': 'My-Boto3-Instance'}]
    }]
)

instance = response['Instances'][0]
print(f"✅ Instance Created: {instance['InstanceId']}")
```

### Run boto3 script:
```cmd
python boto3_test.py
```

---

## 7. Setup Elastic Beanstalk Environment

Elastic Beanstalk is AWS's **Platform as a Service (PaaS)** — it manages servers, load balancing, and scaling for you.

### Step 1: Go to AWS Console

1. Go to your **Cloud Labs portal** → Click **"AWS"** to open console
2. Search for **"Elastic Beanstalk"** in the search bar
3. Click **"Create application"**

### Step 2: Configure the application

Fill in these settings:

```
Application name  : your-app-name
Environment name  : your-app-env
Platform          : PHP 8.5 running on 64bit Amazon Linux 2023
Application code  : Sample application
```

### Step 3: Configure service access (Important!)

Click **"Configure more options"** → scroll to **Security** → click **Edit**:

```
Service role        : LabRole
EC2 key pair        : vockey
IAM instance profile: LabInstanceProfile
```

Click **Save** → Click **Create application**

> ⏳ Wait 3-5 minutes for environment to launch.

### Step 4: Verify environment is running

You should see:
```
Health    : ✅ Ok
Domain    : your-app-env.eba-xxxxxxxx.us-east-1.elasticbeanstalk.com
Platform  : PHP 8.5 running on 64bit Amazon Linux 2023
```

---

## 8. Setup CI/CD Pipeline with GitHub Actions

This automatically deploys your code to Elastic Beanstalk every time you push to GitHub!

```
You push code → GitHub Actions triggers → Deploys to Elastic Beanstalk → Site is live!
```

### Step 1: Project structure

Your project should look like this:

```
your-project/
├── .ebextensions/
│   └── nginx.config        ← tells server where your files are
├── .github/
│   └── workflows/
│       └── deploy.yml      ← CI/CD pipeline config
├── images/                 ← your images
├── index.html              ← main page
├── script.js               ← javascript
├── style.css               ← styles
└── README.md
```

### Step 2: Create `.ebextensions/nginx.config`

```cmd
mkdir .ebextensions
notepad .ebextensions\nginx.config
```

Paste exactly:
```yaml
option_settings:
  "aws:elasticbeanstalk:container:php:phpini":
    document_root: "/"
```

> ⚠️ Use **2 spaces** for indentation — no tabs!

### Step 3: Create `.github/workflows/deploy.yml`

```cmd
mkdir .github
mkdir .github\workflows
notepad .github\workflows\deploy.yml
```

Paste exactly (replace `YOUR_APP_NAME` and `YOUR_ENV_NAME`):

```yaml
name: Deploy to AWS Elastic Beanstalk

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Zip application
        run: |
          zip -r deploy.zip . \
            -x "*.git*" \
            -x ".github/*"

      - name: Deploy to Elastic Beanstalk
        uses: einaregilsson/beanstalk-deploy@v22
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws_session_token: ${{ secrets.AWS_SESSION_TOKEN }}
          region: us-east-1
          application_name: YOUR_APP_NAME      # ← change this!
          environment_name: YOUR_ENV_NAME      # ← change this!
          version_label: v-${{ github.run_number }}
          deployment_package: deploy.zip
          wait_for_deployment: false
```

### Step 4: Add GitHub Secrets

1. Go to your GitHub repo
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **"New repository secret"**
4. Add these 3 secrets:

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your access key from AWS Details |
| `AWS_SECRET_ACCESS_KEY` | Your secret key from AWS Details |
| `AWS_SESSION_TOKEN` | Your session token from AWS Details |

### Step 5: Push to GitHub

```cmd
git add .
git commit -m "Add website files and CI/CD pipeline"
git push origin main
```

### Step 6: Watch the pipeline run

Go to your GitHub repo → click **"Actions"** tab

You will see:
```
✅ Checkout code       (~1 second)
✅ Zip application     (~5 seconds)
✅ Deploy to EB        (~60 seconds)
```

### Step 7: Open your live website

```
http://YOUR_ENV_NAME.eba-xxxxxxxx.us-east-1.elasticbeanstalk.com
```

---

## 9. Project Structure

```
your-project/
│
├── .ebextensions/              # Elastic Beanstalk config
│   └── nginx.config            # Document root setting
│
├── .github/                    # GitHub config
│   └── workflows/
│       └── deploy.yml          # CI/CD pipeline
│
├── images/                     # Images folder
│
├── index.html                  # Main HTML file (MUST be in root!)
├── script.js                   # JavaScript file
├── style.css                   # CSS file
└── README.md                   # This file
```

> ⚠️ **Critical:** Files must be in the **ROOT** of the zip, not inside a subfolder!

---

## 10. Every New Lab Session Checklist

Since AWS Learner Lab credentials expire every ~4 hours, do this every session:

```
□ Step 1: Start Lab → click "AWS Details" → copy credentials
□ Step 2: Run aws configure → enter new Access Key + Secret Key
□ Step 3: Run aws configure set aws_session_token NEW_TOKEN
□ Step 4: Run aws sts get-caller-identity → verify connection
□ Step 5: Update GitHub Secrets with new credentials
□ Step 6: You're ready to work! ✅
```

### Quick commands:
```cmd
# Configure new credentials
aws configure

# Set new session token
aws configure set aws_session_token YOUR_NEW_TOKEN

# Verify connection
aws sts get-caller-identity
```

---

## 11. Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `ExpiredTokenException` | Lab session expired | Get new credentials, run `aws configure` again |
| `InvalidClientTokenId` | Wrong credentials entered | Double-check Access Key and Secret Key |
| `403 Forbidden` on website | Wrong document root | Check `.ebextensions/nginx.config` |
| `Engine execution error` | `Procfile` or `buildspec.yml` conflict | Delete those files from repo |
| Files not showing on site | Files nested in subfolder | Make sure files are in ROOT of project |
| `No Application named X` | Wrong app name in `deploy.yml` | Copy exact name from AWS Console |
| `No Environment named X` | Wrong env name in `deploy.yml` | Copy exact name from AWS Console |
| `AccessDenied` | Wrong/expired credentials | Update GitHub secrets with new credentials |
| Health showing `Grey/No Data` | Lab IAM restriction | Normal in Learner Labs — site still works! |
| `CodePipeline not authorized` | Lab blocks CodePipeline IAM | Use GitHub Actions instead |

---

## Comparison: AWS CLI vs SDK vs Console

| | AWS CLI | AWS SDK (Boto3) | AWS Console |
|---|---|---|---|
| **What it is** | Terminal commands | Python code | Web browser UI |
| **Use case** | Quick tasks | Building applications | Visual management |
| **Example** | `aws s3 ls` | `s3.list_objects_v2()` | Click S3 → view buckets |
| **Best for** | Developers | Automation/Apps | Beginners/Visual |

---

## AWS Services Used in This Setup

| Service | Purpose |
|---|---|
| **S3** | Storage for files and pipeline artifacts |
| **EC2** | Virtual server running inside Elastic Beanstalk |
| **Elastic Beanstalk** | PaaS — manages server, scaling, deployment |
| **IAM** | Identity and access management |
| **STS** | Temporary security credentials |

---

## Important Notes for AWS Learner Labs

```
✅ Allowed regions    : us-east-1 and us-west-2 only
✅ Max EC2 instances  : 9 running at once
✅ Max vCPUs          : 32
✅ EBS volume max     : 100GB
✅ Budget             : Monitor carefully! ($50 credits)

❌ Cannot create IAM roles
❌ Cannot modify LabRole trust policy
❌ CodePipeline blocked (use GitHub Actions instead)
❌ CodeBuild blocked

⚠️  Stop/terminate resources when not using to save credits!
⚠️  EC2, RDS, NAT Gateway eat credits fast!
```

---

## Resources

- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/latest/userguide/)
- [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [AWS SDK List](https://aws.amazon.com/tools/)
- [Elastic Beanstalk Documentation](https://docs.aws.amazon.com/elasticbeanstalk/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS CLI v2 Download](https://awscli.amazonaws.com/AWSCLIV2.msi)

---

*Guide created based on practical setup experience with AWS Academy Learner Labs*
