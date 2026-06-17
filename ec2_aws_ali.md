# Launching an EC2 Instance via AWS CLI — Lab Guide

This guide documents the full process of launching an EC2 instance from scratch using the AWS CLI, installing Apache on it, and serving a simple HTML page — exactly as performed in this lab session.

---

## Step 0: Verify AWS CLI Authentication

```bash
aws sts get-caller-identity
```

**Purpose & syntax:**
- `aws sts get-caller-identity` checks which AWS identity the CLI is currently authenticated as.
- `sts` = Security Token Service (handles identity/credentials).
- `get-caller-identity` returns the account ID, user ID, and ARN tied to the active credentials.
- Used to confirm the CLI is properly configured before doing anything else.

**Result:** Confirmed authentication under the AWS Academy Learner Lab role:
```
Account: 048566646797
Arn: arn:aws:sts::048566646797:assumed-role/voclabs/user4735049=Nabin_Nepali
```

---

## Step 1: Create a Key Pair (for SSH access)

```powershell
aws ec2 create-key-pair --key-name MyKeyPair --query "KeyMaterial" --output text | Out-File -Encoding ascii MyKeyPair.pem
```

**Purpose & syntax:**
- `create-key-pair` generates a new SSH key pair on AWS's side; the private key is returned once and never stored again.
- `--key-name MyKeyPair` is the identifier AWS stores this key pair under.
- `--query "KeyMaterial"` filters the JSON response down to just the private key text.
- `--output text` removes quotes/formatting for a clean raw key.
- `Out-File -Encoding ascii` (PowerShell-specific) writes the file without UTF-16/BOM encoding issues that would corrupt the `.pem` format.

**Result:** `MyKeyPair.pem` created locally — the private key used later to SSH into the instance.

---

## Step 2: Create a Security Group (firewall rules)

```powershell
aws ec2 create-security-group --group-name MySecurityGroup --description "Allow SSH and HTTP"
```

**Purpose & syntax:**
- `create-security-group` creates a new (initially empty) firewall rule container.
- `--group-name` is a human-readable label.
- `--description` is a required metadata field.

**Result:**
```
GroupId: sg-0feba3dd52de02558
```

### Open Port 22 (SSH)

```powershell
aws ec2 authorize-security-group-ingress --group-id sg-0feba3dd52de02558 --protocol tcp --port 22 --cidr 0.0.0.0/0
```

### Open Port 80 (HTTP)

