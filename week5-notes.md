# Week 5 — VPC Design, Subnets, Gateways, NACLs & VPC Peering

> Training notes: VPC & CIDR design → subnet types → Internet Gateway → public subnet + SSH practical → NAT Gateway → private subnet + SG tweaks practical → NACL vs Security Group → cross-region VPC Peering.

---

## Part 1: VPC Design

### 1.1 What a VPC Is

A **VPC (Virtual Private Cloud)** is a logically isolated, software-defined network within an AWS region — your own private slice of the AWS network, where you control IP addressing, routing, subnetting, and connectivity to the outside world. Every resource with networking (EC2, RDS, Lambda-in-VPC, etc.) lives inside a VPC.

Key facts:
- A VPC is **region-scoped** — it spans all AZs within that region, but never crosses regions.
- You define its IP address range as a **CIDR block** at creation (e.g., `10.0.0.0/16`), which can be primary + secondary CIDR blocks added later.
- AWS provides a **Default VPC** per region (already has an IGW, public subnets in every AZ, ready to use) — most production setups instead build a **Custom VPC** for full control over addressing and segmentation.

### 1.2 CIDR & Address Planning

CIDR notation (`10.0.0.0/16`) defines the network address plus how many bits are fixed (`/16` = first 16 bits fixed = 65,536 addresses).

Design considerations learned in practice:
- Plan the VPC CIDR big enough to carve into multiple subnets across multiple AZs without running out (a `/16` giving room for many `/24` subnets is a common starting point).
- **AWS reserves 5 IP addresses in every subnet** (network address, VPC router, DNS, future use, and broadcast-equivalent) — so a `/24` subnet (256 addresses) really only gives you 251 usable IPs.
- Avoid overlapping CIDR ranges across VPCs you might later need to **peer or connect** — overlapping ranges make peering/routing impossible without additional NAT tricks.

### 1.3 Core VPC Components

- **Subnets** — subdivisions of the VPC CIDR, each tied to exactly one AZ.
- **Route Tables** — control where traffic from a subnet is directed; each subnet is associated with exactly one route table (though one route table can serve multiple subnets).
- **Internet Gateway (IGW)** — the door to the public internet.
- **NAT Gateway** — outbound-only internet access for private resources.
- **NACLs** — stateless subnet-level firewall.
- **Security Groups** — stateful instance-level firewall.
- **VPC Peering / Transit Gateway / VPN / Direct Connect** — connectivity between VPCs or to on-prem.

---

## Part 2: Subnets and Types

A **subnet** is a range of IP addresses within the VPC's CIDR, and it lives entirely inside **one Availability Zone** (never spans AZs). Whether a subnet is "public" or "private" is not an inherent AWS setting — it's purely determined by **what its route table points to.**

### 2.1 Public Subnet

A subnet is **public** if its route table has a route sending `0.0.0.0/0` (all internet-bound traffic) to an **Internet Gateway**.

```
Destination        Target
10.0.0.0/16         local
0.0.0.0/0           igw-0123456789abcdef
```

Instances here can be reached from (and reach out to) the internet directly — but only if they also have a **public IP/Elastic IP** and permissive enough Security Group/NACL rules. Typically hosts: load balancers, bastion/jump hosts, NAT gateways.

### 2.2 Private Subnet

A subnet is **private** if its route table has **no route to an Internet Gateway** — outbound internet traffic, if needed at all, is instead routed through a **NAT Gateway** sitting in a public subnet.

```
Destination        Target
10.0.0.0/16         local
0.0.0.0/0           nat-0123456789abcdef
```

Instances here have no direct inbound path from the internet at all — this is where you put application servers, databases, internal services: reachable only from inside the VPC (or via the NAT Gateway for their own outbound calls, e.g., pulling OS updates).

### 2.3 Isolated Subnet (a third, less-discussed variant)

A subnet with **no default route to anywhere outside the VPC at all** — not even via NAT. Used for the most sensitive workloads (e.g., databases that should never need direct internet access even outbound), reachable only via VPC-internal traffic or through VPC Endpoints for specific AWS services (S3, DynamoDB, etc., accessed without ever leaving the AWS network).

---

## Part 3: Internet Gateway (IGW)

An **Internet Gateway** is a horizontally scaled, highly available VPC component that:
- Provides a target for internet-routable traffic in route tables.
- Performs **1:1 NAT** for instances with a public/Elastic IP — translating between the instance's private IP and its public IP.
- Is **attached to exactly one VPC** at a time (one IGW per VPC).
- Has no bandwidth constraints or availability risk of its own to manage — unlike a NAT Gateway, it's not something you provision "capacity" for.

