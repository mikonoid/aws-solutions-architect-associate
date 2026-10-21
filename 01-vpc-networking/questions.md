# 01 — VPC & Networking: practice questions

> Format: question → options → **correct answer** with an explanation of why it's
> right and why the others are wrong. Try to answer first, then expand the answer.

---

### Q1. An instance in a private subnet must download data from the internet (OS updates) but must not be reachable from outside. What do you configure?

- A. Assign an Elastic IP to the instance
- B. Add a `0.0.0.0/0 → IGW` route to the subnet's route table
- C. Place a NAT Gateway in a public subnet and add a route to it
- D. Enable VPC Peering

<details><summary>Answer</summary>

**C.** A NAT Gateway gives private instances **outbound-only** internet access without
opening inbound connections. The NAT goes in a **public** subnet with an Elastic IP, and
the private subnet's route table gets `0.0.0.0/0 → nat-...`.

- A/B would make the instance publicly reachable (inbound connections) — violates the requirement.
- D — peering is unrelated to internet access.
</details>

---

### Q2. You need private (no internet, no NAT) and free read/write access to S3 from private instances. What do you choose?

- A. Interface Endpoint (PrivateLink) for S3
- B. Gateway Endpoint for S3
- C. NAT Gateway
- D. VPC Peering to a VPC that has S3

<details><summary>Answer</summary>

**B.** For **S3** and **DynamoDB** there's a **Gateway Endpoint** — free, works via a
route table entry, and never goes to the internet.

- A — an Interface Endpoint for S3 exists but is **paid**; the "free" requirement → Gateway.
- C — NAT costs money and goes through the internet.
- D — S3 doesn't live in a VPC, so peering is irrelevant.
</details>

---

### Q3. You need to explicitly block inbound traffic from a specific attacker IP at the subnet level. What do you use?

- A. Security Group deny rule
- B. Network ACL deny rule
- C. IAM policy
- D. Route table blackhole

<details><summary>Answer</summary>

**B.** A Security Group **doesn't support deny** (allow only). Explicit IP blocking is
done via a **NACL deny** rule at the subnet level (with a lower rule number so it's
evaluated before any allow).

- A — SG has no deny.
- C — IAM doesn't filter network traffic.
- D — a blackhole route affects CIDR routing; it's not a security mechanism for a single inbound IP.
</details>

---

### Q4. A company has 15 VPCs and an on-prem data center and needs centralized any-to-any connectivity. What do you choose?

- A. A full mesh of VPC Peering between all VPCs
- B. Transit Gateway
- C. A separate IGW in each VPC
- D. Direct Connect between VPCs

<details><summary>Answer</summary>

**B.** **Transit Gateway** is hub-and-spoke, transitive, scales to thousands of VPCs, and
attaches on-prem. For 15 VPCs a full peering mesh = `n(n-1)/2 = 105` connections and is
non-transitive → hard to manage.

- A — unmanageable number of peerings, not transitive.
- C — IGW is about the internet, not VPC-to-VPC connectivity.
- D — DX connects on-prem to AWS, not VPCs to each other.
</details>

---

### Q5. Engineers report packets to an instance are being dropped. You need to find out whether SG/NACL is blocking them, without changing the configuration. What do you enable?

- A. CloudTrail
- B. VPC Flow Logs
- C. Traffic Mirroring
- D. GuardDuty

<details><summary>Answer</summary>

**B.** **VPC Flow Logs** capture traffic metadata tagged **ACCEPT/REJECT** — you can see
exactly what's being rejected. It's the cheapest and standard way to diagnose reachability.

- A — CloudTrail logs **API calls**, not network traffic.
- C — Traffic Mirroring gives **packet contents** (overkill and more expensive here).
- D — GuardDuty is threat detection, not drop diagnosis.
</details>

---

### Q6. What is the maximum number of usable (available) IPs in a subnet with CIDR `/24`?

- A. 256
- B. 255
- C. 254
- D. 251

