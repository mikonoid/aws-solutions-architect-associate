# 02 — Compute

> EC2 is the core of most exam scenarios: pricing models, placement, networking,
> and images. Around it sit containers (ECS/EKS/ECR), serverless (Lambda), managed
> platforms (Beanstalk, App Runner, Lightsail), batch, and hybrid/edge compute.

---

## 1. EC2 — core concepts

- **AMI** — image with the root volume, launch permissions, and block device mapping.
  AMIs are **regional**; copy them to use in another region. Can be shared with other accounts.
- **Instance type** — `family + generation + size`, e.g. `m7g.large`.

| Family | Optimized for | Example use |
|---|---|---|
| **T** | Burstable CPU (credits) | Dev/test, small web apps |
| **M** | General purpose | App servers, balanced workloads |
| **C** | Compute | Batch, gaming, HPC, encoding |
| **R / X / z** | Memory | In-memory DBs, caches, SAP |
| **I / D / H** | Storage (local NVMe / HDD) | NoSQL, data warehouses, Kafka |
| **P / G / Inf / Trn** | Accelerated (GPU/ML chips) | ML training/inference, graphics |

- **User data** — script run at **first boot** (as root). Used for bootstrapping.
- **Instance metadata (IMDS)** — `169.254.169.254`. Use **IMDSv2** (session token) to block SSRF-style credential theft.
- **IAM role (instance profile)** — the right way to give an instance AWS permissions.
  Never store access keys on the instance.
- **Instance states**: stop/start keeps EBS but usually changes the public IP (use an **Elastic IP** to keep it) and may move the host.
  Reboot keeps everything. Terminate deletes the root EBS by default.
- **Hibernate** — saves RAM to the (encrypted) EBS root volume and resumes fast. Max 60 days. Root must be EBS and large enough.

### Purchasing options

| Option | Discount | Commitment | Choose when |
|---|---|---|---|
| **On-Demand** | none | none | Short, unpredictable, can't be interrupted |
| **Reserved Instances** | up to ~72% | 1 or 3 years, specific instance attributes | Steady-state, known type (Standard RI); Convertible RI lets you change family |
| **Savings Plans** | up to ~72% | 1 or 3 years, $/hour commitment | Steady spend but flexible type/region (Compute SP also covers Fargate and Lambda) |
| **Spot** | up to ~90% | none, **2-minute interruption notice** | Fault-tolerant, stateless, flexible jobs (batch, CI, big data) |
| **Dedicated Host** | — | physical server for you | **BYOL** licensing tied to sockets/cores, compliance |
| **Dedicated Instance** | — | hardware not shared with other accounts | Isolation requirement without host-level control |
| **Capacity Reservation** | none by itself | reserve capacity in an AZ | Guarantee capacity for a known event (combine with Savings Plans/RI for discount) |

> Gotcha: "Cheapest for a job that can be interrupted and restarted" → **Spot**.
> Gotcha: "License bound to physical cores/sockets" → **Dedicated Host**.
> Gotcha: Reserved Instances discount the bill but **do not guarantee capacity** unless zonal.

### Placement groups

| Strategy | Layout | Use |
|---|---|---|
| **Cluster** | Close together in **one AZ** | Lowest latency, highest throughput between nodes (HPC). Higher blast radius |
| **Spread** | Each instance on distinct hardware, max **7 instances per AZ** per group | Small number of critical instances that must not fail together |
| **Partition** | Groups of instances on separate racks, up to **7 partitions per AZ** | Large distributed systems (HDFS, Cassandra, Kafka) that are rack-aware |

### Networking on EC2

- **Elastic IP** — static public IPv4 you own in a region. Charged (all public IPv4 is billed now).
- **Enhanced networking (ENA)** — higher bandwidth, lower latency, no extra cost.
- **EFA (Elastic Fabric Adapter)** — ENA + OS-bypass for tightly coupled HPC/MPI on Linux.

---

## 2. Elastic Network Interfaces (ENI)

