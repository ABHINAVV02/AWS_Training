# Week 3 — Linux Administration, EC2, Load Balancing, Auto Scaling & DNS

> Training notes: Linux internals → EC2/AMI mechanics → Load Balancers (HAProxy & AWS) → Auto Scaling → Route 53/DNS.

---

## Part 1: Linux Fundamentals

### 1.1 httpd vs nginx

Both are web servers, but they're built on fundamentally different architectures.

| Aspect | httpd (Apache) | nginx |
|---|---|---|
| Architecture | Process/thread-per-connection (prefork, worker, event MPMs) | Event-driven, asynchronous, single-threaded workers |
| Concurrency model | Spawns a process or thread per request | One worker handles thousands of connections via an event loop (epoll/kqueue) |
| Memory under load | Higher — each connection consumes a process/thread | Lower — connections are cheap, non-blocking |
| Static file serving | Good | Excellent (its original design goal) |
| Dynamic content | Native module support (mod_php, mod_python) | Typically proxies to PHP-FPM, uWSGI, etc. |
| .htaccess support | Yes, per-directory overrides | No — all config is centralized |
| Config style | Verbose, directory-context blocks | Compact, declarative `server {}` blocks |
| Best fit | Legacy apps needing per-directory config, module-heavy setups | High-concurrency, reverse proxy, load balancing, static content, microservices |

**Why it matters:** nginx's event loop means one process can hold open thousands of idle keep-alive connections cheaply (C10K problem solved), whereas Apache's prefork model ties up a full process/thread per connection — which is why nginx is the default choice as a reverse proxy / load balancer in front of application servers, while Apache is still common where `.htaccess`-level flexibility or deep module ecosystems (mod_security, mod_php) are needed.

### 1.2 Types of Processes in Linux

Every running program in Linux is a process, tracked by a PID (Process ID). Processes are classified by:

- **Foreground process** — tied to the terminal; the shell waits for it to finish (e.g., `vim file.txt`).
- **Background process** — runs detached from terminal control, launched with `&` (e.g., `sleep 100 &`); shell returns immediately.
- **Daemon process** — a background process that runs continuously, typically started at boot, detached from any terminal (no controlling TTY), usually name-suffixed with `d` (`sshd`, `httpd`, `crond`).
- **Orphan process** — a child whose parent has terminated; gets re-parented to `init`/`systemd` (PID 1).
- **Zombie process** — a process that has finished execution but still has an entry in the process table because its parent hasn't yet read its exit status via `wait()`. Shows as `<defunct>` in `ps`. Harmless in small numbers, but a symptom of buggy parent code if it accumulates.

Useful commands: `ps aux`, `top`/`htop`, `pstree`, `kill -9 <pid>`, `nohup command &`, `jobs`, `fg`/`bg`.

### 1.3 Users in Linux

A user is an account identified by a **UID (User ID)**. Every process runs with the privileges of the user that started it. User info lives in `/etc/passwd` (username, UID, GID, home dir, shell) and password hashes in `/etc/shadow`.

```
username:x:UID:GID:comment:home_dir:shell
```

Types of users:
- **Root (superuser)** — UID 0, unrestricted access to the entire system.
- **System users** — created by the OS/packages to run services (e.g., `nginx`, `mysql`), not meant for interactive login.
- **Regular/normal users** — human accounts created for interactive use.

### 1.4 Groups in Linux and UID/GID Classification

Groups let you assign permissions to multiple users at once instead of managing each user individually. Group data lives in `/etc/group`.

**UID Classification (standard on RHEL/CentOS/Ubuntu-family systems):**

| UID Range | Category |
|---|---|
| 0 | root |
| 1–99 (or 1–999 on Debian/Ubuntu) | Reserved system/service accounts (statically assigned by packages) |
| 100–999 (or up to 999) | Dynamically allocated system accounts |
| 1000+ | Regular human users (first human user typically gets UID 1000) |

