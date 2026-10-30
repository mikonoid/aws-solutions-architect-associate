# 03 — Storage

> Block, file, and object storage, plus backup and hybrid. On the exam this is
> mostly "which service / class / pattern fits the constraint": durability, shared
> access, OS, latency, cost, and whether data must survive an instance stop.

---

## 1. EBS — Elastic Block Store

Network-attached **block** volumes for **one EC2 instance in the same AZ**.
Think of it as a virtual disk: you format it, mount it, and it survives stop/start.

### Volume types

| Type | Media | Boot? | When to choose |
|---|---|---|---|
| **gp3** | SSD | yes | Default general purpose. Set IOPS and throughput **independently** of size. Baseline 3,000 IOPS / 125 MB/s. |
| **gp2** | SSD | yes | Legacy general purpose. IOPS = 3 × size in GB (burst to 3,000). To get more IOPS you must grow the volume. |
| **io1 / io2** | SSD | yes | Provisioned IOPS for databases. **io2 Block Express** is the high end (up to 256,000 IOPS, 99.999% durability). |
| **st1** | HDD | **no** | Throughput-optimized, big sequential workloads (logs, big data, data warehouses). |
| **sc1** | HDD | **no** | Cold HDD. Cheapest, infrequent sequential access. |

> Note: "Need more IOPS without growing the disk" → **gp3** or **io1/io2**, not gp2.
> Note: Boot volume must be **gp2, gp3, io1, or io2**. st1/sc1 cannot be a root volume.

### Exam facts

- **AZ-scoped.** A volume can attach only to an instance in the **same AZ**.
- To move data to another AZ or Region: **snapshot → create a volume from the snapshot** in the target AZ (copy the snapshot first if cross-region).
- **Snapshots** are incremental, stored in S3 (you do not see them as a bucket), and are **regional**.
  - First snapshot is full; later ones store only changed blocks.
  - You can copy snapshots cross-region and cross-account.
  - Deleting a snapshot does not break later snapshots (AWS keeps the blocks they need).
- **Delete on termination**: root volume defaults to **yes**; extra data volumes default to **no**.
- **Encryption** uses KMS. A snapshot of an encrypted volume is always encrypted.
  - To encrypt an existing unencrypted volume: snapshot → **copy the snapshot with encryption** → create a new volume.
  - You cannot turn encryption on/off on a live volume in place.
- **Multi-Attach** is only for **io1/io2**, same AZ, and the application/filesystem must be **cluster-aware**.
  Do not multi-attach ext4/XFS and expect it to be safe.
- EBS is **durable within the AZ** (replicated inside the AZ). It is **not** multi-AZ by itself.
  For AZ failure protection you need snapshots (or a multi-AZ service such as EFS/FSx).

### Moving a volume between instances

1. Stop (or unmount and detach) the source instance's volume.
2. Detach the volume.
3. Attach it to the target instance **in the same AZ**.
4. Mount it. Data is intact because EBS is independent of the instance lifecycle (except delete-on-termination).

> Gotcha: "Instance was stopped and the extra disk is gone" is usually **delete on termination**
> or an **instance store** volume, not normal EBS behavior. Stop/start keeps EBS data.

---

## 2. Instance Store

**Ephemeral** block storage physically attached to the host. Included with some instance types
(for example storage-optimized `i3`, `i4i`, `d3`).

| | EBS | Instance Store |
|---|---|---|
| Persistence | survives stop/start | **lost** on stop, terminate, or host failure |
| Survives reboot? | yes | **yes** (reboot only) |
| Detach and move | yes (same AZ) | no |
| Performance | high, network-attached | highest, local disk |
| Use | OS, databases, anything you must keep | cache, buffers, scratch, temporary processing |

> Gotcha: "Highest IOPS, data can be rebuilt, cost matters" → **instance store**.
> "Data must survive a stop" → **EBS** (and snapshots).

---

## 3. EFS — Elastic File System

Managed **NFS** file system for **Linux**. Regional, multi-AZ by default, mounted by many
EC2 instances, ECS tasks, and Lambda at the same time.

- Protocol: **NFSv4**. Not a native Windows share (use FSx for Windows for SMB/AD).
- **Mount targets**: one ENI per AZ. Instances mount the target in their AZ. Control access with the mount target's **security group**.
- Storage classes: **Standard**, **Infrequent Access**, **Archive**. Lifecycle policies move cold files automatically.
- **One Zone** EFS is cheaper but tied to a single AZ (data lost if that AZ fails). Regional EFS is the HA default.
- Throughput modes: **Elastic** (default, scales with workload), **Provisioned**, **Bursting**.
- Encryption at rest (KMS) and in transit (TLS mount option).
- **Access points** enforce a POSIX user/group and a root directory — useful for sharing one file system across apps without giving each app the whole tree.
- Pay for storage used, not provisioned capacity (unlike EBS).

> Note: "Many Linux instances need the same files, across AZs" → **EFS**.
> "One instance, one disk, a database" → **EBS**, not EFS.

