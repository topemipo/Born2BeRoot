## Key Concepts for Evaluation

### Debian vs Rocky
| | Debian | Rocky |
|---|---|---|
| Type | Community driven | Enterprise grade |
| Package manager | apt / aptitude | dnf |
| Security module | AppArmor | SELinux |
| Difficulty | Beginner friendly | Complex |
| Cost | Free | Free |
| Based on | Independent | Red Hat |

### apt vs aptitude
- `apt` = simple, fast, does exactly what you tell it
- `aptitude` = smarter, tries harder to resolve dependency conflicts, has a visual menu interface
- Both install/remove packages but `aptitude` is better at solving complex package conflicts

### AppArmor vs SELinux
- Both implement MAC (Mandatory Access Control)
- AppArmor uses file paths for profiles (simpler)
- SELinux uses labels on every file and process (more powerful but much more complex)

### VirtualBox vs UTM
- VirtualBox = Type 2 hypervisor for x86_64 machines
- UTM = uses QEMU emulation, works on Apple M1/M2 ARM machines
- ARM Macs cannot natively virtualise x86 Debian — they need emulation

### UFW vs firewalld
- UFW = Debian/Ubuntu default, simpler syntax
- firewalld = Rocky/Red Hat default, more complex, zone-based

### Type 1 vs Type 2 Hypervisor
- Type 1 (bare metal) = runs directly on hardware, no host OS (e.g. VMware ESXi)
- Type 2 (hosted) = runs on top of host OS (e.g. VirtualBox) — what you used

### MBR Partition Limits
- Maximum 4 primary partitions
- Or 3 primary + 1 extended partition
- Extended partition acts as a container for unlimited logical partitions
- Modern alternative: GPT (supports 128 primary partitions, no extended partition needed)

### LVM Benefits
- Resize partitions without reformatting
- Combine multiple physical disks into one logical volume
- Easy snapshots
- More flexible than traditional fixed partitions

---

## Evaluation Commands Cheatsheet

### Simple Setup
```bash
# Check no graphical interface — just boot the VM
# Check UFW
sudo ufw status
# Check SSH
sudo service ssh status
# Check OS
head -2 /etc/os-release
```

### User Section
```bash
groups tadeyelu                          # verify sudo and user42 groups
sudo chage -l tadeyelu                   # verify password expiry rules
sudo chage -l root                       # verify root password expiry
grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs
grep pam_pwquality /etc/pam.d/common-password
sudo adduser newuser                     # create new user
sudo addgroup evaluating                 # create group
sudo adduser newuser evaluating          # add user to group
groups newuser                           # verify group membership
```

### Hostname & Partitions
```bash
hostname                                 # check current hostname
sudo hostnamectl set-hostname newname    # change hostname
sudo reboot                             # apply hostname change
lsblk                                   # view partition structure
sudo fdisk -l                           # view detailed partition table
```

### Sudo
```bash
which sudo                              # confirm sudo installed
sudo cat /etc/sudoers.d/sudo_config     # view sudo rules
sudo ls /var/log/sudo/                  # verify log directory exists
sudo cat /var/log/sudo/sudo_config      # read sudo text log
sudo journalctl _COMM=sudo | grep COMMAND | wc -l  # count sudo commands
```

### UFW
```bash
sudo ufw status numbered                # list rules with numbers
sudo ufw allow 8080                     # add rule
sudo ufw status numbered                # verify new rule
sudo ufw delete allow 8080              # delete rule
sudo ufw status numbered                # verify deletion
```

### SSH
```bash
grep Port /etc/ssh/sshd_config          # verify port 4242
grep PermitRootLogin /etc/ssh/sshd_config  # verify root login disabled
sudo sshd -T | grep permitrootlogin     # check active config
ssh newuser@localhost -p 4243           # SSH in as new user (from host)
ssh root@localhost -p 4243              # attempt root SSH (should fail)
```

### Monitoring Script
```bash
cat /usr/local/bin/monitoring.sh        # view the script
sudo bash /usr/local/bin/monitoring.sh  # run it manually
sudo crontab -l                         # view cron schedule
sudo crontab -e                         # edit cron (to stop/start script)
```

### AppArmor
```bash
sudo systemctl status apparmor          # check AppArmor service
sudo aa-status                          # check loaded profiles
```
