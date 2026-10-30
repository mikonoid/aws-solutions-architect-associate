# 02 — Compute: practice questions

> Format: question → options → correct answer with why the others are wrong.
> Try to answer first, then expand the answer.

---

### Q1. A nightly data processing job takes 3 hours, can be restarted from checkpoints, and must be as cheap as possible. What EC2 option?

- A. On-Demand
- B. Reserved Instances (3-year)
- C. Spot Instances
- D. Dedicated Hosts

<details><summary>Answer</summary>

**C.** **Spot** gives up to ~90% off. The job tolerates interruption (2-minute notice) because it checkpoints.

- A — works but costs the most.
- B — commitment for something that runs 3 hours a day is wasteful.
- D — for licensing/compliance, most expensive.
</details>

---

### Q2. Software licenses are bound to the number of physical CPU sockets and cores. The company wants to bring these licenses to AWS. What do you use?

- A. Dedicated Instances
- B. Dedicated Hosts
- C. Spot Instances
- D. Compute Savings Plan

<details><summary>Answer</summary>

**B.** **Dedicated Hosts** give visibility and control of sockets/cores on a physical server — required for BYOL licensing.

- A — hardware isolation, but no socket/core visibility.
- C/D — pricing models, not licensing control.
</details>

---

### Q3. An HPC application needs the lowest possible network latency and highest throughput between 20 instances. What do you configure?

- A. Spread placement group across 3 AZs
- B. Partition placement group
- C. Cluster placement group in one AZ, with enhanced networking/EFA
- D. Instances in different regions with peering

<details><summary>Answer</summary>

**C.** **Cluster** puts instances close together in one AZ. EFA adds OS-bypass for MPI workloads.

- A — spread maximizes separation, not latency.
- B — partitions are for rack-aware distributed systems.
- D — cross-region adds latency.
</details>

---

### Q4. Five critical instances must each run on separate underlying hardware so that one hardware failure affects at most one instance. What fits?

- A. Cluster placement group
- B. Spread placement group
- C. Partition placement group with one partition
- D. A single Dedicated Host

<details><summary>Answer</summary>

**B.** **Spread** places each instance on distinct hardware (max 7 per AZ per group).

- A — instances share proximity, a higher blast radius.
- C — one partition means instances share racks.
- D — one host is a single point of failure.
</details>

---

### Q5. An application on EC2 needs to read from S3. What is the most secure way to give it credentials?

- A. Store an IAM user's access keys in a config file on the instance
- B. Put the keys in user data
- C. Attach an IAM role through an instance profile
- D. Make the bucket public

<details><summary>Answer</summary>

**C.** An **IAM role** delivers short-lived credentials through instance metadata, rotated automatically.

- A/B — long-lived keys that can leak (user data is readable from the instance).
- D — exposes data to everyone.
</details>

---

### Q6. A licensed app is tied to a MAC address. If the instance fails, a replacement must take over with the same MAC and private IP. What do you use?

- A. Elastic IP
- B. A secondary ENI moved to the replacement instance in the same AZ
- C. Create a new AMI with the MAC address
- D. Use an instance store volume

<details><summary>Answer</summary>

**B.** A **secondary ENI** keeps its MAC, private IP, and SGs when moved to another instance in the same AZ.

- A — only a public IPv4, not the MAC.
- C — MACs belong to ENIs, not AMIs.
- D — storage, unrelated.
</details>

---

### Q7. Developers want to deploy a Java web app by uploading code. They do not want to design EC2, load balancers, or auto scaling themselves, but they want access to the underlying resources if needed. What do you recommend?

- A. Lambda
- B. Elastic Beanstalk
- C. Lightsail
- D. EC2 with a hand-built ASG

<details><summary>Answer</summary>

**B.** **Elastic Beanstalk** provisions EC2, ASG, ELB, and monitoring for supported platforms, while you keep control over the resources.

- A — would require refactoring into functions; long-running app server isn't a fit.
- C — simple VPS, less suited to auto scaling web apps.
- D — the opposite of "not designing it themselves".
</details>

---

### Q8. A Beanstalk deployment must have no downtime, keep full capacity, and allow quick rollback by terminating new instances if health checks fail. Which policy?

- A. All at once
- B. Rolling
- C. Immutable
- D. Rolling with additional batch

<details><summary>Answer</summary>

**C.** **Immutable** launches a new set of instances in a temporary ASG. If they fail, they are terminated and the old ones keep serving.

- A — downtime.
- B — reduced capacity during deploy, rollback means redeploying.
- D — keeps capacity, but rollback is a redeploy over updated instances.
</details>

---

### Q9. A team wants to run containers without provisioning or patching servers, and pay only for vCPU and memory used by tasks. What do you choose?