<details><summary>Answer</summary>

**D. 251.** A `/24` has 256 addresses total, but AWS **reserves 5** in every subnet
(network, VPC router, DNS, future use, broadcast) → `256 − 5 = 251`.
</details>

---

### Q7. You need a hybrid connection to on-prem with **stable low latency** and high throughput for prod. What do you choose?

- A. Site-to-Site VPN
- B. Direct Connect
- C. VPC Peering
- D. NAT Gateway

<details><summary>Answer</summary>

**B.** **Direct Connect** is a private physical line with stable latency and high
throughput (unlike VPN over the public internet).

- A — VPN is cheap/fast to launch, but latency depends on the internet.
- C/D — not hybrid on-prem solutions.

> For prod reliability, teams often combine **DX + VPN as backup**.
</details>

---

### Q8. Private instances with **IPv6** addresses need outbound-only internet access. What do you use?

- A. NAT Gateway
- B. Egress-only Internet Gateway
- C. Internet Gateway
- D. NAT Instance

<details><summary>Answer</summary>

**B.** For **IPv6**, outbound-only access is provided by an **Egress-only Internet Gateway**.
NAT (Gateway/Instance) works for **IPv4**. IPv6 addresses in AWS are globally routable,
so a regular IGW would allow inbound connections too.
</details>

---

### Q9. VPC-A is peered with VPC-B, and VPC-B is peered with VPC-C. Why can't instances in A reach instances in C?

- A. Security Groups need updating
- B. VPC Peering is not transitive — you need a direct A↔C peering or a Transit Gateway
- C. The CIDR blocks overlap
- D. An IGW is missing

<details><summary>Answer</summary>

**B.** **VPC Peering is not transitive**: traffic does not flow "through" an intermediate
VPC. You need either a direct A↔C peering or a **Transit Gateway** as a hub.
</details>

---

### Q10. A NAT Gateway is deployed in a single AZ. During an outage of that AZ, private instances in **other** AZs lost internet access. How do you fix this for HA?

- A. Increase the NAT Gateway size
- B. Deploy a separate NAT Gateway in each AZ and route locally
- C. Move the NAT Gateway into a private subnet
- D. Replace the NAT Gateway with an IGW

<details><summary>Answer</summary>

**B.** A NAT Gateway is a **zonal** resource. For resilience, create a **NAT Gateway in
each AZ**, and each private subnet's route table points to the NAT **in its own AZ**.

- A — size doesn't protect against an AZ failure.
- C — a NAT must be in a public subnet.
- D — an IGW would make the instances public, breaking privacy.
</details>

---

### Q11. A colleague created a VPC with IPv4 CIDR `10.0.0.0/24` but now needs to host more resources and is running out of IP space. He plans to create a brand-new VPC with a larger range. What's a better, reliable approach?

- A. Delete the existing subnets in the VPC and create new subnets
- B. He is thinking of the right approach (create a new, bigger VPC)
- C. Create a new VPC and connect the old VPC with the new one
- D. Expand the existing VPC by adding a secondary CIDR to it

<details><summary>Answer</summary>

**D.** A VPC supports **secondary CIDR blocks**. You can add more address space to the
existing VPC without recreating it, then add new subnets in the new range.

- A — deleting/recreating subnets doesn't add more address space (still limited by the `/24`).
- B — creating a whole new VPC is unnecessary disruption and migration effort.
- C — peering adds complexity and the CIDRs here would also need to not overlap; not the simplest reliable fix.
</details>

---

### Q12. You have a bastion host EC2 in a public subnet and want to SSH to it. The route table already has an Internet Gateway. What is the **secure and minimal** configuration for the SSH request to work?

- A. Allow SSH (port 22) on SG inbound and SG outbound; allow NACL inbound and outbound for `0.0.0.0/0`
- B. Allow SSH (port 22) on SG inbound and SG outbound; allow NACL inbound for your IP address
- C. Allow SSH (port 22) on SG inbound and NACL inbound for your IP address
- D. Allow SSH (port 22) on SG inbound; allow NACL inbound and NACL outbound for your IP address