Practical checklist for a working public subnet:
1. IGW created and **attached to the VPC**.
2. Subnet's **route table** has `0.0.0.0/0 → igw-xxxx`.
3. Instance has a **public IP or Elastic IP** assigned.
4. **Security Group** on the instance allows the needed inbound port (e.g., `22` for SSH).
5. **NACL** on the subnet allows the traffic both inbound and outbound (NACLs are stateless — see Part 6).

---

## Part 4: Practical — Public Subnet + SSH via CLI

Standing this up end-to-end:

```bash
# 1. Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# 2. Create public subnet in an AZ
aws ec2 create-subnet --vpc-id vpc-xxxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a

# 3. Create + attach Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxxx --internet-gateway-id igw-xxxx

# 4. Create route table, add default route to IGW, associate with the subnet
aws ec2 create-route-table --vpc-id vpc-xxxx
aws ec2 create-route --route-table-id rtb-xxxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxxx
aws ec2 associate-route-table --route-table-id rtb-xxxx --subnet-id subnet-xxxx

# 5. Enable auto-assign public IP on the subnet
aws ec2 modify-subnet-attribute --subnet-id subnet-xxxx --map-public-ip-on-launch

# 6. Launch instance into the subnet with a security group allowing SSH
aws ec2 run-instances --image-id ami-xxxx --instance-type t2.micro \
  --subnet-id subnet-xxxx --security-group-ids sg-xxxx --key-name mykey

# 7. SSH in using the assigned public IP
ssh -i mykey.pem ec2-user@<public-ip>
```

**Debugging note from practice:** the most common reason SSH fails even after all this is a Security Group missing an inbound rule for port 22 from your IP, or a NACL that's blocking the **ephemeral return-traffic port range** (1024–65535) on the outbound side — since SSH's *response* traffic comes back on a high, dynamically chosen port, not port 22.

---

## Part 5: NAT Gateway

### 5.1 What It Does

A **NAT Gateway** lets instances in a **private subnet** initiate outbound connections to the internet (e.g., to download OS patches or call an external API) **while still blocking any inbound connection initiated from the internet.** It's the "outbound-only" counterpart to the IGW's bidirectional access.

### 5.2 Setup

- The NAT Gateway itself is deployed **inside a public subnet** (it needs its own route to the IGW) and is assigned an **Elastic IP**.
- The **private subnet's route table** then points `0.0.0.0/0` at the NAT Gateway (not the IGW directly).
- AWS's managed NAT Gateway is **highly available within its AZ only** — for multi-AZ resilience, best practice is **one NAT Gateway per AZ**, each with its own EIP, so a failure in one AZ doesn't take down egress for the others.

### 5.3 Security Group Tweaks for NAT Gateway + Private Instances

Practical adjustments needed to get a private-subnet instance actually reaching the internet through NAT:

