# 03 — Storage: practice questions

> Format: question → options → correct answer with why the others are wrong.
> Try to answer first, then expand the answer.

---

### Q1. A database on EC2 needs 16,000 IOPS, but you do not want to grow the disk just to get performance. The volume is gp2. What is the best change?

- A. Keep gp2 and increase the volume size until IOPS = 3 × size reaches 16,000
- B. Switch to gp3 or io2 and provision the IOPS independently of size
- C. Switch to st1 and enable bursting
- D. Move the database to instance store so IOPS are free

<details><summary>Answer</summary>

**B.** **gp3** and **io1/io2** let you provision IOPS (and, for gp3, throughput) without
resizing. io2 is the safer pick if you also need high durability and sustained IOPS.

- A — works on gp2, but you pay for capacity you do not need. The question asks for the better change.
- C — st1 is HDD, cannot be a boot volume, and is for sequential throughput, not database IOPS.
- D — instance store is lost on stop or host failure. Wrong for a database.
</details>

---

### Q2. An EBS volume in `us-east-1a` must be attached to a new instance in `us-east-1b`. What do you do?

- A. Detach the volume and attach it to the instance in `us-east-1b`
- B. Snapshot the volume, then create a new volume from that snapshot in `us-east-1b`
- C. Enable Multi-Attach and attach the same volume to both AZs
- D. Copy the volume with AWS Backup and restore it onto instance store

<details><summary>Answer</summary>

**B.** EBS volumes are **AZ-scoped**. Snapshots are regional, so you create a new volume
from the snapshot in the target AZ.

- A — attach only works inside the same AZ.
- C — Multi-Attach is same-AZ only, and only for io1/io2.
- D — instance store is the wrong durability target, and Backup is unnecessary for a one-time move.
</details>

---

### Q3. A team uses a storage-optimized EC2 instance for video transcoding. Scratch files are huge and can be regenerated. They want the highest throughput and no extra storage bill. What storage do they use?

- A. gp3
- B. EFS Standard
- C. Instance store
- D. S3 Standard mounted with Mountpoint, required for all scratch

<details><summary>Answer</summary>

**C.** **Instance store** is local to the host, very fast, and included with the instance.
Data loss on stop is acceptable because the files can be regenerated.

- A — durable and billed, but not the fastest or the cheapest for disposable scratch.
- B — shared and durable, overkill and slower for local scratch.
- D — possible for some pipelines, but not the best match for "highest throughput, no extra bill".
</details>

---

### Q4. Twenty Linux EC2 instances across three AZs must read and write the same configuration and content files. What do you choose?

- A. One gp3 volume with Multi-Attach
- B. EFS with a mount target in each AZ
- C. FSx for Windows File Server
- D. Instance store replicated by a cron job

<details><summary>Answer</summary>

**B.** **EFS** is a regional NFS file system. Instances in each AZ mount the local mount
target and see the same files.

- A — Multi-Attach is io1/io2 only, same AZ, and needs a cluster-aware filesystem.
- C — Windows/SMB/AD, not the Linux NFS answer.
- D — not durable or strongly shared.
</details>

---

### Q5. A Windows application must move to AWS unchanged. Users authenticate with Active Directory and access shares over SMB with Windows ACLs. What do you choose?

- A. EFS
- B. FSx for Lustre
- C. FSx for Windows File Server
- D. S3 with a bucket policy per user

<details><summary>Answer</summary>

**C.** **FSx for Windows File Server** speaks SMB, joins Active Directory, and enforces
Windows ACLs. Multi-AZ is available when the share must survive an AZ failure.

- A — NFS for Linux, not AD/SMB.
- B — parallel POSIX for HPC, not Windows shares.
- D — object storage, not a Windows file share.
</details>

---

### Q6. A genomics pipeline must process terabytes that already sit in S3, with thousands of parallel readers and high throughput. Results should be written back to S3. What fits?

