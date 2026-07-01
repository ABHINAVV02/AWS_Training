# AWS Solutions Architect — Week 1: Compute & OS Fundamentals

A detailed, step-by-step study guide and lab runbook covering virtualization theory, EC2 provisioning, Linux OS orientation, the `vim` editor, the Linux file system hierarchy, resource monitoring, package management, data streams, and file immutability.

---

## Table of Contents

1. [Cloud Computing & Virtualization Theory](#1-cloud-computing--virtualization-theory)
2. [AWS EC2 Provisioning](#2-aws-ec2-provisioning)
3. [OS Architecture & System Orientation](#3-os-architecture--system-orientation)
4. [The `vim` Text Editor](#4-the-vim-text-editor)
5. [The Linux File System Hierarchy](#5-the-linux-file-system-hierarchy)
6. [File Management & System Resources](#6-file-management--system-resources)
7. [Data Streams & OS-Level Security](#7-data-streams--os-level-security)

---

## 1. Cloud Computing & Virtualization Theory

Before provisioning any AWS resource, it's important to understand the layer of abstraction that makes cloud computing possible in the first place: virtualization.

### 1.1 The Operating System's role

An Operating System sits between raw hardware (CPU, RAM, disk, network interfaces) and the applications that want to use it. It schedules CPU time, allocates memory, and manages I/O so that multiple programs can run concurrently without stepping on each other.

### 1.2 Virtualization and hypervisors

Virtualization is software that lets multiple independent virtual machines (VMs) share a single physical server, each believing it has dedicated hardware. This is what allows a cloud provider to sell many small VMs out of one large physical machine efficiently.

**Two hypervisor types to know:**

| Type | Description | Example | Typical Use |
|---|---|---|---|
| **Type 1 (Bare Metal)** | Installed directly on the physical hardware, no host OS underneath | VMware ESXi, AWS Nitro | Production data centers — low overhead, high performance |
| **Type 2 (Hosted)** | Installed on top of a regular OS, which then hosts guest VMs | Oracle VirtualBox, VMware Workstation | Local development and testing on a laptop/desktop |

AWS's EC2 fleet runs on the **AWS Nitro System**, a Type 1 (bare-metal) hypervisor architecture, which is why EC2 instances get near bare-metal performance despite being virtual machines.

### 1.3 Cloud service models

**Steps to reason about which model you're using for a given AWS service:**
1. Ask: *"What layer do I manage, and what does AWS manage?"*
2. **IaaS (Infrastructure as a Service)** — AWS provides the physical hardware and hypervisor; you manage everything above that: OS, patching, runtime, and application code. **Example: EC2.**
3. **PaaS (Platform as a Service)** — AWS manages the OS and runtime; you only deploy application code. **Example: Elastic Beanstalk, AWS Lambda (function-level).**
4. **SaaS (Software as a Service)** — Fully managed, ready-to-use software; you don't manage any infrastructure. **Example: Amazon Chime, WorkMail.**

EC2 sits at the IaaS end of the spectrum, which is exactly why Week 1's hands-on labs focus so heavily on OS-level administration — with IaaS, that responsibility falls on you.

---

## 2. AWS EC2 Provisioning

### 2.1 Instance architecture decisions

**Steps taken to provision the instance:**
1. Chose an **AMI (Amazon Machine Image)** — the pre-baked template containing the OS and initial software. Selected **Red Hat Enterprise Linux (RHEL)** as the base OS.
2. Chose an **Instance Type**: `t3.micro`, selected specifically to stay within the AWS Free Tier. This type provides:
   - 2 vCPUs
   - 1 GiB of RAM
   - Burstable ("T-series") CPU performance, meaning it accrues CPU credits during idle periods and spends them during bursts of activity
3. Left **Storage** at its default setting, letting AWS auto-provision the root **EBS (Elastic Block Store)** volume needed to boot the OS — no secondary data volumes were attached at this stage.
4. Launched the instance and waited for its status checks to pass (2/2 checks) before connecting.

### 2.2 Security & networking

**Steps to lock down access with a Security Group:**
1. During launch (or afterward via **EC2 > Security Groups**), create or edit a Security Group attached to the instance.
2. Security Groups are **stateful, instance-level firewalls** — if you allow inbound traffic on a port, the corresponding outbound response traffic is automatically allowed back out; you don't need a separate outbound rule for replies.
3. Add an inbound rule:
   - Type: **SSH**
   - Protocol: TCP
   - Port: **22**
   - Source: **My IP** (restricts SSH access to your current public IP address only, rather than `0.0.0.0/0` which would open it to the entire internet)
4. Save the rule. Any connection attempt on port 22 from outside your IP will now be dropped at the firewall level before it ever reaches the instance's OS.

---

## 3. OS Architecture & System Orientation

After connecting to the instance via SSH, the next step is to orient yourself: who you are, where you are, and what system you're on.

### 3.1 Identify the active user

```
whoami
```
```
# Output:
ec2-user
```
`ec2-user` is the default administrative user pre-configured by AWS's RHEL AMI. It has passwordless `sudo` access, which is how you perform privileged operations without logging in as `root` directly — a security best practice, since it preserves an audit trail of *which* user ran a privileged command.

### 3.2 Identify the internal network name

```
hostname
```
```
# Output:
ip-172-31-16-122.ec2.internal
```
AWS automatically names EC2 instances based on their private IP address by default. This internal DNS name is resolvable from within the VPC, which is useful when instances need to reference each other without hardcoding IPs.

### 3.3 Identify the current working directory

```
pwd
```
```
# Output:
/home/ec2-user
```
`pwd` ("print working directory") confirms your current location in the file system tree — by default, you land in your user's home directory right after login.

### 3.4 Identify system architecture and kernel version

```
uname -a
```
```
# Output:
Linux ip-172-31-16-122.ec2.internal 5.14.0-284.11.1.el9_2.x86_64 #1 SMP PREEMPT_DYNAMIC Wed Apr 12 10:45:03 EDT 2023 x86_64 x86_64 x86_64 GNU/Linux
```
Breaking this output down:
- `Linux` — the kernel name
- `5.14.0-284.11.1.el9_2.x86_64` — the specific kernel version (useful when checking driver/module compatibility)
- `x86_64` (appearing three times) — CPU architecture, kernel build, and OS build all confirmed as 64-bit
- `GNU/Linux` — confirms this is a GNU userland running on top of the Linux kernel

---

## 4. The `vim` Text Editor

Cloud servers rarely have a graphical interface, so editing configuration files means using a terminal-based editor. `vim` is the standard choice on nearly every Linux distribution.

### 4.1 Why `vim` instead of a GUI editor

`vim` runs entirely inside the terminal over your existing SSH session — no extra ports, no GUI forwarding, no additional software needed on the server. It's also virtually guaranteed to be present (or trivially installable) on any Linux box you SSH into.

### 4.2 Opening a file

```
vim server-info.txt
```
If the file doesn't exist yet, `vim` creates a new, empty buffer with that filename ready to be saved on your first write.

### 4.3 Understanding `vim`'s modes

`vim` is a **modal editor** — the same keys do different things depending on which mode you're in. This is the single most important concept to internalize.

**Steps to move between modes:**
1. On opening a file, `vim` starts in **Normal (Command) Mode** — keystrokes are interpreted as commands (navigation, deletion, copy/paste), not typed text.
2. Press `i` to enter **Insert Mode** — now keystrokes are typed directly into the file as text, just like a normal text editor.
3. Press `Esc` to return to **Normal Mode** from Insert Mode at any time.
4. From Normal Mode, press `:` to enter **Command-Line Mode** — this is where you type save/quit/search instructions at the bottom of the screen.

### 4.4 Core Normal Mode navigation

| Key | Action |
|---|---|
| `h` `j` `k` `l` | Move left, down, up, right (arrow keys also work) |
| `w` | Jump forward to the start of the next word |
| `b` | Jump backward to the start of the previous word |
| `0` | Jump to the start of the current line |
| `$` | Jump to the end of the current line |
| `gg` | Jump to the first line of the file |
| `G` | Jump to the last line of the file |
| `:n` | Jump to line number `n` (e.g., `:25`) |

### 4.5 Basic editing commands (Normal Mode)

| Key | Action |
|---|---|
| `x` | Delete the character under the cursor |
| `dd` | Delete (cut) the current line |
| `yy` | Yank (copy) the current line |
| `p` | Paste after the cursor/line |
| `u` | Undo the last change |
| `Ctrl + r` | Redo |
| `/searchterm` | Search forward for `searchterm`; press `n` to jump to the next match |

### 4.6 Saving and exiting (Command-Line Mode)

**Steps:**
1. Press `Esc` to make sure you're in Normal Mode.
2. Press `:` to open the command line at the bottom of the screen.
3. Type one of the following and press `Enter`:
   - `:w` — write (save) the file without exiting
   - `:q` — quit (only works if there are no unsaved changes)
   - `:wq` — write and quit in one step (the most common combo)
   - `:q!` — force-quit **without** saving, discarding all changes
4. Example full sequence for editing `/etc/fstab` safely:
   ```
   sudo vim /etc/fstab
   ```
   Press `i` → make your edits → press `Esc` → type `:wq` → press `Enter`.

---

## 5. The Linux File System Hierarchy

Linux organizes everything — files, devices, configuration, running processes — into a single unified tree rooted at `/`. Understanding this layout (the **Filesystem Hierarchy Standard**, or FHS) is essential for knowing *where* things live and *why*.

### 5.1 Exploring the hierarchy

**Steps:**
1. View the top level of the tree:
   ```
   ls -l /
   ```
2. Key directories and what they hold:

| Path | Purpose |
|---|---|
| `/` | The root of the entire file system — every other directory branches from here |
| `/home` | Personal directories for regular users (e.g., `/home/ec2-user`) |
| `/root` | The home directory for the `root` user specifically (separate from `/home` for security/isolation reasons) |
| `/etc` | System-wide configuration files (e.g., `/etc/fstab`, `/etc/passwd`, `/etc/os-release`) |
| `/var` | Variable data that changes frequently during runtime — logs (`/var/log`), spool files, caches |
| `/usr` | Installed user programs, libraries, and shared resources — the bulk of installed software lives here (`/usr/bin`, `/usr/lib`) |
| `/bin` and `/sbin` | Essential command binaries needed for basic system operation (on modern systems these are often symlinked into `/usr/bin` and `/usr/sbin`) |
| `/lib` | Shared libraries needed by the binaries in `/bin` and `/sbin` |
| `/opt` | Optional, third-party, or self-contained application packages |
| `/tmp` | Temporary files — typically cleared on reboot |
| `/dev` | Device files representing hardware (disks, terminals, etc.) as if they were regular files |
| `/proc` | A virtual file system exposing real-time kernel and process information (doesn't exist on disk — generated on the fly) |
| `/boot` | Kernel image and bootloader files needed to start the system |
| `/mnt` and `/media` | Standard mount points for manually mounted or removable file systems |

3. Practical example — locate where a specific config file lives using this mental model:
   ```
   ls -l /etc/os-release
   ```
   Since it's system-wide configuration/metadata (not user data, not a log), `/etc` is exactly where you'd expect to find it.
4. Practical example — confirm `/proc` is virtual, not on-disk:
   ```
   cat /proc/cpuinfo
   ```
   This returns live CPU details generated by the kernel at the moment you read it — there's no real file being read from a physical disk.

### 5.2 Absolute vs. relative paths

**Steps to practice both:**
1. An **absolute path** always starts from `/` and works regardless of your current location:
   ```
   cd /home/ec2-user/project-files
   ```
2. A **relative path** is interpreted from your current working directory:
   ```
   pwd
   cd project-files
   ```
3. Two useful shortcuts:
   - `.` refers to the current directory
   - `..` refers to the parent directory
   ```
   cd ..
   ```

---

## 6. File Management & System Resources

### 6.1 Resource monitoring with `free -h`

Because the `t3.micro` instance is capped at 1 GiB of RAM, memory pressure can cause AWS health checks (and the instance itself) to fail if left unmonitored.

**Steps:**
1. Check memory usage in human-readable form:
   ```
   free -h
   ```
   ```
   # Output:
                  total        used        free      shared  buff/cache   available
   Mem:           960Mi       150Mi       500Mi       8.0Mi       310Mi       650Mi
   Swap:             0B          0B          0B
   ```
2. Reading the columns:
   - **used** — memory actively held by running processes
   - **free** — completely unused memory
   - **buff/cache** — memory the kernel is using for disk caching; this is reclaimable on demand, so it isn't "wasted" memory
   - **available** — the realistic amount available for new processes, accounting for reclaimable cache (this is the number that actually matters, not "free")
3. Note that `Swap` shows `0B` — this `t3.micro` has no swap space configured, meaning if RAM is fully exhausted, the OOM (Out-Of-Memory) killer will start terminating processes rather than falling back to disk-based swap.

### 6.2 Package management with `dnf`

**Steps to install software on RHEL:**
1. Install a package (Apache HTTP Server, in this example):
   ```
   sudo dnf install httpd -y
   ```
2. `dnf` resolves and installs dependencies automatically — in this case pulling in `httpd-tools` and `redhat-logos-httpd` alongside the main `httpd` package.
3. Confirm installation succeeded by checking the package is registered:
   ```
   dnf list installed httpd
   ```
4. Other useful `dnf` operations:
   ```
   sudo dnf update -y          # update all installed packages
   sudo dnf remove httpd -y    # uninstall a package
   dnf search <keyword>        # search available packages
   ```

### 6.3 Filesystem operations

**Steps:**
1. Create a dedicated directory:
   ```
   mkdir /home/ec2-user/project-files
   ```
2. Create multiple empty files at once using brace expansion (the shell expands `{1..5}` into `1 2 3 4 5` before running the command):
   ```
   touch file{1..5}.conf
   ```
3. List all files, including hidden ones (dotfiles), with full permission/ownership detail:
   ```
   ls -lah
   ```
   ```
   # Output:
   total 12K
   drwxr-xr-x. 2 ec2-user ec2-user  93 Jun  4 09:20 .
   drwx------. 4 ec2-user ec2-user 115 Jun  4 09:18 ..
   -rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file1.conf
   -rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file2.conf
   -rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file3.conf
   -rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file4.conf
   -rw-r--r--. 1 ec2-user ec2-user   0 Jun  4 09:20 file5.conf
   ```
   - The `-l` flag shows the long format (permissions, owner, group, size, modified date).
   - The `-a` flag includes hidden entries — anything starting with `.`, including `.` (this directory) and `..` (the parent directory).
   - The `-h` flag makes file sizes human-readable (K/M/G instead of raw bytes).
4. Decode the permission string `-rw-r--r--`:
   - 1st character: file type (`-` = regular file, `d` = directory)
   - Next 3: owner permissions (`rw-` = read + write, no execute)
   - Next 3: group permissions (`r--` = read only)
   - Last 3: everyone-else permissions (`r--` = read only)
5. Remove a specific file:
   ```
   rm file1.conf
   ```

### 6.4 Copying, moving, renaming, and deleting — files and directories

`vim` and `dnf` get most of the attention, but day-to-day administration is dominated by basic file/directory manipulation. Linux uses the **same commands** for files and directories in most cases, just with different flags.

**Copying files:**
```
cp file1.conf file1-backup.conf        # copy a file, giving it a new name
cp file1.conf /home/ec2-user/backups/  # copy a file into another directory (keeps original name)
```

**Copying directories:**
```
cp -r project-files/ project-files-backup/
```
- The `-r` (recursive) flag is **required** for directories — `cp` will refuse to copy a directory without it, since a plain `cp` only knows how to handle individual files.
- Add `-v` (verbose) to any of these commands to print each file as it's copied — useful for confirming large copy operations are actually progressing:
  ```
  cp -rv project-files/ project-files-backup/
  ```

**Moving and renaming — files and directories:**
```
mv file2.conf file2-renamed.conf              # rename a file in place
mv file2-renamed.conf /home/ec2-user/backups/ # move a file into another directory
mv project-files/ project-files-old/          # rename a directory
mv project-files/ /home/ec2-user/backups/     # move a directory into another directory
```
- Unlike `cp`, `mv` does **not** need a `-r` flag for directories — moving/renaming doesn't require reading through the directory's contents, so the same command works for both files and directories unmodified.
- `mv` is also how you rename anything on Linux — there is no separate `rename` command in common use; renaming is just "moving" a file to a new name in the same location.

**Deleting files:**
```
rm file3.conf           # delete a single file
rm -i file3.conf        # prompt for confirmation before deleting (safer)
rm file*.conf           # delete multiple files matching a wildcard pattern
```

**Deleting directories:**
```
rmdir empty-folder/            # remove a directory, but ONLY if it is completely empty
rm -r project-files-old/       # remove a directory and everything inside it, recursively
rm -rf project-files-old/      # same as above, but force it (no confirmation prompts, ignores nonexistent files)
```
- `rmdir` is the safe option — it fails loudly if the directory still has files in it, preventing accidental data loss.
- `rm -rf` is powerful and destructive: it deletes recursively (`-r`) and forces (`-f`) the operation without asking for confirmation. There is **no trash/recycle bin** on a Linux server — anything removed this way is gone immediately. Always double-check the path before running it, especially with `sudo`.

**Quick reference table:**

| Action | On a file | On a directory |
|---|---|---|
| Copy | `cp source dest` | `cp -r source/ dest/` |
| Move / rename | `mv source dest` | `mv source/ dest/` |
| Delete | `rm file` | `rmdir dir/` (empty only) or `rm -r dir/` |
| Force delete | `rm -f file` | `rm -rf dir/` |

---

## 7. Data Streams & OS-Level Security

### 7.1 Reading files and streams

**Steps:**
1. View an entire short file at once:
   ```
   cat /etc/os-release
   ```
2. View just the first N lines of a large file (useful for headers or checking a file's beginning without loading all of it):
   ```
   sudo head -n 5 /var/log/messages
   ```
3. View just the last N lines (the most common way to check recent activity):
   ```
   sudo tail -n 5 /var/log/messages
   ```
4. Follow a log file live, printing new lines as they're written — critical for real-time troubleshooting of an application as you reproduce an issue:
   ```
   sudo tail -f /var/log/messages
   ```
   Press `Ctrl + C` to stop following.

### 7.2 Pattern matching with `grep`

**Steps:**
1. Search a file for a specific string, case-insensitively (`-i`), to audit failed SSH login attempts:
   ```
   sudo grep -i "failed" /var/log/secure
   ```
   ```
   # Output:
   Jun  4 08:45:12 ip-172-31-16-122 sshd[1204]: Failed password for invalid user admin from 198.51.100.23 port 54321 ssh2
   ```
2. Useful `grep` flags to know:
   ```
   grep -r "pattern" /some/directory/    # recursive search through a directory
   grep -c "pattern" file.txt            # count matching lines instead of printing them
   grep -v "pattern" file.txt            # invert match — show lines that do NOT match
   ```

### 7.3 I/O redirection and pipelining

**Steps:**
1. **Overwrite redirection (`>`)** — create/replace a file with new content:
   ```
   echo "EC2 Server Config v1" > server-info.txt
   ```
2. **Append redirection (`>>`)** — add to the end of a file without erasing existing content:
   ```
   echo "Instance Type: t3.micro" >> server-info.txt
   ```
3. **Pipelining (`|`)** — feed the output of one command directly into another as input, rather than manually copying it:
   ```
   ps -aux | grep httpd
   ```
   ```
   # Output:
   root        1522  0.0  0.5  17764  5320 ?        Ss   09:25   0:00 /usr/sbin/httpd -DFOREGROUND
   apache      1523  0.0  0.4  18020  4112 ?        S    09:25   0:00 /usr/sbin/httpd -DFOREGROUND
   apache      1524  0.0  0.4  18020  4112 ?        S    09:25   0:00 /usr/sbin/httpd -DFOREGROUND
   ec2-user    1550  0.0  0.2   6408  2148 pts/0    S+   09:27   0:00 grep --color=auto httpd
   ```
   `ps -aux` lists every running process on the system; piping it into `grep httpd` filters that massive list down to just the Apache-related processes — far faster than scrolling through the full output manually.

### 7.4 File immutability with `chattr`

Accidentally deleting or overwriting a critical boot file (like `/etc/fstab`) can leave an EC2 instance unbootable. The `chattr` command sets special file attributes at the file system level that override normal permission checks — even `root` can't modify or delete an immutable file without first removing the lock.

**Steps:**
1. Create a critical configuration file to protect:
   ```
   touch /home/ec2-user/critical-config.conf
   ```
2. Apply the immutable attribute (`+i`):
   ```
   sudo chattr +i /home/ec2-user/critical-config.conf
   ```
3. Verify the attribute was applied — look for the `i` flag in the output:
   ```
   lsattr /home/ec2-user/critical-config.conf
   ```
   ```
   # Output:
   ----i----------------- /home/ec2-user/critical-config.conf
   ```
4. Confirm the protection works by attempting to delete the file (this should fail with "Operation not permitted"):
   ```
   sudo rm /home/ec2-user/critical-config.conf
   ```
5. To make changes again later, remove the immutable attribute first:
   ```
   sudo chattr -i /home/ec2-user/critical-config.conf
   ```

---

*This document serves as a structured technical reference and portfolio summary of core AWS and Linux system administration skills covered in Week 1.*
