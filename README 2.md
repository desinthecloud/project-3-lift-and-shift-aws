# PROJECT 3: AWS Lift and Shift (Manual Deployment)

## Project Goal

Deploy the vprofile multi tier Java application to AWS using EC2 instances, Application Load Balancer, Auto Scaling, S3, and Route 53. This is a **manual deployment** where you'll use the AWS Console and CLI to build everything step by step.

**What "done" looks like:**

- Backend services on EC2 (Ubuntu 22.04):
  - MySQL database
  - Memcached caching
  - RabbitMQ message broker
- Application tier:
  - Tomcat app server serving your WAR file
- Frontend:
  - Application Load Balancer handling HTTP traffic
- Infrastructure:
  - Private Route 53 zone for backend DNS
  - S3 bucket for artifact storage
  - Auto Scaling Group for Tomcat (optional but recommended)
- Optional:
  - Public domain with Route 53
  - SSL certificate with ACM

---

## Prerequisites

**What you need before starting:**

1. AWS account with admin access
2. AWS CLI installed and configured on your M4 MacBook Air
3. vprofile application code (from PROJECT 1/2)
4. Maven installed locally to build the WAR file
5. SSH client
6. Budget: ~$15/day while running, ~$190 credits = 11 days

**Verify AWS CLI:**
```bash
aws --version
aws sts get-caller-identity
```

---

## Architecture Overview

```
Internet
   ↓
Application Load Balancer (Public)
   ↓
Tomcat EC2 (Private Subnet)
   ↓
MySQL / Memcached / RabbitMQ (Private Subnet)
```

**Network:**
- Default VPC (or create new VPC)
- Public subnets for ALB
- Private subnets for app and backend
- Route 53 private zone for backend DNS
- Security groups controlling traffic flow

---

## Step 0: Create a Key Pair

**Purpose:** SSH access to all EC2 instances

1. Go to **EC2 Console** → **Key Pairs**
2. Click **Create key pair**
3. Name: `vprofile-key`
4. Type: **RSA**
5. Format: **.pem** (for Mac/Linux)
6. Click **Create**
7. Save `vprofile-key.pem` to `~/.ssh/`
8. Set permissions:

```bash
chmod 400 ~/.ssh/vprofile-key.pem
```

---

## Step 1: Create Security Groups

Create **3 security groups** in this order.

### A) Backend Security Group

**Name:** `vprofile-backend-sg`

**Description:** Security group for MySQL, Memcached, RabbitMQ

**Inbound Rules:**

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| MySQL | TCP | 3306 | vprofile-app-sg | From Tomcat |
| Custom TCP | TCP | 11211 | vprofile-app-sg | Memcached from Tomcat |
| Custom TCP | TCP | 5672 | vprofile-app-sg | RabbitMQ from Tomcat |
| All Traffic | All | All | vprofile-backend-sg | Backend to backend |
| SSH | TCP | 22 | My IP | SSH access |

**Outbound Rules:** All traffic to 0.0.0.0/0

### B) App Security Group

**Name:** `vprofile-app-sg`

**Description:** Security group for Tomcat application servers

**Inbound Rules:**

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom TCP | TCP | 8080 | vprofile-alb-sg | From ALB |
| SSH | TCP | 22 | My IP | SSH access |

**Outbound Rules:** All traffic to 0.0.0.0/0

### C) ALB Security Group

**Name:** `vprofile-alb-sg`

**Description:** Security group for Application Load Balancer

**Inbound Rules:**

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| HTTP | TCP | 80 | 0.0.0.0/0 | Public HTTP |
| HTTP | TCP | 80 | ::/0 | Public HTTP IPv6 |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Public HTTPS |
| HTTPS | TCP | 443 | ::/0 | Public HTTPS IPv6 |

**Outbound Rules:** All traffic to 0.0.0.0/0

**Note:** You need to create the backend SG first, then app SG (referencing backend), then ALB SG (referencing app).

---

## Step 2: Launch Backend EC2 Instances

Launch **3 instances** with these settings:

### Common Settings for All Backend Instances:

