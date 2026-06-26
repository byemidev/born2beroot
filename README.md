# Born2BeRoot — Activity Summary

> **42 School Project** | System Administration & Security  
> **OS:** Debian 12 (stable) | **Hypervisor:** VirtualBox | **Date:** June 2026

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Virtual Machine Setup](#virtual-machine-setup)
3. [Operating System Installation](#operating-system-installation)
4. [System Configuration](#system-configuration)
5. [Security Hardening](#security-hardening)
6. [Monitoring Script](#monitoring-script)
7. [Bonus: Web Stack](#bonus-web-stack)
8. [Evaluation Preparation](#evaluation-preparation)
9. [Key Concepts & Definitions](#key-concepts--definitions)
10. [Evaluation Commands Cheatsheet](#evaluation-commands-cheatsheet)

---

## Project Overview

**Born2BeRoot** is a system administration project from the 42 School curriculum. It involves configuring a secure virtual machine from scratch using **Debian** or **Rocky Linux**, with emphasis on:

- Virtualization fundamentals
- OS installation and partitioning (LVM)
- SSH hardening and remote access
- Firewall configuration (UFW)
- User/group management and sudo policies
- Password policies with `libpam-pwquality`
- System monitoring via cron jobs
- Bonus: WordPress + MariaDB + Lighttpd stack

> **Key Constraint:** No graphical interface. All configuration is done via command line.

---

## Virtual Machine Setup

### Hypervisor Configuration (VirtualBox)

| Setting | Value |
|---------|-------|
| **Type** | Type 2 (Hosted) Hypervisor |
| **RAM** | 4–8 GiB |
| **vCPUs** | 2–8 cores |
| **Disk** | 10–30 GiB (pre-allocated) |
| **Network** | NAT with Port Forwarding |
| **Storage** | VDI (VirtualBox Disk Image) |

### Port Forwarding Rules

| Name | Protocol | Host Port | Guest Port | Purpose |
|------|----------|-----------|------------|---------|
| SSH | TCP | `2222` (or available) | `4242` | Remote SSH access |
| HTTP | TCP | `8080` (or available) | `80` | Web server (bonus) |

---

## Operating System Installation

### Debian Installation Steps

1. Download **Debian 12 (bookworm) netinst ISO**
2. Boot VM → Select **Install** (not graphical install)
3. Configure:
   - **Hostname:** `<login>42`
   - **Domain:** (leave empty)
   - **Root password:** strong password
   - **User account:** `<login>` with strong password

### Partitioning (Bonus: Encrypted LVM)

```
Guided → Use entire disk and set up encrypted LVM
         → Separate /home, /var, and /tmp partitions
```

**Bonus Partitions:**

| Mount Point | Size | Filesystem | Notes |
|-------------|------|------------|-------|
| `/` | ~5 GiB | ext4 | Root system |
| `/home` | ~5 GiB | ext4 | User home directories |
| `/var` | ~3 GiB | ext4 | Variable data |
| `/tmp` | ~2 GiB | ext4 | Temporary files |
| `/srv` | ~2 GiB | ext4 | **Bonus** — Service data |
| `/var/log` | ~2 GiB | ext4 | **Bonus** — Log files |
| `swap` | ~1 GiB | swap | Virtual memory |

> **LVM (Logical Volume Manager)** allows dynamic resizing and management of disk partitions without reformatting.

### Software Selection

Only select:
- ✅ **SSH server**
- ✅ **standard system utilities**

Everything else is installed manually later.

---

## System Configuration

### 1. System Update

```bash
su -
apt update && apt upgrade
```

### 2. Hostname Verification

```bash
hostnamectl                          # Check current hostname
hostnamectl set-hostname <login>42   # Set if needed
```

### 3. Sudo Installation & Configuration

```bash
apt install sudo
```

Edit sudoers file with `visudo`:

```
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    requiretty
Defaults    badpass_message="WRONG PASSWORD"
Defaults    logfile="/var/log/sudo/sudo.log"
Defaults    log_input
Defaults    log_output
Defaults    iolog_dir=/var/log/sudo
Defaults    passwd_tries=3
```

**Add user to groups:**

```bash
groupadd user42
usermod -aG user42,sudo <username>
```

---

## Security Hardening

### SSH Configuration (`/etc/ssh/sshd_config`)

```bash
Port 4242                          # Change from default 22
PermitRootLogin no                 # Disable root login
PasswordAuthentication yes         # (or no if using keys)
```

```bash
systemctl restart ssh
systemctl status ssh
```

### UFW Firewall

```bash
apt install ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow 4242                     # SSH port
ufw allow http                     # Bonus: port 80
ufw enable
ufw status numbered
```

### Password Policy

**File: `/etc/login.defs`**

```
PASS_MAX_DAYS   30
PASS_MIN_DAYS   2
PASS_WARN_AGE   7
```

**File: `/etc/pam.d/common-password`** (install `libpam-pwquality` first)

```bash
apt install libpam-pwquality
```

```
password    requisite    pam_pwquality.so     minlen=10 lcredit=-1 ucredit=-1 dcredit=-1     maxrepeat=3 usercheck=1 difok=7 enforce_for_root
```

| Parameter | Meaning |
|-----------|---------|
| `minlen=10` | Minimum 10 characters |
| `lcredit=-1` | At least 1 lowercase |
| `ucredit=-1` | At least 1 uppercase |
| `dcredit=-1` | At least 1 digit |
| `maxrepeat=3` | Max 3 consecutive identical chars |
| `usercheck=1` | Cannot contain username |
| `difok=7` | 7 chars different from previous |
| `enforce_for_root` | Applies to root too |

---

## Monitoring Script

### `monitoring.sh` (`/usr/local/bin/monitoring.sh`)

The script displays system information every 10 minutes via `wall` (broadcast to all logged-in users).

**Typical information displayed:**

| Metric | Command |
|--------|---------|
| Architecture | `uname -a` |
| CPU Physical Cores | `nproc --all` |
| CPU Virtual Cores | `nproc` |
| RAM Usage | `free -m` |
| Disk Usage | `df -h` |
| CPU Load | `top -bn1` |
| Last Boot | `who -b` |
| LVM Status | `lsblk` |
| TCP Connections | `ss -t` |
| Logged Users | `users` |
| Network (IP/MAC) | `hostname -I`, `ip link` |
| Sudo Commands | `journalctl _COMM=sudo` |

### Cron Job Setup

```bash
sudo crontab -u root -e
```

```cron
*/10 * * * * /usr/local/bin/monitoring.sh
```

**To stop/start without modifying the script:**

```bash
sudo /etc/init.d/cron stop
sudo /etc/init.d/cron start
```

---

## Bonus: Web Stack

### Services Installed

| Service | Purpose | Package |
|---------|---------|---------|
| **Lighttpd** | Web server | `lighttpd` |
| **MariaDB** | Database | `mariadb-server` |
| **PHP** | Server-side scripting | `php`, `php-cgi`, `php-mysql`, etc. |
| **WordPress** | CMS | Downloaded from wordpress.org |

### Setup Commands

```bash
# Install packages
apt install lighttpd mariadb-server
apt install php php-pdo php-mysql php-zip php-gd php-mbstring     php-curl php-xml php-pear php-bcmath php-opcache php-json php-cgi

# Enable PHP in Lighttpd
lighttpd-enable-mod fastcgi fastcgi-php
systemctl restart lighttpd

# Configure MariaDB
mariadb
```

```sql
CREATE DATABASE <login>_db;
CREATE USER <login>@localhost;
GRANT ALL PRIVILEGES ON <login>_db.* TO <login>@localhost;
FLUSH PRIVILEGES;
EXIT;
```

```bash
# Download and configure WordPress
cd /var/www/html
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
# Configure wp-config.php with DB credentials
```

---

## Evaluation Preparation

### Signature File

1. **Power off** the VM (do NOT save state)
2. On host terminal:

```bash
cd /sgoinfre/students/<intra_username>/<VM_name>
shasum <VM_name>.vdi
```

3. Copy the SHA256 hash into `signature.txt` at the root of your Git repository
4. **Clone the VM** in VirtualBox for evaluation
5. Verify the clone's signature matches before pushing

### Pre-Evaluation Checklist

- [ ] `signature.txt` exists and matches `.vdi` hash
- [ ] VM boots without GUI
- [ ] Login requires password (not root)
- [ ] `sudo ufw status` → `active`
- [ ] `sudo systemctl status ssh` → active on port 4242
- [ ] `getent group sudo` → contains your user
- [ ] `getent group user42` → contains your user
- [ ] `hostnamectl` → `<login>42`
- [ ] `lsblk` → shows LVM partitions
- [ ] `sudo chage -l <user>` → password policy enforced
- [ ] `sudo visudo` → sudoers config visible
- [ ] `/var/log/sudo/sudo.log` exists and logs commands
- [ ] `monitoring.sh` runs every 10 minutes (check `sudo crontab -u root -l`)
- [ ] SSH connection works: `ssh <user>@127.0.0.1 -p 4242`
- [ ] Root SSH denied: `ssh root@127.0.0.1 -p 4242` → "Permission denied"
- [ ] Bonus: WordPress accessible via browser

---

## Key Concepts & Definitions

### Virtual Machine (VM)
A software-based emulation of a physical computer. It runs an isolated OS on top of a host system, allowing safe testing and experimentation without affecting the main machine.

### LVM (Logical Volume Manager)
A storage management system that allows dynamic resizing, creation, and deletion of disk partitions (logical volumes) without reformatting the entire disk.

### SSH (Secure Shell)
A network protocol that provides encrypted communication between client and server. Replaces insecure protocols like Telnet by encrypting all traffic.

### UFW (Uncomplicated Firewall)
A user-friendly interface for managing `iptables` firewall rules. Simplifies allowing/blocking ports and traffic directions.

### Sudo (Superuser Do)
A command that allows authorized users to execute commands with elevated (root) privileges. Configured via the `sudoers` file.

### AppArmor
A Linux security module providing Mandatory Access Control (MAC). Restricts programs to a limited set of resources. Included by default in Debian. Check with `aa-status`.

### Cron
A time-based job scheduler. `crontab` is the configuration file where scheduled tasks are defined using a specific syntax:

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, Sun=0 or 7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

### Aptitude vs APT
| Feature | APT | Aptitude |
|---------|-----|----------|
| Level | Lower-level | Higher-level |
| Dependencies | Basic resolution | Smarter conflict resolution |
| Interface | Command-line only | Interactive TUI available |
| Auto-remove | No | Yes (suggests cleanup) |

### Debian vs Rocky Linux
| Aspect | Debian | Rocky Linux |
|--------|--------|-------------|
| Development | Community-driven | Rocky Enterprise Software Foundation |
| Focus | Universal OS (desktop + server) | Enterprise-grade server |
| Community | Larger, older | CentOS successor |
| Use Case | Personal servers, beginners | Large-scale business deployments |

---

## Evaluation Commands Cheatsheet

### General System
```bash
lsb_release -a              # OS info
cat /etc/os-release         # OS info (alternative)
hostnamectl                 # Hostname info
lsblk                       # Partition layout
```

### Users & Groups
```bash
getent group sudo           # Check sudo members
getent group user42         # Check user42 members
groups <username>           # List user's groups
cut -d: -f1 /etc/passwd   # List all users
sudo adduser <new_user>   # Create user
sudo groupadd <group>       # Create group
sudo usermod -aG <group> <user>  # Add user to group
sudo chage -l <user>        # Check password expiry rules
```

### SSH
```bash
sudo systemctl status ssh   # SSH service status
sudo service ssh status     # Alternative
ssh <user>@127.0.0.1 -p 4242   # Connect via SSH
```

### UFW
```bash
sudo ufw status             # Firewall status
sudo ufw status numbered    # Rules with numbers
sudo ufw allow <port>       # Open port
sudo ufw delete <number>    # Delete rule by number
```

### Sudo
```bash
sudo visudo                 # Edit sudoers file
cd /var/log/sudo && ls      # Check sudo logs
cat sudo.log                # View sudo command history
```

### Monitoring
```bash
cd /usr/local/bin && cat monitoring.sh   # View script
sudo crontab -u root -l                  # List cron jobs
sudo crontab -u root -e                  # Edit cron jobs
```

### Bonus
```bash
sudo systemctl status lighttpd   # Web server status
sudo systemctl status mariadb    # Database status
```

---

## 📁 Project Files Structure

```
born2beroot/
├── signature.txt              # SHA256 hash of .vdi file
├── monitoring.sh              # System monitoring script
├── README.md                  # This file
└── (bonus files if applicable)
```

---

## Credits
[VirtualBox User Guide](https://www.virtualbox.org/manual/ch01.html#create-vm-wizard)
[Debian tutorials](https://www.debian.org/doc/manuals/debian-reference/ch01.en.html)
[B2beroot activity gitbook](https://noreply.gitbook.io/born2beroot)

> **Author:** emi arevalo 
> **42 Login:** garevalo  
> **Repository:** https://github.com/byemidev/born2beroot
