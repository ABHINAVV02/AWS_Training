# AWS & Linux System Administration — Week 2 Course Notes

A detailed, step-by-step reference covering core Linux system administration, storage management, AWS EBS integration, LVM, and security/troubleshooting practices, capped off with a real-world deployment project.

---

## Table of Contents

1. [System Administration & Log Management](#1-system-administration--log-management)
2. [File System & Disk Management](#2-file-system--disk-management)
3. [AWS Storage & EBS](#3-aws-storage--ebs)
4. [Logical Volume Management (LVM)](#4-logical-volume-management-lvm)
5. [Advanced Troubleshooting & Security](#5-advanced-troubleshooting--security)
6. [Project Overview](#6-project-overview)

---

## 1. System Administration & Log Management

Effective system administration starts with knowing where the system records its activity and how to control the services generating that activity.

### 1.1 Reading log files and the log directory structure

Almost all system and application logs live under `/var/log`. Knowing the layout lets you find the right log quickly instead of guessing.

**Steps:**
1. List the log directory to see what's available:
   ```
   ls -lh /var/log
   ```
2. Key files/folders to know:
   - `/var/log/messages` (RHEL/CentOS) or `/var/log/syslog` (Debian/Ubuntu) — general system messages
   - `/var/log/secure` (RHEL/CentOS) or `/var/log/auth.log` (Debian/Ubuntu) — authentication and login attempts
   - `/var/log/cron` — scheduled task execution logs
   - `/var/log/dmesg` — kernel ring buffer / boot-time hardware messages
   - `/var/log/httpd/` or `/var/log/nginx/` — web server access and error logs
3. View a log file live (most recent entries as they're written):
   ```
   tail -f /var/log/messages
   ```
4. Search a log for a specific keyword or error:
   ```
   grep -i "error" /var/log/messages
   ```
5. View the last N lines of a log without following it:
   ```
   tail -n 100 /var/log/secure
   ```

### 1.2 Monitoring network activity with `netstat`

`netstat` shows active connections, listening ports, and routing information — essential for confirming a service is actually listening where you expect.

**Steps:**
1. Install `netstat` if it isn't present (it ships in the `net-tools` package):
   ```
   sudo yum install net-tools -y      # RHEL/CentOS
   sudo apt install net-tools -y      # Debian/Ubuntu
   ```
2. List all listening TCP ports along with the owning process:
   ```
   sudo netstat -tulnp
   ```
   - `-t` TCP, `-u` UDP, `-l` listening only, `-n` numeric (no DNS lookups), `-p` show process/PID
3. Check if a specific port (e.g., 8000 for Gunicorn) is listening:
   ```
   sudo netstat -tulnp | grep 8000
   ```
4. View all active established connections:
   ```
   netstat -an | grep ESTABLISHED
   ```
5. View the routing table:
   ```
   netstat -r
   ```

### 1.3 Managing service files at `/usr/lib/systemd`

This is where package-installed `systemd` unit files (the default definitions for each service) live, as distinct from `/etc/systemd/system`, which holds local overrides and custom units.

**Steps:**
1. List the default service unit files:
   ```
   ls /usr/lib/systemd/system
   ```
2. View the contents of an existing unit file to understand its structure:
   ```
   cat /usr/lib/systemd/system/sshd.service
   ```
3. A typical unit file has three sections:
   ```
   [Unit]
   Description=My Custom Service
   After=network.target

   [Service]
   ExecStart=/usr/bin/python3 /opt/myapp/app.py
   Restart=always
   User=myappuser

   [Install]
   WantedBy=multi-user.target
   ```
4. **Never edit files directly in `/usr/lib/systemd/system`** — they can be overwritten on package updates. Instead, place custom or overriding units in `/etc/systemd/system`.
5. After creating or editing a unit file, reload `systemd`'s configuration cache so it recognizes the change:
   ```
   sudo systemctl daemon-reload
   ```

### 1.4 Using `systemctl` for service management

**Steps:**
1. Check the current status of a service:
   ```
   sudo systemctl status sshd
   ```
2. Start a stopped service:
   ```
   sudo systemctl start sshd
   ```
3. Stop a running service:
   ```
   sudo systemctl stop sshd
   ```
4. Enable a service to start automatically at boot:
   ```
   sudo systemctl enable sshd
   ```
5. Disable a service from starting at boot:
   ```
   sudo systemctl disable sshd
   ```
6. Check whether a service is enabled at boot:
   ```
   systemctl is-enabled sshd
   ```
7. List all currently active services:
   ```
   systemctl list-units --type=service --state=running
   ```

### 1.5 Service reload vs. restart

**Steps to test the difference:**
1. Reload a service's configuration without dropping active connections (used after a config file change that the service supports re-reading live):
   ```
   sudo systemctl reload nginx
   ```
2. Fully restart a service — stops the process completely, then starts it fresh (needed when a change can't be hot-reloaded, or the process is unresponsive):
   ```
   sudo systemctl restart nginx
   ```
3. If you're unsure whether a service supports reload, use the combined command, which reloads if supported and restarts if not:
   ```
   sudo systemctl reload-or-restart nginx
   ```
4. Confirm the change took effect by checking the service's uptime in its status output:
   ```
   sudo systemctl status nginx
   ```
   - After a **restart**, "Active since" resets to the current time.
   - After a **reload**, "Active since" stays at the original start time (the process never stopped).

---

## 2. File System & Disk Management

### 2.1 Managing automount and safe mount via `/etc/fstab`

`/etc/fstab` defines which file systems should be mounted automatically at boot, and how.

**Steps:**
1. View the current fstab entries:
   ```
   cat /etc/fstab
   ```
2. A typical entry has six fields:
   ```
   UUID=xxxx-xxxx  /data  ext4  defaults  0  2
   ```
   - Field 1: device (by UUID, label, or path)
   - Field 2: mount point
   - Field 3: file system type
   - Field 4: mount options (`defaults`, `nofail`, `noauto`, etc.)
   - Field 5: dump flag (backup utility, usually `0`)
   - Field 6: `fsck` pass order (`0` = skip, `1` = root, `2` = others)
3. Add a new automount entry (edit as root):
   ```
   sudo nano /etc/fstab
   ```
   Append a line, for example:
   ```
   /dev/xvdf1  /data  ext4  defaults,nofail  0  2
   ```
   - `nofail` is the "safe mount" option — it tells the boot process not to hang or fail if the device isn't present, which is critical for cloud instances where a volume might be detached.
4. Apply the fstab changes without rebooting:
   ```
   sudo mount -a
   ```
5. Verify the mount succeeded:
   ```
   df -h
   ```

### 2.2 Troubleshooting `fstab` failures (detaching a drive with a stale entry)

If a drive is detached (or fails) while `/etc/fstab` still references it **without** `nofail`, the system can hang at boot or drop into emergency mode.

**Steps to fix:**
1. If the system boots into emergency/rescue mode, log in with the root password when prompted.
2. Open `/etc/fstab` (it's typically mounted read-write even in emergency mode):
   ```
   nano /etc/fstab
   ```
3. Either comment out the offending line:
   ```
   # /dev/xvdf1  /data  ext4  defaults  0  2
   ```
   or add the `nofail` option so future detachments don't block boot:
   ```
   /dev/xvdf1  /data  ext4  defaults,nofail  0  2
   ```
4. Save and exit, then reboot:
   ```
   sudo reboot
   ```
5. To prevent this in the future, always test new fstab entries before rebooting:
   ```
   sudo mount -a
   ```
   If this command errors out, fix the entry **before** rebooting — a failed `mount -a` today is a failed boot tomorrow.

### 2.3 Checking disk space with `df -h`

**Steps:**
1. View disk usage for all mounted file systems in human-readable form:
   ```
   df -h
   ```
2. Check usage for a specific mount point:
   ```
   df -h /data
   ```
3. Include file system type in the output:
   ```
   df -hT
   ```
4. If you also need to know which directories are consuming space (not just which disks), use:
   ```
   du -sh /var/* | sort -rh | head -10
   ```

### 2.4 Partitioning schemes: MBR

**Steps to view and create an MBR partition:**
1. Identify the disk you want to partition:
   ```
   lsblk
   ```
2. Launch `fdisk` on the target disk (MBR is `fdisk`'s native scheme):
   ```
   sudo fdisk /dev/xvdf
   ```
3. Inside the `fdisk` prompt:
   - Press `n` to create a new partition
   - Choose `p` for primary (MBR supports up to 4 primary partitions)
   - Accept or specify the partition number, start sector, and end sector (or size, e.g., `+10G`)
   - Press `w` to write the changes to disk and exit
4. Confirm the new partition exists:
   ```
   lsblk
   sudo fdisk -l /dev/xvdf
   ```
5. Note the MBR limitation: only 4 primary partitions are allowed. To get more, one primary must be converted into an **extended partition** containing logical partitions (see Section 4.3).

### 2.5 Creating a partition, creating a file system, and mounting it (including permanent mounts)

**Full end-to-end steps:**
1. Create the partition (see 2.4 above), resulting in a device like `/dev/xvdf1`.
2. Format the partition with a file system:
   ```
   sudo mkfs.ext4 /dev/xvdf1
   ```
   or, for XFS:
   ```
   sudo mkfs.xfs /dev/xvdf1
   ```
3. Create a mount point directory:
   ```
   sudo mkdir -p /data
   ```
4. Mount it temporarily (lasts until reboot):
   ```
   sudo mount /dev/xvdf1 /data
   ```
5. Verify:
   ```
   df -h /data
   ```
6. To unmount:
   ```
   sudo umount /data
   ```
7. To make the mount **permanent** (survives reboot), find the partition's UUID:
   ```
   sudo blkid /dev/xvdf1
   ```
8. Add an entry to `/etc/fstab` using that UUID:
   ```
   sudo nano /etc/fstab
   ```
   ```
   UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /data  ext4  defaults,nofail  0  2
   ```
9. Test the entry without rebooting:
   ```
   sudo mount -a
   df -h
   ```

### 2.6 Why usable storage is less than the advertised size (metadata)

**How to demonstrate this on a real disk:**
1. Check the raw size of the block device:
   ```
   lsblk -b /dev/xvdf1
   ```
2. Format it and immediately check usable space:
   ```
   sudo mkfs.ext4 /dev/xvdf1
   sudo mount /dev/xvdf1 /data
   df -h /data
   ```
3. Compare the two numbers — the `df -h` figure will be smaller. This gap is consumed by:
   - The **superblock** (file system metadata describing the volume)
   - **Inode tables** (one inode per file/directory, reserved up front)
   - The **journal** (ext4/xfs journaling for crash recovery)
   - Reserved blocks (ext-family file systems reserve ~5% for root by default)
4. View reserved-block percentage on an ext4 volume:
   ```
   sudo tune2fs -l /dev/xvdf1 | grep -i "reserved"
   ```
5. Reduce (or remove) the reserved-block percentage if needed:
   ```
   sudo tune2fs -m 1 /dev/xvdf1
   ```

### 2.7 Working principle of a hard drive

**Steps to inspect this on your own system:**
1. View physical/geometry details of a disk (platters, heads, cylinders — most meaningful on physical drives):
   ```
   sudo fdisk -l /dev/xvdf
   ```
2. Conceptual model to know:
   - A hard drive stores data on spinning **platters**, divided into concentric **tracks**.
   - Each track is divided into **sectors** (traditionally 512 bytes, now often 4K).
   - A **read/write head** moves across the platter to access specific tracks — this seek time is why sequential access is faster than random access on spinning disks.
   - Partitioning and file systems map logical block addresses on top of this physical sector layout, which is why disk alignment matters for performance.
3. On cloud EBS volumes, this physical model is abstracted away (EBS is virtualized network storage), but the logical addressing concepts — sectors, blocks, and inode-to-block mapping — still apply at the file system layer.

---

## 3. AWS Storage & EBS

### 3.1 Key Management Service (KMS)

**Steps to view and use a KMS key for EBS encryption:**
1. In the AWS Console, go to **KMS > Customer managed keys**.
2. Click **Create key**, choose "Symmetric," and follow the wizard to name and set permissions for the key.
3. Via AWS CLI, list existing keys:
   ```
   aws kms list-keys
   ```
4. Use a KMS key when creating an encrypted EBS volume (Console: check "Encrypt this volume" and select the key; CLI shown in 3.3 below).

### 3.2 Encryption limitation: encrypted volumes can't be manually decrypted elsewhere

**What to know / how to observe this in practice:**
1. Create an encrypted volume tied to a specific KMS key.
2. Attempt to attach that volume to a different account or region without KMS key access permissions — the attach will succeed, but reading the data will fail because decryption is handled transparently by AWS using the KMS key, not by a manual decrypt operation.
3. Practical implication: if you need to share an encrypted volume/snapshot across accounts, you must **share the KMS key permissions** (grant the target account `kms:Decrypt` on the key) — there's no CLI command like `decrypt-volume` that strips encryption.
4. To copy an encrypted snapshot and re-encrypt with a key you control:
   ```
   aws ec2 copy-snapshot \
     --source-region us-east-1 \
     --source-snapshot-id snap-0123456789abcdef0 \
     --encrypted \
     --kms-key-id alias/my-key
   ```

### 3.3 Managing EBS: attach, detach, and mount

**Steps:**
1. Create a new EBS volume (must be in the same Availability Zone as the target instance):
   ```
   aws ec2 create-volume \
     --availability-zone us-east-1a \
     --size 20 \
     --volume-type gp3
   ```
2. Attach it to a running instance:
   ```
   aws ec2 attach-volume \
     --volume-id vol-0123456789abcdef0 \
     --instance-id i-0123456789abcdef0 \
     --device /dev/xvdf
   ```
3. On the instance, confirm the new device appeared:
   ```
   lsblk
   ```
4. Format and mount it (see Section 2.5 for the full partition/format/mount workflow).
5. Detach the volume when finished (unmount first!):
   ```
   sudo umount /data
   aws ec2 detach-volume --volume-id vol-0123456789abcdef0
   ```

### 3.4 NAS, SAN, and Object Storage

**How to explore each in AWS:**
1. **NAS-style storage** — Amazon EFS (Elastic File System). Mount it over NFS:
   ```
   sudo mount -t nfs4 fs-0123456.efs.us-east-1.amazonaws.com:/ /mnt/efs
   ```
2. **SAN-style storage** — Amazon EBS itself behaves like a SAN: block-level, one volume per attaching instance, presented as a raw block device (`/dev/xvdf`) that you partition and format yourself (see 3.3 above).
3. **Object Storage** — Amazon S3. Upload/download objects via the CLI rather than mounting a file system:
   ```
   aws s3 cp myfile.txt s3://my-bucket/myfile.txt
   aws s3 ls s3://my-bucket/
   ```
4. Refer to the comparison table below to decide which fits a given workload.

### 3.5 iSCSI

**Steps to conceptually/practically set up an iSCSI target-initiator pair (on-prem or self-managed equivalent of SAN):**
1. On the storage server (target), install the iSCSI target utilities:
   ```
   sudo yum install targetcli -y
   ```
2. Use `targetcli` to create a backing store and export it as an iSCSI target (LUN).
3. On the client (initiator), install the initiator tools:
   ```
   sudo yum install iscsi-initiator-utils -y
   ```
4. Discover available targets on the storage server:
   ```
   sudo iscsiadm -m discovery -t sendtargets -p <target-server-ip>
   ```
5. Log in to the discovered target so it appears as a local block device:
   ```
   sudo iscsiadm -m node --login
   ```
6. Confirm the new iSCSI device:
   ```
   lsblk
   ```
   From here, it's partitioned, formatted, and mounted exactly like any other block device (Section 2.5) — this is why iSCSI served the same purpose as SAN before purpose-built SAN hardware/services became common.

### 3.6 Generating a new UUID to attach a root volume to another instance for troubleshooting

**Steps:**
1. Stop the problem instance (do not terminate):
   ```
   aws ec2 stop-instances --instance-ids i-0123456789abcdef0
   ```
2. Detach its root volume:
   ```
   aws ec2 detach-volume --volume-id vol-0123456789abcdef0
   ```
3. Attach it to a healthy "rescue" instance as a secondary device:
   ```
   aws ec2 attach-volume \
     --volume-id vol-0123456789abcdef0 \
     --instance-id i-0rescueinstance000 \
     --device /dev/xvdf
   ```
4. On the rescue instance, mount it to inspect/fix files:
   ```
   sudo mkdir -p /mnt/rescue
   sudo mount /dev/xvdf1 /mnt/rescue
   ```
5. If the rescue instance already has a file system with the **same UUID** (common when both volumes came from the same AMI), the mount can fail or misbehave. Generate a new UUID for the attached volume:
   ```
   sudo tune2fs -U random /dev/xvdf1     # ext4
   ```
   or for XFS:
   ```
   sudo xfs_admin -U generate /dev/xvdf1
   ```
6. Retry the mount:
   ```
   sudo mount /dev/xvdf1 /mnt/rescue
   ```
7. After troubleshooting, unmount, detach, and reattach the volume back to the original instance as `/dev/xvda` (or its original root device name), then start the instance again.

### 3.7 Lifecycle Manager for automated snapshots

**Steps (AWS Console):**
1. Go to **EC2 > Elastic Block Store > Lifecycle Manager**.
2. Click **Create lifecycle policy**.
3. Choose the policy type: **EBS snapshot policy** (or EBS-backed AMI policy).
4. Select target resources by tag (e.g., tag EBS volumes with `Backup=true` and target that tag).
5. Define the schedule (e.g., daily at 03:00 UTC) and retention count (e.g., keep the last 7 snapshots).
6. Assign an IAM role that grants the Lifecycle Manager permission to create/delete snapshots.
7. Review and create the policy — snapshots will now be created and pruned automatically going forward.

**Equivalent via CLI:**
```
aws dlm create-lifecycle-policy \
  --description "Daily EBS backups" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/DLMRole \
  --policy-details file://policy-details.json
```

### Storage Model Comparison

| Storage Type | Access Method | AWS Equivalent | Typical Use Case |
|---|---|---|---|
| **NAS** | File-level access over a network (NFS/SMB) | Amazon EFS | Shared file access across multiple servers |
| **SAN** | Block-level access over a dedicated network | Amazon EBS | High-performance, low-latency block storage for servers/databases |
| **Object Storage** | API/HTTP-based access | Amazon S3 | Storing large volumes of unstructured data (backups, media, static assets) |

---

## 4. Logical Volume Management (LVM)

### 4.1 Creating logical volumes

**Full step-by-step LVM setup:**
1. Identify the raw disk/partition to use:
   ```
   lsblk
   ```
2. Initialize it as a Physical Volume (PV):
   ```
   sudo pvcreate /dev/xvdf
   ```
3. Confirm the PV was created:
   ```
   sudo pvdisplay
   ```
4. Create a Volume Group (VG) from the PV:
   ```
   sudo vgcreate data_vg /dev/xvdf
   ```
5. Confirm the VG:
   ```
   sudo vgdisplay data_vg
   ```
6. Create a Logical Volume (LV) inside the VG (e.g., 15G out of the pool):
   ```
   sudo lvcreate -L 15G -n data_lv data_vg
   ```
7. Confirm the LV:
   ```
   sudo lvdisplay /dev/data_vg/data_lv
   ```
8. Format and mount the LV exactly like a normal partition:
   ```
   sudo mkfs.ext4 /dev/data_vg/data_lv
   sudo mkdir -p /data
   sudo mount /dev/data_vg/data_lv /data
   ```
9. Make it permanent via `/etc/fstab` (see Section 2.5, steps 7–9).

### 4.2 Why logical volumes are necessary

**Steps to demonstrate the key benefit — resizing without repartitioning:**
1. Suppose `/data` (the LV from 4.1) is running low on space. Check current usage:
   ```
   df -h /data
   ```
2. Extend the underlying Volume Group with a brand-new disk (no need to touch the existing partition layout):
   ```
   sudo pvcreate /dev/xvdg
   sudo vgextend data_vg /dev/xvdg
   ```
3. Grow the Logical Volume using the newly added space:
   ```
   sudo lvextend -L +10G /dev/data_vg/data_lv
   ```
4. Grow the file system to use the new space (ext4 example):
   ```
   sudo resize2fs /dev/data_vg/data_lv
   ```
   or for XFS:
   ```
   sudo xfs_growfs /data
   ```
5. Confirm the extra space is now usable — no downtime, no repartitioning, no data migration:
   ```
   df -h /data
   ```

### 4.3 Extended partitions

**Steps to create an extended partition (needed to exceed MBR's 4-primary-partition limit):**
1. Launch `fdisk` on the target disk:
   ```
   sudo fdisk /dev/xvdf
   ```
2. Press `n` for a new partition, then choose `e` for **extended** instead of `p` for primary.
3. Accept the default start/end (typically sized to use all remaining disk space).
4. Press `n` again to add a **logical** partition inside the extended partition — `fdisk` will offer `l` (logical) automatically once an extended partition exists.
5. Repeat step 4 for each additional logical partition needed.
6. Press `w` to write the partition table.
7. Confirm the layout:
   ```
   sudo fdisk -l /dev/xvdf
   lsblk
   ```
   Logical partitions will appear numbered from 5 onward (e.g., `/dev/xvdf5`, `/dev/xvdf6`), since 1–4 are reserved for primary/extended partitions in MBR.

### 4.4 Combining logical partitions from different hard drives into one logical volume

**Steps:**
1. Prepare each physical disk/partition as a PV:
   ```
   sudo pvcreate /dev/xvdf1
   sudo pvcreate /dev/xvdg1
   sudo pvcreate /dev/xvdh1
   ```
2. Confirm all PVs are recognized:
   ```
   sudo pvs
   ```
3. Create a single Volume Group spanning all three physical volumes:
   ```
   sudo vgcreate combined_vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
   ```
4. Confirm the combined capacity:
   ```
   sudo vgdisplay combined_vg
   ```
5. Create one Logical Volume that draws space from across all the underlying disks:
   ```
   sudo lvcreate -l 100%FREE -n combined_lv combined_vg
   ```
6. Format and mount it as a single, unified volume:
   ```
   sudo mkfs.ext4 /dev/combined_vg/combined_lv
   sudo mkdir -p /bigdata
   sudo mount /dev/combined_vg/combined_lv /bigdata
   ```
7. Confirm total usable space now reflects all combined disks:
   ```
   df -h /bigdata
   ```

---

## 5. Advanced Troubleshooting & Security

### 5.1 Troubleshooting a lost instance key (EC2)

**Steps to regain access without the original SSH key:**
1. Stop the affected instance (don't terminate):
   ```
   aws ec2 stop-instances --instance-ids i-0123456789abcdef0
   ```
2. Detach its root EBS volume:
   ```
   aws ec2 detach-volume --volume-id vol-0123456789abcdef0
   ```
3. Attach that volume as a **secondary** device to a healthy "rescue" instance you already have access to:
   ```
   aws ec2 attach-volume \
     --volume-id vol-0123456789abcdef0 \
     --instance-id i-0rescueinstance000 \
     --device /dev/xvdf
   ```
4. On the rescue instance, mount the attached volume:
   ```
   sudo mkdir -p /mnt/recover
   sudo mount /dev/xvdf1 /mnt/recover
   ```
5. Inject your own public key into the recovered volume's `authorized_keys` file so the original instance will trust your key pair:
   ```
   sudo mkdir -p /mnt/recover/home/ec2-user/.ssh
   sudo cp ~/.ssh/authorized_keys /mnt/recover/home/ec2-user/.ssh/authorized_keys
   sudo chown -R 1000:1000 /mnt/recover/home/ec2-user/.ssh
   sudo chmod 700 /mnt/recover/home/ec2-user/.ssh
   sudo chmod 600 /mnt/recover/home/ec2-user/.ssh/authorized_keys
   ```
6. Unmount and detach the volume from the rescue instance:
   ```
   sudo umount /mnt/recover
   aws ec2 detach-volume --volume-id vol-0123456789abcdef0
   ```
7. Reattach the volume to the original instance as its root device (typically `/dev/xvda` or `/dev/sda1`):
   ```
   aws ec2 attach-volume \
     --volume-id vol-0123456789abcdef0 \
     --instance-id i-0123456789abcdef0 \
     --device /dev/xvda
   ```
8. Start the original instance and SSH in using your new key pair:
   ```
   aws ec2 start-instances --instance-ids i-0123456789abcdef0
   ssh -i my-new-key.pem ec2-user@<instance-ip>
   ```

### 5.2 Troubleshooting mount/detach failures in `fstab`

*(Full walkthrough already covered in Section 2.2 — summarized here as the security/recovery checklist.)*

**Quick-reference recovery checklist:**
1. If boot hangs, wait for the prompt to drop into **emergency mode**, or reboot and give the root password when asked.
2. Remount the root file system as read-write if needed:
   ```
   mount -o remount,rw /
   ```
3. Edit `/etc/fstab` and comment out or fix the broken entry:
   ```
   nano /etc/fstab
   ```
4. Add `nofail` (and optionally `noauto` for drives that shouldn't block boot at all) to any non-critical mounts going forward.
5. Validate before rebooting:
   ```
   mount -a
   ```
6. Only reboot once `mount -a` returns no errors:
   ```
   reboot
   ```

---

## 6. Project Overview

Alongside the topics above, the week also included a hands-on project: deploying a **Django** web application on an AWS EC2 instance, using **Gunicorn** as the WSGI application server and **Nginx** as the reverse proxy in front of it — Nginx received client requests and forwarded dynamic traffic to Gunicorn, which ran the Django app itself.

---

*This document serves as a structured technical reference and portfolio summary of core Linux and AWS system administration skills covered in Week 2.*