<details><summary>Answer</summary>

**D.** Security Groups are **stateful** — allowing SSH **inbound** is enough; the return
traffic is automatically allowed (no outbound SG rule needed). NACLs are **stateless**, so
you need both **inbound and outbound** rules. Scoping to **your IP** is the secure/minimal choice.

- A — allowing NACL for `0.0.0.0/0` is not minimal/secure (opens to the whole internet).
- B — missing the NACL **outbound** rule (NACL is stateless, so return traffic would be blocked).
- C — same problem: no NACL outbound rule for the return traffic.

> Key trap: SG stateful (inbound only) vs NACL stateless (needs inbound **and** outbound,
> on ephemeral ports for the response).
</details>

---

### Q13. A MAC-address-dependent application (licensing is tied to the MAC) will run on an on-demand `r4.2xlarge` EC2 instance. How do you ensure the MAC address does **not change** even if the instance is restarted or rebooted?

- A. Assign a static MAC address to the EC2 instance while setting up the server
- B. Use a VPC with an Elastic Network Interface that has a fixed MAC address
- C. Use a VPC with a private subnet for the EC2 — the MAC will be fixed by default
- D. Use a VPC with a private subnet and configure the MAC address to be tied to that subnet

<details><summary>Answer</summary>

**B.** An **Elastic Network Interface (ENI)** keeps a **fixed MAC address** for its
lifetime. Attach an ENI to the instance; across stop/start/reboot the ENI (and its MAC)
stays the same.

- A — you can't manually assign an arbitrary static MAC in EC2.
- C — a private subnet doesn't guarantee a fixed MAC by itself.
- D — MAC addresses aren't "tied to a subnet"; they belong to the ENI.
</details>

---

### Q14. You want a direct, **private** communication mechanism to S3 from within your VPC. What feature provides this connectivity?

- A. Direct Connect
- B. S3 PrivateLink
- C. S3 VPC Endpoint
- D. S3 Peering

<details><summary>Answer</summary>

**C.** An **S3 VPC Endpoint** (Gateway Endpoint) provides direct, private access to S3
from within the VPC, without going over the internet.

- A — Direct Connect links on-prem to AWS, not a VPC-to-S3 private path.
- B — the common/expected answer here is the VPC Endpoint; while S3 does support interface
  endpoints, "S3 PrivateLink" is not the standard name for the direct S3 connectivity feature.
- D — "S3 Peering" doesn't exist.
</details>

---

### Q15. Where are security groups applied within the AWS VPC?

- A. Subnets
- B. EC2 Instances
- C. Elastic Network Interfaces
- D. Route Tables

<details><summary>Answer</summary>

**C.** Security groups are applied at the **Elastic Network Interface (ENI)** level. An
EC2 instance "has" a security group because the SG is attached to the instance's ENI.

- A — NACLs are applied at the subnet level, not SGs.
- B — effectively via the ENI, but the precise attachment point is the ENI.
- D — route tables control routing, not security filtering.
</details>

---

### Q16. You set up a peering connection between two VPCs with proper Security Groups and Route Tables, launched EC2 instances in both, but communication times out. What could be the reason?

- A. Security groups of the EC2 instances are not configured to allow traffic from the peered VPC
- B. Network ACLs have been configured not to allow traffic from the peered VPC
- C. Peered VPCs are in different regions
- D. Route tables of both VPCs only contain a specific IP range for the peering connection, and one of the EC2 instances is outside the configured IP ranges

<details><summary>Answer</summary>

**D.** The stem says SGs and route tables are configured "properly." The most likely
remaining cause is that the route tables cover only a **specific IP range**, and one EC2
instance's IP is **outside** that range, so its traffic isn't routed over the peering
connection and times out.

