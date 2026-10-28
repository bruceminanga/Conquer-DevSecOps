# 🏗️ Production AWS Architecture: The Complete Engineering Guide & Runbook for a Multi-Tier VPC

Running production workloads inside AWS’s default VPC, or placing compute instances and databases in subnets with direct internet routes, is the cloud infrastructure equivalent of leaving your data center's front door unlocked.

Whether preparing for **SOC 2 Type II**, **PCI-DSS (Req 1.3)**, or **HIPAA**, or simply hardening infrastructure against automated perimeter sweeps, an isolated **multi-tier Virtual Private Cloud (VPC)** is the baseline enterprise standard.

Below is an engineering breakdown of the architectural rationale, followed by an end-to-end AWS CLI runbook to provision a hardened VPC across two Availability Zones — covering flow logs, VPC endpoints, egress controls, default security group neutralization, custom NACLs, and an order-safe teardown script.

---

## 🏛️ PART 1: The Architectural “Why”

### 1. The Multi-Tier Model (Defense-in-Depth)

Rather than maintaining a flat network topology, isolate workloads into three distinct planes:

```text
INTERNET (Untrusted External Traffic)
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│ PUBLIC INGRESS TIER (0.0.0.0/0 -> Internet Gateway)         │
│   [ NAT Gateway ]            [ Application Load Balancer ]  │
└──────────────┬───────────────────────────────┬──────────────┘
               │ (Outbound Only)               │ (Forward HTTP/S)
               ▼                               ▼
┌─────────────────────────────────────────────────────────────┐
│ PRIVATE APPLICATION TIER (0.0.0.0/0 -> NAT Gateway)         │
│   [ Backend Compute: EKS / ECS / EC2 / Microservices ]      │
│   [ VPC Endpoints: S3, SSM, ECR, CloudWatch ]               │
└──────────────────────────────┬──────────────────────────────┘
                               │ (Restricted Port: 5432/3306)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ ISOLATED DATA TIER (Local Route 10.0.0.0/16 Only)           │
│   [ Persistent Systems: RDS / Aurora / ElastiCache ]        │
└─────────────────────────────────────────────────────────────┘
```

* 🔹 **Public / Ingress Tier:** Hosts customer-facing ingress endpoints (ALBs) and egress translation proxies (NAT Gateways). No application logic or database engines reside here.
* 🔹 **Private Application Tier:** Hosts internal business logic (EKS worker nodes, ECS tasks, EC2 instances). Accepts routed requests from the ALB and initiates outbound calls to public APIs via the NAT Gateway, while remaining entirely unreachable from inbound internet traffic.
* 🔹 **Isolated Persistence Tier:** Hosts databases and stateful caches (RDS, Aurora, ElastiCache). Has zero route to the internet in either direction.

---

### 2. Public vs. Private Subnets: The Under-the-Hood Truth

A subnet is classified as "public" or "private" strictly by the configuration of its associated **Route Table**:

* **Public Subnet:** Contains an explicit route mapping `0.0.0.0/0` to an **Internet Gateway (IGW)**.
* **Private Subnet:** Has no route pointing to an IGW. Even if an instance in this subnet is assigned a public IPv4 address, return traffic lacks an egress route, keeping the instance network-isolated from the internet.

---

### 3. Why NAT Gateways?

Backend microservices frequently require outbound internet access to download OS patches, pull container base images, and call external vendor APIs (Stripe, Twilio, SendGrid).

A **NAT Gateway** translates the instance's private IPv4 (`10.0.x.x`) to an assigned Elastic IP (EIP). Because the NAT Gateway is **stateful**, responses to outbound-initiated TCP connections are permitted back in, while unsolicited inbound connection attempts are dropped at the edge.

---

### 4. Independent Route Tables per Tier and AZ

