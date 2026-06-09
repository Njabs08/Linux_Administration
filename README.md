#  PromisedLand.com – Linux Server Deployment
### IT 210 | Linux Administration | Final Project
**AWS EC2 | Ubuntu Server 22.04 LTS | Samba | UFW | ACLs | Cron**

---

##  Project Overview

This project is a full Linux server deployment for **PromisedLand.com's** new secondary data center in Independence, Missouri. The facility houses **38 employees** across four departments and serves as the IT secondary data center supporting hybrid call centers across the United States and Canada.
<img width="1064" height="668" alt="image" src="https://github.com/user-attachments/assets/323e4ce7-e573-4dc9-8df1-e17b46f7c821" />

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
---

##  Step 2 — System Update

Updated all repositories and packages to ensure the server is current and secure.

```bash
sudo apt update
sudo apt upgrade -y
```
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
---

## Step 6 — Sudo Access

Restricted sudo access to the Infrastructure Team only via `/etc/sudoers`:

```bash
sudo visudo
# Added: %it_admin   ALL=(ALL:ALL) ALL
```

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
---

##  Step 14 — CLI Package Installation (tree)

Installed `tree` — a non-default command-line utility for visualizing directory structures:

```bash
sudo apt install tree -y
tree /srv/shares/
```

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
---

##  Course Outcomes Met

| Outcome | Description | Demonstrated By |
|---|---|---|
| CO1 | Linux fundamentals applied to real-world problems | Users, groups, permissions, ACLs, firewall, SSH, cron, Samba |
| CO2 | Designed, installed, and set up a virtualized Linux system | AWS EC2 + partitioned EBS + Ubuntu 22.04 LTS |
| CO3 | Managed, maintained, and troubleshot a Linux system | Automated updates, scheduled reboots, security hardening, MOTD |

---

*IT 210 – Linux Administration | BYU Pathway Worldwide | Njabulo P Tshuma*
