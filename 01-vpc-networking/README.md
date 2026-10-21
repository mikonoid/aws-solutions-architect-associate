# 01 — VPC & Networking

> The foundation of every AWS architecture. On the exam, networking shows up in
> almost every question: public/private subnets, routing, internet access,
> security (SG/NACL), hybrid connectivity, and private access to services.

---

## 1. VPC — core concepts

- **VPC (Virtual Private Cloud)** — an isolated virtual network within **a single region**.
  A VPC cannot span regions, but it **covers all AZs** of the region.
- **CIDR block** — an IP range (e.g. `10.0.0.0/16`). You can add a **secondary CIDR**.
  - Allowed private ranges (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
  - VPC CIDR size: **/16 (65536 IPs) … /28 (16 IPs)**.
  - Limits (soft): **5 VPCs per region**, **5 CIDR blocks per VPC** (both adjustable).
- **Subnet** — lives **in a single AZ** (subnet ≠ multi-AZ). The subnet CIDR is a subset of the VPC CIDR.
  - AWS **reserves 5 IPs** in every subnet (first 4 + last one).
    E.g. in `10.0.0.0/24`: `.0 .1 .2 .3` and `.255` are taken → 251 usable, not 256.
  - Sizing tip: subtract the 5 reserved IPs. Need 29 usable? `/27` gives 32−5=27 (too few) → use `/26` (64−5=59).
- **Default VPC** — created automatically in every region; all subnets are public
  (have auto-assign public IP).

### Public vs private subnet

| | Public | Private |
|---|---|---|
| Route table | has a `0.0.0.0/0 → IGW` route | **no** route to IGW |
| Public IP | resources can have public/Elastic IP | private IPs only |
| Internet access | directly via IGW | outbound via **NAT**, no inbound |

> Note: A subnet becomes "public" NOT via a flag, but via a **route to the IGW** in its route table.

### Route tables

- Every subnet is associated with exactly one route table; if none is set, it uses the VPC's **main** route table.
- **Local route** — every route table has an unremovable local route for the VPC CIDR (intra-VPC communication).
- **Longest-prefix match** — the most specific route wins (e.g. `10.0.1.0/24` beats `10.0.0.0/16` for `10.0.1.50`).
- Common targets: `0.0.0.0/0 → IGW` (public), `0.0.0.0/0 → NAT` (private outbound), peering connection, Virtual Private Gateway (VPN/DX).
- Best practice: a **dedicated route table per subnet** so you don't accidentally make a subnet public via the main table.

---

## 2. Internet Gateway, NAT, Egress-only IGW

- **Internet Gateway (IGW)** — provides two-way internet access for public subnets.
  One IGW per VPC. For an instance to reach the internet you need: public IP + route to IGW + SG/NACL allow.
- **NAT Gateway** — outbound internet access for **private** subnets (IPv4).
  - AWS-managed, scales from 5 Gbps up to **100 Gbps**, **lives in a single AZ** → for HA put a NAT in each AZ.
  - Placed in a **public** subnet, requires an **Elastic IP**. Security groups do **not** apply to a NAT Gateway.
  - Costs money (per hour + per GB). Needs an IGW in the VPC to actually reach the internet.
- **NAT Instance** — a cheap legacy, self-managed alternative (a regular EC2 doing NAT).
  - Must **disable source/destination check**; security groups **do** apply; can double as a bastion host.
  - Not HA by default (use an ASG across AZs). Bandwidth depends on the instance type.
- **Egress-only Internet Gateway** — same idea as NAT, but for **IPv6** (outbound access,
  inbound connections blocked). IPv6 addresses in AWS are always public, so NAT for IPv6 doesn't exist.

| Need | Solution |
|---|---|
| Inbound+outbound internet (IPv4/IPv6) | Internet Gateway |
| Outbound-only internet for private (IPv4) | NAT Gateway / NAT Instance |
| Outbound-only internet for private (IPv6) | Egress-only IGW |

> Gotcha: "private subnet needs internet access without a public IP" → **NAT Gateway**,
> not IGW directly.

- **Bastion host** — a hardened EC2 in a **public** subnet used as an SSH jump host to reach
  instances in **private** subnets. Lock its SG inbound to your IP only; private instances
  accept SSH only from the bastion's SG.
- **Multi-tier pattern** — ALB in public subnet → app servers in private subnet (SG allows only the
  ALB SG) → database in private subnet (SG allows only the app SG). A chain of trust via SG references.

---

## 3. Security Groups vs Network ACL

| | Security Group | Network ACL (NACL) |
|---|---|---|
| Level | **ENI / instance** level | **subnet** level |
| Stateful? | **Stateful** (response allowed automatically) | **Stateless** (needs explicit inbound+outbound) |
| Rules | **allow** only | **allow and deny** |
| Rule evaluation | all rules together | **by rule number order**, first match wins |
| Defaults | deny all inbound, allow all outbound | default NACL: allow all; custom NACL: deny all |

> Note: To **explicitly block** a specific IP (e.g. an attacker) you need a **NACL deny**,
> because SG can't deny.
> Note: For stateless NACL, don't forget **ephemeral ports** (1024–65535) for return traffic.

- **SG referencing**: a security group rule can reference **another security group** (same region), enabling
  multi-tier "chain of trust": ALB SG → app SG → DB SG, instead of hardcoding IPs.
- **NACL rule numbers**: `1–32766`, lower number = higher precedence, first match wins; AWS convention is
  increments of **100**. The final `*` rule denies anything unmatched. Custom NACL denies all until you add rules.

---

## 4. VPC connectivity: Peering, Transit Gateway, VPN, Direct Connect

- **VPC Peering** — a direct connection between two VPCs (even cross-region/cross-account).
  - **NOT transitive**: A↔B and B↔C does not give A↔C. CIDRs **must not overlap**.
  - You must update route tables on both sides.
- **Transit Gateway (TGW)** — a hub connecting many VPCs and on-prem (**hub-and-spoke**).
  - Transitive, scales to thousands of VPCs, supports multicast, cross-region (peering).
  - The right choice when many VPCs need "any-to-any" connectivity.
- **Site-to-Site VPN** — an IPsec tunnel over the internet to on-prem. Fast, cheap, but
  depends on the internet (variable latency).
  - Components: **Virtual Private Gateway (VGW)** on the AWS side + **Customer Gateway (CGW)** on-prem.
  - **2 tunnels** per connection for redundancy, ~1.25 Gbps per tunnel. Enable **route propagation** on the route table.
  - **Client VPN** — a separate TLS-based VPN for individual remote users (not site-to-site).
- **Direct Connect (DX)** — a private physical line to AWS. Stable latency, high
  throughput, but takes a long time to provision (weeks).
  - **Not encrypted by default** → run a VPN over DX if you need encryption.
  - **Private VIF** → reach a VPC; **public VIF** → reach public AWS services (e.g. S3). Uses BGP.
  - **Direct Connect Gateway** → connect one DX to VPCs in multiple regions.
  - Speeds: Dedicated **1/10/100 Gbps**; Hosted **50 Mbps–10 Gbps**.
  - **DX + VPN** as backup — the typical pattern for reliable hybrid connectivity.
  - Fast HA backup for DX while waiting for a second line → **Site-to-Site VPN**.

| Need | Solution |
|---|---|
| 2 VPCs directly | VPC Peering |
| Many VPCs + on-prem, centralized | Transit Gateway |
| Fast/cheap hybrid connection | Site-to-Site VPN |
| Stable latency, high throughput | Direct Connect |
| Reliable hybrid (prod) | Direct Connect + VPN backup |

> Gotcha: "Transitive peering" doesn't exist — if a question needs many VPCs linked → **TGW**.

---

## 5. VPC Endpoints — private access to AWS services

Let you reach AWS services **without going to the internet** (no IGW/NAT).

- **Gateway Endpoint** — only for **S3** and **DynamoDB**.
  - Free. Works via an entry in the **route table**. Doesn't leave the region.
- **Interface Endpoint (AWS PrivateLink)** — for most other services (SQS, SNS, KMS,
  API Gateway, etc.) and for your own services.
  - Creates an **ENI with a private IP** in the subnet. Paid (per hour + per GB).

| Service | Endpoint type |
|---|---|
| S3, DynamoDB | Gateway (free) or Interface |
| Other AWS / your own services | Interface (PrivateLink) |

> Note: "Private access to S3 without NAT/internet, cheaply" → **Gateway Endpoint**.

---

## 6. DNS and Route 53 (networking part)

- A VPC has a built-in DNS resolver at `VPC base +2` (`169.254.169.253` / `.2`).
- VPC flags: `enableDnsSupport` and `enableDnsHostnames` — needed for DNS names
  and for **Interface Endpoints** to work via private DNS names.
- **Route 53 Resolver**:
  - **Inbound endpoint** — on-prem → resolves AWS DNS.
  - **Outbound endpoint** — AWS → resolves on-prem DNS (conditional forwarding).
- Route 53 routing policies (more in the DNS topic): Simple, Weighted, Latency,
  Failover, Geolocation, Geoproximity, Multivalue.

---

## 7. Flow Logs and network monitoring

- **VPC Flow Logs** — logs traffic **metadata** (accept/reject), NOT the payload.
  - Level: VPC / subnet / ENI. Destinations: **CloudWatch Logs**, **S3**, Firehose.
  - Use: diagnosing "why traffic isn't flowing", auditing, security.
  - Immutable: you **can't edit a flow log after creation** — delete and recreate.
  - Does **not** capture: Amazon DNS traffic, DHCP, or traffic to instance metadata (`169.254.169.254`).
- **Traffic Mirroring** — copies **actual packets** from an ENI for deep analysis (IDS/IPS).
  - When you need the traffic content itself, not just metadata.

> Note: "Why are packets dropped / who is blocking traffic" → **Flow Logs** (look for REJECT).
> Gotcha: Flow Logs don't show packet contents → if you need the payload, use **Traffic Mirroring**.

---

## 8. Quick recap (TL;DR)

- VPC = region; subnet = one AZ; AWS reserves **5 IPs** per subnet.
- Public subnet = **route to IGW**.
- Outbound internet for private: IPv4 → **NAT Gateway**, IPv6 → **Egress-only IGW**.
- NAT Gateway — **per-AZ**, in a public subnet, with an Elastic IP, up to 100 Gbps (no SG).
- **SG** = stateful, allow only, instance level, can reference other SGs. **NACL** = stateless, allow+deny, subnet level.
- Block a specific IP → **NACL deny**.
- Route tables: unremovable **local route**, **longest-prefix match** wins, one RT per subnet (best practice).
- Many VPCs "any-to-any" → **Transit Gateway** (peering is not transitive).
- Private access to S3/DynamoDB without NAT → **Gateway Endpoint** (free).
- Other services privately → **Interface Endpoint / PrivateLink**.
- Hybrid: fast/cheap → **Site-to-Site VPN** (VGW+CGW, 2 tunnels); stable/high-throughput → **Direct Connect** (not encrypted by default).
- Traffic diagnostics → **VPC Flow Logs** (metadata), contents → **Traffic Mirroring**.

## 9. Questions

See [`questions.md`](questions.md)