* **Tier Segregation:** The public tier routes outbound traffic to the IGW, the application tier routes to a NAT Gateway, and the persistence tier routes only locally. Sharing route tables across tiers dissolves security boundaries.
* **Failure Domain Isolation (High Availability):** If all private subnets route outbound traffic through a single NAT Gateway located in `us-east-1a`, an outage in Availability Zone A eliminates internet access for instances running in Availability Zone B. **Deploying one NAT Gateway per AZ keeps failure domains completely independent.**

---

### 5. Why VPC Endpoints?

Without VPC Endpoints, internal traffic destined for AWS services (S3, DynamoDB, SSM, ECR) must traverse NAT Gateways out to the public internet before looping back to AWS.

* **Cost Optimization:** NAT Gateways incur hourly runtime charges plus per-GB data processing fees. **Gateway Endpoints (S3, DynamoDB) are free.**
* **Security & Reliability:** Traffic remains within the private AWS network backbone, eliminating internet egress dependencies for infrastructure management (e.g., AWS Systems Manager Session Manager).

---

### 6. Why Flow Logs?

Routing tables and security groups define access controls, but they do not provide audit proof. **VPC Flow Logs** capture network telemetry (accepted and rejected IP packets) to satisfy compliance controls (SOC 2, PCI-DSS) and support security incident forensics.

---

## 🛠️ PART 2: The Implementation Runbook

### Network Addressing Architecture

* **VPC CIDR:** `10.0.0.0/16` (65,536 addresses)
* **Region:** `us-east-1` (`us-east-1a`, `us-east-1b`)

| Subnet Name           | CIDR Block     | Type     | Purpose                      |
| :-------------------- | :------------- | :------- | :--------------------------- |
| `prod-public-1a`      | `10.0.1.0/24`  | Public   | Ingress ALB + NAT Gateway A  |
| `prod-public-1b`      | `10.0.2.0/24`  | Public   | Ingress ALB + NAT Gateway B  |
| `prod-private-app-1a` | `10.0.11.0/24` | Private  | Microservices / Compute AZ-A |
| `prod-private-app-1b` | `10.0.12.0/24` | Private  | Microservices / Compute AZ-B |
| `prod-private-db-1a`  | `10.0.21.0/24` | Isolated | Primary RDS / Redis Master   |
| `prod-private-db-1b`  | `10.0.22.0/24` | Isolated | Read Replica / Redis Replica |

> [!TIP]
> Ensure your VPC CIDR does not overlap with existing cloud networks, corporate VPNs, or on-premises data center subnets to prevent IP conflicts during future VPC Peering or AWS Transit Gateway integrations.

---

### Shell Session Initialization

```bash
set -euo pipefail
export AWS_REGION=us-east-1
```

---

### Step 1: Create the VPC & Enable DNS Resolution

Both DNS flags must be set to `true` to ensure RDS endpoints, private hosted zones, and Interface VPC Endpoints resolve correctly via the Route 53 Resolver (`10.0.0.2`).

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]' \
  --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support "{\"Value\":true}"
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames "{\"Value\":true}"
```

---

### Step 2: Provision Subnets Across Availability Zones

```bash
# Public Tier (Tier 1)
PUB_SUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-public-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PUB_SUB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-public-1b}]' \
  --query 'Subnet.SubnetId' --output text)

aws ec2 modify-subnet-attribute --subnet-id $PUB_SUB_1A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB_SUB_1B --map-public-ip-on-launch

# Private Application Tier (Tier 2)
APP_SUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-app-1a}]' \
  --query 'Subnet.SubnetId' --output text)

APP_SUB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-app-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Isolated Database Tier (Tier 3)
DB_SUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.21.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-db-1a}]' \
  --query 'Subnet.SubnetId' --output text)

DB_SUB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.22.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-db-1b}]' \
  --query 'Subnet.SubnetId' --output text)
```

---

### Step 3: Attach the Internet Gateway (IGW)

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

---

### Step 4: Allocate Elastic IPs & Deploy High-Availability NAT Gateways

Provision one NAT Gateway per public subnet to preserve multi-AZ availability:

```bash
EIP_1A=$(aws ec2 allocate-address --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=prod-nat-eip-1a}]' \
  --query 'AllocationId' --output text)