- A. EBS io2 on one large instance
- B. FSx for Lustre linked to the S3 bucket
- C. S3 Glacier Flexible Retrieval
- D. Volume Gateway Stored mode

<details><summary>Answer</summary>

**B.** **FSx for Lustre** is the parallel file system for HPC/ML. Linked to S3, objects
are presented as files and results can be exported back to the bucket.

- A — one instance cannot give thousands of parallel readers shared access.
- C — archive class, wrong performance tier.
- D — hybrid block gateway, not an HPC file system.
</details>

---

### Q7. Objects are accessed unpredictably: some daily, some never. You want automatic cost optimization and no retrieval fees. What storage class?

- A. S3 Standard-IA
- B. S3 Intelligent-Tiering
- C. S3 Glacier Deep Archive
- D. S3 One Zone-IA

<details><summary>Answer</summary>

**B.** **Intelligent-Tiering** moves objects between frequent and infrequent tiers for you.
There is a small monitoring fee and **no retrieval fee**.

- A — you must already know access is infrequent, and reads have a retrieval fee.
- C — hours to restore, retrieval fees, 180-day minimum.
- D — single AZ and a retrieval fee. Wrong if access is unknown.
</details>

---

### Q8. Compliance archives are read at most once a quarter, must survive an AZ loss, and analysts need the object in milliseconds when they do open it. Which class?

- A. S3 Glacier Deep Archive
- B. S3 Glacier Flexible Retrieval
- C. S3 Glacier Instant Retrieval
- D. S3 One Zone-IA

<details><summary>Answer</summary>

**C.** **Glacier Instant Retrieval** is archive pricing with **millisecond** access and
multi-AZ durability. Minimum storage duration is 90 days.

- A — 12–48 hours, cheapest, too slow.
- B — minutes to hours, not milliseconds.
- D — milliseconds, but one AZ, so it fails the durability requirement.
</details>

---

### Q9. A user deletes an object from a versioned bucket and then says the file is gone. The bucket owner can still see prior versions. What happened, and how do you restore it?

- A. Versioning permanently destroyed every version. Restore from a snapshot only.
- B. The delete created a delete marker. Delete the delete marker (or GET a specific version ID) to restore the object.
- C. S3 is eventually consistent, so the object will reappear in 24 hours.
- D. Block Public Access hid the object. Disable it.

<details><summary>Answer</summary>

**B.** A delete **without a version ID** writes a **delete marker**. The data versions
remain. Remove the delete marker and the latest version becomes visible again.

- A — versioning exists specifically so a simple delete does not destroy history.
- C — S3 is strongly consistent. This is not a consistency delay.
- D — Block Public Access does not hide objects from the owner.
</details>

---

### Q10. A private bucket must let a partner download one object for 15 minutes. The bucket must stay private. What do you use?

- A. Disable Block Public Access and add a public ACL on the object
- B. A pre-signed URL generated by an IAM principal that can read the object
- C. Enable static website hosting
- D. An S3 Access Point with a policy of Principal "*"

<details><summary>Answer</summary>

**B.** A **pre-signed URL** grants the signer's permissions for a limited time. The bucket
stays private.

- A — makes the object public. Opposite of the requirement.
- C — website hosting is for public HTTP sites, not a one-object private share.
- D — `"*"` is public access, and an access point does not by itself expire in 15 minutes.
</details>

---

### Q11. You enabled S3 static website hosting. `https://` to the website endpoint fails, while `http://` works. What is the right fix if the site needs HTTPS and a custom domain?

- A. S3 website endpoints support HTTPS. Request an ACM certificate in the bucket's region and attach it to the bucket.
- B. Put CloudFront in front of the website endpoint and attach an ACM certificate in us-east-1
- C. Switch the bucket to One Zone-IA
- D. Use a pre-signed URL as the site homepage

<details><summary>Answer</summary>

