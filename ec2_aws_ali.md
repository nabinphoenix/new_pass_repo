# Launching an EC2 Instance via AWS CLI — Lab Guide (incl. HaqDeskAI EC2)

This guide documents two full EC2 launches performed in this lab session:

- **Part 1:** A basic instance (`MyFirstInstance`) launched into the account's default VPC, with Apache installed manually over SSH.
- **Part 2: HaqDeskAI EC2** — a second instance launched into a **custom-built VPC** (own subnet, internet gateway, route table, and security group), with an attached IAM role and Apache auto-installed via a user data script — no manual SSH setup required.

---

# Part 1: MyFirstInstance (Default VPC)

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

## Summary of Resources Created (Part 1)

| Resource | Identifier |
|---|---|
| Key Pair | `MyKeyPair` |
| Security Group | `sg-0feba3dd52de02558` |
| AMI Used | `ami-083eb03ee8ae87bb1` (Amazon Linux 2023) |
| Instance ID | `i-0c9e92c81a6543c56` |
| Public IP | `100.53.2.71` |
| Web Server | Apache (httpd) serving `index.html` |

---

# Part 2: HaqDeskAI EC2 — Custom VPC Launch

This part builds a fully custom network from scratch (instead of relying on the default VPC) and reuses the existing key pair, while creating a new VPC-specific security group and attaching an IAM role. Apache and the HTML page are installed automatically via a **user data** script — no manual SSH setup needed.

### What carries over vs. what needed to be rebuilt

- **Key Pair** (`MyKeyPair`) — reusable as-is; not tied to any VPC.
- **IAM Role** — reusable as-is; not tied to any VPC.
- **Security Group** — NOT reusable; security groups are tied to a specific VPC, so a new one had to be created inside the new custom VPC.

---

## Step 13: Find an Available IAM Instance Profile

```powershell
aws iam list-instance-profiles --query "InstanceProfiles[].InstanceProfileName" --output table
```

**Purpose & syntax:**
- EC2 doesn't attach an IAM *role* directly — it attaches an **instance profile**, a wrapper/container around a role.
- `list-instance-profiles` lists every instance profile available in the account.
- `--query` extracts just the names for a quick readable list.

**Result:** Found two available profiles — `EMR_EC2_DefaultRole` and `LabInstanceProfile`. Chose **`LabInstanceProfile`**.

---

## Step 14: Create the User Data Script (auto-installs Apache on boot)

```powershell
$script = @'
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd
echo '<html><head><title>HaqDeskAI Server</title></head><body><h1>Hello from HaqDeskAI!</h1><p>Apache was auto-installed via user data.</p></body></html>' > /var/www/html/index.html
'@
$script = $script -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\userdata.sh", $script)
```

**Purpose & syntax:**
- "User data" is a script EC2 automatically runs as `root` the very first time an instance boots — this is what lets Apache and the HTML page get set up with zero manual SSH steps.
- The `@' ... '@` block is a PowerShell here-string, letting multi-line text (including nested single quotes) be written literally without PowerShell trying to interpret it.
- `-replace "\`r\`n", "\`n"` strips Windows-style line endings (CRLF) down to Unix-style (LF), since the script runs inside Linux and mismatched line endings can cause boot script issues.
- `[System.IO.File]::WriteAllText` writes the cleaned-up script to `userdata.sh`, bypassing PowerShell's usual file-write cmdlets (which would reintroduce CRLF).
- Inside the script itself: no `sudo` is needed because user data scripts already run as `root` automatically.

**Result:** `userdata.sh` created locally, ready to be passed into `run-instances`.

---

## Step 15: Create a Custom VPC

```powershell
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query "Vpc.VpcId" --output text
```

**Purpose & syntax:**
- `create-vpc` creates a new isolated virtual network.
- `--cidr-block 10.0.0.0/16` defines the private IP range for the whole network (65,536 addresses: `10.0.0.0`–`10.0.255.255`).

**Result:**
```
VpcId: vpc-06894f02beeaa1fb7
```

---

## Step 16: Create a Subnet Inside the VPC

```powershell
aws ec2 create-subnet --vpc-id vpc-06894f02beeaa1fb7 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query "Subnet.SubnetId" --output text
```

