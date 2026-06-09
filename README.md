#  PromisedLand.com – Linux Server Deployment
### IT 210 | Linux Administration | Final Project
**AWS EC2 | Ubuntu Server 22.04 LTS | Samba | UFW | ACLs | Cron**

---

##  Project Overview

This project is a full Linux server deployment for **PromisedLand.com's** new secondary data center in Independence, Missouri. The facility houses **38 employees** across four departments and serves as the IT secondary data center supporting hybrid call centers across the United States and Canada.

**Organizational Structure:**
-  **Carrie Shumaker** — CIO/Director
-  **Linda McLachlan** — Director of Finance
-  **Richard Durant** — Operations Manager
-  **Brian LaGoe** — Business Applications Manager
-  **Joseph Lubomirski** — Infrastructure Manager
-  **Cole Motley** — CRM Solution Architect

---

##  Technologies Used

| Tool | Purpose |
|---|---|
| AWS EC2 | Cloud hosting platform |
| Ubuntu Server 22.04 LTS | Operating system |
| Samba | Cross-platform file sharing |
| UFW | Firewall management |
| ACLs (setfacl/getfacl) | Fine-grained file permissions |
| Cron | Scheduled maintenance tasks |
| ext4 | Filesystem for all partitions |

---

##  Step 1 — AWS EC2 Instance Launch & Access

Launched an Ubuntu Server 22.04 LTS EC2 instance on AWS with a 120 GB EBS volume attached as a secondary disk.

```bash
ssh -i "njabs.key.pem" ubuntu@18.236.212.245
```

<!-- SCREENSHOT: AWS Console showing EC2 instance in running state -->

<!-- SCREENSHOT: Terminal showing successful SSH connection + MOTD -->

---

##  Step 2 — System Update

Updated all repositories and packages to ensure the server is current and secure.

```bash
sudo apt update
sudo apt upgrade -y
```

<!-- SCREENSHOT: Terminal showing apt update && apt upgrade output -->
![System Update](screenshots/03_apt_update.png)

---

##  Step 3 — Disk Partitioning & Mounting

Partitioned the 120 GB EBS volume (`/dev/nvme1n1`) into 5 dedicated sections:

| Partition | Size | Mount Point | Purpose |
|---|---|---|---|
| nvme1n1p1 | 30 GB | /home | User home directories |
| nvme1n1p2 | 15 GB | /var | Logs & variable data |
| nvme1n1p3 | 50 GB | /srv | Shared department files |
| nvme1n1p4 | 5 GB | /tmp | Temporary files |
| swapfile | 4 GB | swap | Virtual memory |

```bash
sudo fdisk /dev/nvme1n1
sudo mkfs.ext4 /dev/nvme1n1p1
sudo mkfs.ext4 /dev/nvme1n1p2
sudo mkfs.ext4 /dev/nvme1n1p3
sudo mkfs.ext4 /dev/nvme1n1p4
sudo fallocate -l 4G /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

**Verification:**
```bash
lsblk
df -h
free -h
```

<!-- SCREENSHOT: lsblk output showing all partitions -->

<!-- SCREENSHOT: df -h output showing mounted partitions -->

<!-- SCREENSHOT: free -h output showing 4G swap active -->

---

##  Step 4 — Groups

Created department groups and sub-groups mapped directly to the organizational chart:

```bash
sudo groupadd operations
sudo groupadd operation_service.desk
sudo groupadd application
sudo groupadd application_DBA
sudo groupadd application_programming
sudo groupadd infrastructure
sudo groupadd infrastructure_systems
sudo groupadd infrastructure_network
sudo groupadd infrastructure_project.manager
sudo groupadd infrastructure_security
sudo groupadd CRM
sudo groupadd CRM_system.analyst
sudo groupadd CRM_project.coordinator
sudo groupadd CRM_intern
sudo groupadd finance
sudo groupadd executive
```

<!-- SCREENSHOT: cat /etc/group output showing all groups -->
![Groups](screenshots/07_groups.png)

---

## Step 5 — User Accounts

Created all 35 user accounts with `/bin/bash` shell, home directories, and forced password change on first login.

```bash
sudo useradd -m -s /bin/bash -G executive,finance Carrie_Shumaker
sudo useradd -m -s /bin/bash -G finance,executive Linda_McLachlan
sudo useradd -m -s /bin/bash -G infrastructure Joseph_Lubomirski
# ... (all 35 users)