**B.** The S3 **website endpoint is HTTP only**. **CloudFront** terminates HTTPS. ACM
certificates for CloudFront must be in **us-east-1**.

- A — you cannot attach an ACM cert directly to an S3 website endpoint.
- C — storage class does not enable HTTPS.
- D — pre-signed URLs are for temporary private access, not site hosting.
</details>

---

### Q12. An existing unencrypted EBS volume must become encrypted. The instance can be stopped briefly. What is the path?

- A. Modify the volume and toggle encryption on
- B. Snapshot → copy the snapshot with encryption enabled → create a new volume → swap it in
- C. Enable default EBS encryption. The existing volume encrypts in place within an hour
- D. Detach the volume and attach it to a KMS key

<details><summary>Answer</summary>

**B.** Encryption is set when the volume (or snapshot copy) is **created**. You cannot
flip the flag on a live unencrypted volume. Default encryption only affects **new** volumes.

- A — no in-place toggle.
- C — default encryption is not retroactive.
- D — KMS keys are not attached to volumes that way.
</details>

---

### Q13. A company wants one backup policy for EC2, EFS, and FSx, with copies in a second region that nobody can delete before 1 year. What do you use?

- A. Independent EBS snapshots plus a calendar reminder
- B. AWS Backup plans with cross-region copy and a locked backup vault
- C. Elastic Disaster Recovery only
- D. S3 Lifecycle to Glacier Deep Archive

<details><summary>Answer</summary>

**B.** **AWS Backup** centralizes plans across those services. Cross-region copy sends
recovery points to another region. **Vault Lock** (or a locked vault in compliance mode)
makes them immutable for the retention period.

- A — not centralized, not immutable, easy to miss a service.
- C — DRS replicates live servers for failover. It is not the backup-policy answer.
- D — lifecycle does not back up EBS, EFS, or FSx.
</details>

---

### Q14. On-prem backup software must keep writing to tapes, but the company wants to stop buying tape hardware and archive to AWS. What gateway mode?

- A. S3 File Gateway
- B. Volume Gateway Cached
- C. Tape Gateway
- D. FSx for OpenZFS

<details><summary>Answer</summary>

**C.** **Tape Gateway** presents an iSCSI virtual tape library. Existing backup software
keeps its tape workflow. Tapes land in S3 and can archive to Glacier.

- A — file share (NFS/SMB) to S3 objects. The backup app would have to change off tapes.
- B — block volumes, not tapes.
- D — a cloud file system, not a VTL.
</details>

---

### Q15. RPO must be a few seconds and RTO a few minutes for hundreds of on-prem servers, including the OS and installed applications. What do you choose?

- A. AWS Backup nightly
- B. Elastic Disaster Recovery
- C. S3 Sync every hour
- D. Storage Gateway Stored volumes

<details><summary>Answer</summary>

**B.** **Elastic Disaster Recovery** continuously replicates block devices of whole
servers. You can drill and fail over to EC2 with a very low RPO/RTO.

- A — nightly backup cannot give a seconds-level RPO.
- C — file sync misses the OS and is too slow.
- D — hybrid volume backup, not full server orchestration with minute-level RTO.
</details>

---

### Q16. Two applications share one data lake bucket. Each app must be limited to its own prefix, and one app must reach S3 only from a VPC. What helps?

- A. Two buckets, because a bucket can have only one policy
- B. S3 Access Points, one per app, with the sensitive app's access point restricted to the VPC
- C. Object ACLs only
- D. Pre-signed URLs baked into the AMI

<details><summary>Answer</summary>

**B.** **Access Points** give each application its own hostname and policy, including a
policy that allows traffic only from a VPC. The bucket can still have a guardrail policy.

- A — a bucket has one bucket policy, but access points exist so you do not need a new bucket per app.
- C — ACLs are legacy and a poor fit for prefix-and-VPC rules. Prefer Bucket owner enforced.
- D — pre-signed URLs expire and are not an access-control design.
</details>
