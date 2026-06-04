# Week 1: AWS Solutions Architect - Compute & OS Fundamentals

## Overview
This document serves as the comprehensive study guide and lab runbook for Week 1 of my AWS Certified Solutions Architect (SAA-C03) training. 

The objective of this week was to bridge the gap between physical hardware virtualization and cloud infrastructure. We then transitioned into practical execution by provisioning an Amazon EC2 instance and performing core Enterprise Linux administration to understand the compute layer from the inside out.

---

## Part 1: Cloud Computing & Virtualization Theory
Before provisioning AWS resources, it is critical to understand the architecture that makes cloud computing possible.

### Virtualization & Hypervisors
* **The OS Role:** An Operating System manages underlying bare-metal hardware (CPU, RAM, Disk) and allocates it to applications.
* **Virtualization:** Software that allows multiple independent virtual machines (VMs) to run on a single physical server, maximizing hardware efficiency.
* **Hypervisor Types:**
  * **Type 1 (Bare Metal):** Installed directly on the hardware (e.g., VMware ESXi, AWS Nitro). This high-performance, low-overhead architecture is what powers AWS data centers.
  * **Type 2 (Hosted):** Installed on top of a standard OS (e.g., VirtualBox). Primarily used for local development and testing.

### Cloud Service Models
* **IaaS (Infrastructure as a Service):** AWS provides the raw hardware and hypervisor (e.g., EC2). The architect manages the OS, patching, networking, and applications.
* **PaaS (Platform as a Service):** AWS manages the underlying OS and runtime. The user focuses solely on deploying application code.
* **SaaS (Software as a Service):** Fully managed end-user applications hosted in the cloud.

---

## Part 2: AWS EC2 Provisioning
We transitioned from theory to practice by launching our first compute resource in the AWS Cloud.

### Instance Architecture
Provisioned a virtual machine utilizing the following parameters:
* **AMI (Amazon Machine Image):** Selected Red Hat Enterprise Linux (RHEL) to act as the base operating system.
* **Instance Type (`t3.micro`):** Selected to utilize the AWS Free Tier. The `t3.micro` provides 2 vCPUs and 1 GiB of RAM.
* **Storage:** Allowed AWS to automatically provision the default root Elastic Block Store (EBS) volume required to boot the OS. No secondary drives were attached at this stage.

### Security & Networking (Domain 1: Secure Architectures)
* **Security Groups:** Configured a stateful, instance-level firewall. 
  * *Rule implemented:* Allowed Inbound SSH (Port 22) traffic strictly from my local IP address to enable remote terminal access.

---

## Part 3: OS Architecture & System Orientation
After securely connecting to the `t3.micro` instance via SSH, I executed core commands to understand the system identity and filesystem hierarchy.

### Core Navigation Commands

* **Identify the active user profile:**
```bash
whoami

# Output:
ec2-user
```
*(Note: `ec2-user` is the default administrative user provisioned by the AWS RHEL AMI).*

* **Identify the internal AWS network name:**
```bash
hostname

# Output:
ip-172-31-16-122.ec2.internal
```

* **Identify the current directory path:**
```bash
pwd

# Output:
/home/ec2-user
```

* **Identify System Architecture & Kernel:**
```bash
uname -a

# Output:
Linux ip-172-31-16-122.ec2.internal 5.14.0-284.11.1.el9_2.x86_64 #1 SMP PREEMPT_DYNAMIC Wed Apr 12 10:45:03 EDT 2023 x86_64 x86_64 x86_64 GNU/Linux
```

### Command-Line Text Editing
Introduced to the `vim` text editor for modifying system configurations without a GUI. Mastered the transition between Command Mode (for navigating, saving, and exiting via `:wq`) and Insert Mode (for text entry).

---

## Part 4: File Management & System Resources
Administering a lightweight cloud instance requires strict resource monitoring and efficient file management.

### Resource Monitoring
Because the `t3.micro` instance is constrained to 1 GiB of RAM, aggressive memory management is critical to prevent the instance from failing AWS health checks.

```bash
free -h

# Output:
               total        used        free      shared  buff/cache   available
Mem:           960Mi       150Mi       500Mi       8.0Mi       310Mi       650Mi
Swap:             0B          0B          0B
```

### Package Management
Installed software directly from the command line using the native RHEL package manager (`dnf`).

```bash
sudo dnf install httpd -y

# Output:
Updating Subscription Management repositories.
Unable to read consumer identity
Last metadata expiration check: 0:15:23 ago on Thu 04 Jun 2026 09:00:00 AM UTC.
Dependencies resolved.
================================================================================
 Package             Arch       Version             Repository             Size
================================================================================
Installing:
 httpd               x86_64     2.4.53-7.el9        rhel-9-appstream      1.4 M
Installing dependencies:
 httpd-tools         x86_64     2.4.53-7.el9        rhel-9-appstream       82 k
 redhat-logos-httpd  noarch     90.4-1.el9          rhel-9-appstream       24 k

Transaction Summary
================================================================================
Install  3 Packages

Total download size: 1.5 M
Installed size: 4.3 M
Downloading Packages:
...
Installed:
  httpd-2.4.53-7.el9.x86_64   httpd-tools-2.4.53-7.el9.x86_64   redhat-logos-httpd-90.4-1.el9.noarch

Complete!
```