# Force password change on first login
sudo chage -d 0 Carrie_Shumaker

# Lock part-time accounts
sudo usermod -L Drew_Dixon
sudo usermod -L Brian_Hoang

# Set temp account expiry
sudo chage -E 2025-12-31 Evan_Casement
```

<!-- SCREENSHOT: cat /etc/passwd output showing all users with /bin/bash -->
![User Accounts](screenshots/08_users.png)

---

## Step 6 — Sudo Access

Restricted sudo access to the Infrastructure Team only via `/etc/sudoers`:

```bash
sudo visudo
# Added: %it_admin   ALL=(ALL:ALL) ALL
```

<!-- SCREENSHOT: sudoers entry showing it_admin group -->
![Sudo Config](screenshots/09_sudo.png)

---

##  Step 7 — Shared Directories & Permissions

Created 10 departmental share directories under `/srv/shares/` with setgid permissions:

```bash
sudo mkdir -p /srv/shares/{executive,operations,service_desk,application,infrastructure,crm,finance,common}

sudo chown root:executive    /srv/shares/executive    && sudo chmod 2770 /srv/shares/executive
sudo chown root:operations   /srv/shares/operations   && sudo chmod 2770 /srv/shares/operations
sudo chown root:operation_service.desk /srv/shares/service_desk && sudo chmod 2770 /srv/shares/service_desk
sudo chown root:application  /srv/shares/application  && sudo chmod 2770 /srv/shares/application
sudo chown root:infrastructure /srv/shares/infrastructure && sudo chmod 2770 /srv/shares/infrastructure
sudo chown root:CRM          /srv/shares/crm          && sudo chmod 2770 /srv/shares/crm
sudo chown root:finance      /srv/shares/finance      && sudo chmod 2770 /srv/shares/finance
sudo chown root:operations   /srv/shares/common       && sudo chmod 2775 /srv/shares/common
```

**Permission Model:**
- `2770` — Group members only (no outside access); setgid active
- `2775` — All authenticated staff can read/write (common folder)
- `s` in group position = setgid bit (new files inherit group automatically)

<!-- SCREENSHOT: ls -la /srv/shares/ showing all directories and permissions -->
![Shared Directories](screenshots/10_shares.png)

---

##  Step 8 — ACLs (Access Control Lists)

Granted Carrie Shumaker (CIO/Director) read-only access to all department shares for organizational oversight, without full group membership:

```bash
sudo setfacl -m u:Carrie_Shumaker:rx /srv/shares/operations
sudo setfacl -m u:Carrie_Shumaker:rx /srv/shares/service_desk
sudo setfacl -m u:Carrie_Shumaker:rx /srv/shares/application
sudo setfacl -m u:Carrie_Shumaker:rx /srv/shares/infrastructure
sudo setfacl -m u:Carrie_Shumaker:rx /srv/shares/crm
sudo setfacl -m u:Carrie_Shumaker:rx /srv/shares/finance
sudo setfacl -m u:Sherie_Modelski:rx /srv/shares/infrastructure
```

```bash
sudo getfacl /srv/shares/finance
```

<!-- SCREENSHOT: getfacl /srv/shares/finance showing Carrie's r-x ACL entry -->
![ACL Finance](screenshots/11_acl_finance.png)

<!-- SCREENSHOT: getfacl /srv/shares/operations showing Carrie's r-x ACL entry -->
![ACL Operations](screenshots/12_acl_operations.png)

---

##  Step 9 — Samba File Server

Installed and configured Samba for cross-platform file sharing across all departments:

```bash
sudo apt install samba -y
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
sudo nano /etc/samba/smb.conf
```

**Sample share definition:**
```ini
[operations]
   path = /srv/shares/operations
   valid users = @operations
   read only = no
   browsable = yes
```

```bash
sudo systemctl enable smbd
sudo systemctl start smbd
```

<!-- SCREENSHOT: systemctl status smbd showing active (running) -->
![Samba Status](screenshots/13_samba_status.png)

---

##  Step 10 — Firewall (UFW)

Configured UFW with default-deny inbound policy, allowing only SSH and Samba:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow samba
sudo ufw enable
sudo ufw status verbose
```