EIP_1B=$(aws ec2 allocate-address --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=prod-nat-eip-1b}]' \
  --query 'AllocationId' --output text)

NAT_GW_1A=$(aws ec2 create-nat-gateway --subnet-id $PUB_SUB_1A --allocation-id $EIP_1A \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-natgw-1a}]' \
  --query 'NatGateway.NatGatewayId' --output text)

NAT_GW_1B=$(aws ec2 create-nat-gateway --subnet-id $PUB_SUB_1B --allocation-id $EIP_1B \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-natgw-1b}]' \
  --query 'NatGateway.NatGatewayId' --output text)

# Block until gateways transition to available status
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_1A $NAT_GW_1B
```

---

### Step 5: Configure Routing Tables

#### A. Public Route Table
```bash
RT_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-rt-public}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB_SUB_1A
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB_SUB_1B
```

#### B. Private Application Route Tables (Per-AZ NAT Egress)
```bash
# AZ-A App Route Table
RT_PRIV_1A=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-rt-private-1a}]' \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RT_PRIV_1A --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_1A
aws ec2 associate-route-table --route-table-id $RT_PRIV_1A --subnet-id $APP_SUB_1A

# AZ-B App Route Table
RT_PRIV_1B=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-rt-private-1b}]' \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RT_PRIV_1B --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_1B
aws ec2 associate-route-table --route-table-id $RT_PRIV_1B --subnet-id $APP_SUB_1B
```

#### C. Isolated Database Route Table (Local Only)
```bash
RT_DB=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-rt-database}]' \
  --query 'RouteTable.RouteTableId' --output text)

# Explicitly contains only the implicit local route (10.0.0.0/16)
aws ec2 associate-route-table --route-table-id $RT_DB --subnet-id $DB_SUB_1A
aws ec2 associate-route-table --route-table-id $RT_DB --subnet-id $DB_SUB_1B
```

#### D. Quarantine the Main Route Table
Any subnet lacking an explicit association automatically defaults to the VPC's Main Route Table. Ensure it remains isolated with no routes pointing to an IGW or NAT:

```bash
MAIN_RT=$(aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=$VPC_ID Name=association.main,Values=true \
  --query 'RouteTables[0].RouteTableId' --output text)

aws ec2 create-tags --resources $MAIN_RT --tags Key=Name,Value=prod-rt-main-isolated
```

---

### Step 6: Chained Security Groups & Egress Hardening

Security groups enforce stateful firewall rules at the elastic network interface (ENI) layer. Chain security groups by referencing IDs rather than static CIDRs:

```bash
ALB_SG=$(aws ec2 create-security-group --group-name prod-alb-sg \
  --description "Public ingress for ALB" --vpc-id $VPC_ID --query 'GroupId' --output text)
APP_SG=$(aws ec2 create-security-group --group-name prod-app-sg \
  --description "Ingress from ALB only" --vpc-id $VPC_ID --query 'GroupId' --output text)