- **Private instance's Security Group** — outbound rule allowing the traffic type needed (e.g., `443` outbound for HTTPS package repos); Security Groups being **stateful** means the *return* traffic is automatically allowed back in without a separate inbound rule.
- **NAT Gateway** itself does **not** use a Security Group (it's a managed AWS resource, not an EC2 instance) — so all the actual filtering happens on the **private instance's SG** and the subnet's **NACL**, not on the NAT Gateway.
- If a bastion/jump host in the public subnet is used to reach the private instance, the **private instance's SG** must explicitly allow inbound SSH **from the bastion's security group** (referencing the SG ID as the source, not a CIDR) — a common, cleaner pattern than allowing a specific IP.

### 5.4 NAT Gateway vs NAT Instance (context)

| | NAT Gateway (managed) | NAT Instance (self-managed EC2) |
|---|---|---|
| Management | Fully managed by AWS | You patch/manage the EC2 instance |
| Availability | HA within its AZ | Single point of failure unless you build HA yourself |
| Bandwidth | Scales automatically up to very high throughput | Limited by the instance type's network performance |
| Cost model | Hourly + per-GB data processing charge | Just the EC2 instance cost |
| Security Group | None (not an EC2 instance) | Has its own SG like any instance, must allow the private subnet's traffic |

---

## Part 6: NACL (Network Access Control List)

### 6.1 NACL vs Security Group

| | Security Group | NACL |
|---|---|---|
| Applies at | Instance/ENI level | Subnet level (applies to every instance in the subnet) |
| State | **Stateful** — return traffic automatically allowed, no matter the rule direction | **Stateless** — inbound and outbound rules must each be explicitly defined; return traffic is NOT automatically allowed |
| Rule type | **Allow only** — you cannot write a Deny rule | Supports both **Allow and Deny** rules |
| Rule evaluation | All rules evaluated together (most permissive wins, since there's no explicit Deny) | Rules evaluated **in numbered order**, first match wins, then stops |
| Default behavior | Default SG denies all inbound, allows all outbound | Default NACL allows all traffic; a **custom NACL** denies all by default until you add rules |

### 6.2 Why NACL Statelessness Matters in Practice

Because NACLs don't track connection state, you must explicitly allow **both directions** for any traffic to work — this is exactly what trips people up with SSH: allowing inbound `22` isn't enough, you also need an **outbound rule allowing the ephemeral port range (1024–65535)** so the SSH session's response traffic can leave the subnet.

```
Inbound rules:
100  ALLOW  TCP  22      0.0.0.0/0
*    DENY   ALL  ALL     0.0.0.0/0   (implicit final rule)

Outbound rules:
100  ALLOW  TCP  1024-65535   0.0.0.0/0   (ephemeral return traffic)
*    DENY   ALL  ALL          0.0.0.0/0
```

### 6.3 When to Use NACLs

Since Security Groups already provide fine-grained, stateful control per instance, NACLs are typically used as a **coarser, subnet-wide backstop** — e.g., explicitly blocking a known-malicious CIDR range at the subnet boundary regardless of what any individual instance's SG allows, since NACL Deny rules apply before traffic ever reaches the instance's SG evaluation.

---

## Part 7: VPC Peering (Cross-Region)

### 7.1 What VPC Peering Is

A **VPC Peering Connection** creates a private, direct network route between two VPCs, letting resources in each communicate using **private IP addresses** as if they were on the same network — traffic never traverses the public internet.

- Peering is **not transitive** — if VPC A peers with VPC B, and VPC B peers with VPC C, A **cannot** reach C through B. Each pair needing connectivity needs its own explicit peering connection (or a **Transit Gateway** for hub-and-spoke at scale instead).
- Works both **intra-region** and **cross-region** (Inter-Region VPC Peering) — cross-region traffic goes over AWS's private global backbone, encrypted, never hitting the public internet.

### 7.2 Setup Flow

1. **Requester** side initiates the peering connection, specifying the peer VPC ID (and region, for cross-region).
2. **Accepter** side (could be the same account or a different one) must **accept** the request before it becomes active.
3. **Update route tables on both sides** — each VPC's route table needs a route pointing the other VPC's CIDR at the peering connection ID (`pcx-xxxx`) — peering doesn't work automatically just by accepting; routing must be added manually on both ends.
4. **Update Security Groups** to allow traffic from the peer VPC's CIDR range (or specific SG, if same-account/region).
5. **Update NACLs** if using custom (non-default) NACLs, same statelessness caveat as before.

```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaa --peer-vpc-id vpc-bbbb --peer-region us-west-2

aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxxx   # run in the peer account/region

aws ec2 create-route --route-table-id rtb-in-vpc-a \
  --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-xxxx

aws ec2 create-route --route-table-id rtb-in-vpc-b \
  --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-xxxx
```

### 7.3 Prerequisites & Gotchas

- **Non-overlapping CIDR blocks** are mandatory — this is exactly why VPC CIDR planning (Part 1.2) matters up front; two VPCs with overlapping ranges simply cannot be peered.
- **DNS resolution across the peering connection** is off by default — must explicitly enable "DNS resolution support" on the peering connection if you want private DNS hostnames to resolve to private IPs across the peer.
- Cross-region peering has **no bandwidth bottleneck concept like VPN** but does incur **data transfer charges** for cross-region traffic, unlike same-region peering.
- Security Groups **cannot reference a peer VPC's Security Group ID directly across regions** — cross-region peering requires using **CIDR-based** SG rules instead of SG-ID references (SG-to-SG references only work same-region, and only for VPCs in the same or a permitted account).

---

## Summary — Week 5 Mental Model

```
VPC (region-scoped, CIDR block, e.g., 10.0.0.0/16)
   │
   ├── Public Subnet (route: 0.0.0.0/0 → IGW)
   │      └── bastion host / NAT Gateway / ALB
   │             SSH tested end-to-end: SG allows 22 inbound + NACL allows both directions
   │
   ├── Private Subnet (route: 0.0.0.0/0 → NAT Gateway)
   │      └── app/db instances — outbound-only internet via NAT
   │             reached only via bastion SG-to-SG reference, never directly from internet
   │
   ├── NACL — stateless, subnet-wide, evaluated in rule order, allow + deny
   └── Security Group — stateful, instance-wide, allow-only

VPC Peering (cross-region)
   - non-overlapping CIDRs required
   - route tables updated manually on BOTH sides
   - SG rules must be CIDR-based across regions (not SG-ID references)
   - private, encrypted, over AWS backbone — never public internet
```