<!-- SCREENSHOT: ufw status verbose showing all rules -->
![UFW Firewall](screenshots/14_ufw_status.png)

---

##  Step 11 — SSH Hardening

Disabled root login and password authentication in `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
sudo grep -E "PermitRootLogin|PasswordAuthentication" /etc/ssh/sshd_config
```

<!-- SCREENSHOT: grep output showing PermitRootLogin no and PasswordAuthentication no -->
![SSH Hardening](screenshots/15_ssh_hardening.png)

---

##  Step 12 — Message of the Day (MOTD)

Configured a custom MOTD displayed at every login:

```
*******************************************************************
*   PROMISEDLAND.COM – Independence, MO Data Center               *
*   AUTHORIZED USERS ONLY. All activity is logged and monitored.  *
*   Unauthorized access is a violation of company policy and      *
*   may be subject to disciplinary or legal action.               *
*                                                                  *
*   IT Support: itsupport@promisedland.com  |  Ext. 4200          *
*   Infrastructure Manager: Joseph Lubomirski                     *
*******************************************************************
```

<!-- SCREENSHOT: cat /etc/motd showing the full MOTD -->
![MOTD](screenshots/16_motd.png)

---

##  Step 13 — Scheduled Reboot (Cron)

Configured a cron job to reboot the server every two years at 2:00 AM on January 1st:

```bash
sudo nano /etc/cron.d/scheduled_reboot
```

```
0 2 1 1 */2  root  /sbin/reboot
```

```bash
cat /etc/cron.d/scheduled_reboot
sudo systemctl status cron
```

<!-- SCREENSHOT: cat /etc/cron.d/scheduled_reboot showing the entry -->
![Cron Job](screenshots/17_cron.png)

---

##  Step 14 — CLI Package Installation (tree)

Installed `tree` — a non-default command-line utility for visualizing directory structures:

```bash
sudo apt install tree -y
tree /srv/shares/
```

<!-- SCREENSHOT: tree /srv/shares/ showing full directory structure -->
![Tree Output](screenshots/18_tree.png)

---

##  Final Verification

```bash
lsblk && df -h && free -h
cat /etc/group | grep -E "operations|application|CRM|infrastructure|finance|executive"
ls -la /srv/shares/
sudo getfacl /srv/shares/finance
sudo systemctl status smbd
sudo ufw status verbose
sudo grep -E "PermitRootLogin|PasswordAuthentication" /etc/ssh/sshd_config
cat /etc/motd
cat /etc/cron.d/scheduled_reboot
tree /srv/shares/
```

<!-- SCREENSHOT: Full verification output -->
![Final Verification](screenshots/19_verification.png)

---

##  Repository Structure

```
IT210-Linux-Deployment/
├── README.md               ← This file
├── screenshots/            ← All verification screenshots
│   ├── 01_ec2_instance.png
│   ├── 02_ssh_connection.png
│   ├── 03_apt_update.png
│   ├── 04_lsblk.png
│   ├── 05_df_h.png
│   ├── 06_free_h.png
│   ├── 07_groups.png
│   ├── 08_users.png
│   ├── 09_sudo.png
│   ├── 10_shares.png
│   ├── 11_acl_finance.png
│   ├── 12_acl_operations.png
│   ├── 13_samba_status.png
│   ├── 14_ufw_status.png
│   ├── 15_ssh_hardening.png
│   ├── 16_motd.png
│   ├── 17_cron.png
│   ├── 18_tree.png
│   └── 19_verification.png
├── PromisedLand_Deployment_Proposal.docx  ← Written proposal
└── Video_Script.md         ← Video demo script
```

---

##  Course Outcomes Met

| Outcome | Description | Demonstrated By |
|---|---|---|
| CO1 | Linux fundamentals applied to real-world problems | Users, groups, permissions, ACLs, firewall, SSH, cron, Samba |
| CO2 | Designed, installed, and set up a virtualized Linux system | AWS EC2 + partitioned EBS + Ubuntu 22.04 LTS |
| CO3 | Managed, maintained, and troubleshot a Linux system | Automated updates, scheduled reboots, security hardening, MOTD |

---

*IT 210 – Linux Administration | BYU Pathway Worldwide | Njabulo Thusi*
