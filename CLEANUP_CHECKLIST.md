# PROJECT 3: Cleanup Checklist

Use this checklist to delete all AWS resources and stop charges.

**Estimated time:** 15-20 minutes

---

## Automated Cleanup (Recommended)

Run the cleanup script:

```bash
cd PROJECT_3/scripts
chmod +x cleanup.sh
./cleanup.sh
```

The script will delete everything automatically.

---

## Manual Cleanup (If Script Fails)

Follow these steps in order to avoid dependency errors.

### Step 1: Delete Auto Scaling Group

**Console:**
1. EC2 → Auto Scaling Groups
2. Select `vprofile-app-asg`
3. Actions → Delete
4. Confirm deletion
5. Wait for instances to terminate

**CLI:**
```bash
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name vprofile-app-asg \
  --force-delete
```

**Verify:**
- [ ] ASG deleted
- [ ] ASG instances terminated

---

### Step 2: Delete Launch Template

**Console:**
1. EC2 → Launch Templates
2. Select `vprofile-app-lt`
3. Actions → Delete template
4. Confirm

**CLI:**
```bash
aws ec2 delete-launch-template \
  --launch-template-name vprofile-app-lt
```

**Verify:**
- [ ] Launch template deleted

---

### Step 3: Deregister AMI and Delete Snapshots

**Console:**
1. EC2 → AMIs
2. Select `vprofile-app-ami`
3. Actions → Deregister AMI
4. Confirm
5. Go to **Snapshots**
6. Find snapshots associated with the AMI
7. Select and delete

**CLI:**
```bash
# Get AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners self \
  --filters "Name=name,Values=vprofile-app-ami" \
  --query 'Images[0].ImageId' \
  --output text)

# Deregister AMI
aws ec2 deregister-image --image-id $AMI_ID

# Get snapshot IDs
SNAPSHOT_IDS=$(aws ec2 describe-snapshots \
  --owner-ids self \
  --filters "Name=description,Values=*$AMI_ID*" \
  --query 'Snapshots[].SnapshotId' \
  --output text)

# Delete snapshots
for SNAP in $SNAPSHOT_IDS; do
  aws ec2 delete-snapshot --snapshot-id $SNAP
done
```

**Verify:**
- [ ] AMI deregistered
- [ ] Snapshots deleted

---

### Step 4: Delete Application Load Balancer

**Console:**
1. EC2 → Load Balancers
2. Select `vprofile-alb`
3. Actions → Delete
4. Confirm
5. **Wait 2-3 minutes for deletion to complete**

**CLI:**
```bash
# Get ALB ARN
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --names vprofile-alb \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Delete ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# Wait for deletion
aws elbv2 wait load-balancers-deleted --load-balancer-arns $ALB_ARN
```

**Verify:**
- [ ] ALB deleted (not in list)
- [ ] Waited for deletion to complete

---

### Step 5: Delete Target Group

**Console:**
1. EC2 → Target Groups
2. Select `vprofile-tg`
3. Actions → Delete
4. Confirm

**CLI:**
```bash
# Get target group ARN
TG_ARN=$(aws elbv2 describe-target-groups \
  --names vprofile-tg \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Delete target group
aws elbv2 delete-target-group --target-group-arn $TG_ARN
```

**Verify:**
- [ ] Target group deleted

---

### Step 6: Terminate EC2 Instances

**Console:**
1. EC2 → Instances
2. Select all instances with tag Project=vprofile:
   - vprofile-mysql
   - vprofile-memcached
   - vprofile-rabbitmq
   - vprofile-tomcat (if not already terminated by ASG)
3. Instance state → Terminate instance
4. Confirm
5. **Wait 2-3 minutes for termination**

**CLI:**
```bash
# Get all instance IDs
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=vprofile" \
            "Name=instance-state-name,Values=running,stopped" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text)

# Terminate instances
aws ec2 terminate-instances --instance-ids $INSTANCE_IDS

# Wait for termination
aws ec2 wait instance-terminated --instance-ids $INSTANCE_IDS
```

**Verify:**
- [ ] All instances terminated
- [ ] Waited for termination to complete

---

### Step 7: Delete Security Groups

**IMPORTANT:** Delete in this order to avoid dependency errors.

**Console:**
1. EC2 → Security Groups
2. Delete in order:
   a. `vprofile-alb-sg`
   b. `vprofile-app-sg`
   c. `vprofile-backend-sg`
3. For each: Select → Actions → Delete → Confirm

**CLI:**
```bash
# Delete ALB security group
aws ec2 delete-security-group --group-name vprofile-alb-sg

# Delete App security group
aws ec2 delete-security-group --group-name vprofile-app-sg

# Delete Backend security group
aws ec2 delete-security-group --group-name vprofile-backend-sg
```

**If you get dependency errors:**
- Wait longer for instances to fully terminate
- Check for any remaining ENIs (network interfaces)

**Verify:**
- [ ] vprofile-alb-sg deleted
- [ ] vprofile-app-sg deleted
- [ ] vprofile-backend-sg deleted

---

### Step 8: Delete Route 53 Records

**Console:**
1. Route 53 → Hosted zones
2. Select `vprofile.in`
3. Select each A record (db01, mc01, rmq01)
4. Delete → Confirm
5. **Do NOT delete NS and SOA records yet**