- A/B — the question states SGs are properly configured; NACLs aren't mentioned as misconfigured.
- C — cross-region peering is supported and wouldn't by itself cause a timeout.
</details>

---

### Q17. For route tables, what is a best practice when creating routes for multiple subnets?

- A. Put all routes into the MAIN route table so all subnets dynamically use it (fewest route tables to manage)
- B. Create a unique route table for each subnet
- C. Place a default route to the IGW in the MAIN route table and allow all subnets, minus the exceptions, to use it
- D. Place a default route to the corporate HQ over Direct Connect in the MAIN route table, minus the exceptions

<details><summary>Answer</summary>

**B.** Best practice is to **create a dedicated (custom) route table per subnet**. This
avoids accidentally making a subnet public via the MAIN route table and gives explicit,
auditable control over each subnet's routing.

- A/C — relying on the MAIN route table with a default IGW route risks unintentionally
  exposing subnets to the internet (anything using MAIN becomes public).
- D — a specific hybrid design, not a general best practice for multiple subnets.
</details>

---

### Q18. You peered VPC A↔B and VPC B↔C, and created a NAT Gateway in VPC B. You want VPC A and VPC C to use VPC B's NAT Gateway to reach the internet, but only VPC B's resources can. Why?

- A. Route tables in VPC A and VPC C are not configured to use VPC B's NAT Gateway
- B. Using another VPC's NAT Gateway is not supported in AWS
- C. VPC B's subnet containing the NAT Gateway is not configured in VPC A and VPC C route tables
- D. The NAT Gateway is not created inside VPC B's public subnet

<details><summary>Answer</summary>

**B.** You **cannot use a NAT Gateway across a VPC peering connection**. Edge-to-edge
routing through a peered VPC's NAT Gateway (also IGW, VPN, DX) is **not supported**. Each
VPC needs its own NAT Gateway for internet access.

- A/C — no route-table change makes cross-VPC NAT work; it's an architectural limitation.
- D — the stem implies VPC B's resources already reach the internet, so the NAT is placed correctly.
</details>

---

### Q19. Select **two** statements that are true about Security Groups.

- A. A Security Group has allow rules and deny rules
- B. By default, incoming traffic is allowed regardless of any rules
- C. By default, outgoing traffic is allowed regardless of any rules
- D. A Security Group is the first layer of security

<details><summary>Answer</summary>

**C and D.** A default SG **allows all outbound** traffic, and the SG (at the instance/ENI
level) acts as the **first line of defense** for the instance.

- A — SGs are **allow-only** (no deny rules); that's NACLs.
- B — default SG **denies all inbound** until you add allow rules.
</details>

---

### Q20. A VPC was initially created with CIDR `20.0.0.0/16`. A route table shows a route for `30.0.0.0/20` pointing to a **local** target. Which statement is correct?

- A. It's a VPC peering connection route for `30.0.0.0/20`
- B. It's a VPN connection route for a remote network `30.0.0.0/20`
- C. It's a Direct Connect route for a remote network `30.0.0.0/20`
- D. `30.0.0.0/20` is a secondary CIDR range added to the VPC, with a local route

<details><summary>Answer</summary>

**D.** A **local** target means the destination is inside the VPC. Since `30.0.0.0/20` is
outside the primary `20.0.0.0/16`, it must be a **secondary CIDR block** added to the VPC
— AWS automatically adds a local route for it.

- A/B/C — peering, VPN, and DX routes point to a peering connection, VGW, etc., **not** `local`.
</details>

---

### Q21. How does a Lambda function communicate with resources inside your VPC?

- A. A VPC Gateway is configured
- B. A VPC Endpoint is configured
- C. An ENI is established for Lambda
- D. You cannot do this

<details><summary>Answer</summary>

**C.** When you enable VPC access for a Lambda function, Lambda creates an **Elastic
Network Interface (ENI)** in the chosen subnets, giving it a private IP to reach VPC
resources (RDS, ElastiCache, etc.).