- A virtual NIC in a **subnet (one AZ)**: private IPs, optional public/Elastic IP, security groups, **MAC address**.
- The **primary ENI** (eth0) cannot be detached. **Secondary ENIs** can be detached and attached to another instance **in the same AZ**.
- The ENI keeps its private IP, Elastic IP, SGs, and **MAC** when moved. That is the answer for:
  - Licensing tied to a MAC address.
  - Fast failover: move the ENI (and its IP) to a standby instance.
  - Management network on a separate subnet (dual-homed instance).
- Number of ENIs and IPs per instance depends on the instance type.

---

## 3. EC2 Image Builder

- Managed pipeline that **builds, hardens, tests, and distributes AMIs** (and container images) on a schedule.
- Recipe = base image + components (install/configure/test). Distribute to regions and accounts.
- Exam trigger: "automate golden AMI creation and keep it patched".

---

## 4. Elastic Beanstalk

- **PaaS** for web apps (Java, .NET, Node.js, Python, PHP, Ruby, Go, Docker). You upload code; Beanstalk creates EC2, ASG, ELB, and monitoring.
- You still own the underlying resources and can tune them. You pay for those resources, not Beanstalk.
- Environment tiers: **Web server** and **Worker** (pulls from SQS).
- Deployment policies:

| Policy | Downtime | Extra capacity | Rollback |
|---|---|---|---|
| All at once | yes | no | redeploy |
| Rolling | no, reduced capacity | no | redeploy |
| Rolling with additional batch | no, full capacity | small | redeploy |
| Immutable | no | full new ASG | fast (terminate new ASG) |
| Traffic splitting | no | full new set | canary, fast |
| Blue/green (swap URLs) | no | new environment | swap back |

- Exam trigger: "developers want to deploy code without managing infrastructure, using a familiar platform".

---

## 5. Lightsail

- Simple VPS with predictable monthly price: instances, containers, managed DBs, load balancer, DNS.
- Choose for simple websites, small apps, dev/test, or users without AWS experience.
- Not for complex, highly scalable architectures (limited integration compared with EC2/VPC).

---

## 6. Containers: ECS, EKS, ECR, App Runner

### ECS

- AWS-native container orchestrator.
- **Task definition** — containers, CPU/memory, ports, env, IAM roles. **Task** — running copy. **Service** — keeps N tasks running, integrates with ELB and auto scaling.
- **Launch types**:
  - **EC2** — you manage the container instances (patching, scaling, capacity). Cheaper at steady scale, Spot capacity providers, GPUs.
  - **Fargate** — serverless, no instances to manage, pay per task vCPU/memory.
- IAM:
  - **Task role** — permissions for the application code (call S3, DynamoDB).
  - **Task execution role** — permissions for ECS agent to pull from ECR and write logs.
- Networking mode `awsvpc` gives each task its own ENI and SG (required for Fargate).
- Persistent shared storage: **EFS** (works with EC2 and Fargate).

### EKS

- Managed **Kubernetes** control plane (multi-AZ). Worker options: managed node groups, self-managed nodes, **Fargate**, Karpenter.
- Choose when the team already uses Kubernetes or needs portability across clouds/on-prem.
- Pods get VPC IPs (VPC CNI). Use **IRSA / Pod Identity** to give pods IAM permissions.

### ECR

- Private Docker/OCI registry. Integrated with IAM, encryption at rest, **image scanning**, lifecycle policies, cross-region/cross-account replication.
- Private access from a VPC: interface endpoints for ECR API and DKR plus an **S3 gateway endpoint** (image layers live in S3).

### App Runner

- Fully managed: point at source code or a container image → HTTPS service with auto scaling and load balancing.
- Least operational effort for a web app/API in a container. Less control than ECS/EKS.

| Need | Choose |
|---|---|
| Run containers, least ops, AWS-native | ECS on Fargate |
| Kubernetes required | EKS |
| Web app from code/image, zero infra decisions | App Runner |
| Cost-optimize steady container fleet, GPUs, custom AMI | ECS/EKS on EC2 (+ Spot) |

---

## 7. AWS Batch

- Managed batch scheduler: job definitions, job queues, compute environments (EC2, **Spot**, Fargate).
- Provisions the right capacity, runs containers, scales to zero.
- Choose for long-running or large-scale batch jobs (> 15 minutes, big memory/GPU) where Lambda limits don't fit.

