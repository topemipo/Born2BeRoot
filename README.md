*This project has been created as part of the 42 curriculum by tadeyelu*

## Table of Contents
1. [Project Overview](#project-overview)
2. [Virtual Machine Setup](#virtual-machine-setup)
3. [Disk Partitions & LVM](#disk-partitions--lvm)
4. [AppArmor](#apparmor)
5. [SSH Configuration](#ssh-configuration)
6. [UFW Firewall](#ufw-firewall)
7. [User Management](#user-management)
8. [Password Policy](#password-policy)
9. [Sudo Configuration](#sudo-configuration)
10. [Monitoring Script](#monitoring-script)
11. [Crontab](#crontab)
12. [Extras](#extras)
---

## Project Overview

Born2beRoot is a 42 system administration project where you set up a secure Linux server from scratch inside a virtual machine. The goal is to learn how servers work — partitioning, encryption, firewalls, user management, security policies, and automation.

**Key requirements:**
- Debian GNU/Linux (no graphical interface)
- Encrypted LVM partitions
- SSH on port 4242 (no root login)
- UFW firewall (only port 4242 open)
- Strong password policy
- Strict sudo configuration
- Automated monitoring script running every 10 minutes

---

## Virtual Machine Setup

**Hypervisor:** VirtualBox (Type 2 hypervisor — runs on top of host OS)

**VirtualBox vs UTM**
- VirtualBox = Type 2 hypervisor for x86_64 machines
- UTM = uses QEMU emulation, works on Apple M1/M2 ARM machines
- ARM Macs cannot natively virtualise x86 Debian — they need emulation

**Port Forwarding (NAT):**
- Host Port: `4243`
- Guest Port: `4242`
- This is needed because the VM uses NAT networking and the host couldn't use port 4242 directly

**To SSH into VM from host machine:**
```bash
ssh tadeyelu@localhost -p 4243
```

**Debian vs Rocky**
| | Debian | Rocky |
|---|---|---|
| Type | Community driven | Enterprise grade |
| Package manager | apt / aptitude | dnf |
| Security module | AppArmor | SELinux |
| Difficulty | Beginner friendly | Complex |
| Cost | Free | Free |
| Based on | Independent | Red Hat |

---

## Disk Partitions & LVM

### Partition Layout
```
sda (12G — whole virtual disk)
├─ sda1   791M   part   /boot        (primary partition — bootloader)
├─ sda2   1K     part                (extended partition — container)
└─ sda5   11.2G  part                (logical partition inside sda2)
   └─ sda5_crypt  11.2G  crypt       (decrypted layer after password entry)
      ├─ tadeyelu42--vg-root   7.1G  lvm   /
      ├─ tadeyelu42--vg-swap_1 620M  lvm   [SWAP]
      └─ tadeyelu42--vg-home   3.4G  lvm   /home
```

### Key Concepts

**MBR (Master Boot Record):** Old partitioning scheme with a maximum of 4 partitions. To get more, one slot is used as an extended partition (container) which can hold unlimited logical partitions inside it.

- `sda1` = primary partition (boot)
- `sda2` = extended partition (the container, 1K metadata only)
- `sda5` = logical partition living INSIDE sda2 (proof: same Start/End sectors in `fdisk -l`)

**Why sda2 looks like a sibling of sda5 in lsblk:** `lsblk` doesn't visualise the extended/logical relationship clearly. `fdisk -l` shows the truth — sda5's sectors fall entirely within sda2's boundaries.

**Encryption:** `sda5` is encrypted. At boot you enter a passphrase which decrypts it and presents it as `sda5_crypt`. The boot partition (`sda1`) is NOT encrypted because the system needs to read it before asking for the passphrase.

**LVM (Logical Volume Manager):** Creates flexible virtual partitions on top of the encrypted layer. Unlike fixed partitions, LVM volumes can be resized, merged, or snapshotted without reformatting.

**The full chain:**
```
Physical disk → Encrypted partition → Decrypted layer → LVM volumes
```

### Useful Commands
```bash
lsblk                    # view partition structure
sudo fdisk -l            # view detailed partition table with sectors
```

---

## AppArmor

**What it is:** AppArmor (Application Armor) is a Linux security system implementing MAC (Mandatory Access Control). It confines applications to only access what their profile explicitly allows.

**DAC vs MAC:**
- DAC (Discretionary Access Control) = traditional Linux. File owner decides who can access files via `chmod`. If a program is compromised it has all the permissions of the user running it.
- MAC (Mandatory Access Control) = AppArmor. The OS itself enforces rules regardless of what the user wants. Even a compromised program cannot escape its profile.

**AppArmor Profiles:** Text files in `/etc/apparmor.d/` defining what each application can and cannot do (read, write, network access etc.). Two modes:
- Enforce mode = rules actively enforced, violations blocked
- Complain mode = violations logged but not blocked

**Why `active (exited)` is normal:** AppArmor loads its profiles into the kernel then exits. The kernel handles enforcement directly — AppArmor doesn't need to keep running as a process.

**AppArmor vs SELinux**
- Both implement MAC (Mandatory Access Control)
- AppArmor uses file paths for profiles (simpler)
- SELinux uses labels on every file and process (more powerful but much more complex)

### Useful Commands
```bash
sudo systemctl status apparmor    # check AppArmor service status
sudo aa-status                    # check loaded profiles
```

---

## SSH Configuration

**What SSH is:** Secure Shell — lets you remotely control another computer via terminal with everything encrypted. Without SSH you'd have to physically sit at the server to manage it.

**Configuration file:** `/etc/ssh/sshd_config`

**Key settings:**
```
Port 4242
PermitRootLogin no
```

**Why no root SSH:** If root could SSH in, a brute force attack on the password would give full system control. By disabling it, attackers must first know a valid username AND password for a regular user, then escalate separately.

### Useful Commands
```bash
sudo systemctl status ssh                  # check SSH service
grep Port /etc/ssh/sshd_config             # verify port 4242
grep PermitRootLogin /etc/ssh/sshd_config  # verify root login disabled
sudo sshd -T | grep permitrootlogin        # check active running config

# SSH in as regular user (from host machine)
ssh tadeyelu@localhost -p 4243

# SSH in as root (should be denied)
ssh root@localhost -p 4243
```

---

## UFW Firewall

**What UFW is:** Uncomplicated Firewall — controls which network ports accept incoming connections. Your server has 65,535 ports. UFW locks all of them except the ones you explicitly allow.

**Why control ports:** Every open port is a potential entry point for attackers. By only allowing port 4242, you reduce the attack surface to just SSH.

**Current rules:** Only port 4242 is open (IPv4 and IPv6).

**UFW vs firewalld**
- UFW = Debian/Ubuntu default, simpler syntax
- firewalld = Rocky/Red Hat default, more complex, zone-based

### Useful Commands
```bash
sudo ufw status                  # check UFW status and rules
sudo ufw status numbered         # show rules with numbers
sudo ufw allow 8080              # add a new rule (for evaluation demo)
sudo ufw delete allow 8080       # delete rule by name
sudo ufw delete 3                # delete rule by number
sudo systemctl status ufw        # check UFW service
```

---

## User Management

**Users on the system:**
- `root` — superuser, full system access
- `tadeyelu` — regular user, member of `sudo` and `user42` groups

**Groups:**
- `sudo` — allows user to run commands with elevated privileges using `sudo`
- `user42` — project-specific group required by the subject

### Useful Commands
```bash
groups tadeyelu                        # check user's groups
cat /etc/passwd | grep /bin/bash       # list real human users

# Create new user
sudo adduser username

# Create new group
sudo addgroup groupname

# Add user to group
sudo adduser username groupname

# Delete user
sudo deluser username
sudo deluser --remove-home username    # also deletes home directory

# Change hostname
sudo hostnamectl set-hostname newhostname
sudo reboot                            # hostname updates after reboot
hostname                               # verify hostname
```

---

## Password Policy

### Configuration Files
- `/etc/login.defs` — time-based rules (expiry, minimum days, warning)
- `/etc/pam.d/common-password` — complexity rules

### Rules Implemented

**Time-based (in `/etc/login.defs`):**
```
PASS_MAX_DAYS   30     # password expires every 30 days
PASS_MIN_DAYS   2      # minimum 2 days between password changes
PASS_WARN_AGE   7      # warning 7 days before expiry
```

**Complexity (in `/etc/pam.d/common-password`):**
```
password requisite pam_pwquality.so retry=3 minlen=10 ucredit=-1 dcredit=-1 lcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

Breaking down the complexity line:
- `retry=3` — 3 attempts to enter valid password
- `minlen=10` — minimum 10 characters
- `ucredit=-1` — at least 1 uppercase letter
- `dcredit=-1` — at least 1 digit/number
- `lcredit=-1` — at least 1 lowercase letter
- `maxrepeat=3` — no more than 3 consecutive identical characters
- `reject_username` — password cannot contain the username
- `difok=7` — at least 7 characters different from previous password
- `enforce_for_root` — root must also comply

**Note:** These defaults apply to newly created users. Existing users must be updated manually with `chage`.

### Useful Commands
```bash
sudo chage -l tadeyelu          # view password expiry info for user
sudo chage -l root              # view password expiry info for root
sudo chage -M 30 tadeyelu       # set max days to 30
sudo chage -m 2 tadeyelu        # set min days to 2
sudo chage -M 30 root           # set max days for root
sudo chage -m 2 root            # set min days for root

# Verify login.defs settings
grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs

# Verify PAM complexity settings
grep pam_pwquality /etc/pam.d/common-password
```

### Advantages & Disadvantages of Password Policy

**Advantages:**
- Complexity rules make passwords harder to brute force or guess
- Regular expiry (30 days) limits damage if a password is stolen
- Minimum days between changes prevents users from reverting immediately
- `difok=7` ensures each new password is genuinely different
- `reject_username` prevents the most obvious passwords

**Disadvantages:**
- Users find frequent changes annoying and may write passwords down (less secure)
- Complexity rules lead to predictable patterns like `Password1!`
- Frequent expiry causes productivity loss and more IT support requests
- Modern security standards (NIST) actually recommend against forced expiry unless compromised

---

## Sudo Configuration

**What sudo is:** Superuser Do — lets authorised regular users temporarily borrow root's privileges for a single command. Safer than always working as root because one mistake as root can break everything.

**Configuration file:** `/etc/sudoers.d/sudo_config`

### Rules Implemented
```
Defaults passwd_tries=3
Defaults badpass_message="Wrong Password, mate! Try again :)"
Defaults logfile="/var/log/sudo/sudo_config"
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo"
Defaults requiretty
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

**Breaking down each rule:**
- `passwd_tries=3` — max 3 password attempts before lockout
- `badpass_message` — custom error message on wrong password
- `logfile` — text log of all sudo commands (who, when, what)
- `log_input, log_output` — records keystrokes typed AND screen output of every sudo session
- `iolog_dir` — directory where input/output recordings are stored
- `requiretty` — sudo can only be run from a real terminal, not scripts or background processes (prevents certain attack vectors)
- `secure_path` — sudo only looks for commands in these trusted directories, preventing path hijacking attacks

### Useful Commands
```bash
sudo cat /etc/sudoers.d/sudo_config      # view sudo rules
sudo ls /var/log/sudo/                   # list sudo log directory
sudo cat /var/log/sudo/sudo_config       # read text log
sudo journalctl _COMM=sudo | grep COMMAND | wc -l  # count sudo commands (run as root for accurate count)
```

---

## Monitoring Script

**Location:** `/usr/local/bin/monitoring.sh`

**Purpose:** Broadcasts system information to all terminals every 10 minutes using `wall`.

### Full Script Explained

```bash
#!/bin/bash

# ARCHITECTURE
# uname -a prints all system info: kernel name, hostname, version, architecture
arch=$(uname -a)

# PHYSICAL CPU
# /proc/cpuinfo is a virtual file the kernel updates with CPU info
# grep "physical id" finds lines for each physical core
# wc -l counts those lines
cpuf=$(grep "physical id" /proc/cpuinfo | wc -l)

# VIRTUAL CPU (logical cores including hyperthreaded)
# grep "processor" finds each logical core entry (numbered 0, 1, 2...)
cpuv=$(grep "processor" /proc/cpuinfo | wc -l)

# RAM
# free --mega shows RAM usage in megabytes
# awk '$1 == "Mem:"' filters only the Mem: line
# $2 = total, $3 = used
ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
ram_use=$(free --mega | awk '$1 == "Mem:" {print $3}')
ram_percent=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')

# DISK
# df -m shows disk usage in MB
# grep "/dev/" keeps only real disk partitions (removes tmpfs, udev)
# grep -v "/boot" removes boot partition from calculation
# awk adds up total column ($2) and used column ($3) across all partitions
# disk_total converts MB to GB (divide by 1024)
disk_total=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} END {printf("%.1fGb\n"), disk_t/1024}')
disk_use=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} END {print disk_u}')
disk_percent=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} {disk_t += $2} END {printf("%d"), disk_u/disk_t*100}')

# CPU LOAD
# vmstat 1 2 samples system stats twice, 1 second apart
# tail -1 takes the last (most stable) sample
# $15 is the idle CPU percentage column
# subtract from 100 to get usage percentage
cpul=$(vmstat 1 2 | tail -1 | awk '{printf $15}')
cpu_op=$(expr 100 - $cpul)
cpu_fin=$(printf "%.1f" $cpu_op)

# LAST BOOT
# who -b shows last system boot time
# awk filters the "system" line and prints date ($3) and time ($4)
lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')

# LVM USE
# lsblk lists all block devices
# grep "lvm" finds LVM volumes
# wc -l counts them
# if count > 0 print yes, else print no
lvmu=$(if [ $(lsblk | grep "lvm" | wc -l) -gt 0 ]; then echo yes; else echo no; fi)

# TCP CONNECTIONS
# ss = socket statistics (modern replacement for netstat)
# -t = TCP only, -a = all states
# grep ESTAB = only established (active) connections
# wc -l counts them
tcpc=$(ss -ta | grep ESTAB | wc -l)

# USER LOG
# users shows usernames of all logged in sessions
# wc -w counts words (each username = one session)
ulog=$(users | wc -w)

# NETWORK
# hostname -I prints all IPs, awk takes only first (IPv4)
# ip link shows network interfaces
# grep "link/ether" finds MAC address line
# awk prints second column (the MAC address)
ip=$(hostname -I | awk '{print $1}')
mac=$(ip link | grep "link/ether" | awk '{print $2}')

# SUDO COUNT
# journalctl _COMM=sudo shows all sudo logs
# grep COMMAND filters only actual command executions (not auth attempts)
# wc -l counts total sudo commands ever run
cmnd=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

# BROADCAST
# wall sends message to all logged in terminals
wall "	#Architecture: $arch
	#Physical CPU: $cpuf
	#vCPU: $cpuv
	#Memory Usage: $ram_use/${ram_total}MB ($ram_percent%)
	#Disk Usage: $disk_use/${disk_total} ($disk_percent%)
	#CPU load: $cpu_fin%
	#Last boot: $lb
	#LVM use: $lvmu
	#TCP Connections: $tcpc ESTABLISHED
	#User log: $ulog
	#Network: IP $ip ($mac)
	#Sudo: $cmnd cmd"
```

### Key Concepts

**wall:** Stands for "write all" — broadcasts a message to every open terminal on the system simultaneously.

**Physical CPU vs Virtual CPU:**
- Physical cores = actual silicon processing units inside the CPU chip
- Virtual/logical cores = what the OS sees, created by hyperthreading (each physical core splits into 2 logical cores)
- Hierarchy: Motherboard → Physical CPU chip(s) → Physical cores → Logical cores

**Why idle percentage instead of usage directly:**
`vmstat`'s column 15 (`id`) is the idle percentage. We subtract from 100 because it's more reliable than reading usage directly.

---

## Crontab

**What cron is:** A time-based job scheduler in Linux. It runs commands automatically at specified times without user intervention.

**Cron syntax:**
```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

**`*/n` = every n units** (e.g. `*/10` = every 10 minutes)
**Just `n` = at exactly that value** (e.g. `0 8 * * *` = at 8:00am every day)

**Current crontab entry:**
```
*/10 * * * * sh /usr/local/bin/monitoring.sh
```
Runs monitoring.sh every 10 minutes, every hour, every day.

**Why `(somewhere)` instead of `(tty1)`:** When cron runs the script it has no terminal attached, so wall doesn't know which tty to display.

### Useful Commands
```bash
sudo crontab -l              # list cron jobs
sudo crontab -e              # edit cron jobs
sudo systemctl status cron   # check cron service
```

### Stopping the script without modifying it (for evaluation)
```bash
sudo crontab -e
# Add # at the start of the monitoring line:
# */10 * * * * sh /usr/local/bin/monitoring.sh
```
The script itself is untouched — only the crontab is modified.

**To restart it:** Remove the `#` from the crontab entry.

## Extras

**To make a copy of your VM's disk**
```
VBoxManage clonemedium /sgoinfre/students/tadeyelu/VirtualBox_VMs/temis_first_vm/temis_first_vm.vdi /sgoinfre/students/tadeyelu/VirtualBox_VMs/temis_first_vm/temis_first_vm_copy2.vdi
```

**To get your VM's signature**
```
sha1sum temis_first_vm.vdi
```

**To switch VirtualBox to use the copy**
- Step 1 — Open VirtualBox and make sure your VM is shut down
- Step 2 — Click on your VM then go to Settings
- Step 3 — Go to Storage
- Step 4 — Under Controller: SATA you'll see your current .vdi file attached. Click on it.
- Step 5 — On the right side click the small disk icon to change the disk
- Step 6 — Click "Choose a disk file" and navigate to:
```
/sgoinfre/students/tadeyelu/VirtualBox_VMs/temis_first_vm/temis_first_vm_copy.vdi
```
- Step 7 — Click OK to save