- A/B — not how Lambda attaches to a VPC.
- D — Lambda can access VPC resources via VPC configuration.
</details>

---

### Q22. If VPC A is peered to VPC B, and VPC B is peered to VPC C, can VPC A and VPC C communicate directly?

- A. Yes, VPC transitive peering is enabled by default
- B. No, VPC transitive peering is not enabled by default
- C. Yes, as long as a networking service in VPC B forwards the packets on A's behalf
- D. No, VPC transitive peering is not a supported feature in AWS

<details><summary>Answer</summary>

**D.** **VPC peering is never transitive** in AWS — there's no setting to enable it. For
A↔C you need a direct peering or a **Transit Gateway**.

- A/B — it's not a toggle; it's simply unsupported.
- C — routing "through" VPC B (edge-to-edge) is not supported.
</details>

---

### Q23. Two VPCs in **different regions** (`172.10.0.0/16` and `192.168.0.0/16`) need their subnets to communicate, and the team explicitly wants **no transitive peering**. Best solution?

- A. Use VPC Peering (AWS-provided connectivity between two VPCs)
- B. Use a Software VPN appliance between the VPCs
- C. Use VPC-to-VPC routing over an IPsec VPN connection
- D. VPC-to-VPC routing in an AWS Direct Connect location

<details><summary>Answer</summary>

**A.** **Inter-Region VPC Peering** directly connects two VPCs with non-overlapping CIDRs,
is AWS-managed (no appliances), and is inherently **non-transitive** — exactly what's asked.

- B/C — software/IPsec VPN adds unnecessary management overhead when peering fits.
- D — Direct Connect is for on-prem to AWS, not the simplest VPC-to-VPC answer here.
</details>

---

### Q24. Two EC2 instances in the **same subnet** each reference the **other's security group ID**. You move Instance B to a different AZ by snapshotting and launching a new instance (new IP), reusing the same security groups. Communication now fails. According to this exam's keyed answer, what's the problem?

- A. VPC routes are misconfigured between AZs
- B. A NACL is misconfigured
- C. The security group must be modified on Instance B since it moved to a new AZ
- D. The security groups for both instances must be updated to reflect the new sources and destinations

<details><summary>Answer</summary>

**D** (keyed answer). The quiz expects that because the instance/IPs changed, the SGs must
be updated to reflect the new sources/destinations.

> Nuance: In real AWS, **security group references are by SG ID**, not IP, so they keep
> working across AZ moves and IP changes — so technically this shouldn't break. Treat D as
> this particular quiz's intended answer, but remember the real behavior: SG-by-ID survives
> IP/AZ changes.
</details>

---

### Q25. You have an S3 **Gateway VPC Endpoint** serving some buckets. You create a new S3 bucket and reuse the endpoint, but requests from EC2 fail with **Access Denied**. Select **two** possible causes.

- A. The VPC endpoint policy is restricted to certain buckets and doesn't include the new bucket
- B. The IAM role/user doesn't have access to the new S3 bucket
- C. AWS's default DENY policy blocks IAM users/roles that already have access to the bucket
- D. You must add the new bucket hostname as destination and the endpoint ID as target in the route table

<details><summary>Answer</summary>

**A and B.** Access to S3 via an endpoint is governed by **both** the **VPC endpoint policy**
and **IAM permissions** (and the bucket policy). If the endpoint policy whitelists specific
buckets, or IAM doesn't allow the new bucket, you get Access Denied.

- C — there's no such "default DENY that blocks already-authorized users" mechanism.
- D — gateway endpoints add a **prefix-list route** automatically; you don't add per-bucket
  hostname routes manually.
</details>

---

### Q26. Select **two** statements that are true about VPCs.

- A. Subnets can span Availability Zones
- B. A VPC can have the same IP address range as another VPC in the same AWS account
- C. A VPC can span AWS Regions
- D. Secondary VPC CIDR blocks can be added after the VPC is created