```powershell
aws ec2 authorize-security-group-ingress --group-id sg-0feba3dd52de02558 --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**Purpose & syntax:**
- `authorize-security-group-ingress` adds an inbound rule to an existing security group.
- `--group-id` targets which security group to modify.
- `--protocol tcp --port 22/80` specifies the traffic type and port allowed (SSH and HTTP respectively).
- `--cidr 0.0.0.0/0` allows traffic from any IP address (fine for a lab; should be restricted to a specific IP in production).

**Result:** Both rules added successfully, confirmed by `"Return": true` and returned `SecurityGroupRuleId` values for each rule.

---

## Step 3: Find an AMI (OS Image)

```powershell
aws ec2 describe-images --owners amazon --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" --query "Images | sort_by(@, &CreationDate) | [-1].ImageId" --output text
```

**Purpose & syntax:**
- `describe-images` searches AWS's catalog of machine images.
- `--owners amazon` restricts results to AWS's official published images.
- `--filters` narrows results by name pattern (Amazon Linux 2023, 64-bit) and availability state.
- `--query "Images | sort_by(@, &CreationDate) | [-1].ImageId"` is a JMESPath expression: sorts matches by creation date and extracts the `ImageId` of the newest one.
- `--output text` returns a clean string instead of JSON.

**Result:**
```
AMI ID: ami-083eb03ee8ae87bb1   (Amazon Linux 2023, x86_64)
```

---

## Step 4: Launch the Instance

```powershell
aws ec2 run-instances --image-id ami-083eb03ee8ae87bb1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids sg-0feba3dd52de02558 --count 1 --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyFirstInstance}]'
```

**Purpose & syntax:**
- `run-instances` is the actual launch command.
- `--image-id` specifies which OS image to boot (the AMI from Step 3).
- `--instance-type t2.micro` sets the hardware size (1 vCPU, 1 GB RAM).
- `--key-name` attaches the SSH key pair created in Step 1.
- `--security-group-ids` attaches the firewall rules from Step 2.
- `--count 1` launches a single instance.
- `--tag-specifications` assigns a `Name` tag so the instance is identifiable by name, not just by ID.

**Result:**
```
InstanceId: i-0c9e92c81a6543c56
State: pending → running
Tag Name: MyFirstInstance
Security Group attached: sg-0feba3dd52de02558
```

---

## Step 5: Retrieve the Instance ID (by tag)

```powershell
aws ec2 describe-instances --filters "Name=tag:Name,Values=MyFirstInstance" --query "Reservations[0].Instances[0].InstanceId" --output text
```

**Purpose & syntax:**
- `describe-instances` retrieves details about EC2 instances.
- `--filters "Name=tag:Name,Values=MyFirstInstance"` finds the instance by its assigned `Name` tag instead of needing to already know the ID.
- `--query` drills into the nested JSON response to extract just the `InstanceId`.

**Result:**
```
i-0c9e92c81a6543c56
```

---

## Step 6: Check Instance State & Public IP

```powershell
aws ec2 describe-instances --instance-ids i-0c9e92c81a6543c56 --query "Reservations[0].Instances[0].[State.Name,PublicIpAddress]" --output table
```

**Purpose & syntax:**
- `--instance-ids` targets the specific instance directly using its ID.
- `--query` extracts two fields: `State.Name` (running/pending/stopped) and `PublicIpAddress`.
- `--output table` renders the result as a readable table.

**Result:**
```
State: running
Public IP: 100.53.2.71
```

---

## Step 7: Fix Key File Permissions (Windows)

The first SSH attempt failed because Windows had inherited broad file permissions (including `Authenticated Users`) on the `.pem` file, and SSH refuses to use a key that's accessible by anyone other than the owner.

```powershell
icacls MyKeyPair.pem /inheritance:r
icacls MyKeyPair.pem /grant:r "$($env:USERNAME):(R)"
```

**Purpose & syntax:**
- `icacls` is Windows' command-line tool for managing file access control lists (the Windows equivalent of Linux's `chmod`).
- `/inheritance:r` removes permissions inherited from the parent folder.
- `/grant:r "$($env:USERNAME):(R)"` grants read-only access to exactly one user (the current Windows account), replacing any other explicit permissions.

**Result:** Permissions corrected — file now only readable by the owner.

---

## Step 8: Connect via SSH

```powershell
ssh -i MyKeyPair.pem ec2-user@100.53.2.71
```

**Purpose & syntax:**
- `ssh` is the Secure Shell client used to remotely log into the instance.
- `-i MyKeyPair.pem` specifies the private key matching the public key installed on the instance at launch.
- `ec2-user` is the default login username for Amazon Linux AMIs.
- `100.53.2.71` is the instance's public IP address.

**Result:** Successfully logged into the instance, landing at the `[ec2-user@ip-172-31-31-137 ~]$` shell prompt.

---

## Step 9: Install Apache Web Server

```bash
sudo dnf update -y
sudo dnf install -y httpd
```

**Purpose & syntax:**
- `dnf` is Amazon Linux 2023's package manager.
- `sudo` runs the command with root/administrator privileges (required to install software).
- `update -y` refreshes package version info and auto-confirms prompts.
- `install -y httpd` installs Apache (named `httpd` on Red Hat-based distros like Amazon Linux).

---

## Step 10: Start and Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

**Purpose & syntax:**
- `systemctl` controls system services.
- `start httpd` launches the Apache web server immediately.
- `enable httpd` configures Apache to auto-start on every future reboot.

**Result:** Apache running and verified via `sudo systemctl status httpd` showing `active (running)`.

---

## Step 11: Create a Simple HTML Page

```bash
echo '<html><head><title>My First EC2 Server</title></head><body><h1>Hello from my EC2 instance!</h1><p>Apache is running successfully.</p></body></html>' | sudo tee /var/www/html/index.html
```

**Purpose & syntax:**
- `/var/www/html/` is Apache's default web root — any file here becomes servable.
- `index.html` is the default filename Apache serves when no specific page is requested.
- Single quotes (`'...'`) are used instead of double quotes to avoid bash's history expansion feature misinterpreting the `!` character.
- `sudo tee` writes the file with elevated privileges (a plain `>` redirect would fail since the regular user lacks write access to `/var/www/html/`).

**Result:** `index.html` created successfully inside the web root.

---

## Step 12: Verify the Page is Served

```bash
curl localhost
```

**Purpose & syntax:**
- `curl localhost` sends an HTTP request to the local Apache server and prints the response — used to confirm Apache is serving the correct file from inside the instance itself.

**Result:** HTML content returned correctly, confirming Apache is serving the page.

**Final check (from local browser):**
```
http://100.53.2.71
```

Page loaded successfully showing:
> **Hello from my EC2 instance!**
> Apache is running successfully.

---

## Summary of Resources Created

| Resource | Identifier |
|---|---|
| Key Pair | `MyKeyPair` |
| Security Group | `sg-0feba3dd52de02558` |
| AMI Used | `ami-083eb03ee8ae87bb1` (Amazon Linux 2023) |
| Instance ID | `i-0c9e92c81a6543c56` |
| Public IP | `100.53.2.71` |
| Web Server | Apache (httpd) serving `index.html` |

---

## Cleanup (when finished with the lab)

```bash
aws ec2 terminate-instances --instance-ids i-0c9e92c81a6543c56
```

**Purpose & syntax:**
- `terminate-instances` permanently shuts down and deletes the instance to stop billing/usage.
- `--instance-ids` accepts one or more instance IDs to terminate.
