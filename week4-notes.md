# Week 4 — Shared Storage (NFS/EFS), IAM, Directory Services & S3

> Training notes: NFS server on EC2 → EFS → IAM (users/groups/policies/roles + JSON) → LDAP/AD overview → S3 (hosting, permissions, versioning, CLI, replication, storage classes).

---

## Part 1: Shared File Storage — NFS then EFS

### 1.1 NFS Server on a Plain EC2 Instance

Before using AWS's managed EFS, standing up **NFS (Network File System)** manually on an EC2 instance shows what a managed file service is actually doing under the hood.

**Server side (the instance exporting storage):**
```bash
sudo yum install -y nfs-utils        # or apt install nfs-kernel-server
sudo mkdir -p /shared
sudo chown nobody:nobody /shared

# /etc/exports — defines what's shared, with whom, and with what rights
echo "/shared 10.0.0.0/16(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

sudo exportfs -arv
sudo systemctl enable --now nfs-server
```

Key `/etc/exports` options:
- `rw` / `ro` — read-write or read-only.
- `sync` — writes flushed to disk before acknowledging the client (safer, slower) vs `async`.
- `no_root_squash` — client's root user retains root privileges on the share (`root_squash`, the default, maps remote root to `nobody` for security).
- `no_subtree_check` — disables extra checks on subdirectory exports; improves reliability for mounts of a whole filesystem.

**Client side (any instance mounting the share):**
```bash
sudo yum install -y nfs-utils
sudo mkdir -p /mnt/shared
sudo mount -t nfs 10.0.1.5:/shared /mnt/shared
```

**Limitations this exposes:** the NFS server is a **single EC2 instance** — single point of failure, fixed EBS capacity you must manually resize, no built-in multi-AZ redundancy, and you own patching/scaling the NFS daemon yourself. This is exactly the gap **EFS** fills.

### 1.2 EFS (Elastic File System)

EFS is AWS's **fully managed, elastic, multi-AZ NFS (v4.1) service** — no server to patch, capacity grows/shrinks automatically, and it can be mounted concurrently by thousands of EC2 instances/containers/Lambda functions across multiple AZs at once.

Setup flow:
1. Create an **EFS file system** (choose Standard or One Zone storage class).
2. Create **Mount Targets** in each subnet/AZ you want to mount from (each mount target gets an ENI + IP in that subnet).
3. Attach a **Security Group** to the mount target allowing NFS (port **2049**) from your instances' security group.
4. Mount on the client exactly like NFS, using the EFS DNS name:
```bash
sudo mount -t efs -o tls fs-0123456789abcdef:/ /mnt/efs
# or via the amazon-efs-utils helper
sudo yum install -y amazon-efs-utils
sudo mount -t efs fs-0123456789abcdef:/ /mnt/efs
```

**EFS vs manual NFS-on-EC2:**

| | Self-managed NFS on EC2 | EFS |
|---|---|---|
| Availability | Single instance = single point of failure | Natively multi-AZ, highly available |
| Scaling | Manual EBS resize | Elastic, pay-per-GB actually stored |
| Patching/ops | You manage the NFS daemon, OS, kernel | Fully managed by AWS |
| Concurrent mounts | Possible but you manage locking/contention | Designed for thousands of concurrent clients |
| Performance modes | Whatever the instance/EBS gives you | General Purpose vs Max I/O; Bursting vs Provisioned throughput |
| Storage classes | N/A | Standard, Standard-IA, One Zone, One Zone-IA, with lifecycle management moving cold files automatically |

**Note:** EFS is NFS — it works with Linux-based POSIX file access. It's not a block-storage replacement for EBS, and not the same as FSx (used for Windows/SMB or Lustre HPC needs).

---

## Part 2: IAM — Users, Groups, Policies, Roles

### 2.1 IAM Users

An **IAM User** is an identity representing a person or application that interacts with AWS, with its own long-term credentials (console password and/or access keys). Best practice: don't use the **root account** for daily work — create individual IAM users, enable **MFA**, and follow **least privilege**.

### 2.2 IAM Groups

A **Group** is a collection of IAM users. You attach policies to the group instead of to each user individually — anyone added to the group inherits its permissions. Groups cannot be nested, and a group is not itself an identity (you can't log in "as a group," and roles can't be assumed by groups).

### 2.3 IAM Policies

A **Policy** is a JSON document that explicitly defines **what actions are allowed or denied on which resources, under what conditions.**

Anatomy of a policy document:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-training-bucket",
        "arn:aws:s3:::my-training-bucket/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

Key fields:
- **Version** — policy language version, almost always `"2012-10-17"`.
- **Effect** — `Allow` or `Deny`. An explicit `Deny` always overrides any `Allow`, anywhere in the evaluation (even from a different attached policy).
- **Action** — the API operations covered (`service:Action`), supports wildcards (`s3:*`).
- **Resource** — the ARN(s) the statement applies to; `*` means all resources of that type.
- **Condition** (optional) — narrows when the statement applies (source IP, MFA present, time window, tags, etc.).