- A. ECS with the EC2 launch type
- B. ECS with the Fargate launch type
- C. EC2 with Docker installed
- D. EKS with self-managed nodes

<details><summary>Answer</summary>

**B.** **Fargate** is serverless compute for containers. No instances to manage.

- A/C/D — all require managing EC2 instances.
</details>

---

### Q10. An ECS task fails to start with an error pulling the image from ECR. The application itself only needs S3 access. Which role must allow ECR pulls and CloudWatch Logs writes?

- A. Task role
- B. Task execution role
- C. EC2 instance profile of the bastion host
- D. Service-linked role for Auto Scaling

<details><summary>Answer</summary>

**B.** The **task execution role** is used by the ECS agent to pull images and send logs.

- A — task role is for the app code (S3 calls).
- C/D — unrelated.
</details>

---

### Q11. A company already runs Kubernetes on-prem and wants the same tooling and manifests on AWS with a managed control plane. What do you choose?

- A. ECS
- B. EKS
- C. App Runner
- D. Elastic Beanstalk

<details><summary>Answer</summary>

**B.** **EKS** runs upstream Kubernetes with a managed, multi-AZ control plane.

- A — AWS-native orchestrator, different API.
- C/D — higher-level platforms, not Kubernetes.
</details>

---

### Q12. A Lambda function must process a video file and needs about 45 minutes. What should you do?

- A. Increase the Lambda timeout to 60 minutes
- B. Increase memory to 10 GB so it finishes in 15 minutes, guaranteed
- C. Run it as an ECS Fargate task or an AWS Batch job
- D. Use provisioned concurrency

<details><summary>Answer</summary>

**C.** Lambda's maximum timeout is **15 minutes**. Long-running work fits Fargate or Batch.

- A — not possible.
- B — more memory may help but cannot guarantee it.
- D — removes cold starts; doesn't extend the timeout.
</details>

---

### Q13. A latency-sensitive API on Lambda shows slow first responses after idle periods. What reduces this?

- A. Reserved concurrency
- B. Provisioned concurrency
- C. Increasing the timeout
- D. Moving the function into a VPC

<details><summary>Answer</summary>

**B.** **Provisioned concurrency** keeps execution environments initialized, removing **cold starts**.

- A — caps/guarantees concurrency but doesn't pre-warm.
- C — unrelated to startup latency.
- D — doesn't remove cold starts.
</details>

---

### Q14. A Lambda function was moved into private subnets to reach an RDS database. Now its calls to a public third-party API time out. What fixes it?

- A. Attach an Internet Gateway to the private subnets' route table directly
- B. Give the Lambda function a public IP
- C. Add a NAT Gateway in a public subnet and route the private subnets' 0.0.0.0/0 to it
- D. Add a VPC gateway endpoint for the third-party API

<details><summary>Answer</summary>

**C.** VPC-attached Lambda ENIs only have private IPs. Internet access requires a **NAT Gateway**.

- A — Lambda ENIs don't get public IPs, so an IGW route alone doesn't work.
- B — not supported.
- D — gateway endpoints exist only for S3 and DynamoDB.
</details>

---

### Q15. A large number of batch jobs need GPUs and should automatically use Spot capacity, scaling down to zero when the queue is empty. What fits best?

- A. Lambda
- B. AWS Batch with a Spot compute environment
- C. Lightsail
- D. Elastic Beanstalk worker tier with On-Demand

<details><summary>Answer</summary>

**B.** **AWS Batch** manages queues and compute environments, supports GPUs and Spot, and scales to zero.

- A — no GPUs, 15-minute limit.
- C — not a batch scheduler.
- D — possible, but more work and not cost-optimized.
</details>

---

### Q16. A manufacturer needs to run EC2 and RDS on-premises next to factory systems for single-digit-millisecond latency, using the same AWS APIs. What do you choose?

- A. VMware Cloud on AWS
- B. AWS Outposts
- C. Snowball Edge
- D. Local Zones

<details><summary>Answer</summary>

**B.** **Outposts** installs AWS-managed infrastructure in your facility with the same AWS services and APIs.

- A — runs VMware on AWS hardware in AWS regions.
- C — rugged edge device for disconnected sites/data transfer, not a full on-prem AWS region extension.
- D — AWS-run sites in metro areas, not inside your factory.
</details>

---

### Q17. A company wants to keep its golden AMIs patched and hardened automatically every week and distribute them to two regions. What do you use?

- A. EC2 Image Builder
- B. Systems Manager Run Command only
- C. Elastic Beanstalk
- D. Copy the AMI manually

<details><summary>Answer</summary>

**A.** **Image Builder** pipelines build, test, and distribute images on a schedule.

- B — runs commands but doesn't manage the image pipeline.
- C — app deployment platform.
- D — manual, error-prone.
</details>
