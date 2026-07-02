# Devops-Assesment-AWS
# AWS Highly Available Web Infrastructure — DevOps Foundation Assignment

This project deploys a **highly available, fault-tolerant web application** on AWS using only **Free Tier** resources. It covers IAM, VPC networking, Security Groups, EC2, EBS, Target Groups, an Application Load Balancer, and Auto Scaling — built, tested, broken, and fixed end-to-end.

## Architecture

```
Internet
   │
   ▼
Application Load Balancer (Public-Subnet-A / Public-Subnet-B)
   │
   ▼
Target Group
   │
   ▼
EC2-1 (AZ-A)        EC2-2 (AZ-B)
   │                     │
   ▼                     ▼
EBS Volume           EBS Volume

Networking : VPC → Subnets → Route Table → Internet Gateway
Security   : IAM → Security Groups
```

## Tech Stack / Services Used
IAM · VPC · Subnets · Internet Gateway · Route Tables · Security Groups · EC2 · EBS · Target Groups · Application Load Balancer · Auto Scaling Group

---

## Table of Contents
1. [Part 1 — IAM](#part-1--iam)
2. [Part 2 — VPC & Networking](#part-2--vpc--networking)
3. [Part 3 — Security Groups](#part-3--security-groups)
4. [Part 4 — EC2](#part-4--ec2)
5. [Part 5 — EBS](#part-5--ebs)
6. [Part 6 — Target Groups](#part-6--target-groups)
7. [Part 7 — Application Load Balancer](#part-7--application-load-balancer)
8. [Part 8 — Failure Testing](#part-8--failure-testing)
9. [Part 9 — Auto Scaling Group](#part-9--auto-scaling-group)
10. [Troubleshooting Challenges](#troubleshooting-challenges)
11. [Q&A](#qa)
12. [Key Learnings](#key-learnings)

---

## Part 1 — IAM

### Step 1: Create IAM Users
Console → **IAM → Users → Create user** → repeat for `developer1`, `developer2`, `devops1`.

Equivalent via AWS CLI:
```bash
aws iam create-user --user-name developer1
aws iam create-user --user-name developer2
aws iam create-user --user-name devops1
```

![IAM Users](PART%201%20%E2%80%94%20IAM/Part1-IAM-step1-Create%20IAM%20Users.png)

### Step 2: Create IAM Groups & Attach Policies
```bash
# Create groups
aws iam create-group --group-name Developers
aws iam create-group --group-name DevOps

# Attach policies
aws iam attach-group-policy --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

aws iam attach-group-policy --group-name DevOps \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

aws iam attach-group-policy --group-name DevOps \
  --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess
```

![Create IAM Groups 1](PART%201%20%E2%80%94%20IAM/Part1-IAM%20-%20Step%202-%20Create%20IAM%20Groups-ss1.png)
![Create IAM Groups 2](PART%201%20%E2%80%94%20IAM/Part1-IAM-Step%202-%20Create%20IAM%20Groups-ss2.png)
![Create IAM Groups 3](PART%201%20%E2%80%94%20IAM/Part1-IAM-Step%202-Create%20IAM%20Groups-ss3.png)

### Step 3: Add Users to Groups
```bash
aws iam add-user-to-group --user-name developer1 --group-name Developers
aws iam add-user-to-group --user-name developer2 --group-name Developers
aws iam add-user-to-group --user-name devops1 --group-name DevOps
```

![Add users to groups 1](PART%201%20%E2%80%94%20IAM/Part1-IAM-Step%203-%20Add%20users%20to%20groups-ss1.png)
![Add users to groups 2](PART%201%20%E2%80%94%20IAM/Part1-IAM-Step%203-Add%20users%20to%20groups-ss2.png)
![Add users to groups 3](PART%201%20%E2%80%94%20IAM/Part1-IAM-Step%203-Add%20users%20to%20groups-ss3.png)

---

## Part 2 — VPC & Networking

### Step 1: Create the VPC
```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=DevOps-VPC}]'
```
![Create VPC](PART%202%20%E2%80%94%20VPC%20%26%20Networking/PART%202%20%E2%80%94%20VPC%20%26%20Networking-step1-Createvpc.png)

### Step 2: Create Public Subnets (in two AZs)
```bash
VPC_ID=<your-vpc-id>

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet-A}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet-B}]'
```
![Create Subnets](PART%202%20%E2%80%94%20VPC%20%26%20Networking/PART%202%20%E2%80%94%20VPC%20%26%20Networking--step-2-Create%20subnets.png)

### Step 3: Create & Attach Internet Gateway
```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=DevOps-IGW}]'

IGW_ID=<your-igw-id>
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
```
![Create and attach IGW](PART%202%20%E2%80%94%20VPC%20%26%20Networking/PART%202%20%E2%80%94%20VPC%20%26%20Networking-step3-Create%20and%20attach%20Internet%20Gateway.png)

### Step 4: Create Route Table + Default Route
```bash
aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]'

RT_ID=<your-route-table-id>
aws ec2 create-route --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
```
![Route Table](PART%202%20%E2%80%94%20VPC%20%26%20Networking/PART%202%20%E2%80%94%20VPC%20%26%20Networking-step4-Route%20Table.png)

### Step 5: Associate Both Subnets with the Route Table
```bash
aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id <Public-Subnet-A-id>
aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id <Public-Subnet-B-id>
```
![Associate Subnets](PART%202%20%E2%80%94%20VPC%20%26%20Networking/PART%202%20%E2%80%94%20VPC%20%26%20Networking-step5-Associate%20Subnets.png)

---

## Part 3 — Security Groups

### Step 1: Create Security Group + Rules
```bash
aws ec2 create-security-group --group-name Web-SG \
  --description "Allow SSH and HTTP" --vpc-id $VPC_ID

SG_ID=<your-sg-id>

aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```
![Create SG](PART%203%20%E2%80%94%20Security%20Groups/PART%203%20%E2%80%94%20Security%20Groups-step1-Create%20SG.png)

### Step 2: Validate Access
```bash
# SSH access
ssh -i my-key.pem ec2-user@<EC2-PUBLIC-IP>

# HTTP access
curl http://<EC2-PUBLIC-IP>
```
![Validate SSH/HTTP 1](PART%203%20%E2%80%94%20Security%20Groups/Part3-Security%20Group-step-2-Validate.png)
![Validate SSH/HTTP 2](PART%203%20%E2%80%94%20Security%20Groups/Part3-Security%20Group-step2-validate-ss2.png)

### Step 3: Challenge — Remove HTTP (80)
```bash
aws ec2 revoke-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Test — this should now time out
curl -m 5 http://<EC2-PUBLIC-IP>
```
**Result:** `curl: (28) Connection timed out` — the browser also shows `ERR_CONNECTION_TIMED_OUT`.

Restore it:
```bash
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```
![Challenge - Remove HTTP](PART%203%20%E2%80%94%20Security%20Groups/PART%203%20%E2%80%94%20Security%20Groups-Step%203-%20Challenge%20%E2%80%94%20Remove%20HTTP-ss1.png)

---

## Part 4 — EC2

### Step 1: Launch EC2-1 (Public-Subnet-A)
```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t2.micro \
  --key-name my-key \
  --subnet-id <Public-Subnet-A-id> \
  --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EC2-1}]' \
  --user-data '#!/bin/bash
yum install -y nginx
echo "Server 1 - $(hostname)" > /usr/share/nginx/html/index.html
systemctl enable nginx
systemctl start nginx'
```

### Step 2: Launch EC2-2 (Public-Subnet-B)
```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t2.micro \
  --key-name my-key \
  --subnet-id <Public-Subnet-B-id> \
  --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EC2-2}]' \
  --user-data '#!/bin/bash
yum install -y nginx
echo "Server 2 - $(hostname)" > /usr/share/nginx/html/index.html
systemctl enable nginx
systemctl start nginx'
```

### Verify
```bash
curl http://<EC2-1-IP>
curl http://<EC2-2-IP>
```
![EC2-1](PART%204%20%E2%80%94%20EC2/Part4-ec2--1-ss1.png)
![EC2-2](PART%204%20%E2%80%94%20EC2/part4-ec2-2-ss1.png)

---

## Part 5 — EBS

### Step 1: Inspect the Root Volume
```bash
aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=<EC2-1-instance-id>
```
![Inspect root volume](PART%205%20%E2%80%94%20EBS/Part5-EBS-step1-Inspectrootvolume1.png)

### Step 2: Create a New 5 GB Volume
```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 5 \
  --volume-type gp2 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=Data-Volume}]'
```
![Create new volume](PART%205%20%E2%80%94%20EBS/Part5-EBS-step2-%20Create%20new%20volume.png)

### Step 3: Attach It to EC2-1
```bash
aws ec2 attach-volume \
  --volume-id <volume-id> \
  --instance-id <EC2-1-instance-id> \
  --device /dev/xvdf
```

### Step 4: Mount the Volume
SSH into EC2-1, then:
```bash
lsblk
sudo mkfs -t ext4 /dev/xvdf
sudo mkdir /mnt/data
sudo mount /dev/xvdf /mnt/data
df -h
```
![Mount volume](PART%205%20%E2%80%94%20EBS/Part5-EBS-step4-mountit.png)

### Step 5: Create a File & Test Persistence
```bash
sudo sh -c 'echo "This is my assignment file" > /mnt/data/assignment.txt'
cat /mnt/data/assignment.txt

# Reboot
sudo reboot
```
Reconnect after ~1 minute:
```bash
ssh -i my-key.pem ec2-user@<EC2-1-IP>
cat /mnt/data/assignment.txt
```
**Result:** File content is still there after reboot — proves EBS persists independently of the instance lifecycle (unlike instance-store/RAM).

![Create file & test persistence](PART%205%20%E2%80%94%20EBS/Part5-EBS-step5-Create%20file%20%26%20test%20persistence.png)
![EBS Persistent Storage](PART%205%20%E2%80%94%20EBS/PART%205%20EBS%20%20Persistent%20Storage.png)

---

## Part 6 — Target Groups

### Step 1: Create Target Group & Register Targets
```bash
aws elbv2 create-target-group \
  --name Web-TG \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID \
  --health-check-protocol HTTP --health-check-path /

TG_ARN=<target-group-arn>

aws elbv2 register-targets --target-group-arn $TG_ARN \
  --targets Id=<EC2-1-instance-id> Id=<EC2-2-instance-id>
```

### Verify Health
```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```
![Target Group healthy](PART%206%20%E2%80%94%20Target%20Groups/Part6-Target%20Group-Create%20Target%20group-ss1.png)

---

## Part 7 — Application Load Balancer

### Step 1: Create the ALB
```bash
aws elbv2 create-load-balancer \
  --name Web-ALB \
  --subnets <Public-Subnet-A-id> <Public-Subnet-B-id> \
  --security-groups $SG_ID \
  --scheme internet-facing \
  --type application

ALB_ARN=<load-balancer-arn>
```

### Step 2: Create Listener → Forward to Target Group
```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```
![ALB created](PART%207%20%E2%80%94%20Application%20Load%20Balancer/Part7-Application%20Load%20Balancer-ss1.png)

### Step 3: Validate
```bash
ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

# Refresh multiple times to see both servers respond
curl http://$ALB_DNS
curl http://$ALB_DNS
curl http://$ALB_DNS
```
![Validate ALB 1](PART%207%20%E2%80%94%20Application%20Load%20Balancer/Part7-Application%20Load%20Balancer-step-2-validate.png)
![Validate ALB 2](PART%207%20%E2%80%94%20Application%20Load%20Balancer/Part7-Application%20Load%20Balancer-step-2-validate-ss2.png)

---

## Part 8 — Failure Testing

### Step 1: Stop EC2-1
```bash
aws ec2 stop-instances --instance-ids <EC2-1-instance-id>
```
![EC2-1 stopped](PART%208%20%E2%80%94%20Failure%20Testing/part8-failiuretesting-ss1.png)

### Step 2: Confirm It Goes Unhealthy
```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```
![Target unhealthy](PART%208%20%E2%80%94%20Failure%20Testing/part8-failiuretesting-ss2.png)

### Step 3: Confirm the Site Is Still Up (via EC2-2)
```bash
curl http://$ALB_DNS
```
![Site still up](PART%208%20%E2%80%94%20Failure%20Testing/part8-failiure%20testing-ss3.png)

### Step 4: Start EC2-1 Again & Confirm Recovery
```bash
aws ec2 start-instances --instance-ids <EC2-1-instance-id>
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```
![Both healthy again](PART%208%20%E2%80%94%20Failure%20Testing/part8-Failiuretesting-ss4.png)

---

## Part 9 — Auto Scaling Group

### Step 1: Create a Launch Template
```bash
aws ec2 create-launch-template \
  --launch-template-name Web-LT \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t2.micro",
    "SecurityGroupIds": ["'"$SG_ID"'"],
    "UserData": "'"$(base64 -w0 <<'EOF'
#!/bin/bash
yum install -y nginx
echo "Server - $(hostname)" > /usr/share/nginx/html/index.html
systemctl enable nginx
systemctl start nginx
EOF
)"'"
  }'
```
![Launch Template](PART%209%20%E2%80%94%20Auto%20Scaling%20Group/part9-Autoscaling-step1-launchtemplate-ss.png)

### Step 2: Create the Auto Scaling Group
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name Web-ASG \
  --launch-template LaunchTemplateName=Web-LT,Version='$Latest' \
  --min-size 1 --desired-capacity 1 --max-size 2 \
  --target-group-arns $TG_ARN \
  --vpc-zone-identifier "<Public-Subnet-A-id>,<Public-Subnet-B-id>"
```
![Create ASG](PART%209%20%E2%80%94%20Auto%20Scaling%20Group/part9-Autoscaling-step2-Create%20ASG-ss.png)

### Step 3: Verify Attachment & Activity
```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names Web-ASG
aws autoscaling describe-scaling-activities --auto-scaling-group-name Web-ASG
```
![ASG Details](PART%209%20%E2%80%94%20Auto%20Scaling%20Group/part9-Autoscaling-step2-%20ASG%20Details-ss3.png)
![ASG Activity](PART%209%20%E2%80%94%20Auto%20Scaling%20Group/part9-Autoscaling-step2-%20ASG%20Activity-ss4.png)

---

## Troubleshooting Challenges

### Challenge 1 — Delete the Default Route
```bash
aws ec2 delete-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0
curl -m 5 http://$ALB_DNS      # times out / unreachable

# Fix:
aws ec2 create-route --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
curl http://$ALB_DNS           # works again
```

### Challenge 2 — Remove Port 80 from Security Group
```bash
aws ec2 revoke-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
curl -m 5 http://$ALB_DNS      # times out

# Fix:
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### Challenge 3 — Stop One EC2 Instance
```bash
aws ec2 stop-instances --instance-ids <EC2-1-instance-id>
curl http://$ALB_DNS           # still works — served by EC2-2
```
**Why:** the ALB routes only to healthy targets in the Target Group; the failed instance is automatically taken out of rotation.

### Challenge 4 — Detach the EBS Volume
```bash
aws ec2 detach-volume --volume-id <volume-id>

# On the instance:
df -h                          # /mnt/data no longer listed
cat /mnt/data/assignment.txt   # No such file or directory
```
![EBS detached impact](PART%205%20%E2%80%94%20EBS/PART%205%20EBS%20%20Persistent%20Storage.png)

### Challenge 5 — Remove All Targets from the Target Group
```bash
aws elbv2 deregister-targets --target-group-arn $TG_ARN \
  --targets Id=<EC2-1-instance-id> Id=<EC2-2-instance-id>

curl http://$ALB_DNS           # 503 Service Unavailable — No healthy targets

# Fix:
aws elbv2 register-targets --target-group-arn $TG_ARN \
  --targets Id=<EC2-1-instance-id> Id=<EC2-2-instance-id>
```

---

## Q&A

### Part 1 — IAM
- **Why not use the root account daily?** Root has unrestricted access to every service and billing setting; a leaked root credential is a full account takeover. Day-to-day work should use least-privilege IAM identities instead.
- **IAM User vs IAM Role?** A User is a permanent identity with long-term credentials tied to a specific person/app. A Role is an identity with temporary credentials that anything (a user, service, or AWS resource) can assume — no long-term keys involved.
- **Why Groups over per-user permissions?** Groups let you manage permissions once for many people; adding/removing a user from a group is far simpler and less error-prone than editing individual policies.

### Part 2 — VPC
- **Why a custom VPC instead of default?** Full control over CIDR ranges, subnet layout, and routing — needed for intentional AZ placement and isolation, which the default VPC doesn't guarantee.
- **Why multiple AZs?** AZs are physically isolated; spreading resources across them protects against a single data-center-level failure.
- **If the IGW is detached?** Public subnets lose internet connectivity — no inbound or outbound traffic to/from the internet.
- **If the default route is removed?** Instances keep their public IPs but become unreachable from the internet, since there's no path out of the VPC.

### Part 3 — Security Groups
- **Purpose of a Security Group?** A stateful, instance-level virtual firewall controlling inbound/outbound traffic.
- **Security Group vs NACL?** SGs are stateful (return traffic auto-allowed) and apply at the instance level; NACLs are stateless (return traffic must be explicitly allowed) and apply at the subnet level.
- **Why not open SSH to the world in production?** It exposes the instance to brute-force and exploit attempts from any IP; production access should be limited to a bastion host, VPN, or specific trusted IP ranges.

### Part 4 — EC2
- **Why separate AZs?** So the loss of one AZ doesn't take down the whole application.
- **If one AZ goes down?** The instance(s) in that AZ become unreachable, but traffic is served from the surviving AZ via the ALB/Target Group — the app stays up.

### Part 5 — EBS
- **Why does the file survive reboot?** EBS is persistent, network-attached block storage independent of the instance's lifecycle — a reboot doesn't wipe it (only terminating the instance with "delete on termination" enabled would).
- **RAM vs EBS?** RAM is volatile, in-instance memory wiped on any power loss/reboot; EBS is durable, persistent block storage that survives reboots and can be detached/reattached.
- **If EBS is detached?** Any mounted paths become inaccessible immediately; unsaved reads/writes to that volume fail until it's reattached.

### Part 6 — Target Groups
- **Purpose of a Target Group?** Groups backend targets (EC2 instances, IPs, Lambdas) and continuously health-checks them so traffic only routes to healthy ones.
- **Why does an ALB use Target Groups?** It decouples routing rules from targets — the ALB just forwards to whichever targets in the group are currently healthy.

### Part 7 — Load Balancer
- **Why is a Load Balancer required?** It distributes traffic across multiple instances/AZs, removing any single instance as a point of failure and enabling horizontal scaling.
- **If one EC2 instance fails?** The ALB detects the failed health check and stops sending it traffic — the other instance(s) keep serving requests.
- **Purpose of Health Checks?** They let the ALB detect unhealthy targets in real time and automatically route around them.

### Part 8 — Failure Testing
- **Why was the app still accessible?** Because a second healthy instance existed in the Target Group behind the ALB — the load balancer simply stopped sending traffic to the failed one.
- **Which component handled the failure?** The Application Load Balancer + Target Group health checks.

### Part 9 — Auto Scaling
- **What is Desired Capacity?** The number of instances the ASG actively tries to maintain at any time (between Min and Max).
- **Purpose of a Launch Template?** A reusable blueprint (AMI, instance type, security group, user data, etc.) the ASG uses to launch new instances consistently.
- **How does Auto Scaling improve availability?** It automatically replaces unhealthy/terminated instances and can scale out under load, keeping capacity at the desired level without manual intervention.

---

## Full Request Flow (Demo Summary)

```
User request
   → Route 53 / direct DNS resolves ALB DNS name
   → Application Load Balancer (in Public-Subnet-A & B, via IGW-routed subnets)
   → Target Group (health-checks decide which targets are eligible)
   → EC2 instance (Nginx, protected by Security Group allowing port 80)
   → EBS volume (serves/stores the instance's data, e.g. logs or mounted app data)

Supporting layers throughout the request lifecycle:
   - IAM        → controls who/what can manage these resources (not part of the data path itself)
   - VPC/Subnets/Route Tables/IGW → provide the network path in and out of AWS
   - Security Groups → gatekeep traffic at the instance/ALB level
```

## Key Learnings
- High availability comes from redundancy across AZs + automated health-based routing, not from any single "HA service."
- Security Groups (stateful, instance-level) and NACLs (stateless, subnet-level) serve different, complementary purposes.
- EBS volumes are independent lifecycle objects from EC2 instances — this is what makes data persistence and volume portability possible.
- ALB + Target Group + Auto Scaling Group together form a self-healing pattern: failures are detected and routed around automatically.
- Breaking things on purpose (routes, SGs, targets) is the fastest way to actually understand what each component is responsible for.
