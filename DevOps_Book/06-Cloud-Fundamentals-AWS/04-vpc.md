# Chapter 4 — VPC & Networking

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what a VPC is and why you need a custom one for production
- Design a multi-AZ VPC with public and private subnets
- Configure Internet Gateways, NAT Gateways, and route tables
- Differentiate Security Groups from NACLs and know when to use each
- Choose between VPC Peering and Transit Gateway
- Implement VPC Endpoints to keep traffic off the public internet

---

## 4.1 What Is a VPC?

A **VPC (Virtual Private Cloud)** is your own isolated network within AWS. Think of it as a private data center network that you define and control — but running inside AWS infrastructure.

Key facts:

- Every AWS account gets a **default VPC** per region — but always create custom VPCs for production
- You control: IP address range, subnets, route tables, gateways, and security
- VPCs are **regional** — they span all Availability Zones in a region
- Resources in a VPC are isolated from other AWS customers by default

> **Why not use the default VPC?** The default VPC has all subnets public, which is a security risk. Production workloads need private subnets for databases and app servers.

---

## 4.2 VPC Components

```
VPC: 10.0.0.0/16
├── AZ: us-east-1a
│   ├── Public Subnet:  10.0.1.0/24  (IGW route → internet)
│   └── Private Subnet: 10.0.2.0/24  (NAT Gateway route → internet)
└── AZ: us-east-1b
    ├── Public Subnet:  10.0.3.0/24
    └── Private Subnet: 10.0.4.0/24

Internet Gateway ─── attached to VPC (enables public internet access)
NAT Gateway ──────── in public subnet (enables private subnet outbound internet)
```

Core components:

| Component | Purpose |
|---|---|
| **CIDR Block** | IP address range for the VPC (e.g., `10.0.0.0/16` = 65,536 IPs) |
| **Subnet** | Subdivision of VPC IP range, tied to one AZ |
| **Internet Gateway (IGW)** | Enables VPC resources to reach the public internet |
| **NAT Gateway** | Lets private subnet resources initiate outbound internet connections |
| **Route Table** | Rules that control where network traffic is directed |
| **Security Group** | Stateful firewall at the instance level |
| **NACL** | Stateless firewall at the subnet level |
| **VPC Endpoint** | Private connection to AWS services without internet |

---

## 4.3 Subnets

- **Public subnet**: has a route to the Internet Gateway — resources can get public IPs and be reached from the internet
- **Private subnet**: no direct internet route — accessible only from within the VPC or via VPN/Direct Connect

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'

# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'

# Create private subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'
```

> **CIDR sizing tip**: Use `/16` for the VPC and `/24` for subnets. A `/24` gives you 251 usable IPs (AWS reserves 5 per subnet). Plan for growth — you can add subnets but cannot change the VPC CIDR.

---

## 4.4 Internet Gateway and Route Tables

An **Internet Gateway** is a horizontally scaled, redundant, highly available VPC component that enables communication between your VPC and the internet. There is no bandwidth constraint.

```bash
# Create and attach Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-12345678 \
  --vpc-id vpc-12345678

# Create public route table with IGW route
aws ec2 create-route-table --vpc-id vpc-12345678
aws ec2 create-route \
  --route-table-id rtb-12345678 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-12345678

# Associate public subnets with this route table
aws ec2 associate-route-table \
  --route-table-id rtb-12345678 \
  --subnet-id subnet-public-1a
```

Route table logic: traffic matching `0.0.0.0/0` (all internet traffic) is sent to the IGW. More specific routes (like `10.0.0.0/16` for local VPC traffic) take precedence.

---

## 4.5 NAT Gateway (Private Subnet Internet Access)

A **NAT Gateway** lets instances in private subnets initiate outbound connections to the internet (e.g., to download packages) without being reachable from the internet.

```bash
# Allocate Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc

# Create NAT Gateway in PUBLIC subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a \
  --allocation-id eipalloc-12345678

# Create route table for private subnets with NAT Gateway
aws ec2 create-route-table --vpc-id vpc-12345678
aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-12345678

aws ec2 associate-route-table \
  --route-table-id rtb-private \
  --subnet-id subnet-private-1a
```

> **Cost warning**: NAT Gateway costs ~$0.045/hr + $0.045/GB of data processed — the biggest surprise bill item for beginners. Create one NAT Gateway per AZ for high availability. For dev environments, consider NAT Instance (EC2-based, cheaper but not managed).

---

## 4.6 Security Groups vs NACLs

| Feature | Security Group | NACL |
|---|---|---|
| **Level** | Instance level | Subnet level |
| **State** | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| **Rules** | Allow only | Allow and Deny |
| **Evaluation** | All rules evaluated | Rules evaluated in order (lowest number first) |
| **Default** | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| **Use case** | Primary firewall for instances | Additional subnet-level control |

**Security Group example:**

```bash
# Create security group for web servers
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group" \
  --vpc-id vpc-12345678