- **AMI:** Ubuntu Server 22.04 LTS (free tier eligible)
- **Instance Type:** t2.micro
- **Network:** Default VPC
- **Subnet:** Any private subnet (or public for now, we'll fix routing later)
- **Auto assign Public IP:** Enable (for now, to download packages)
- **Security Group:** `vprofile-backend-sg`
- **Key Pair:** `vprofile-key`
- **Storage:** 8 GB gp3

### Instance 1: MySQL

**Name tag:** `vprofile-mysql`

**User Data:** Copy the entire contents of `PROJECT_3/userdata/mysql.sh`

### Instance 2: Memcached

**Name tag:** `vprofile-memcached`

**User Data:** Copy the entire contents of `PROJECT_3/userdata/memcache.sh`

### Instance 3: RabbitMQ

**Name tag:** `vprofile-rabbitmq`

**User Data:** Copy the entire contents of `PROJECT_3/userdata/rabbitmq.sh`

**Launch all 3 instances and wait 5-10 minutes for user data scripts to complete.**

---

## Step 3: Verify Backend Services

SSH into each backend instance and verify the services are running.

### Verify MySQL:

```bash
ssh -i ~/.ssh/vprofile-key.pem ubuntu@<mysql-public-ip>

# Check status
sudo systemctl status mysql

# Verify database
sudo mysql -uroot -padmin123 -e "SHOW DATABASES;"

# Should see "accounts" database

# Check setup completion
cat /tmp/db-setup-complete.txt

exit
```

### Verify Memcached:

```bash
ssh -i ~/.ssh/vprofile-key.pem ubuntu@<memcached-public-ip>

# Check status
sudo systemctl status memcached

# Test connection
echo "stats" | nc localhost 11211

# Should see memcached stats

exit
```

### Verify RabbitMQ:

```bash
ssh -i ~/.ssh/vprofile-key.pem ubuntu@<rabbitmq-public-ip>

# Check status
sudo systemctl status rabbitmq-server

# List users
sudo rabbitmqctl list_users

# Should see "test" user

exit
```

---

## Step 4: Create Route 53 Private Hosted Zone

**Purpose:** Give backend services stable DNS names instead of IPs.

1. Go to **Route 53** → **Hosted zones**
2. Click **Create hosted zone**
3. Settings:
   - **Domain name:** `vprofile.in`
   - **Type:** Private hosted zone
   - **VPCs:** Select your VPC (usually default VPC)
   - **Region:** us-east-1
4. Click **Create hosted zone**

### Create DNS Records:

Click **Create record** for each backend service:

**Record 1: MySQL**
- Record name: `db01`
- Record type: A
- Value: <MySQL private IP>
- TTL: 300

**Record 2: Memcached**
- Record name: `mc01`
- Record type: A
- Value: <Memcached private IP>
- TTL: 300

**Record 3: RabbitMQ**
- Record name: `rmq01`
- Record type: A
- Value: <RabbitMQ private IP>
- TTL: 300

**Test DNS resolution from any EC2 instance:**

```bash
ssh -i ~/.ssh/vprofile-key.pem ubuntu@<mysql-public-ip>
nslookup db01.vprofile.in
nslookup mc01.vprofile.in
nslookup rmq01.vprofile.in
```

Each should resolve to the correct private IP.

---

## Step 5: Build and Upload Application Artifact

### A) Update Application Configuration

On your local M4 MacBook Air:

```bash
cd /path/to/your/vprofile-project
cp PROJECT_3/config/application.properties.template src/main/resources/application.properties
```

Edit `src/main/resources/application.properties`:

```properties
jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
jdbc.username=admin
jdbc.password=admin123

memcached.active.host=mc01.vprofile.in
memcached.active.port=11211

rabbitmq.address=rmq01.vprofile.in
rabbitmq.port=5672
rabbitmq.username=test
rabbitmq.password=test
```

### B) Build the Application

```bash
mvn clean install

# WAR file will be at: target/vprofile-v2.war
```

### C) Create S3 Bucket

```bash
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create bucket with unique name
BUCKET_NAME="vprofile-artifact-storage-$ACCOUNT_ID"
aws s3 mb s3://$BUCKET_NAME

# Upload WAR file
aws s3 cp target/vprofile-v2.war s3://$BUCKET_NAME/vprofile-v2.war

# Verify upload
aws s3 ls s3://$BUCKET_NAME/
```

---

## Step 6: Create IAM Role for EC2 S3 Access

**Purpose:** Allow Tomcat EC2 to download WAR from S3.

### A) Create IAM Role

1. Go to **IAM** → **Roles**
2. Click **Create role**
3. **Trusted entity type:** AWS service
4. **Use case:** EC2
5. Click **Next**
6. **Permissions:** Create inline policy
7. Click **JSON** tab
8. Paste contents of `PROJECT_3/iam/s3-access-policy.json` (update bucket name)
9. Name policy: `vprofile-s3-access`
10. Click **Next**
11. **Role name:** `vprofile-ec2-s3-role`
12. Click **Create role**

---

## Step 7: Launch Tomcat Application Instance

**AMI:** Ubuntu Server 22.04 LTS

**Instance Type:** t2.micro

**Network Settings:**
- VPC: Default
- Subnet: Any private subnet (or public)
- Auto assign Public IP: Enable

**IAM Role:** `vprofile-ec2-s3-role` (attach the role you just created)

**Security Group:** `vprofile-app-sg`

**Key Pair:** `vprofile-key`

**User Data:** Copy contents of `PROJECT_3/userdata/tomcat.sh`

**Name tag:** `vprofile-tomcat`

**Launch instance and wait 5 minutes.**

---

## Step 8: Deploy Application to Tomcat

SSH into the Tomcat instance:

```bash
ssh -i ~/.ssh/vprofile-key.pem ubuntu@<tomcat-public-ip>
```

Download and deploy the WAR:

```bash
# Switch to root
sudo su -

# Verify Tomcat is running
systemctl status tomcat9

# Get your bucket name
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="vprofile-artifact-storage-$ACCOUNT_ID"

# Download WAR from S3
aws s3 cp s3://$BUCKET_NAME/vprofile-v2.war /tmp/ROOT.war

# Deploy to Tomcat
cp /tmp/ROOT.war /var/lib/tomcat9/webapps/ROOT.war

# Set permissions
chown tomcat9:tomcat9 /var/lib/tomcat9/webapps/ROOT.war

# Restart Tomcat
systemctl restart tomcat9

# Watch logs
tail -f /var/log/tomcat9/catalina.out

# Wait for "Server startup in [XXXX] milliseconds"
# Press Ctrl+C to exit log view

# Test locally
curl http://localhost:8080

# Should see HTML response
```

If you see errors in the logs, check:
- Backend services are running
- Route 53 DNS is resolving correctly
- Security groups allow traffic

---

## Step 9: Create Application Load Balancer

### A) Create Target Group

1. Go to **EC2** → **Target Groups**
2. Click **Create target group**
3. Settings:
   - **Target type:** Instances
   - **Target group name:** `vprofile-tg`
   - **Protocol:** HTTP
   - **Port:** 8080
   - **VPC:** Default VPC
   - **Health check path:** /login
   - **Health check interval:** 30 seconds
4. Click **Next**
5. **Register targets:**
   - Select `vprofile-tomcat` instance
   - Port: 8080
   - Click **Include as pending below**
6. Click **Create target group**

Wait 1-2 minutes, then check target health:
- Should show "healthy" status

### B) Create Application Load Balancer

1. Go to **EC2** → **Load Balancers**
2. Click **Create load balancer**
3. Select **Application Load Balancer**
4. Settings:
   - **Name:** `vprofile-alb`
   - **Scheme:** Internet-facing
   - **IP address type:** IPv4
5. **Network mapping:**
   - VPC: Default VPC
   - Subnets: Select at least 2 availability zones
6. **Security groups:**
   - Select `vprofile-alb-sg`
7. **Listeners:**
   - Protocol: HTTP
   - Port: 80
   - Default action: Forward to `vprofile-tg`
8. Click **Create load balancer**

**Get ALB DNS name:**

```bash
aws elbv2 describe-load-balancers \
  --names vprofile-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```

**Test your application:**

Open browser: `http://<alb-dns-name>`

You should see the vprofile login page.

**Login with:**
- Username: `admin_vp`
- Password: `admin_vp`

---

## Step 10: Create Auto Scaling Group (Optional)

**Purpose:** Automatically scale Tomcat instances based on load.

### A) Create AMI from Tomcat Instance

1. Go to **EC2** → **Instances**
2. Select `vprofile-tomcat`
3. **Actions** → **Image and templates** → **Create image**
4. Settings:
   - **Image name:** `vprofile-app-ami`
   - **Image description:** Tomcat with vprofile app deployed
5. Click **Create image**
6. Wait 5-10 minutes for AMI to be available

### B) Create Launch Template

1. Go to **EC2** → **Launch Templates**
2. Click **Create launch template**
3. Settings:
   - **Name:** `vprofile-app-lt`
   - **Description:** Launch template for vprofile app
4. **Application and OS Images:**
   - **My AMIs**
   - Select `vprofile-app-ami`
5. **Instance type:** t2.micro
6. **Key pair:** `vprofile-key`
7. **Network settings:**
   - Security groups: `vprofile-app-sg`
8. **Advanced details:**
   - IAM instance profile: `vprofile-ec2-s3-role`
9. Click **Create launch template**

### C) Create Auto Scaling Group

1. Go to **EC2** → **Auto Scaling Groups**
2. Click **Create Auto Scaling group**
3. **Name:** `vprofile-app-asg`
4. **Launch template:** Select `vprofile-app-lt`
5. Click **Next**
6. **Network:**
   - VPC: Default
   - Subnets: Select private subnets (or same as Tomcat)
7. Click **Next**
8. **Load balancing:**
   - Attach to existing load balancer
   - Choose from target groups
   - Select `vprofile-tg`
9. **Health checks:**
   - ELB health checks: Enable
10. Click **Next**
11. **Group size:**
    - Desired: 2
    - Minimum: 1
    - Maximum: 4
12. **Scaling policies:**
    - Target tracking
    - Metric: Average CPU utilization
    - Target value: 50
13. Click **Next** → **Next** → **Create Auto Scaling group**

**Verify:**
- Auto Scaling should launch 2 new instances
- They should register with the target group
- ALB should distribute traffic across all healthy instances

**Now you can terminate the original Tomcat instance:**

```bash
aws ec2 terminate-instances --instance-ids <original-tomcat-instance-id>
```

---

## Step 11 (Optional): Add Custom Domain with Route 53

**If you don't have a domain, skip this step and use the ALB DNS name.**

### A) Register Domain in Route 53

1. Go to **Route 53** → **Registered domains**
2. Click **Register domain**
3. Search for available domain: `yourname-devops.com`
4. Add to cart and complete registration
5. Wait for email verification (can take up to 3 days for approval)

### B) Create Public Hosted Zone

Route 53 automatically creates a hosted zone when you register a domain.

1. Go to **Route 53** → **Hosted zones**
2. Find your domain
3. Click **Create record**
4. Settings:
   - Record name: Leave blank (apex) or use `app`
   - Record type: A
   - Alias: Yes
   - Route traffic to: Application and Classic Load Balancer
   - Region: us-east-1
   - Load balancer: Select `vprofile-alb`
5. Click **Create records**

**Test:** `http://yourname-devops.com` or `http://app.yourname-devops.com`

### C) Add SSL Certificate (Optional)

1. Go to **ACM (Certificate Manager)**
2. Click **Request certificate**
3. **Certificate type:** Public
4. **Domain names:** `yourname-devops.com` and `*.yourname-devops.com`
5. **Validation:** DNS validation
6. Click **Request**
7. Click **Create records in Route 53** (automatic)
8. Wait 5-10 minutes for validation

### D) Add HTTPS Listener to ALB

1. Go to **EC2** → **Load Balancers**
2. Select `vprofile-alb`
3. **Listeners** tab → **Add listener**
4. Settings:
   - Protocol: HTTPS
   - Port: 443
   - Default action: Forward to `vprofile-tg`
   - SSL certificate: Select your ACM cert
5. Click **Add**

**Test:** `https://yourname-devops.com`

---

## Validation Checklist

Run the validation script:

```bash
cd PROJECT_3/scripts
chmod +x validate-stack.sh
./validate-stack.sh
```

**Manual checks:**

- [ ] All 3 backend instances running
- [ ] Tomcat instances running (via ASG)
- [ ] Route 53 private zone has 3 A records
- [ ] ALB is active
- [ ] Target group shows healthy targets
- [ ] Application loads via ALB DNS
- [ ] Can login to app (admin_vp / admin_vp)
- [ ] User profile pages load (data from MySQL)
- [ ] No errors in Tomcat logs

---

## Cost Tracking

**Daily costs while running:**
- 3 backend instances (t2.micro): $3.60/day
- 2 app instances (t2.micro via ASG): $2.40/day
- Application Load Balancer: $5.00/day
- Data transfer: ~$1.00/day
- **Total: ~$12/day**

**With $190 credits = ~15 days of runtime**

**Track your spending:**

```bash
aws ce get-cost-and-usage \
  --time-period Start=2025-02-01,End=2025-02-15 \
  --granularity DAILY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE
```

---

## Cleanup

**IMPORTANT:** Delete all resources when done to stop charges.

Run the cleanup script:

```bash
cd PROJECT_3/scripts
chmod +x cleanup.sh
./cleanup.sh
```

Or manually delete in this order:

1. Auto Scaling Group
2. Launch Template
3. AMI and snapshots
4. Application Load Balancer (wait for deletion)
5. Target Group
6. EC2 instances (all 4)
7. Security groups (delete in reverse order: ALB → App → Backend)
8. Route 53 records (delete A records first)
9. Route 53 hosted zone (private)
10. S3 bucket (empty first, then delete)
11. IAM role and policies
12. Key pair

See `CLEANUP_CHECKLIST.md` for detailed steps.

---

## Troubleshooting

### Issue: Backend services not starting

**Check user data logs:**
```bash
ssh -i ~/.ssh/vprofile-key.pem ubuntu@<instance-ip>
sudo cat /var/log/cloud-init-output.log
```

Look for errors during package installation.

### Issue: Tomcat can't connect to backend

**Check DNS resolution:**
```bash
nslookup db01.vprofile.in
nslookup mc01.vprofile.in
nslookup rmq01.vprofile.in
```

**Check security groups:**
- Backend SG allows port 3306 from App SG
- Backend SG allows port 11211 from App SG
- Backend SG allows port 5672 from App SG

### Issue: ALB returns 502 Bad Gateway

**Check target health:**
```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

If unhealthy:
- Verify Tomcat is running on port 8080
- Check App SG allows port 8080 from ALB SG
- Check health check path is correct (/login)

### Issue: Can't SSH to instances

**Check security groups:**
- SSH rule allows port 22 from your IP
- Your IP hasn't changed

**Verify key permissions:**
```bash
chmod 400 ~/.ssh/vprofile-key.pem
```

### Issue: S3 download fails on Tomcat

**Check IAM role:**
- Role is attached to instance
- Policy allows s3:GetObject on your bucket

**Test from instance:**
```bash
aws s3 ls s3://your-bucket-name/
```

---

## Next Steps

**PROJECT 4:** Refactor to AWS managed services
- RDS instead of EC2 MySQL
- ElastiCache instead of EC2 Memcached
- Amazon MQ instead of EC2 RabbitMQ
- CloudFront for CDN

**PROJECT 5:** Add CI/CD
- Jenkins pipeline
- GitHub integration
- Automated deployments

---

## Files in This Project

```
PROJECT_3/
├── README.md (this file)
├── userdata/
│   ├── mysql.sh (MySQL setup for Ubuntu 22.04)
│   ├── memcache.sh (Memcached setup)
│   ├── rabbitmq.sh (RabbitMQ setup)
│   └── tomcat.sh (Tomcat setup)
├── config/
│   └── application.properties.template (App config)
├── iam/
│   ├── s3-access-policy.json (S3 permissions)
│   └── ec2-trust-policy.json (EC2 trust policy)
├── scripts/
│   ├── validate-stack.sh (Validation script)
│   └── cleanup.sh (Cleanup script)
└── CLEANUP_CHECKLIST.md (Detailed cleanup steps)
```

---

## Learning Outcomes

You now understand:
- Multi tier application deployment on AWS
- Security group design and network isolation
- Application Load Balancer configuration
- Route 53 private hosted zones for internal DNS
- IAM roles for EC2 service access
- S3 artifact storage
- Auto Scaling Groups for high availability
- User data scripts for EC2 bootstrapping
- SSL/TLS with ACM
- AWS cost management

---

## Resources

- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [AWS ELB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/)
- [Route 53 Documentation](https://docs.aws.amazon.com/route53/)
- [Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)