### Filesystem Operations

```bash
# Create a dedicated directory for application files
mkdir /home/ec2-user/project-files

# Create multiple empty configuration files instantly using brace expansion
touch file{1..5}.conf

# List all files in the directory, including hidden files, with detailed permission sets
ls -lah

# Output:
total 12K
drwxr-xr-x. 2 ec2-user ec2-user  93 Jun  4 09:20 .
drwx------. 4 ec2-user ec2-user 115 Jun  4 09:18 ..
-rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file1.conf
-rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file2.conf
-rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file3.conf
-rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file4.conf
-rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file5.conf

# Safely remove a specific file
rm file1.conf
```

### Reading Data streams

```bash
cat /etc/os-release

# Output:
NAME="Red Hat Enterprise Linux"
VERSION="9.2 (Plow)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="9.2"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Red Hat Enterprise Linux 9.2 (Plow)"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:redhat:enterprise_linux:9::baseos"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
```

```bash
sudo head -n 5 /var/log/messages

# Output:
Jun  4 09:15:01 ip-172-31-16-122 systemd[1]: Starting system activity accounting tool...
Jun  4 09:15:01 ip-172-31-16-122 systemd[1]: sysstat-collect.service: Deactivated successfully.
Jun  4 09:15:01 ip-172-31-16-122 systemd[1]: Finished system activity accounting tool.
Jun  4 09:15:04 ip-172-31-16-122 systemd[1]: Starting Network Manager Script Dispatcher Service...
Jun  4 09:15:04 ip-172-31-16-122 systemd[1]: Started Network Manager Script Dispatcher Service.
```

```bash
sudo tail -n 5 /var/log/messages

# Output:
Jun  4 09:18:22 ip-172-31-16-122 systemd[1]: Started Session 3 of User ec2-user.
Jun  4 09:18:22 ip-172-31-16-122 systemd-logind[788]: New session 3 of user ec2-user.
Jun  4 09:18:24 ip-172-31-16-122 systemd[1]: Starting dnf makecache...
Jun  4 09:18:25 ip-172-31-16-122 dnf[1422]: Metadata cache refreshed recently.
Jun  4 09:18:25 ip-172-31-16-122 systemd[1]: dnf-makecache.service: Deactivated successfully.
```
*(A critical technique for troubleshooting real-time application errors on the instance).*

---

## Part 5: Data Streams & OS-Level Security
The final phase covered advanced text processing to filter standard outputs, and the enforcement of file immutability to protect critical instance configurations.

### Pattern Matching (Grep)
Filtering massive system logs to isolate specific data points.

```bash
# Search the secure log file for a specific string (case-insensitive) to audit failed logins
sudo grep -i "failed" /var/log/secure

# Output:
Jun  4 08:45:12 ip-172-31-16-122 sshd[1204]: Failed password for invalid user admin from 198.51.100.23 port 54321 ssh2
```

### I/O Redirection & Pipelining
Chaining discrete commands together and directing their standard output (stdout) into files.

```bash
# Overwrite (>): Create a new file and inject a string of text
echo "EC2 Server Config v1" > server-info.txt

# Append (>>): Add new data to the end of the file without overwriting the previous lines
echo "Instance Type: t3.micro" >> server-info.txt

# Pipelining (|): List all running processes, and pass that exact output to grep to filter for 'httpd'
ps -aux | grep httpd

# Output:
root        1522  0.0  0.5  17764  5320 ?        Ss   09:25   0:00 /usr/sbin/httpd -DFOREGROUND
apache      1523  0.0  0.4  18020  4112 ?        S    09:25   0:00 /usr/sbin/httpd -DFOREGROUND
apache      1524  0.0  0.4  18020  4112 ?        S    09:25   0:00 /usr/sbin/httpd -DFOREGROUND
ec2-user    1550  0.0  0.2   6408  2148 pts/0    S+   09:27   0:00 grep --color=auto httpd
```

### File Immutability (Domain 1: Secure Architectures)
Accidentally deleting critical boot files (such as `/etc/fstab`) will permanently break an EC2 instance. We utilized `chattr` to make files immutable, ensuring that not even the root administrator can delete or modify them without explicitly removing the lock first.

```bash
# Create a critical configuration file on the t3.micro instance
touch /home/ec2-user/critical-config.conf

# Add the immutable (+i) attribute
sudo chattr +i /home/ec2-user/critical-config.conf

# Verify the attribute is applied (Look for the 'i' in the output string)
lsattr /home/ec2-user/critical-config.conf

# Output: 
----i----------------- /home/ec2-user/critical-config.conf
```
