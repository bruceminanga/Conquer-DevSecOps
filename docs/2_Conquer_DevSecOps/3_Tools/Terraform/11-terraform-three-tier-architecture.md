# 🏛️ Architecting Resilience: A Production 3-Tier AWS Runbook

*Why the classic Web-App-Database pattern remains the enterprise gold standard, and an end-to-end deployment runbook you can actually execute.*

---

Microservices, serverless, and edge runtimes dominate architecture discussions. But when external compliance auditors arrive for **SOC 2 Type II**, **HIPAA**, or **PCI-DSS**, or when sudden production traffic surges 100x, the architectural design that holds the line predictably without ballooning budgets is still the classic **3-Tier Architecture**.

Below is an engineering analysis of why this model excels, followed by an end-to-end, CLI-driven AWS deployment runbook.

---

## 🧭 Part 1: Why 3-Tier Still Wins

A 3-tier architecture physically segregates the stack into three network planes:

```text
               [ Internet ]
                    │
                    ▼
┌───────────────────────────────────────┐
│ 1. Web Tier: ALB + AWS WAF            │  Public Subnets
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ 2. App Tier: Auto Scaling Group (ASG) │  Private Subnets
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ 3. Data Tier: Multi-AZ RDS Engine     │  Isolated Subnets
└───────────────────────────────────────┘
```

---

### 1. Network Isolation and Defense-in-Depth

In a flat network topology, a Remote Code Execution (RCE) vulnerability in an exposed web server places the attacker directly adjacent to your database layer. The 3-tier model mitigates this blast radius:

* **Web Tier:** The sole public ingress point. Terminates TLS, filters malicious payloads via AWS WAF, and forwards requests internally.
* **App Tier:** Deployed in private subnets with no public IP routes. Accepts traffic strictly from the Application Load Balancer's security group.
* **Database Tier:** Deployed in isolated subnets with no route to an Internet Gateway or NAT Gateway. Listens on PostgreSQL port `5432` strictly for connections initiated by the application tier's security group.

> Compromising the front door does not yield direct access to the persistent data plane.

---

### 2. Independent Scalability

Each layer encounters fundamentally distinct performance bottlenecks:

| Tier          | Primary Bottlenecks                                         | Scaling Strategy                                         |
| :------------ | :---------------------------------------------------------- | :------------------------------------------------------- |
| **Web Tier**  | Concurrent TCP connections, TLS handshakes, network I/O     | Managed automatically by ALB elasticity                  |
| **App Tier**  | CPU utilization, thread contention, application heap memory | Horizontal Auto Scaling (EC2 / ASG)                      |
| **Data Tier** | Disk I/O (IOPS), buffer cache, connection pools             | Vertical scaling, storage autoscaling, and Read Replicas |

This decoupling prevents over-provisioning an entire server fleet to resolve a bottleneck in a single subsystem.

---

### 3. Decoupled Failure Domains

* A memory leak in application code cannot exhaust host RAM on the database node or trigger the Linux Out-Of-Memory (OOM) killer on the persistence engine.
* Application deployments and blue/green rollouts execute independently without altering persistent database state.
* Frontend, backend, and platform/DBA teams maintain clear boundaries of operational ownership.

---

### 4. Managed Persistence Engine (Amazon RDS / Aurora)

Deploying a managed database tier provides enterprise guarantees:

* **Synchronous Multi-AZ Standby:** Automated failover completes in 60 to 120 seconds with zero manual DNS intervention.
* **Continuous Backups:** Automated transaction log replication enables point-in-time recovery (PITR).
* **Automated Maintenance:** OS patching occurs within managed maintenance windows.
* **Storage Autoscaling:** Elastic disk allocation prevents outages caused by full storage volumes.

---

### 5. Audit & Compliance Readiness

Security frameworks (PCI-DSS Req 1.3, HIPAA, SOC 2) mandate that sensitive and regulated data cannot reside on hosts directly reachable from the public internet. Physical subnet segmentation paired with security-group chaining provides clear technical proof for audit controls.

---

## 🛠️ Part 2: The Deployment Runbook

* **Reference Target:** AWS CLI v2, Region `us-east-1`.
* **Execution Strategy:** Execute phases sequentially within a single terminal session to preserve exported environment variables.

> [!WARNING]
> **Cost Warning:** High-availability multi-AZ architectures with redundant NAT Gateways and Multi-AZ RDS incur hourly costs. Tear down sandbox environments promptly after testing.

---

### 📍 Phase 1: Network Topology

Deploy across two Availability Zones (`us-east-1a`, `us-east-1b`) spanning six subnets:

* **VPC CIDR:** `10.0.0.0/16`
* **Web Tier (Public):** `10.0.1.0/24` (AZ-1), `10.0.2.0/24` (AZ-2) $\rightarrow$ Default route to Internet Gateway.
* **App Tier (Private):** `10.0.10.0/24` (AZ-1), `10.0.11.0/24` (AZ-2) $\rightarrow$ Egress route to AZ-local NAT Gateway.
* **Database Tier (Isolated):** `10.0.20.0/24` (AZ-1), `10.0.21.0/24` (AZ-2) $\rightarrow$ Local VPC route only (`10.0.0.0/16`).

```bash
# 1. Base VPC & Internet Gateway
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-3tier-vpc}]' \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support '{"Value":true}'

IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-3tier-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# 2. Subnet Helper Function & Allocation
mk_subnet() {
  aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block $1 \
    --availability-zone $2 \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$3}]" \
    --query 'Subnet.SubnetId' --output text
}

SUBNET_WEB_AZ1=$(mk_subnet 10.0.1.0/24  us-east-1a subnet-web-az1)
SUBNET_WEB_AZ2=$(mk_subnet 10.0.2.0/24  us-east-1b subnet-web-az2)
SUBNET_APP_AZ1=$(mk_subnet 10.0.10.0/24 us-east-1a subnet-app-az1)
SUBNET_APP_AZ2=$(mk_subnet 10.0.11.0/24 us-east-1b subnet-app-az2)
SUBNET_DB_AZ1=$(mk_subnet  10.0.20.0/24 us-east-1a subnet-db-az1)
SUBNET_DB_AZ2=$(mk_subnet  10.0.21.0/24 us-east-1b subnet-db-az2)

# Enable Public IP Assignment for Web Subnets
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_WEB_AZ1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_WEB_AZ2 --map-public-ip-on-launch

# 3. High-Availability NAT Gateways (One per AZ)
EIP1=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
EIP2=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)

NAT1=$(aws ec2 create-nat-gateway --subnet-id $SUBNET_WEB_AZ1 --allocation-id $EIP1 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-az1}]' \
  --query NatGateway.NatGatewayId --output text)
NAT2=$(aws ec2 create-nat-gateway --subnet-id $SUBNET_WEB_AZ2 --allocation-id $EIP2 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-az2}]' \
  --query NatGateway.NatGatewayId --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT1 $NAT2

# 4. Routing Architecture
# Public Tier (To IGW)
RT_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $SUBNET_WEB_AZ1
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $SUBNET_WEB_AZ2

# App Tier AZ-1 (To NAT1)
RT_APP1=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT_APP1 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT1
aws ec2 associate-route-table --route-table-id $RT_APP1 --subnet-id $SUBNET_APP_AZ1

# App Tier AZ-2 (To NAT2)
RT_APP2=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT_APP2 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT2
aws ec2 associate-route-table --route-table-id $RT_APP2 --subnet-id $SUBNET_APP_AZ2

# Isolated DB Tier (Local Route Only)
RT_DB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 associate-route-table --route-table-id $RT_DB --subnet-id $SUBNET_DB_AZ1
aws ec2 associate-route-table --route-table-id $RT_DB --subnet-id $SUBNET_DB_AZ2
```

---

### 📍 Phase 2: Chained Security Group Perimeter

Chain security groups by ID to maintain dynamic trust boundaries across scaling events:

$$\text{Internet} \xrightarrow[\text{Port 80/443}]{} \mathbf{\text{ALB\_SG}} \xrightarrow[\text{Port 8080}]{} \mathbf{\text{APP\_SG}} \xrightarrow[\text{Port 5432}]{} \mathbf{\text{DB\_SG}}$$

```bash
# 1. Create Security Groups
ALB_SG=$(aws ec2 create-security-group --group-name prod-alb-sg \
  --description "Public ingress for ALB" --vpc-id $VPC_ID --query GroupId --output text)
APP_SG=$(aws ec2 create-security-group --group-name prod-app-sg \
  --description "Ingress from ALB only" --vpc-id $VPC_ID --query GroupId --output text)
DB_SG=$(aws ec2 create-security-group --group-name prod-db-sg \
  --description "Ingress from app tier only" --vpc-id $VPC_ID --query GroupId --output text)

# 2. Ingress Chaining
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80  --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 8080 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG  --protocol tcp --port 5432 --source-group $APP_SG

# 3. Egress Hardening (DB Never Initiates Outbound Connections)
aws ec2 revoke-security-group-egress --group-id $DB_SG \
  --ip-permissions 'IpProtocol=-1,IpRanges=[{CidrIp=0.0.0.0/0}]'
```

---

### 📍 Phase 3: The Persistence Tier (Multi-AZ RDS)

Provision a hardened, managed PostgreSQL database:

```bash
# 1. DB Subnet Group across isolated subnets
aws rds create-db-subnet-group \
  --db-subnet-group-name prod-db-subnet-group \
  --db-subnet-group-description "Isolated subnets for Multi-AZ RDS" \
  --subnet-ids $SUBNET_DB_AZ1 $SUBNET_DB_AZ2

# 2. Resolve default engine version
PG_VERSION=$(aws rds describe-db-engine-versions --engine postgres --default-only \
  --query 'DBEngineVersions[0].EngineVersion' --output text)

# 3. Provision Multi-AZ PostgreSQL Engine
aws rds create-db-instance \
  --db-instance-identifier prod-database \
  --db-name appdb \
  --engine postgres --engine-version $PG_VERSION \
  --db-instance-class db.m6i.large \
  --allocated-storage 100 --max-allocated-storage 500 --storage-type gp3 \
  --master-username dbadmin \
  --manage-master-user-password \
  --vpc-security-group-ids $DB_SG \
  --db-subnet-group-name prod-db-subnet-group \
  --multi-az --no-publicly-accessible --storage-encrypted \
  --backup-retention-period 30 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "Sun:04:30-05:30" \
  --enable-performance-insights \
  --enable-cloudwatch-logs-exports postgresql \
  --copy-tags-to-snapshot \
  --deletion-protection

# Block until database initialization finishes
aws rds wait db-instance-available --db-instance-identifier prod-database

# Extract database endpoint and managed secret ARN
DB_HOST=$(aws rds describe-db-instances --db-instance-identifier prod-database \
  --query 'DBInstances[0].Endpoint.Address' --output text)
DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier prod-database \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)
```

> [!TIP]
> The `--manage-master-user-password` flag delegates credential generation and rotation to **AWS Secrets Manager**, ensuring that database passwords never appear in shell histories, process listings, or code.

---

### 📍 Phase 4: The Application Tier

#### 4.1 IAM Instance Profile (Zero Bastions, Zero SSH Keys)
Use AWS Systems Manager (SSM) Session Manager for all host shell interactions:

```bash
aws iam create-role --role-name prod-app-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name prod-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Restrict Secrets Manager access strictly to the RDS secret ARN
aws iam put-role-policy --role-name prod-app-role --policy-name read-db-secret \
  --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"secretsmanager:GetSecretValue\",\"Resource\":\"$DB_SECRET_ARN\"}]}"

aws iam create-instance-profile --instance-profile-name prod-app-profile
aws iam add-role-to-instance-profile --instance-profile-name prod-app-profile --role-name prod-app-role
```

#### 4.2 Application User Data Bootstrap
```bash
cat << 'EOF' > userdata.sh
#!/bin/bash
set -eu
dnf update -y
dnf install -y jq postgresql15

# Provision local application directory and healthcheck endpoint
mkdir -p /srv/app && echo ok > /srv/app/healthz
chown -R ec2-user:ec2-user /srv/app

# Pull database credentials dynamically from AWS Secrets Manager
SECRET=$(aws secretsmanager get-secret-value --secret-id "__SECRET_ARN__" \
  --region us-east-1 --query SecretString --output text)

install -m 600 -o ec2-user -g ec2-user /dev/null /etc/app.env
{
  echo "DB_HOST=__DB_HOST__"
  echo "DB_USER=$(echo "$SECRET" | jq -r .username)"
  echo "DB_PASS=$(echo "$SECRET" | jq -r .password)"
  echo "PORT=8080"
} > /etc/app.env

# Configure systemd unit
cat << 'SERVICE' > /etc/systemd/system/app.service
[Unit]
Description=Application Tier Service
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/srv/app
EnvironmentFile=/etc/app.env
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=always

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable --now app.service
EOF

sed -i "s|__SECRET_ARN__|$DB_SECRET_ARN|; s|__DB_HOST__|$DB_HOST|" userdata.sh
USERDATA_B64=$(base64 -w 0 userdata.sh)
```

#### 4.3 EC2 Launch Template (IMDSv2 Enforced)
```bash
aws ec2 create-launch-template \
  --launch-template-name prod-app-template \
  --version-description "v1.0" \
  --launch-template-data "{
    \"ImageId\": \"resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64\",
    \"InstanceType\": \"c6i.large\",
    \"SecurityGroupIds\": [\"$APP_SG\"],
    \"IamInstanceProfile\": {\"Name\": \"prod-app-profile\"},
    \"MetadataOptions\": {\"HttpTokens\": \"required\", \"HttpPutResponseHopLimit\": 1},
    \"BlockDeviceMappings\": [{\"DeviceName\": \"/dev/xvda\", \"Ebs\": {\"Encrypted\": true, \"VolumeType\": \"gp3\"}}],
    \"UserData\": \"$USERDATA_B64\"
  }"
```

---

### 📍 Phase 5: The Web Tier (ALB + WAF + ASG)

