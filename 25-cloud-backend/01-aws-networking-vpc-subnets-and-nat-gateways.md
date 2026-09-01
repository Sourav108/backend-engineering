# AWS Networking: VPCs, Public vs Private Subnets, and NAT Gateways

---

## 1. What Is It?
An **Amazon Virtual Private Cloud (VPC)** is an isolated, logically partitioned private software-defined network provisioned within the AWS cloud where backend engineers deploy computing resources (EC2 instances, EKS Kubernetes clusters, RDS databases) with full control over IP address ranges, subnets, routing tables, and network gateways.

---

## 2. Why Does It Exist?
Without a private VPC:
- Every database and backend server would receive a public IPv4 address directly exposed to the open internet, vulnerable to automated port scans, brute-force SSH attacks, and zero-day exploits.
- **VPC Subnet Isolation** enforces physical network zoning: only public ingress load balancers are reachable from the internet; all application microservices and databases reside in **Private Subnets with zero public IP addresses**.

---

## 3. Mental Model: The Production Multi-AZ 3-Tier VPC Topology

```mermaid
flowchart TD
    subgraph Internet["The Public Internet"]
        Users["End Users & External APIs"]
    end

    subgraph AWS_VPC["AWS VPC (CIDR: 10.0.0.0/16)"]
        IGW["Internet Gateway (IGW)"]
        Users <--> IGW

        subgraph PublicSubnets["1. Public Subnets (CIDR: 10.0.1.0/24 & 10.0.2.0/24)"]
            ALB["Application Load Balancer (ALB)"]
            NAT1["NAT Gateway AZ-1"]
            NAT2["NAT Gateway AZ-2"]
            IGW <--> ALB & NAT1 & NAT2
        end

        subgraph AppPrivateSubnets["2. Private Application Subnets (CIDR: 10.0.10.0/24 & 10.0.20.0/24)"]
            Pod1["Spring Boot Pods (AZ-1)"]
            Pod2["Spring Boot Pods (AZ-2)"]
            ALB --> Pod1 & Pod2
            Pod1 -->|Outbound HTTPS (API calls)| NAT1
            Pod2 -->|Outbound HTTPS (API calls)| NAT2
        end

        subgraph DataPrivateSubnets["3. Isolated Database Subnets (CIDR: 10.0.100.0/24 & 10.0.200.0/24)"]
            PrimaryDB[("PostgreSQL Primary (AZ-1)")]
            ReplicaDB[("PostgreSQL Standby (AZ-2)")]
            Pod1 & Pod2 --> PrimaryDB & ReplicaDB
            Note over DataPrivateSubnets: ZERO internet route! 100% isolated!
        end
    end
```

---

## 4. How Does It Work?

### 1. Public vs Private Subnet Mechanics
- **Public Subnet**: Its Route Table contains a default route pointing directly to the **Internet Gateway (IGW)**:
  $$\texttt{0.0.0.0/0} \longrightarrow \texttt{igw-xxxx}$$
  Instances in this subnet can receive direct inbound traffic from the internet (e.g. Load Balancers).
- **Private Subnet (with Outbound Internet)**: Its Route Table points default internet traffic to a **NAT Gateway**:
  $$\texttt{0.0.0.0/0} \longrightarrow \texttt{nat-xxxx}$$
  Instances in this subnet **cannot be reached from the internet**, but can initiate outbound requests (e.g. downloading Stripe SDKs or calling OpenAI APIs).
- **Isolated Database Subnet**: Contains **no route to IGW and no route to NAT Gateway**.

---

### 2. The NAT Gateway High-Availability Architecture
- **Single NAT Gateway Failure Risk**: If you deploy 1 NAT Gateway in AZ-1 and point Private Subnets in AZ-1 and AZ-2 to it:
  - If AZ-1 suffers an AWS datacenter outage, **all pods in AZ-2 immediately lose outbound internet connectivity**.
  - Cross-AZ data transfer fees ($\$0.01/\text{GB}$) accumulate between AZ-2 pods and AZ-1 NAT.
- **Production Standard**: Always deploy **1 NAT Gateway per Availability Zone**.

---

## 5. Implementation: Terraform Production VPC Definition

```hcl
# 1. AWS VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "production-backend-vpc" }
}

# 2. Internet Gateway for Inbound Public Traffic
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

# 3. Public Subnet (AZ 1a)
resource "aws_subnet" "public_1a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
}

# 4. Elastic IP & NAT Gateway in Public Subnet
resource "aws_eip" "nat_1a" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat_1a" {
  allocation_id = aws_eip.nat_1a.id
  subnet_id     = aws_subnet.public_1a.id
}

# 5. Private Application Subnet (AZ 1a)
resource "aws_subnet" "private_app_1a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-east-1a"
}

# 6. Route Table routing Private Subnet outbound traffic through NAT Gateway
resource "aws_route_table" "private_app_rt_1a" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_1a.id
  }
}

resource "aws_route_table_association" "private_app_assoc_1a" {
  subnet_id      = aws_subnet.private_app_1a.id
  route_table_id = aws_route_table.private_app_rt_1a.id
}
```

---

## 6. Performance & Security Comparison

| Subnet Tier | Inbound Internet Access | Outbound Internet Access | Target Infrastructure |
|---|---|---|---|
| **Public Subnet** | ✅ Direct via IGW | ✅ Direct via IGW | ALBs, Bastion Hosts, NAT Gateways |
| **Private Subnet** | ❌ Blocked | ✅ Via NAT Gateway | Spring Boot Pods, EKS Nodes, Workers |
| **Isolated Subnet**| ❌ Blocked | ❌ Blocked | PostgreSQL Primary/Replica, Redis |

---

## 7. Interview Questions

### Q1: Why must a NAT Gateway always reside in a Public Subnet rather than a Private Subnet?
<details>
<summary>Reveal Answer</summary>

**Answer**:
A **NAT (Network Address Translation) Gateway** translates private IP addresses (`10.0.10.15`) into a public Elastic IP to communicate with external internet services on behalf of private instances.
- To transmit packets to and receive reply packets from the internet, the NAT Gateway itself must have an internet-routable public IP and a route table path directly to an **Internet Gateway (IGW)**.
- If you place a NAT Gateway in a private subnet whose route table has no path to an IGW, the NAT Gateway has no physical internet connectivity and cannot perform NAT translation for other nodes.
</details>

---

## 8. Quick Revision
- **VPC**: Isolated software-defined private cloud network.
- **Public Subnet**: Route table points `0.0.0.0/0` to Internet Gateway (IGW).
- **Private Subnet**: Route table points `0.0.0.0/0` to NAT Gateway.
- **Isolated Subnet**: Contains zero internet routes; ideal for databases.
- **Multi-AZ NAT**: Deploy 1 NAT Gateway per AZ to eliminate single points of failure and avoid cross-AZ data fees.