GID follows the same broad convention — each user by default gets a **primary group** matching their username with the same numeric ID (user-private-group scheme).

Group types:
- **Primary group** — every user has exactly one; new files created by the user are owned by this group by default; recorded in `/etc/passwd`.
- **Secondary/supplementary groups** — additional groups a user belongs to, granting extra permissions (e.g., adding a user to the `docker` or `sudo` group); recorded in `/etc/group`.

### 1.5 User vs Groups

| | User | Group |
|---|---|---|
| Represents | A single identity/account | A collection of users |
| Identifier | UID | GID |
| Purpose | Authentication, owning processes/files individually | Simplifying permission management across many users |
| File ownership | Every file has exactly one owning user | Every file has exactly one owning group |
| Config file | `/etc/passwd`, `/etc/shadow` | `/etc/group` |

In short: **users authenticate, groups authorize at scale.** You `chown` a file to a user, and `chgrp`/group permissions to share access among a team without editing per-user ACLs.

### 1.6 find vs locate

Both search for files, but with very different mechanisms:

| | `find` | `locate` |
|---|---|---|
| Method | Walks the filesystem live, in real time | Queries a pre-built index database (`mlocate.db`) |
| Speed | Slower (especially on large trees) | Very fast |
| Freshness | Always accurate, reflects current state | Can be stale until the DB is updated (`updatedb`, usually via cron) |
| Power | Extremely flexible — filter by name, size, time, permissions, type, and execute actions | Simple filename substring/pattern matching only |
| Example | `find /var/log -name "*.log" -mtime -1 -size +10M` | `locate nginx.conf` |

**Rule of thumb:** use `locate` for a quick "where is this file roughly" lookup; use `find` when you need precision, freshness, or to act on results (`-exec`, `-delete`).

### 1.7 Permissions in Linux

Every file/directory has an owner, a group, and permission bits for three classes: **owner (u)**, **group (g)**, **others (o)**.

Permission types:
- **r (read)** = 4 — view file contents / list directory contents
- **w (write)** = 2 — modify file / create-delete entries in directory
- **x (execute)** = 1 — run file as a program / enter (`cd` into) a directory

Displayed as `-rwxr-xr--` (type + owner + group + others), or numerically as `754` etc.

Special permissions:
- **SUID (4000)** — executable runs with the file owner's privileges, not the caller's (e.g., `passwd`).
- **SGID (2000)** — on a file, runs with the group's privileges; on a directory, new files inherit the directory's group.
- **Sticky bit (1000)** — on a directory (e.g., `/tmp`), only the file's owner (or root) can delete/rename it even if others have write access.

Commands: `chmod`, `chown`, `chgrp`, `umask` (default permission mask for newly created files).

---

## Part 2: AMI, EC2 & Instance Provisioning

### 2.1 AMI vs Snapshot

| | AMI (Amazon Machine Image) | Snapshot |
|---|---|---|
| What it is | A complete template to **launch an EC2 instance** — includes root volume data, launch permissions, block device mapping | A **point-in-time backup of an EBS volume** (block-level, incremental) |
| Scope | Can reference one or more snapshots (root + attached volumes) plus metadata | Just the raw volume data |
| Purpose | Used to **boot new instances** | Used to **restore/create new EBS volumes**, or as the backing data for building an AMI |
| Relationship | An AMI is *built from* snapshot(s) | A snapshot is a *component* of an AMI |

In short: a snapshot is the storage-level backup; an AMI is a bootable, launchable wrapper around one or more snapshots plus instance-level configuration.

### 2.2 Golden AMI

A **Golden AMI** is a pre-baked, hardened, standardized AMI that already contains your OS patches, security agents, monitoring tools, base packages, and configuration — so that new instances launch already compliant and ready, instead of running lengthy bootstrap scripts every time.