<details><summary>Answer</summary>

**B and D.** Two VPCs can use the **same CIDR** (they're isolated; it only matters if you
try to peer them). You can also add **secondary CIDR blocks** to a VPC after creation.

- A — a subnet lives in exactly **one AZ**; it cannot span AZs.
- C — a VPC is **regional** and cannot span regions (use peering/TGW for cross-region).
</details>

---

### Q27. You're troubleshooting communication problems between EC2 instances in a VPC. Where can you see the **accepted and denied** firewall traffic for Security Groups and NACLs?

- A. CloudTrail
- B. CloudWatch Logs
- C. Application Load Balancer Logs
- D. VPC Flow Logs

<details><summary>Answer</summary>

**D.** **VPC Flow Logs** record IP traffic metadata with an **ACCEPT/REJECT** action, so
you can see what SGs/NACLs allowed or denied.

- A — CloudTrail logs **API activity**, not packet accept/reject.
- B — CloudWatch Logs is a **destination** for Flow Logs, but the feature that captures the
  accept/deny data is VPC Flow Logs.
- C — ALB access logs cover HTTP requests to the load balancer, not SG/NACL decisions.
</details>

---

### Q28. How is a VPC charged?

- A. By the number of subnets provisioned
- B. By the size of the VPC CIDR block
- C. By the number of routes defined
- D. A VPC does not have a fee

<details><summary>Answer</summary>

**D.** A **VPC itself is free**. You pay for resources/traffic inside it (NAT Gateways,
Elastic IPs in some cases, data transfer, interface endpoints, etc.), but not for the VPC,
subnets, route tables, IGW, SGs, or NACLs.
</details>

---

### Q29. A VPC `10.10.0.0/16` has a public subnet and two private subnets (`10.10.2.0/24` = private-1, `10.10.3.0/24` = private-2), each private subnet with its own route table. An EC2 in private-1 cannot reach an RDS MySQL DB in private-2. Select **two** possible reasons.

- A. One private subnet route table's **local route** was changed to restrict access to within the subnet IP range
- B. The RDS security group **inbound** rule is misconfigured with `10.10.1.0/24` instead of `10.10.2.0/24`
- C. The `10.10.3.0/24` subnet's NACL is modified to **deny inbound on port 3306** from `10.10.2.0/24`
- D. The RDS security group **outbound** doesn't contain a rule for ALL traffic or port 3306 for `10.10.2.0/24`

<details><summary>Answer</summary>

**B and C.** The RDS **inbound** SG must allow the EC2's subnet (`10.10.2.0/24`); pointing
it at `10.10.1.0/24` blocks the EC2 (B). A **NACL deny** on 3306 inbound to private-2 from
private-1 also blocks it (C).

- A — the VPC **local route cannot be modified or removed**, so this isn't a valid cause.
- D — SGs are **stateful**; the RDS outbound rule is irrelevant since response traffic is
  automatically allowed for an accepted inbound connection.
</details>

---

### Q30. You created VPC `10.10.0.0/16` and subnet `10.10.1.0/24`. A `/24` has 256 addresses, but the console shows only **251 available** IPs, with no resources launched yet. Why?

- A. The first four IP addresses and the last IP address in each subnet CIDR are reserved by AWS
- B. AWS reserves 5 IPs for every VPC, taken from the first subnet you create
- C. AWS launches monitoring resources in the new VPC when the first subnet is created, reserving 5 IPs
- D. None of the above

<details><summary>Answer</summary>

**A.** AWS **reserves 5 IPs in every subnet**: the first four (`.0` network, `.1` VPC
router, `.2` DNS, `.3` future use) and the last (`.255` broadcast). `256 − 5 = 251`.

- B/C — the reservation is **per subnet**, not a per-VPC or monitoring reservation.
</details>

---

## Notes / mistakes I made

> Record topics where you often make mistakes, so you can come back to them.

-