#### 5.1 Target Group & Application Load Balancer
```bash
TG_ARN=$(aws elbv2 create-target-group --name prod-app-tg \
  --protocol HTTP --port 8080 --vpc-id $VPC_ID --target-type instance \
  --health-check-protocol HTTP --health-check-port 8080 \
  --health-check-path /healthz \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3 \
  --health-check-timeout-seconds 5 --health-check-interval-seconds 15 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

ALB_ARN=$(aws elbv2 create-load-balancer --name prod-public-alb \
  --subnets $SUBNET_WEB_AZ1 $SUBNET_WEB_AZ2 \
  --security-groups $ALB_SG \
  --scheme internet-facing --type application --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN --attributes \
  Key=routing.http.drop_invalid_header_fields.enabled,Value=true \
  Key=deletion_protection.enabled,Value=true
```

#### 5.2 Listeners (HTTPS with Modern TLS Policy & HTTP Redirect)
```bash
# HTTP -> HTTPS Redirect
aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions 'Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# HTTPS Listener (Replace with valid ACM Certificate ARN)
# aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
#   --protocol HTTPS --port 443 \
#   --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
#   --certificates CertificateArn=arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID \
#   --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

#### 5.3 AWS WAF Regional Web ACL Integration
```bash
WAF_ARN=$(aws wafv2 create-web-acl --name prod-web-acl --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=prod-web-acl \
  --rules '[{"Name":"AWSCommon","Priority":1,"Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesCommonRuleSet"}},"OverrideAction":{"None":{}},"VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"AWSCommon"}}]' \
  --query Summary.ARN --output text)

aws wafv2 associate-web-acl --web-acl-arn $WAF_ARN --resource-arn $ALB_ARN
```

#### 5.4 Auto Scaling Group & Target Tracking Scaling
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name prod-app-asg \
  --launch-template "LaunchTemplateName=prod-app-template,Version=\$Latest" \
  --min-size 2 --max-size 10 --desired-capacity 2 \
  --vpc-zone-identifier "$SUBNET_APP_AZ1,$SUBNET_APP_AZ2" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB --health-check-grace-period 300

# Target tracking scaling: Maintain fleet-wide average CPU utilization at 60%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name prod-app-asg --policy-name cpu-target-60 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":60.0}'
```

---

### 📍 Phase 6: Production Verification

#### 1. Ingress Verification
```bash
ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

curl -I "http://${ALB_DNS}/healthz"
# Expected: HTTP/1.1 200 OK (or HTTP/1.1 301 Moved Permanently if redirecting)
```

#### 2. Perimeter Isolation Audit (Run from External Internet)
```bash
# 1. Attempt connection to RDS endpoint
nc -zv -w 3 "$DB_HOST" 5432
# Expected: Connection timed out

# 2. Attempt connection to internal App Tier private IP
nc -zv -w 3 10.0.10.25 8080
# Expected: Connection timed out (Non-routable RFC 1918 address)
```

#### 3. App-to-Database Connection Audit
Start an AWS Systems Manager session to an application instance:

```bash
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names prod-app-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)

aws ssm start-session --target "$INSTANCE_ID"
```

Inside the Session Manager shell:
```bash
source /etc/app.env
PGPASSWORD="$DB_PASS" psql -h "$DB_HOST" -U "$DB_USER" -d appdb -c '\conninfo'
# Expected: Confirms encrypted SSL connection to the PostgreSQL database
```

---

### 📍 Phase 7: Day-2 Operations & Incident Triage

#### Database Failover Chaos Test
Test database failover during a designated maintenance window:

```bash
aws rds reboot-db-instance --db-instance-identifier prod-database --force-failover
```
*The standby instance assumes primary status within 60 to 120 seconds. Ensure application connection pools configure automated retries and short DNS TTL resolution.*

#### Zero-Downtime Rolling Release
```bash
# 1. Author new Launch Template Version with updated AMI
aws ec2 create-launch-template-version \
  --launch-template-name prod-app-template --source-version 1 \
  --launch-template-data '{"ImageId":"ami-NEW_AMI_ID"}'

# 2. Trigger automated Instance Refresh
aws autoscaling start-instance-refresh --auto-scaling-group-name prod-app-asg \
  --preferences '{"MinHealthyPercentage": 50, "InstanceWarmup": 300}'
```

---

### 📍 Phase 8: Observability & Audit Baseline

* [ ] **VPC Flow Logs:** Capture all traffic metadata and stream to an encrypted S3 audit bucket.
* [ ] **CloudWatch Alarms:** Configure alerts for ALB `HTTPCode_Target_5XX_Count`, `UnHealthyHostCount`, RDS `CPUUtilization`, and `FreeStorageSpace`.
* [ ] **AWS CloudTrail:** Enable multi-region trail logging for all control plane and data plane API calls.
* [ ] **AWS Backup:** Automate daily EBS volume and RDS cross-region snapshots to satisfy enterprise Recovery Point Objectives (RPO).