**Purpose & syntax:**
- `create-subnet` carves a smaller IP range out of the VPC where instances actually live.
- `--cidr-block 10.0.1.0/24` gives this subnet 256 addresses.
- `--availability-zone` pins the subnet to a specific physical data center location, as AWS requires.

**Result:**
```
SubnetId: subnet-00ca6f168cf35ea5d
```

---

## Step 17: Enable Auto-Assign Public IP on the Subnet

```powershell
aws ec2 modify-subnet-attribute --subnet-id subnet-00ca6f168cf35ea5d --map-public-ip-on-launch
```

**Purpose & syntax:**
- `modify-subnet-attribute` changes a setting on an existing subnet.
- `--map-public-ip-on-launch` ensures any instance launched into this subnet automatically gets a public IP, instead of only a private one.
- No output on success.

---

## Step 18: Create an Internet Gateway

```powershell
aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text
```

**Purpose & syntax:**
- An Internet Gateway (IGW) is what lets traffic flow between a VPC and the public internet — a new VPC has none by default.
- This creates the gateway object (not yet connected to anything).

**Result:**
```
InternetGatewayId: igw-0e395f0107387fbc2
```

---

## Step 19: Attach the Internet Gateway to the VPC

```powershell
aws ec2 attach-internet-gateway --vpc-id vpc-06894f02beeaa1fb7 --internet-gateway-id igw-0e395f0107387fbc2
```

**Purpose & syntax:**
- `attach-internet-gateway` connects the gateway to the VPC — without this, the gateway exists but does nothing.
- No output on success.

---

## Step 20: Create a Route Table

```powershell
aws ec2 create-route-table --vpc-id vpc-06894f02beeaa1fb7 --query "RouteTable.RouteTableId" --output text
```

**Purpose & syntax:**
- A route table is a set of rules deciding where network traffic gets directed.
- This creates a new, empty table tied to the VPC.

**Result:**
```
RouteTableId: rtb-0bbd1a9ddab658f04
```

---

## Step 21: Add a Route to the Internet Gateway

```powershell
aws ec2 create-route --route-table-id rtb-0bbd1a9ddab658f04 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0e395f0107387fbc2
```

**Purpose & syntax:**
- `create-route` adds a rule to the table.
- `--destination-cidr-block 0.0.0.0/0` matches "any IP whatsoever" — the catch-all for internet-bound traffic.
- `--gateway-id` sends matching traffic out through the Internet Gateway.

**Result:** `"Return": true` — route added successfully.

---

## Step 22: Associate the Route Table with the Subnet

```powershell
aws ec2 associate-route-table --subnet-id subnet-00ca6f168cf35ea5d --route-table-id rtb-0bbd1a9ddab658f04
```

**Purpose & syntax:**
- A route table only takes effect on subnets it's explicitly linked to.
- This connects the routing rules to the actual subnet instances will launch into.

**Result:** Associated successfully (`"AssociationState": {"State": "associated"}`).

---

## Step 23: Create a Security Group Inside the New VPC

```powershell
aws ec2 create-security-group --group-name HaqDeskAI-SG --description "Allow SSH and HTTP" --vpc-id vpc-06894f02beeaa1fb7 --query "GroupId" --output text
```

**Purpose & syntax:**
- Same command as before, but `--vpc-id` explicitly ties this new group to the custom VPC instead of the default one — security groups can't be shared across VPCs.

**Result:**
```
GroupId: sg-09e1265d8d4b1e208
```

### Open Port 22 (SSH) and Port 80 (HTTP)