Benefits:
- Faster boot/scale-out (Auto Scaling launches are near-instant vs. running config management on every boot).
- Consistency — every instance is identical, reducing configuration drift.
- Security — pre-hardened, pre-patched baseline reduces attack surface.
- Often built via automated pipelines (e.g., HashiCorp Packer + CI/CD) so golden AMIs are versioned and rebuilt regularly with the latest patches.

### 2.3 User Data on Instance Creation & Reading cloud-init Logs

**User data** is a script (bash, cloud-config YAML, etc.) passed at launch time that EC2 feeds to the instance's first-boot process — used for bootstrapping (installing packages, pulling app code, setting configs) without needing to bake everything into the AMI.

- Runs as **root**, only on **first boot** by default (unless explicitly configured to run every boot).
- Processed by **cloud-init**, the industry-standard multi-distro initialization tool AWS AMIs ship with.

**Reading cloud-init logs** (for debugging failed user-data scripts):

```bash
cat /var/log/cloud-init.log          # cloud-init's own execution log
cat /var/log/cloud-init-output.log   # stdout/stderr of your user-data script — most useful for debugging
```

You can also re-inspect what user data was passed via the instance metadata service:
```bash
curl http://169.254.169.254/latest/user-data
```

### 2.4 Advanced Settings of EC2 Instance

Key advanced launch options beyond AMI/instance type/key pair:

- **VPC/Subnet/Auto-assign public IP** — network placement.
- **IAM Instance Profile** — attaches an IAM role so the instance can call AWS APIs without hardcoded credentials.
- **Shutdown behavior** — Stop vs Terminate on OS shutdown.
- **Termination protection** — prevents accidental termination via console/CLI.
- **Monitoring** — Basic (5-min) vs Detailed CloudWatch monitoring (1-min).
- **Tenancy** — Shared / Dedicated Instance / Dedicated Host.
- **Elastic Inference / Credit specification** (for T-series burstable instances — Standard vs Unlimited).
- **File systems** — attaching EFS at launch.
- **User data** field.
- **Placement group** assignment.
- **EBS optimization** — dedicated throughput to EBS, separate from network traffic.
- **Metadata options (IMDSv1 vs IMDSv2)** — enforcing token-based metadata requests for security (mitigates SSRF attacks against the metadata service).

### 2.5 Placement Groups and Types

A **placement group** controls how EC2 instances are physically placed on underlying hardware, to optimize for either performance or fault tolerance.

| Type | Behavior | Use Case |
|---|---|---|
| **Cluster** | Packs instances close together in a single AZ, on the same low-latency, high-throughput network spine | HPC, tightly-coupled node-to-node workloads (e.g., MPI clusters) |
| **Spread** | Spreads each instance across distinct underlying hardware (max 7 instances per AZ per group) | Small numbers of critical instances that must not share a single point of hardware failure |
| **Partition** | Divides instances into logical partitions, each on separate racks (separate power/network) | Large distributed systems that natively handle partition-level failure (HDFS, Cassandra, Kafka) |

### 2.6 Pricing in AWS (EC2)

- **On-Demand** — pay per second/hour, no commitment; highest per-unit cost; best for unpredictable/short-term workloads.
- **Reserved Instances (RI)** — 1 or 3-year commitment for a specific instance family/region in exchange for up to ~72% discount; Standard (less flexible, cheaper) vs Convertible (can change instance attributes) RIs.
- **Savings Plans** — commit to a $/hour spend for 1 or 3 years; more flexible than RIs (applies across instance families/sizes/regions automatically).
- **Spot Instances** — bid on spare AWS capacity for up to ~90% discount; AWS can reclaim with a 2-minute warning; best for fault-tolerant, interruptible workloads (batch jobs, CI runners, stateless workers).
- **Dedicated Hosts** — pay for an entire physical server, billed per host.
- **Dedicated Instances** — pay per instance, but guaranteed to run on hardware dedicated to your account only.

### 2.7 Types of Instances / Provisioning

EC2 instance families are grouped by workload profile:

- **General Purpose (T, M series)** — balanced CPU/memory/network; T-series is burstable (CPU credits).
- **Compute Optimized (C series)** — high vCPU-to-memory ratio; batch processing, media transcoding, gaming servers.
- **Memory Optimized (R, X, z1d)** — high memory-to-vCPU ratio; in-memory databases, caches.
- **Storage Optimized (I, D, H series)** — high sequential/random I/O against local NVMe; data warehousing, distributed file systems.
- **Accelerated Computing (P, G, Inf series)** — GPU/FPGA/ML-inference hardware.

**Provisioning models:**
- On-Demand launch (manual/API/console).
- Auto Scaling Group-driven provisioning.
- Spot Fleet/Spot requests.
- Launch Templates/Configurations for repeatable provisioning.

### 2.8 Dedicated Host vs Dedicated Instance

| | Dedicated Host | Dedicated Instance |
|---|---|---|
| Isolation unit | An entire **physical server** allocated to you | Just your **instance** guaranteed not to share hardware with other AWS accounts |
| Visibility | You see host-level details (sockets, cores) — useful for **BYOL (bring-your-own-license)** software licensed per-socket/per-core | No visibility into underlying host |
| Control over placement | You can control exactly which instances land on which host | No control — AWS places it on *some* dedicated (non-shared) hardware |
| Billing | Pay for the host itself, regardless of instance count on it | Pay per instance, plus a small dedicated-tenancy premium |
| Best fit | Licensing compliance (Windows Server, SQL Server core-based licensing), regulatory requirements needing physical server visibility | Simpler compliance need — "just don't put me on shared hardware" |

---

## Part 3: Load Balancing

### 3.1 Types of Load Balancers in AWS

AWS Elastic Load Balancing (ELB) offers four types:

| Type | Layer | Key Traits |
|---|---|---|
| **ALB (Application Load Balancer)** | L7 (HTTP/HTTPS) | Content-based routing (path/host/header/query-string based), WebSocket & HTTP/2 support, native integration with ECS/Lambda targets |
| **NLB (Network Load Balancer)** | L4 (TCP/UDP/TLS) | Ultra-low latency, millions of req/sec, **static IP per AZ**, preserves client source IP, ideal for extreme performance or non-HTTP protocols |
| **GLB (Gateway Load Balancer)** | L3 (IP) | Transparently routes traffic **through** a fleet of third-party virtual appliances (firewalls, IDS/IPS, DPI) using GENEVE encapsulation, for traffic inspection use cases |
| **CLB (Classic Load Balancer)** | L4/L7 (legacy) | Predecessor to ALB/NLB, EC2-Classic era, largely deprecated for new workloads |

### 3.2 Load Balancer via HAProxy (Manual Setup)

Before using AWS's managed ALB, standing up **HAProxy** manually illustrates what a load balancer actually does under the hood.

Core HAProxy config sections:
```
global
    # process-wide settings (logging, max connections, user/group to run as)

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    server web1 10.0.1.10:80 check
    server web2 10.0.1.11:80 check
```

- **frontend** — defines the listening IP/port that receives client traffic.
- **backend** — defines the pool of real servers traffic is distributed to, plus the **balance algorithm** and **health checks** (`check`).
- Running this manually teaches the underlying primitives (frontend/backend separation, health checks, algorithm selection) that AWS ALB abstracts away behind a managed console.

### 3.3 AWS ALB — Options Explored

Setting up a standard AWS Application Load Balancer surfaces these key configuration areas:

- **Listeners** — protocol/port ALB listens on (e.g., HTTP:80, HTTPS:443); each listener has **rules** evaluated in priority order.
- **Target Groups** — logical pool of targets (EC2 instances, IPs, Lambda functions) plus health check settings (path, interval, healthy/unhealthy threshold).
- **Security Groups** — attached to the ALB itself, controlling what can reach it.
- **Cross-zone load balancing** — evenly distributes traffic across targets in *all* enabled AZs, not just the AZ traffic arrived in.
- **Idle timeout** — how long ALB keeps an idle connection open.
- **Deletion protection** and **access logs** (to S3) for auditing.
- **SSL/TLS termination** — ALB can terminate HTTPS and talk plain HTTP to targets, offloading crypto work from backend instances.
- **Sticky sessions (session affinity)** — via cookies, so a client keeps hitting the same target.