---

## 8. Lambda

- Event-driven functions, pay per request and duration (ms). No servers.
- Limits to remember:
  - Timeout **max 15 minutes**.
  - Memory **128 MB – 10 GB** (CPU scales with memory).
  - Ephemeral `/tmp` up to **10 GB**.
  - Package: 50 MB zipped / 250 MB unzipped, or container image up to **10 GB**.
  - Default concurrency **1,000 per region** (soft limit).
- **Reserved concurrency** — caps and guarantees concurrency for one function.
- **Provisioned concurrency** — pre-warmed environments, removes **cold starts**.
- Invocation types: synchronous (API Gateway, ALB), asynchronous (S3, SNS, EventBridge — retries + DLQ/destinations), poll-based event source mappings (SQS, Kinesis, DynamoDB Streams).
- **VPC access**: Lambda creates ENIs in your subnets. To reach the internet from a VPC Lambda you need a **NAT Gateway** (a public subnet alone doesn't help). Use RDS Proxy for databases.
- Versions and aliases for safe deployments (weighted alias = canary).

> Gotcha: "Job runs 40 minutes" → not Lambda. Use Fargate, Batch, or Step Functions to split work.

### SAM and Serverless Application Repository

- **SAM** — CloudFormation extension (shorthand for Lambda, API Gateway, DynamoDB) + CLI to build, test locally, and deploy.
- **Serverless Application Repository** — catalog of published SAM apps you can deploy (public or shared privately).

### Amplify

- Build and host full-stack web/mobile apps: hosting with CI/CD from Git, auth (Cognito), APIs (AppSync/API Gateway), storage.
- Exam trigger: "front-end developers need to quickly build and host a web/mobile app with auth and backend".

---

## 9. Hybrid and edge compute

| Service | What it is | Choose when |
|---|---|---|
| **Outposts** | AWS-managed racks/servers in your data center running EC2, EBS, ECS, EKS, RDS | Low latency to on-prem systems or **data residency**, same AWS APIs on-prem |
| **ECS Anywhere / EKS Anywhere** | Run ECS tasks / EKS-distro clusters on your own hardware | Containers on-prem with AWS management, no Outposts hardware |
| **VMware Cloud on AWS** | VMware SDDC on dedicated AWS bare metal | Migrate/extend vSphere workloads without refactoring |
| **Snow Family** | Rugged devices with storage and compute (Snowball Edge) | Edge processing or offline data transfer with poor/no connectivity |
| **Local Zones / Wavelength** | AWS compute close to metros / inside 5G networks | Single-digit ms latency to end users |

> Note: Snowcone and Snowmobile are retired. Snowball Edge is the remaining option; for online transfer prefer DataSync.

---

## 10. Quick picker

| Need | Choose |
|---|---|
| Full control of the OS | EC2 |
| Interruptible batch, lowest cost | EC2 Spot / Batch on Spot |
| Steady usage, flexible instance family | Compute Savings Plan |
| BYOL per socket | Dedicated Host |
| Lowest latency between nodes | Cluster placement group (+ EFA) |
| Few critical instances, separate hardware | Spread placement group |
| MAC/IP must survive instance replacement | Secondary ENI |
| Deploy web app code, AWS manages infra | Elastic Beanstalk |
| Simple VPS, fixed monthly price | Lightsail |
| Containers without servers | ECS on Fargate |
| Kubernetes | EKS |
| Container web app, minimum config | App Runner |
| Event-driven code < 15 min | Lambda |
| AWS on-prem | Outposts |

## 11. Quick recap

- AMIs are regional. Use IAM roles, not keys. Prefer IMDSv2.
- Spot = 2-minute notice. Dedicated Host = BYOL. Savings Plans = flexible commitment.
- Cluster = performance, Spread = availability (7 per AZ), Partition = big distributed systems.
- Secondary ENI moves with its IP and MAC, same AZ only.
- Beanstalk Immutable / blue-green for safe deployments.
- ECS task role = app permissions; execution role = pull image and write logs.
- Lambda: 15 min, 10 GB memory, cold starts → provisioned concurrency, VPC internet → NAT.

## 12. Questions

See [`questions.md`](questions.md)