```powershell
aws ec2 authorize-security-group-ingress --group-id sg-09e1265d8d4b1e208 --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-09e1265d8d4b1e208 --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**Result:** Both rules confirmed via:
```powershell
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=sg-09e1265d8d4b1e208" --query "SecurityGroupRules[].[IpProtocol,FromPort,ToPort,CidrIpv4]" --output table
```
```
tcp   22   22   0.0.0.0/0
tcp   80   80   0.0.0.0/0
-1    -1   -1   0.0.0.0/0   (default outbound allow-all rule, created automatically)
```

---

## Step 24: Launch HaqDeskAI Into the Custom VPC

```powershell
aws ec2 run-instances --image-id ami-083eb03ee8ae87bb1 --instance-type t2.micro --key-name MyKeyPair --subnet-id subnet-00ca6f168cf35ea5d --security-group-ids sg-09e1265d8d4b1e208 --iam-instance-profile Name=LabInstanceProfile --user-data file://userdata.sh --count 1 --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=HaqDeskAI}]'
```

**Purpose & syntax:**
- `--subnet-id subnet-00ca6f168cf35ea5d` places the instance directly into the custom subnet — this single flag ties it to the entire custom network (VPC, routing, internet gateway).
- `--security-group-ids` points to the new VPC-specific security group.
- `--iam-instance-profile Name=LabInstanceProfile` attaches the IAM role wrapper chosen in Step 13.
- `--user-data file://userdata.sh` tells EC2 to run the script from Step 14 automatically on first boot — replacing all manual SSH setup.
- Storage was left at default (no `--block-device-mappings` flag), so it uses the AMI's standard 8 GB root volume.

**Result:**
```
InstanceId: i-0fe585cea18544122
Tag Name: HaqDeskAI
```

---

## Step 25: Get Instance ID, State & Public IP

```powershell
aws ec2 describe-instances --filters "Name=tag:Name,Values=HaqDeskAI" --query "Reservations[0].Instances[0].InstanceId" --output text
aws ec2 describe-instances --instance-ids i-0fe585cea18544122 --query "Reservations[0].Instances[0].[State.Name,PublicIpAddress]" --output table
```

**Result:**
```
State: running
Public IP: 34.200.214.18
```

---

## Step 26: Verify in Browser

After waiting 1–2 minutes for the user data script to finish running (`dnf update`, install Apache, write the HTML page) in the background:

```
http://34.200.214.18
```

Page loaded successfully showing:
> **Hello from HaqDeskAI!**
> Apache was auto-installed via user data.

---

## Summary of Resources Created (Part 2 — HaqDeskAI EC2)

| Resource | Identifier |
|---|---|
| VPC | `vpc-06894f02beeaa1fb7` (CIDR `10.0.0.0/16`) |
| Subnet | `subnet-00ca6f168cf35ea5d` (CIDR `10.0.1.0/24`, `us-east-1a`) |
| Internet Gateway | `igw-0e395f0107387fbc2` |
| Route Table | `rtb-0bbd1a9ddab658f04` |
| Security Group | `sg-09e1265d8d4b1e208` (`HaqDeskAI-SG`) |
| IAM Instance Profile | `LabInstanceProfile` |
| Key Pair (reused) | `MyKeyPair` |
| User Data Script | `userdata.sh` (auto-installs Apache) |
| AMI Used | `ami-083eb03ee8ae87bb1` (Amazon Linux 2023) |
| Instance ID | `i-0fe585cea18544122` |
| Public IP | `34.200.214.18` |
| Web Server | Apache (httpd), auto-installed via user data |

---

## Cleanup (when finished with the lab)

```bash
# Terminate both instances
aws ec2 terminate-instances --instance-ids i-0c9e92c81a6543c56 i-0fe585cea18544122
```

**Purpose & syntax:**
- `terminate-instances` permanently shuts down and deletes instances to stop billing/usage.
- `--instance-ids` accepts a space-separated list, so both instances can be terminated in one call.

If cleaning up the custom network as well (optional, only after both instances are terminated):

```bash
aws ec2 delete-security-group --group-id sg-09e1265d8d4b1e208
aws ec2 detach-internet-gateway --vpc-id vpc-06894f02beeaa1fb7 --internet-gateway-id igw-0e395f0107387fbc2
aws ec2 delete-internet-gateway --internet-gateway-id igw-0e395f0107387fbc2
aws ec2 delete-subnet --subnet-id subnet-00ca6f168cf35ea5d
aws ec2 delete-route-table --route-table-id rtb-0bbd1a9ddab658f04
aws ec2 delete-vpc --vpc-id vpc-06894f02beeaa1fb7
```

These reverse each creation step in order: detach/delete the gateway, delete the subnet, delete the route table, then finally delete the VPC itself (a VPC can't be deleted while it still has attached resources).
