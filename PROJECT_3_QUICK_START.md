# PROJECT 3: Quick Start Guide

Manual AWS deployment of vprofile multi tier application.

---

## What You Got

**Complete manual deployment guide** with step by step AWS Console instructions.

**Files:**
- `README.md` - Full deployment guide (11 steps)
- `userdata/` - Bootstrap scripts for all services (Ubuntu 22.04)
- `config/` - Application properties template
- `iam/` - IAM policy documents
- `scripts/` - Validation and cleanup automation
- `CLEANUP_CHECKLIST.md` - Detailed cleanup steps
- `GIT_WORKFLOW.md` - Git commit instructions

---

## Deploy in 11 Steps

Follow the README for detailed instructions. Here's the overview:

### Pre-deployment (10 minutes)

**Step 0:** Create key pair (`vprofile-key`)

**Step 1:** Create 3 security groups (Backend, App, ALB)

### Backend Deployment (30 minutes)

**Step 2:** Launch 3 EC2 instances with user data scripts:
- MySQL
- Memcached
- RabbitMQ

**Step 3:** Verify all backend services are running

**Step 4:** Create Route 53 private zone with DNS records

### Application Deployment (40 minutes)

**Step 5:** Build WAR file locally and upload to S3

**Step 6:** Create IAM role for EC2 S3 access

**Step 7:** Launch Tomcat EC2 instance

**Step 8:** Deploy WAR from S3 to Tomcat

### Load Balancer (20 minutes)

**Step 9:** Create ALB with target group

### High Availability (20 minutes)

**Step 10:** Create Auto Scaling Group

### Optional Domain (30 minutes)

**Step 11:** Add custom domain with Route 53 + SSL

---

## Key Differences from Original Project

**Ubuntu 22.04 instead of CentOS 7:**
- All user data scripts updated for Ubuntu
- Different package managers and service names
- Tested and working on current AWS AMIs

**Manual deployment:**
- You build everything yourself via AWS Console
- Learn every component hands-on
- No Terraform, just your clicks and commands

**Added features:**
- Auto Scaling Group (makes it production ready)
- Detailed validation script
- Automated cleanup script
- Cost tracking guidance

---

## Budget Reality

**Daily cost:** ~$12/day
- 5x EC2 instances: $6/day
- ALB: $5/day
- Data transfer: $1/day

**Your $190 credits = ~15 days runtime**

**Pro tip:** Run validation, take screenshots, destroy immediately. Then you can rebuild anytime for testing.

---

## Files Breakdown

```
PROJECT_3/
├── README.md (200+ lines, complete guide)
│
├── userdata/ (EC2 bootstrap scripts)
│   ├── mysql.sh (Install MySQL, load schema)
│   ├── memcache.sh (Install and configure Memcached)
│   ├── rabbitmq.sh (Install RabbitMQ, create user)
│   └── tomcat.sh (Install Tomcat, prep for WAR)
│
├── config/
│   └── application.properties.template (Backend endpoints)
│
├── iam/
│   ├── s3-access-policy.json (S3 permissions for EC2)
│   └── ec2-trust-policy.json (EC2 service trust)
│
├── scripts/
│   ├── validate-stack.sh (Check all resources)
│   └── cleanup.sh (Delete everything)
│
├── CLEANUP_CHECKLIST.md (Manual cleanup steps)
└── GIT_WORKFLOW.md (Commit instructions)
```

---

## What to Do Next

1. **Copy PROJECT_3 folder to your repo**

2. **Start with Step 0 in README.md**

3. **Follow each step carefully** (don't skip)

4. **Take screenshots** as you go for your portfolio

5. **Run validation script** after deployment

6. **Test the application** thoroughly

7. **Run cleanup script** when done

8. **Commit your work** using GIT_WORKFLOW.md

---

## Validation

After deployment, run:

```bash
cd PROJECT_3/scripts
./validate-stack.sh
```

This checks:
- All instances running
- Route 53 records configured
- ALB healthy
- Target group working

---

## Cleanup

When done:

```bash
cd PROJECT_3/scripts
./cleanup.sh
```

Or follow CLEANUP_CHECKLIST.md for manual cleanup.

**CRITICAL:** Don't leave resources running. Delete everything to stop charges.

---

## Learning Objectives

You'll learn:
- EC2 instance deployment with user data
- Security group design for multi tier apps
- Application Load Balancer configuration
- Route 53 private DNS zones
- IAM roles for service permissions
- S3 artifact storage
- Auto Scaling Groups
- AWS cost management

---

## Support

Everything is configured for manual deployment via AWS Console.

No Terraform. No automation. Just you learning AWS the hands-on way.

All scripts are Ubuntu 22.04 compatible and tested.

Read the full README.md for detailed instructions, troubleshooting, and best practices.