**CLI:**
```bash
# Get hosted zone ID
ZONE_ID=$(aws route53 list-hosted-zones \
  --query 'HostedZones[?Name==`vprofile.in.`].Id' \
  --output text | cut -d'/' -f3)

# Delete each A record
for RECORD in db01.vprofile.in mc01.vprofile.in rmq01.vprofile.in; do
  # Get record value
  VALUE=$(aws route53 list-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --query "ResourceRecordSets[?Name=='$RECORD.'].ResourceRecords[0].Value" \
    --output text)
  
  # Delete record
  aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch "{
      \"Changes\": [{
        \"Action\": \"DELETE\",
        \"ResourceRecordSet\": {
          \"Name\": \"$RECORD.\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$VALUE\"}]
        }
      }]
    }"
done
```

**Verify:**
- [ ] db01.vprofile.in deleted
- [ ] mc01.vprofile.in deleted
- [ ] rmq01.vprofile.in deleted

---

### Step 9: Delete Route 53 Hosted Zone

**Console:**
1. Route 53 → Hosted zones
2. Select `vprofile.in`
3. Delete hosted zone → Confirm

**CLI:**
```bash
# Delete hosted zone
aws route53 delete-hosted-zone --id $ZONE_ID
```

**Verify:**
- [ ] vprofile.in hosted zone deleted

---

### Step 10: Empty and Delete S3 Bucket

**Console:**
1. S3 → Buckets
2. Select `vprofile-artifact-storage-<account-id>`
3. Empty bucket → Confirm
4. Delete bucket → Confirm

**CLI:**
```bash
# Get bucket name
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="vprofile-artifact-storage-$ACCOUNT_ID"

# Empty bucket
aws s3 rm s3://$BUCKET_NAME --recursive

# Delete bucket
aws s3 rb s3://$BUCKET_NAME
```

**Verify:**
- [ ] Bucket emptied
- [ ] Bucket deleted

---

### Step 11: Delete IAM Role and Policies

**Console:**
1. IAM → Roles
2. Select `vprofile-ec2-s3-role`
3. Delete → Confirm

**CLI:**
```bash
# Detach policy from role
aws iam delete-role-policy \
  --role-name vprofile-ec2-s3-role \
  --policy-name vprofile-s3-access

# Delete instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name vprofile-ec2-s3-role \
  --role-name vprofile-ec2-s3-role

aws iam delete-instance-profile \
  --instance-profile-name vprofile-ec2-s3-role

# Delete role
aws iam delete-role --role-name vprofile-ec2-s3-role
```

**Verify:**
- [ ] IAM role deleted
- [ ] Inline policy deleted

---

### Step 12: Delete Key Pair

**Console:**
1. EC2 → Key Pairs
2. Select `vprofile-key`
3. Actions → Delete → Confirm
4. **Also delete local copy:**

```bash
rm ~/.ssh/vprofile-key.pem
```

**CLI:**
```bash
# Delete from AWS
aws ec2 delete-key-pair --key-name vprofile-key

# Delete local copy
rm ~/.ssh/vprofile-key.pem
```

**Verify:**
- [ ] Key pair deleted from AWS
- [ ] Local .pem file deleted

---

### Step 13 (Optional): Delete ACM Certificate

If you created an SSL certificate:

**Console:**
1. Certificate Manager → Certificates
2. Select your certificate
3. Actions → Delete → Confirm

**CLI:**
```bash
# List certificates
aws acm list-certificates

# Delete certificate (use ARN from list)
aws acm delete-certificate --certificate-arn <certificate-arn>
```

**Verify:**
- [ ] ACM certificate deleted

---

### Step 14 (Optional): Delete Route 53 Domain

If you registered a domain and no longer need it:

**Console:**
1. Route 53 → Registered domains
2. Select domain
3. Actions → Delete domain
4. Follow prompts

**Note:** You may be charged for the full registration period even if deleted early.

---

## Final Verification

Run these commands to verify everything is deleted:

```bash
# Check for running instances
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=vprofile" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}' \
  --output table

# Check for load balancers
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?LoadBalancerName==`vprofile-alb`]' \
  --output table

# Check for Auto Scaling groups
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[?AutoScalingGroupName==`vprofile-app-asg`]' \
  --output table

# Check for S3 buckets
aws s3 ls | grep vprofile

# Check for security groups
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=vprofile*" \
  --query 'SecurityGroups[].GroupName' \
  --output table
```

**All should return empty or "None".**

---

## Cleanup Complete Checklist

- [ ] Auto Scaling Group deleted
- [ ] Launch Template deleted
- [ ] AMI deregistered
- [ ] Snapshots deleted
- [ ] Application Load Balancer deleted
- [ ] Target Group deleted
- [ ] All EC2 instances terminated
- [ ] All security groups deleted (3)
- [ ] Route 53 A records deleted (3)
- [ ] Route 53 private hosted zone deleted
- [ ] S3 bucket emptied and deleted
- [ ] IAM role and policies deleted
- [ ] Key pair deleted (AWS and local)
- [ ] ACM certificate deleted (if created)
- [ ] Verified all resources deleted

---

## Cost After Cleanup

**Expected charges:** $0/day

**Possible residual charges:**
- EBS snapshots if not deleted: ~$0.05/GB/month
- Route 53 hosted zone if public domain: $0.50/month
- ACM certificates: Free

**Final check:** Wait 24 hours, then check AWS billing dashboard.

---

## If You Get Stuck

**Common issues:**

**Can't delete security group:**
- Wait longer for instances to terminate
- Check for ENIs still using the SG

**Can't delete hosted zone:**
- Delete all A records first
- Leave NS and SOA records (deleted automatically)

**Can't delete S3 bucket:**
- Empty bucket first
- Check for versioned objects

**Can't delete IAM role:**
- Detach all policies first
- Remove from instance profile first

**Need help?**
- Check AWS Console for error messages
- Review CloudTrail logs for deletion events
- Contact AWS support if charges continue