---

## 4. FSx

Managed third-party / specialized file systems. Pick by **protocol and workload**, not by "it's shared storage" alone (that is usually EFS).

| Service | Protocol | Choose when |
|---|---|---|
| **FSx for Windows File Server** | SMB, Active Directory, Windows ACLs, DFS | Windows file shares, AD-joined users, lift-and-shift Windows apps |
| **FSx for Lustre** | POSIX, parallel file system | HPC, ML training, media processing. Can be **linked to an S3 bucket** so objects appear as files and results can be written back |
| **FSx for NetApp ONTAP** | NFS, SMB, and iSCSI | Migrate on-prem NetApp. Snapshots, clones, SnapMirror, multi-protocol |
| **FSx for OpenZFS** | NFS | Linux workloads that need ZFS snapshots, clones, and compression |

- Windows FSx and ONTAP can be **Single-AZ or Multi-AZ**.
- Lustre is the answer when the stem says **high throughput parallel processing of data already in S3**.
- EFS is still the default simple shared Linux NFS answer. FSx is for a specific OS, protocol, or migration.

---

## 5. S3 — object storage

Regional buckets, global names, **11 nines durability** (99.999999999%). Objects are immutable
key/value blobs plus metadata. Max object size **5 TB**. A single PUT is limited to **5 GB**;
use **multipart upload** above 100 MB and it is required above 5 GB.

Consistency: S3 is **strongly consistent** for PUT and DELETE (read-after-write and list).
You no longer need to design around eventual consistency.

### Storage classes

| Class | AZs | Retrieval | Min storage duration | Use |
|---|---|---|---|---|
| **Standard** | 3+ | ms, no retrieval fee | none | Frequent access |
| **Intelligent-Tiering** | 3+ | ms for frequent/IA tiers | none for those tiers | Unknown or changing access. Small monitoring fee, **no retrieval fee**. Archive tiers are opt-in |
| **Standard-IA** | 3+ | ms + retrieval fee | 30 days, 128 KB min | Infrequent, but must survive AZ loss |
| **One Zone-IA** | **1** | ms + retrieval fee | 30 days | Infrequent and **recreatable** (thumbnails, secondary copies) |
| **Glacier Instant Retrieval** | 3+ | **milliseconds** | 90 days | Archive accessed about once a quarter, still needs instant access |
| **Glacier Flexible Retrieval** | 3+ | minutes to hours | 90 days | Expedited (1–5 min), Standard (3–5 h), Bulk (5–12 h) |
| **Glacier Deep Archive** | 3+ | **12–48 hours** | 180 days | Cheapest, rarely accessed compliance archives |

> Gotcha: Moving objects to Glacier/IA and deleting them early still bills the **minimum duration**.
> Gotcha: "Millisecond access to archive" is **Glacier Instant Retrieval**, not Flexible or Deep Archive.
> Gotcha: One Zone-IA is wrong if the stem says the data cannot be recreated or must survive AZ failure.

**Lifecycle rules** transition objects between classes and can expire current versions, noncurrent versions, and incomplete multipart uploads.

### Versioning

- Off by default. Once enabled, you can **suspend** it but not fully turn it off.
- Overwrites keep the old version. A delete without a version ID creates a **delete marker**; the bytes are still there.
- To really remove an object, delete by **version ID**.
- Required for **replication** (CRR/SRR).
- **MFA Delete** (enabled by the root user) protects against accidental permanent deletes.

### Access control

- **IAM policies** — who (principal) can call S3.
- **Bucket policy** — resource policy on the bucket. This is how you grant cross-account, public website, or VPC-endpoint-only access.
- **ACLs** — legacy per-object grants. Best practice is **Bucket owner enforced** (ACLs disabled). New buckets default this way.
- **Block Public Access** — account and bucket guardrail. It overrides a bucket policy that would make objects public. Turn it off only when you truly need public reads (static website).
- Effective permission is the **intersection** of IAM and the bucket policy, unless an explicit deny exists (deny always wins).

### Static website hosting

- Enable website hosting and set an index document (and optional error document).
- The website endpoint is **HTTP only**. For HTTPS, put **CloudFront** in front (custom domain + ACM cert in us-east-1).
- Public reads need a bucket policy and Block Public Access relaxed for that bucket.
- Do not confuse the REST endpoint (`bucket.s3.region.amazonaws.com`) with the website endpoint (`bucket.s3-website-region.amazonaws.com`).

### Pre-signed URLs

- Temporary URL that carries the **signer's** permissions. Anyone with the URL can perform that one action until it expires.
- Works for **download and upload**.
- The URL stops working if the signer's credentials expire first (watch IAM role session length).
- Use this for private objects you want to share briefly. Do not make the bucket public.

### Access Points

- Named entry points on a bucket, each with its own policy. Useful when many teams/apps share one bucket.
- An access point can be restricted to a **VPC** (traffic stays on PrivateLink / the VPC endpoint).
- **Multi-Region Access Points** give one global hostname and route to the nearest bucket replica (needs Cross-Region Replication).