### 3.4 Load Balancing Algorithms

- **Round Robin** — requests distributed sequentially across targets in rotation; simple, assumes equal target capacity.
- **Least Outstanding Requests (LOR)** — ALB default; sends the next request to whichever target currently has the fewest in-flight requests — better for uneven request durations.
- **Weighted Round Robin / Weighted Target Groups** — targets/target-groups get proportional traffic based on assigned weight (e.g., canary deployments: 90% to v1, 10% to v2).
- **Flow Hash / Consistent Hashing (NLB)** — routes based on a hash of protocol/source-dest IP/port, ensuring the same client flow consistently reaches the same target.
- **IP Hash** (common in HAProxy/nginx) — client IP determines target, giving basic session persistence without cookies.

### 3.5 Path-Based and Host-Based Routing

Both are **Layer 7 (ALB-only)** routing rule types that let one ALB serve multiple applications:

- **Path-based routing** — routes based on the URL path, e.g.:
  - `/api/*` → target-group-api
  - `/images/*` → target-group-static
  - `/` → target-group-webapp

- **Host-based routing** — routes based on the `Host` header (domain name), e.g.:
  - `api.example.com` → target-group-api
  - `shop.example.com` → target-group-shop

  This is what lets **one ALB with one set of listeners** front multiple distinct domains/microservices, instead of needing a separate load balancer per application — a major cost and management simplification. Rules can also combine host + path conditions together.

---

## Part 4: Auto Scaling Groups (ASG)

### 4.1 Types of Scaling

- **Manual scaling** — operator manually changes desired capacity.
- **Scheduled scaling** — scale up/down at predictable times (e.g., scale up before business hours, down overnight).
- **Dynamic scaling**:
  - **Target Tracking** — you pick a metric (e.g., avg CPU = 50%) and ASG automatically adds/removes instances to hold that target — simplest and most common.
  - **Step Scaling** — define scaling steps tied to CloudWatch alarm breach magnitude (e.g., CPU 50–70% → +1 instance, CPU >70% → +3 instances).
  - **Simple Scaling** — one alarm, one scaling action, with a cooldown period before the next action — legacy, less responsive than step/target tracking.
- **Predictive scaling** — uses ML on historical load patterns to provision ahead of anticipated demand.

Scaling is also bounded by **Min / Max / Desired capacity** settings that define the ASG's operating envelope.

### 4.2 Configuring an ASG with an ALB

The standard elastic architecture pattern:

1. Create a **Launch Template** (AMI, instance type, security group, user data, key pair).
2. Create a **Target Group** and register it with an **ALB**.
3. Create the **ASG**, referencing the launch template, and **attach it to the target group** (not directly to instances) — so ASG automatically registers/deregisters instances with the ALB as it scales in/out.
4. ASG uses the target group's **health checks** (in addition to EC2 status checks) — if ALB marks an instance unhealthy, ASG can terminate and replace it automatically.
5. Attach a **scaling policy** (typically target tracking on `ALBRequestCountPerTarget` or `CPUUtilization`).

This gives you: traffic hits ALB → ALB distributes across whatever instances currently exist → ASG adds/removes instances behind the scenes based on load → ALB automatically adapts its target list.

### 4.3 Elastic IP (Brief)

An **Elastic IP (EIP)** is a static, public IPv4 address you allocate to your AWS account and can attach/detach/re-attach to instances at will — unlike a normal public IP, it **doesn't change** when you stop/start the instance. Useful for a single fixed-IP endpoint (e.g., a NAT instance, DNS A record you don't want to keep updating). AWS charges for EIPs that are **allocated but not attached to a running instance**, to discourage IP hoarding. Note: in an ASG/ALB architecture, you typically *don't* use EIPs per-instance — the ALB's own (rotating) DNS/IP fronts everything instead.

