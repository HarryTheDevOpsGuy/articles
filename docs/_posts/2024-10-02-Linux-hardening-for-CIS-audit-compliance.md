---
title: Setup Production-ready Redis Sentinel
key: 20241002
tags: howtos
author: Harry - The DevOps Guy
#show_author_profile: true
aside:
  toc: true
---

Hardening a Linux system for CIS (Center for Internet Security) compliance ensures that the system is more secure and protected against potential attacks. CIS provides detailed security benchmarks for different operating systems, including Linux, which include steps for configuration, auditing, and ongoing monitoring.

Here’s a step-by-step guide to harden a Linux OS according to the CIS benchmarks:

### Step 1: **Initial Setup**

#### 1.1 Install Updates and Patches
Ensure your system is fully up-to-date with the latest security patches and updates.
```bash
sudo yum update -y  # For CentOS/RHEL/Amazon Linux
sudo apt update && sudo apt upgrade -y  # For Ubuntu/Debian
```

#### 1.2 Remove Unnecessary Packages
Remove any unused or unnecessary software to reduce potential attack vectors.
```bash
sudo yum remove package-name -y
sudo apt-get remove --purge package-name -y
```

#### 1.3 Enable Automatic Updates
For better security, enable automatic updates.
- For Ubuntu/Debian:
  ```bash
  sudo apt install unattended-upgrades
  sudo dpkg-reconfigure -plow unattended-upgrades
  ```
- For CentOS/RHEL/Amazon Linux, install `yum-cron`:
  ```bash
  sudo yum install yum-cron
  sudo systemctl enable yum-cron
  sudo systemctl start yum-cron
  ```

### Step 2: **File System Configuration**

#### 2.1 Create Separate Partitions
Ensure sensitive data and logs have separate partitions to limit the damage if compromised. Some important partitions include:
- `/var`
- `/tmp`
- `/home`
- `/var/log`

Edit `/etc/fstab` to mount the partitions with appropriate options like `nosuid`, `nodev`, `noexec`.

Example for `/tmp`:
```bash
UUID=<partition-uuid> /tmp ext4 defaults,nosuid,noexec,nodev 0 0
```

#### 2.2 Disable Unused File Systems
Disable file systems that aren’t needed. Add the following lines to `/etc/modprobe.d/CIS.conf`:
```bash
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
```

#### 2.3 Disable Mounting of USB Storage
Add the following to `/etc/modprobe.d/usb-storage.conf`:
```bash
install usb-storage /bin/true
```

### Step 3: **Network Configuration**

#### 3.1 Disable Unnecessary Network Services
Stop and disable unnecessary services like `telnet`, `rlogin`, and `rsh`. Use `systemctl` to stop and disable services:
```bash
sudo systemctl disable telnet.socket
sudo systemctl stop telnet.socket
```

#### 3.2 Configure Hostname and DNS Settings
Edit `/etc/hostname` to set your hostname, and update `/etc/hosts` to resolve the hostname to an IP address.

#### 3.3 Configure the Firewall
Set up a basic firewall configuration using `firewalld` (for CentOS/RHEL) or `ufw` (for Ubuntu).

- For CentOS/RHEL:
  ```bash
  sudo yum install firewalld
  sudo systemctl start firewalld
  sudo firewall-cmd --permanent --add-service=ssh
  sudo firewall-cmd --reload
  ```

- For Ubuntu:
  ```bash
  sudo ufw enable
  sudo ufw allow ssh
  ```

#### 3.4 Disable IPv6 if Not Needed
Add the following to `/etc/sysctl.conf` to disable IPv6:
```bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```
Apply changes with:
```bash
sudo sysctl -p
```

#### 3.5 Harden Network Parameters
Modify `/etc/sysctl.conf` for secure network settings:
```bash
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
```
Reload the settings:
```bash
sudo sysctl -p
```

### Step 4: **User and Authentication Configurations**

#### 4.1 Set Password Policies
Modify `/etc/security/pwquality.conf` to enforce strong password policies:
```bash
minlen = 14
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
```

#### 4.2 Configure Account Lockout Policy
Set up account lockout policies in `/etc/pam.d/common-auth` (Ubuntu) or `/etc/pam.d/system-auth` (CentOS/RHEL):
```bash
auth required pam_tally2.so onerr=fail audit silent deny=5 unlock_time=900
```

#### 4.3 Restrict Root Login
- Disable root login via SSH by editing `/etc/ssh/sshd_config`:
  ```bash
  PermitRootLogin no
  ```
- Apply the changes:
  ```bash
  sudo systemctl reload sshd
  ```

#### 4.4 Use SSH Key-based Authentication
Disable password-based SSH authentication by setting the following in `/etc/ssh/sshd_config`:
```bash
PasswordAuthentication no
```
Restart SSH:
```bash
sudo systemctl reload sshd
```

### Step 5: **Logging and Auditing**

#### 5.1 Enable Auditd
Ensure `auditd` is installed and running for logging:
```bash
sudo yum install audit -y  # For CentOS/RHEL
sudo apt install auditd -y  # For Ubuntu
sudo systemctl enable auditd
sudo systemctl start auditd
```

#### 5.2 Configure Auditing for Key Events
Edit `/etc/audit/rules.d/audit.rules` to include rules for monitoring critical system events:
```bash
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/group -p wa -k group_changes
-w /var/log/lastlog -p wa -k logins
```
Reload the auditd service:
```bash
sudo systemctl restart auditd
```

#### 5.3 Enable Logging for Important System Events
Ensure the `rsyslog` service is installed and running:
```bash
sudo yum install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

#### 5.4 Configure Log Rotation
Ensure that logs are rotated properly by checking `/etc/logrotate.conf` and adjusting log retention periods as needed.

### Step 6: **Intrusion Detection and File Integrity**

#### 6.1 Install and Configure AIDE
AIDE (Advanced Intrusion Detection Environment) can help monitor file changes.
```bash
sudo yum install aide -y  # For CentOS/RHEL
sudo apt install aide -y  # For Ubuntu
```
Initialize the AIDE database:
```bash
sudo aide --init
```
Move the generated database to the correct location:
```bash
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

Set up a daily cron job to run AIDE integrity checks:
```bash
sudo crontab -e
```
Add the following line:
```bash
0 5 * * * /usr/sbin/aide --check
```

### Step 7: **Security Audits and Ongoing Monitoring**

#### 7.1 Use Lynis for Security Audits
Lynis is a security auditing tool that can be used to scan and report vulnerabilities.
```bash
sudo apt install lynis -y  # For Ubuntu/Debian
sudo yum install lynis -y  # For CentOS/RHEL
sudo lynis audit system
```

#### 7.2 Monitoring Tools
Install and configure monitoring tools like **Prometheus**, **Telegraf**, and **Grafana** to collect system metrics, log events, and generate security alerts.

### Step 8: **Ongoing Maintenance**

- Regularly **apply patches and updates** to the system.
- Periodically review **user accounts and permissions**.
- Conduct regular **security audits** using tools like `Lynis` or `OpenSCAP` to identify potential vulnerabilities.
- Set up **intrusion detection systems** like **Fail2Ban** for monitoring SSH login attempts and other services.

By following these steps, your Linux system will be hardened according to CIS benchmarks, greatly reducing its attack surface and improving overall security.