### Other exam features

- **Replication** (same-region or cross-region) needs versioning on source and destination. It replicates **new** objects by default; existing objects need **S3 Batch Replication**. Delete markers are optional. Encrypted objects need the right KMS key policy.
- **S3 Replication Time Control (RTC)** gives a 15-minute replication SLA.
- **Event notifications** → Lambda, SNS, SQS, or EventBridge. Use for "on upload, do X".
- **Transfer Acceleration** uploads via CloudFront edge locations. Useful for distant clients and large objects.
- **Object Lock** (WORM): Governance mode (privileged users can bypass) or Compliance mode (nobody can delete until retention ends). Plus **legal hold**.
- **Requester Pays** makes the requester pay transfer costs.
- Encryption: **SSE-S3** (default, AES-256), **SSE-KMS** (CloudTrail audit, CMK, bucket key to cut KMS cost), **SSE-C** (you send the key with each request), or client-side.
- Private access from a VPC without NAT: **Gateway Endpoint** (free) plus an endpoint policy. See the VPC topic.

---

## 6. AWS Backup

Central backup **policies** across services: EC2/EBS, RDS/Aurora, DynamoDB, EFS, FSx, S3, Storage Gateway, and others.

- A **backup plan** says what to protect, how often, and how long to keep it.
- Recovery points land in a **backup vault**. You can copy vaults **cross-region and cross-account**.
- **Vault Lock** makes recovery points immutable (WORM) for compliance — stronger than "we promise not to delete snapshots".
- Use AWS Backup when the requirement is **centralized, auditable, cross-service backup**, not when you only need one EBS snapshot.

---

## 7. Elastic Disaster Recovery (DRS)

Continuous **block-level replication** of whole servers (on-prem or cloud) into a staging area in AWS.

- **RPO** in seconds, **RTO** in minutes (launch recovery instances from the replicated disks).
- Supports non-disruptive **drills**.
- Choose DRS when you must fail over **entire machines** (OS + apps + data), especially from on-prem.
- Choose **AWS Backup** when the need is scheduled backups and restores, not live server replication.

---

## 8. Storage Gateway

Hybrid appliance (VM or hardware) that presents cloud storage to on-prem systems.

| Mode | Presents | Data lives | Choose when |
|---|---|---|---|
| **S3 File Gateway** | NFS or SMB | Objects in S3 | On-prem apps should keep using a file share, but data should land in S3 |
| **Volume Gateway — Cached** | iSCSI | Primary in S3, hot data cached locally | Dataset is larger than local disk |
| **Volume Gateway — Stored** | iSCSI | Full copy on-prem, async backup to S3 | Low-latency local access plus AWS backup |
| **Tape Gateway** | iSCSI VTL | Virtual tapes in S3, archive to Glacier | Replace physical tape libraries without changing backup software |

> Note: File Gateway stores **objects**. Volume Gateway stores **EBS snapshots** of the volumes. Tape Gateway stores **virtual tapes**.

---

## 9. Quick picker

| Need | Choose |
|---|---|
| Single-instance disk, same AZ, survives stop | EBS (gp3 default) |
| Database needing guaranteed IOPS | io2 / io2 Block Express |
| Scratch disk, highest IOPS, data is disposable | Instance store |
| Shared Linux files across AZs | EFS |
| Windows SMB + Active Directory | FSx for Windows File Server |
| HPC / process S3 data as a fast file system | FSx for Lustre |
| NetApp migration, NFS+SMB+iSCSI | FSx for NetApp ONTAP |
| Cheap durable objects, any size, HTTP API | S3 Standard |
| Unknown access pattern | S3 Intelligent-Tiering |
| Rarely accessed, millisecond reads, multi-AZ | Glacier Instant Retrieval or Standard-IA |
| Cheapest long-term archive, hours to restore | Glacier Deep Archive |
| One policy to back up many services, immutable vault | AWS Backup (+ Vault Lock) |
| Fail over whole servers with seconds of data loss | Elastic Disaster Recovery |
| On-prem file share backed by S3 | S3 File Gateway |

## 10. Quick recap

- EBS = one AZ, one instance (except io1/io2 Multi-Attach). Snapshots move data across AZs/regions.
- gp3 sets IOPS and throughput without resizing. st1/sc1 cannot boot.
- Instance store survives reboot only. Stop/terminate wipes it.
- EFS = shared Linux NFS. FSx Windows = SMB/AD. Lustre = HPC + S3. ONTAP = NetApp. OpenZFS = ZFS/NFS.
- S3: 11 nines, strong consistency, multipart over 5 GB, versioning required for replication.
- Block a public bucket with **Block Public Access**. Prefer bucket policies over ACLs.
- Website endpoint is HTTP; HTTPS means CloudFront. Private share means a **pre-signed URL**.
- Backup = scheduled restore points. DRS = continuous server replication. Gateway = hybrid on-prem front door.

## 11. Questions

See [`questions.md`](questions.md)