### 4.4 Scaling Tested via Fake Load on CPU

To validate that a target-tracking policy actually triggers scale-out, you artificially spike CPU on an instance and watch CloudWatch/ASG react:

```bash
# stress-ng or stress package
sudo yum install -y stress
stress --cpu 8 --timeout 300s
```
or
```bash
yes > /dev/null &   # crude busy-loop, run multiple in background
```

You then watch the CloudWatch `CPUUtilization` metric cross the target-tracking threshold, confirm a **scale-out alarm** fires, a new instance launches from the launch template, gets registered with the target group, passes health checks, and starts receiving traffic — then confirm **scale-in** happens (after cooldown) once the fake load is killed and average CPU drops back down.

---

## Part 5: Route 53 & DNS

### 5.1 How DNS Works

DNS (Domain Name System) resolves human-readable domain names into IP addresses, through a hierarchical, distributed lookup:

1. Browser checks local/OS cache.
2. Query goes to a **Recursive Resolver** (typically ISP or public like 8.8.8.8).
3. Resolver queries a **Root nameserver** → tells it which **TLD nameserver** (.com, .org, etc.) to ask.
4. TLD nameserver points to the domain's **Authoritative nameserver** (e.g., Route 53 hosted zone).
5. Authoritative nameserver returns the actual record (e.g., A record IP).
6. Resolver caches and returns the answer to the client, respecting the record's **TTL (Time To Live)**.

### 5.2 Types of DNS (Servers/Roles)

- **Root DNS servers** — top of the hierarchy, direct queries to the correct TLD servers.
- **TLD (Top-Level Domain) servers** — manage `.com`, `.net`, `.org`, country-code TLDs, etc.
- **Authoritative DNS servers** — hold the actual, definitive records for a domain (Route 53 acts as this for zones you host there).
- **Recursive resolvers (Recursive DNS servers)** — do the legwork of querying the chain above on behalf of the client and caching results.

### 5.3 Types of DNS Records

| Record | Purpose |
|---|---|
| **A** | Maps a hostname to an **IPv4** address |
| **AAAA** | Maps a hostname to an **IPv6** address |
| **CNAME** | Maps a hostname to **another hostname** (alias); cannot coexist with other records at the same name, and **cannot be used at the zone apex** (root domain) |
| **MX** | Mail exchange — directs email to mail servers, with priority values |
| **TXT** | Arbitrary text — used for domain verification, SPF/DKIM/DMARC email authentication |
| **NS** | Delegates a (sub)domain to a specific set of authoritative nameservers |
| **SOA** | Start of Authority — holds admin metadata about the zone (primary NS, serial, refresh/retry/expire/TTL) |
| **PTR** | Reverse DNS — maps an IP address back to a hostname |
| **Alias (Route 53-specific)** | AWS's own extension — behaves like a CNAME but *can* be used at the zone apex, and points natively to AWS resources (ALB, CloudFront, S3 website) without an extra DNS lookup, and it's free of charge to query |

### 5.4 Configuring a DNS Server — BIND (Manual) then Route 53

**BIND (manual, self-hosted)** — configuring `named.conf` plus zone files by hand:
```
// named.conf.local
zone "example.local" {
    type master;
    file "/etc/bind/db.example.local";
};
```
```
; db.example.local zone file
$TTL 604800
@   IN  SOA ns1.example.local. admin.example.local. ( 1 604800 86400 2419200 604800 )
@   IN  NS  ns1.example.local.
@   IN  A   192.168.1.10
www IN  A   192.168.1.10
```
This manual exercise makes explicit what a hosted zone abstracts: you own the **zone file**, **SOA record**, **NS delegation**, and every record's TTL — all managed by hand, restart-required on change (`systemctl restart bind9`).