**Policy types:**
- **Identity-based policies** — attached to a user, group, or role (what we wrote above).
- **Resource-based policies** — attached directly to a resource (e.g., an **S3 bucket policy**), specifying who (which principal) can access it — this is how cross-account access to S3 is granted.
- **AWS Managed Policies** — pre-built by AWS (e.g., `AmazonS3ReadOnlyAccess`), not editable.
- **Customer Managed Policies** — your own reusable JSON policies.
- **Inline Policies** — embedded directly on a single user/group/role, not reusable, 1:1 relationship.
- **Permissions Boundaries** — an advanced guardrail policy that caps the *maximum* permissions an identity can ever have, regardless of what other policies grant it.

**Policy evaluation logic (simplified):** default is implicit deny → check for any explicit Deny (wins immediately) → check for any explicit Allow → if none found, implicit deny stands.

### 2.4 IAM Roles

A **Role** is an identity with permissions policies attached — but **no long-term credentials**. Instead, it's **assumed** temporarily (via STS, generating short-lived credentials) by a **trusted principal**: an EC2 instance, a Lambda function, another AWS account, a federated user, or a human via SSO.

Why roles matter in practice:
- Attaching a role to an EC2 instance (**Instance Profile**) lets your app call AWS APIs (e.g., read from S3) **without hardcoding access keys** on the instance — this is the single most important IAM best practice for EC2 workloads.
- Cross-account access works by one account's role **trusting** another account's principal to assume it (`sts:AssumeRole`), instead of sharing IAM users/keys across accounts.
- Roles have a **Trust Policy** (who can assume the role) separate from their **Permissions Policy** (what the role can do once assumed).

### 2.5 Practical Difference: User/Group/Policy/Role Together

```
IAM User "dev-alice"
      │ member of
      ▼
IAM Group "developers"  ── has attached ──▶  Policy "DevS3Access.json" (identity-based, JSON)
      │
      │ (separately) dev-alice can also assume:
      ▼
IAM Role "CI-Deploy-Role"  ── trust policy allows: dev-alice / GitHub Actions OIDC
      │ has attached
      ▼
Policy "DeployPermissions.json"
```

- **User** = who's logging in.
- **Group** = a bucket of users sharing the same baseline permissions.
- **Policy** = the actual JSON rules (the "what's allowed").
- **Role** = a temporary hat anyone/anything trusted can put on, without permanent credentials.

### 2.6 ARN Structure (Amazon Resource Name)

Every resource in AWS — a bucket, a user, an EC2 instance, a role — has a globally unique identifier called an **ARN**. It's what goes in the `Resource` field of every policy you write, so understanding its structure is what makes writing correct JSON policies (and debugging "access denied") straightforward instead of guesswork.

**General format:**
```
arn:partition:service:region:account-id:resource-id
arn:partition:service:region:account-id:resource-type/resource-id
arn:partition:service:region:account-id:resource-type:resource-id
```

Field by field:

| Field | Meaning | Notes |
|---|---|---|
| `arn` | Literal prefix | Always literally `arn` |
| `partition` | Which AWS "universe" | `aws` (standard regions), `aws-cn` (China regions), `aws-us-gov` (GovCloud) |
| `service` | The AWS service namespace | `s3`, `iam`, `ec2`, `lambda`, `rds`, `sts`, etc. |
| `region` | Region the resource lives in | Often **blank** for global services (IAM, S3 buckets, Route 53) |
| `account-id` | 12-digit AWS account ID | Often **blank** for resources without account-level ownership (e.g., S3 buckets, since bucket names are globally unique on their own) |
| `resource-id` / `resource-type` | The specific resource, and how it's separated | Varies per service — some use `/`, some use `:`, some have no type at all |

**Worked examples, side by side:**

```
S3 bucket:            arn:aws:s3:::my-training-bucket
S3 object:             arn:aws:s3:::my-training-bucket/path/to/file.txt
                     (region & account-id fields are BLANK — S3 bucket names are globally unique)

IAM user:              arn:aws:iam::123456789012:user/dev-alice
IAM role:              arn:aws:iam::123456789012:role/CI-Deploy-Role
IAM policy:            arn:aws:iam::123456789012:policy/DevS3Access
                     (IAM is a GLOBAL service → region field is blank, account-id is present)

EC2 instance:          arn:aws:ec2:us-east-1:123456789012:instance/i-0abcd1234efgh5678
Security Group:        arn:aws:ec2:us-east-1:123456789012:security-group/sg-0123456789abcdef0
                     (EC2 is REGIONAL → region field is populated)

Lambda function:       arn:aws:lambda:us-east-1:123456789012:function:my-function
                     (note: uses a COLON before the resource name here, not a slash — service-specific)

STS assumed role:      arn:aws:sts::123456789012:assumed-role/CI-Deploy-Role/session-name
                     (temporary identity, produced when a role is assumed)
```