DB_SG=$(aws ec2 create-security-group --group-name prod-db-sg \
  --description "Ingress from App only" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Chained Ingress Rules
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 8080 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG  --protocol tcp --port 5432 --source-group $APP_SG
```

#### Enforce Egress Restrictions
Replace the standard "allow-all outbound" rule to satisfy zero-trust network baselines:

```bash
ALLOW_ALL_EGRESS='IpProtocol=-1,IpRanges=[{CidrIp=0.0.0.0/0}]'

# Restrict ALB egress strictly to the App Tier
aws ec2 revoke-security-group-egress --group-id $ALB_SG --ip-permissions "$ALLOW_ALL_EGRESS"
aws ec2 authorize-security-group-egress --group-id $ALB_SG \
  --protocol tcp --port 8080 --source-group $APP_SG

# Restrict Database egress entirely (Database initiates zero outbound connections)
aws ec2 revoke-security-group-egress --group-id $DB_SG --ip-permissions "$ALLOW_ALL_EGRESS"
```

#### Neutralize the Default Security Group
CIS AWS Foundations Benchmarks require that the VPC default security group denies all inbound and outbound traffic:

```bash
DEFAULT_SG=$(aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=default \
  --query 'SecurityGroups[0].GroupId' --output text)

# Purge default self-referencing ingress and all outbound rules
aws ec2 revoke-security-group-ingress --group-id $DEFAULT_SG \
  --ip-permissions "IpProtocol=-1,UserIdGroupPairs=[{GroupId=$DEFAULT_SG}]" || true
aws ec2 revoke-security-group-egress --group-id $DEFAULT_SG --ip-permissions "$ALLOW_ALL_EGRESS" || true
```

---

### Step 7: Configure VPC Endpoints

Keep management and storage traffic strictly within the AWS network:

```bash
# 1. Gateway Endpoint for S3 (Attached directly to private app route tables - Free)
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $RT_PRIV_1A $RT_PRIV_1B

# 2. Security Group for Interface Endpoints (Allow HTTPS from within VPC CIDR)
VPCE_SG=$(aws ec2 create-security-group --group-name prod-vpce-sg \
  --description "HTTPS from VPC to interface endpoints" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $VPCE_SG --protocol tcp --port 443 --cidr 10.0.0.0/16

# 3. AWS Systems Manager (SSM) Interface Endpoints (Enables shell access without public IPs)
for SVC in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.$SVC \
    --subnet-ids $APP_SUB_1A $APP_SUB_1B \
    --security-group-ids $VPCE_SG --private-dns-enabled
done
```

---

### Step 8: Enable VPC Flow Logs

Stream network metadata to an encrypted S3 bucket:

```bash
FLOW_LOG_ID=$(aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::YOUR-AUDIT-BUCKET-NAME/vpc-flow-logs/ \
  --max-aggregation-interval 60 \
  --query 'FlowLogIds[0]' --output text)
```

---

### Step 9: Define the Database Subnet Group

RDS requires an explicit DB Subnet Group across multiple Availability Zones:

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name prod-db-subnets \
  --db-subnet-group-description "Isolated persistence tier subnets" \
  --subnet-ids $DB_SUB_1A $DB_SUB_1B
```

---

### Step 10: Architectural Verification & Audit

Test using an EC2 instance launched into the private application tier (configured with the `AmazonSSMManagedInstanceCore` IAM policy):

1. **Verify Outbound NAT Translation:**  
   Execute via SSM Session Manager:
   ```bash
   curl -s https://checkip.amazonaws.com
   ```
   *The returned IP must match `aws ec2 describe-addresses --allocation-ids $EIP_1A`, proving traffic exits through the AZ-local NAT Gateway.*
2. **Verify Inbound Perimeter Isolation:**  
   Run the AWS VPC Reachability Analyzer from the Internet Gateway to the compute instance's ENI. The path must resolve to **Not Reachable**.
3. **Verify Database Isolation:**  
   From an instance in the persistence tier, attempt public egress:
   ```bash
   curl -I --connect-timeout 5 https://aws.amazon.com
   ```
   *The request must immediately time out.*
4. **Audit Flow Logs:**  
   Inspect the S3 bucket to confirm that `REJECT` records are actively captured for blocked egress and unauthorized ingress attempts.

---

### Step 11: Network ACL Hardening (Database Tier)

Security groups provide stateful filtering per instance. Network ACLs (NACLs) provide **stateless, subnet-level packet filtering**:

```bash
NACL_DB=$(aws ec2 create-network-acl --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=prod-nacl-database}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Ingress: Allow PostgreSQL (5432) strictly from Private App Subnets
aws ec2 create-network-acl-entry --network-acl-id $NACL_DB --ingress --rule-number 100 \
  --protocol tcp --port-range From=5432,To=5432 --cidr-block 10.0.11.0/24 --rule-action allow
aws ec2 create-network-acl-entry --network-acl-id $NACL_DB --ingress --rule-number 110 \
  --protocol tcp --port-range From=5432,To=5432 --cidr-block 10.0.12.0/24 --rule-action allow

# Egress: Stateless return traffic to ephemeral ports (1024-65535) on App Subnets
aws ec2 create-network-acl-entry --network-acl-id $NACL_DB --egress --rule-number 100 \
  --protocol tcp --port-range From=1024,To=65535 --cidr-block 10.0.11.0/24 --rule-action allow
aws ec2 create-network-acl-entry --network-acl-id $NACL_DB --egress --rule-number 110 \
  --protocol tcp --port-range From=1024,To=65535 --cidr-block 10.0.12.0/24 --rule-action allow

# Associate NACL with both DB Subnets
for SUB in $DB_SUB_1A $DB_SUB_1B; do
  ASSOC=$(aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=$SUB \
    --query 'NetworkAcls[0].Associations[?SubnetId==`'$SUB'`].NetworkAclAssociationId' --output text)
  aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $NACL_DB
done
```

---

### Step 12: Order-Safe Teardown Script

Cloud resources must be deleted in reverse dependency order. Route table associations and endpoint ENIs must be fully detached before subnets and VPCs can be deleted:

```bash
# 1. Flow Logs, VPC Endpoints & NAT Gateways
aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID
VPCE_IDS=$(aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID \
  --query 'VpcEndpoints[].VpcEndpointId' --output text)
if [ -n "$VPCE_IDS" ]; then
  aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $VPCE_IDS
fi

aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_1A
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_1B
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW_1A $NAT_GW_1B

aws ec2 release-address --allocation-id $EIP_1A
aws ec2 release-address --allocation-id $EIP_1B

# 2. Internet Gateway
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 3. Disassociate and Delete Custom Route Tables
for RT in $RT_PUB $RT_PRIV_1A $RT_PRIV_1B $RT_DB; do
  for ASSOC in $(aws ec2 describe-route-tables --route-table-ids $RT \
    --query 'RouteTables[0].Associations[?Main==`false`].RouteTableAssociationId' --output text); do
    aws ec2 disassociate-route-table --association-id $ASSOC
  done
  aws ec2 delete-route-table --route-table-id $RT
done

# 4. DB Subnet Group, Subnets & NACLs
aws rds delete-db-subnet-group --db-subnet-group-name prod-db-subnets
for SUB in $PUB_SUB_1A $PUB_SUB_1B $APP_SUB_1A $APP_SUB_1B $DB_SUB_1A $DB_SUB_1B; do
  aws ec2 delete-subnet --subnet-id $SUB
done

aws ec2 delete-network-acl --network-acl-id $NACL_DB

# 5. Security Groups and Base VPC
for SG in $DB_SG $APP_SG $ALB_SG $VPCE_SG; do
  aws ec2 delete-security-group --group-id $SG
done

aws ec2 delete-vpc --vpc-id $VPC_ID
```

---

## 📌 PART 3: Production Hardening Notes

* **Scale to 3 Availability Zones:** For mission-critical production platforms, expand this model across three AZs (`us-east-1a`, `us-east-1b`, `us-east-1c`) to maintain capacity during an AZ degradation.
* **Kubernetes (EKS) Subnet Discovery Tags:**  
  When running AWS EKS, attach dynamic discovery tags to public and private subnets so the AWS Load Balancer Controller provisions ALBs accurately:
  ```bash
  aws ec2 create-tags --resources $PUB_SUB_1A $PUB_SUB_1B \
    --tags Key=kubernetes.io/role/elb,Value=1
  aws ec2 create-tags --resources $APP_SUB_1A $APP_SUB_1B \
    --tags Key=kubernetes.io/role/internal-elb,Value=1
  ```
* **Codify in Terraform:** While the AWS CLI provides transparency for how cloud network resources interact, codify this topology into a version-controlled Terraform child module backed by an encrypted remote S3 backend and CI/CD pipelines to prevent manual drift.