**Route 53 (managed)** replaces all of this with API/console-driven **Hosted Zones** — no server to patch/restart, records propagate automatically, integrated health checks and routing policies.

### 5.5 Hosted Zones and Types of Routing Policy

A **Hosted Zone** is a container for all the DNS records for a domain in Route 53.
- **Public Hosted Zone** — resolves domain names on the public internet.
- **Private Hosted Zone** — resolves domain names only within one or more specified VPCs.

**Routing Policies:**

| Policy | Behavior |
|---|---|
| **Simple** | One record, optionally multiple values returned in random order; no health checks |
| **Weighted** | Distributes traffic across multiple records **in proportion to assigned weights** — used for canary releases, A/B testing, gradual migrations |
| **Latency-based** | Routes to the region/resource with the **lowest latency** for the querying user |
| **Failover** | Active-passive setup — routes to a primary, automatically fails over to secondary if primary's health check fails |
| **Geolocation** | Routes based on the **geographic location of the user** (country/continent) |
| **Geoproximity** | Routes based on geographic location **with a bias** you can shift to expand/shrink a region's "pull" (requires Route 53 Traffic Flow) |
| **Multivalue Answer** | Returns multiple healthy records randomly, similar to Simple but **with health checks** — poor-man's load balancing at DNS level |

### 5.6 Weighted Policy — Why Two Separate Records

The key realization: a **weighted routing policy isn't one record with two targets** — you create **two (or more) separate record sets with the same name and type**, and each individual record carries its own **weight** value and its own **Set ID**. Route 53 then returns each record probabilistically, proportional to `weight / sum_of_all_weights`.

Example — sending 80% of traffic to v1, 20% to v2:
```
Record 1: app.example.com  A  1.2.3.4   Weight=80  SetID="v1"
Record 2: app.example.com  A  5.6.7.8   Weight=20  SetID="v2"
```
Setting a weight to `0` stops traffic to that record entirely (useful for pulling a bad canary without deleting the record). This is why it initially feels unintuitive — you're not editing one record's config, you're maintaining **multiple parallel records** that Route 53 arbitrates between at query time.

### 5.7 CNAME — Where the Confusion Comes From

The common confusion points, clarified:

1. **CNAME can't coexist with other records at the same name.** If `www.example.com` is a CNAME, you cannot also have an A or TXT record for `www.example.com` — the CNAME must be the *only* record there.
2. **CNAME cannot be used at the zone apex** (`example.com` itself, no subdomain) — this is a DNS protocol rule, not an AWS limitation, because the apex must also hold the SOA/NS records, which can't coexist with a CNAME.
3. **This is exactly why AWS invented the Alias record** — it looks and acts like a CNAME (points to another DNS name, e.g., an ALB's DNS name) but is a Route53-only pseudo-record type that's legal at the apex, resolves without an extra client-side lookup, and is free to query. So: use **CNAME** for subdomain-to-subdomain aliasing to non-AWS or general targets; use **Alias** specifically when pointing at AWS resources (ALB, CloudFront, S3 static site, another Route 53 record) — especially at the apex, where CNAME is disallowed outright.

---

## Summary — Week 3 Mental Model

```
Linux fundamentals (users/groups/permissions/processes)
        │
        ▼
EC2 instance built from an AMI (backed by snapshots), bootstrapped via user data/cloud-init
        │
        ▼
Traffic enters through a Load Balancer (HAProxy manually → ALB/NLB/GLB managed)
   - L7 routing: path-based / host-based rules on ALB
   - Algorithm: round robin / least outstanding requests / weighted
        │
        ▼
Auto Scaling Group attached to the Target Group
   - scales dynamically (target tracking / step) based on real load (validated via stress testing)
        │
        ▼
Route 53 resolves the domain to the Load Balancer
   - Hosted Zone → Records → Routing Policy (weighted/failover/latency/geo)
   - Alias records preferred over CNAME for AWS targets, especially at the apex
```