# Allow inbound HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow inbound HTTP (for redirect)
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

> **NACLs in practice**: Use security groups for most things. Add NACLs for blocking specific IP ranges — for example, blocking a DDoS source IP at the subnet level before traffic reaches your instances.

---

## 4.7 VPC Peering and Transit Gateway

### VPC Peering

A **VPC Peering** connection is a direct, private network connection between two VPCs (same or different accounts/regions).

- No transitive routing: if A↔B and B↔C, A **cannot** reach C through B
- Low latency, no bandwidth bottleneck
- Best for a few VPCs; does not scale to many

```bash
# Create peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-11111111 \
  --peer-vpc-id vpc-22222222

# Accept the peering request
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-12345678

# Add routes in both VPCs
aws ec2 create-route \
  --route-table-id rtb-vpc1 \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-12345678
```

### Transit Gateway

A **Transit Gateway** acts as a regional hub connecting hundreds of VPCs and on-premises networks.

- Transitive routing: A → TGW → C works
- Single point to manage instead of N×(N-1)/2 peering connections
- Supports VPN and Direct Connect attachments
- Costs per attachment and per GB processed

> Use VPC Peering for simple two-VPC connections. Use Transit Gateway when you have many VPCs, need on-premises connectivity, or need transitive routing.

---

## 4.8 VPC Endpoints (Private AWS Service Access)

**VPC Endpoints** let your VPC resources communicate with AWS services (S3, DynamoDB, etc.) without traversing the public internet.

Two types:

- **Gateway Endpoint**: free; for S3 and DynamoDB; modifies route tables
- **Interface Endpoint**: powered by PrivateLink; for most other AWS services; creates ENI in your subnet; costs ~$0.01/hr

```bash
# S3 Gateway Endpoint — free, traffic stays in AWS network
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private
```

**Without VPC endpoint**: EC2 in private subnet → NAT Gateway → internet → S3 (costs money, leaves AWS network)

**With VPC endpoint**: EC2 in private subnet → VPC endpoint → S3 (free, stays in AWS network)

---

## 4.9 Production VPC Design Pattern

A well-architected 3-tier VPC for production workloads:

```
3-tier architecture:
Public subnets:  ALB, NAT Gateway, Bastion host
Private subnets: EC2 app servers, ECS tasks
Data subnets:    RDS, ElastiCache (no route to internet at all)

Security group chain:
ALB-SG:  inbound 443 from 0.0.0.0/0
App-SG:  inbound 8080 from ALB-SG only
DB-SG:   inbound 5432 from App-SG only
```

This pattern ensures:

- Web traffic only enters through the load balancer
- App servers are never directly exposed to the internet
- Databases can only be reached from app servers
- Even if an app server is compromised, the attacker cannot reach the DB without also compromising the security group rules

---

## Summary

- A VPC is your isolated network in AWS — always use custom VPCs for production
- Design for multiple AZs with both public and private subnets
- Internet Gateway enables inbound/outbound internet for public subnets
- NAT Gateway enables outbound-only internet access from private subnets
- Security Groups are stateful instance-level firewalls; NACLs are stateless subnet-level firewalls
- VPC Peering is simple point-to-point; Transit Gateway is the hub-and-spoke solution for scale
- VPC Endpoints keep traffic to AWS services off the public internet and save NAT Gateway costs

---

## Knowledge Check

1. What is the difference between a public subnet and a private subnet in a VPC?
2. Where must a NAT Gateway be placed — in a public or private subnet? Why?
3. An EC2 instance in a private subnet cannot receive inbound connections from the internet, but traffic is being blocked both ways. What two things might you check?
4. You have 10 VPCs that all need to communicate with each other and with an on-premises data center. Should you use VPC Peering or Transit Gateway? Why?
5. Your application in a private subnet is downloading large files from S3, and your NAT Gateway bill is high. What can you do to reduce costs?

---

## Hands-on Exercise

Create a production-grade VPC from scratch:

1. Create a VPC with CIDR `10.0.0.0/16`
2. Create 4 subnets across 2 AZs:
   - `10.0.1.0/24` — Public, us-east-1a
   - `10.0.2.0/24` — Private, us-east-1a
   - `10.0.3.0/24` — Public, us-east-1b
   - `10.0.4.0/24` — Private, us-east-1b
3. Create and attach an Internet Gateway
4. Create a NAT Gateway in the public subnet (us-east-1a)
5. Create and configure route tables (one for public subnets with IGW, one for private subnets with NAT Gateway)
6. Launch an EC2 instance in the private subnet (no public IP)
7. Verify: the instance can reach the internet (e.g., `curl https://example.com`) via NAT
8. Verify: the instance is NOT directly reachable from the internet
9. Create an S3 VPC Endpoint and verify traffic to S3 no longer flows through the NAT Gateway

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="03-ec2.md">← Previous: EC2 — Elastic Compute Cloud</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="05-s3.md">Next: S3 — Simple Storage Service →</a>
</div>