**Why the inconsistency (blank fields, `/` vs `:`) matters practically:**
- **Global vs regional services** — IAM, S3, Route 53, CloudFront are global, so their ARNs omit the region. EC2, RDS, Lambda are regional, so the region is mandatory in the ARN.
- **Account-scoped vs globally-unique naming** — S3 bucket names are globally unique across all of AWS by design, so the ARN doesn't need an account ID to disambiguate. IAM/EC2 resources are scoped per-account, so the account ID is required.
- **Wildcards work at any segment** — this is what makes ARNs central to policy writing:
  ```json
  "Resource": "arn:aws:s3:::my-training-bucket/*"              // all objects in one bucket
  "Resource": "arn:aws:s3:::my-training-bucket"                 // the bucket itself (needed for s3:ListBucket)
  "Resource": "arn:aws:ec2:us-east-1:123456789012:instance/*"   // all instances in one account/region
  "Resource": "*"                                                // every resource the action can apply to
  ```
- **Bucket-level vs object-level actions need different ARN forms** — a classic policy-writing mistake is granting `s3:GetObject` on the bucket ARN (`.../my-bucket`) instead of the object ARN (`.../my-bucket/*`), or vice versa granting `s3:ListBucket` (a bucket-level action) on the object ARN — each action in the AWS docs specifies which ARN form it requires, and getting this wrong is one of the most common causes of unexpected "access denied" errors.

---

## Part 3: Directory Services — LDAP, Active Directory, OpenLDAP

This is a dense area conceptually, so here it is broken down cleanly.

### 3.1 LDAP (Lightweight Directory Access Protocol)

LDAP is not a product — it's a **protocol** for querying and modifying **directory services**: hierarchical databases optimized for fast lookups of things like users, groups, computers, and organizational structure (read-heavy, write-light workloads — the opposite tradeoff of a relational DB).

Core concepts:
- **DIT (Directory Information Tree)** — the hierarchical structure of entries, similar to a filesystem tree.
- **DN (Distinguished Name)** — the full, unique path to an entry, e.g., `cn=alice,ou=engineering,dc=example,dc=com`.
- **Entry / Attribute** — each object (a user, a group) is an entry with attributes (`cn`=common name, `mail`, `uid`, `memberOf`, etc.), governed by a **schema**.
- **Bind** — the act of authenticating to the LDAP server (simple bind = DN + password; can also do anonymous or SASL binds).

### 3.2 OpenLDAP

**OpenLDAP** is the most common **open-source implementation** of the LDAP protocol/server — i.e., it's *an* LDAP server, the same way nginx is *a* web server. Used heavily on Linux-centric environments to centralize authentication (so you don't manage local `/etc/passwd` users on every single machine — machines instead query the central directory via **PAM/NSS + LDAP** modules).

### 3.3 Active Directory (AD)

**Active Directory** is Microsoft's directory service — it uses LDAP as one of its access protocols, but it's a much larger system than LDAP alone:
- Combines **LDAP** (directory access) + **Kerberos** (authentication) + **DNS** (its own internal DNS is mandatory for AD to function) + **Group Policy** (centralized Windows machine/user policy enforcement).
- Organizes into **Domains**, **Trees**, and **Forests**, with **Domain Controllers (DCs)** holding the directory data.
- Dominant in Windows-centric enterprise environments for centralized identity, device management, and single sign-on.

### 3.4 The Comparison That Usually Clarifies This

| | LDAP | OpenLDAP | Active Directory |
|---|---|---|---|
| What it is | A **protocol** | A specific **open-source server** implementing the LDAP protocol | Microsoft's full **directory service platform** |
| Analogy | Like "HTTP" | Like "nginx" (a server speaking that protocol) | Like a full CMS platform (uses HTTP but is much more) |
| Auth mechanism | Simple bind (or SASL) | Simple bind (or SASL) | Primarily Kerberos, LDAP for directory queries |
| Ecosystem | Protocol-only, vendor neutral | Common on Linux/Unix | Deeply tied to Windows Server, Group Policy, DNS |
| Typical use | Any system needing directory lookups | Centralized Linux auth | Centralized Windows domain auth + device management |

**Where AWS fits in:** AWS Directory Service offers **AWS Managed Microsoft AD** (a real, managed AD) and **AD Connector** (a proxy to your existing on-prem AD) — so you don't have to run your own domain controllers on EC2 just to integrate with corporate identity.

---


## Summary — Week 4 Mental Model

```
Shared storage: NFS on a single EC2 (manual, single-AZ, self-managed)
        │  → same protocol, fully managed, multi-AZ, elastic
        ▼
EFS
        
Identity: IAM Users/Groups (who) + Policies in JSON (what's allowed) + Roles (temporary, assumable identity)
        │  → for enterprise/legacy centralized identity across many machines
        ▼
LDAP (protocol) → OpenLDAP (Linux-world server) / Active Directory (Windows-world platform: LDAP+Kerberos+DNS+GPO)

