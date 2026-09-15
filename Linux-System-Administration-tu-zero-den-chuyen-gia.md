# 🐧 Tài liệu Linux System Administration: Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo quản trị Linux: command line, networking, process, storage, security, performance tuning.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Khởi đầu](#p0) | Cài VM, SSH, command line |
| [P1. Filesystem & Files](#p1) | Phân quyền, ownership, find |
| [P2. User & Group](#p2) | Quản lý user, sudo, PAM |
| [P3. Process & Service](#p3) | systemd, ps, top, signals |
| [P4. Networking](#p4) | IP, route, DNS, firewall |
| [P5. Storage](#p5) | Disk, LVM, mount, fs |
| [P6. Package Management](#p6) | apt/yum/dnf, snap, source |
| [P7. Security](#p7) | SSH, firewall, SELinux, audit |
| [P8. Performance](#p8) | Tuning, monitoring, profiling |
| [P9. Scripting](#p9) | Bash, cron, automation |
| [P10. Troubleshooting](#p10) | Debug, recovery, post-mortem |

---

<a id="p0"></a>
## P0. Khởi đầu

### Bước 1: Môi trường thực hành

```bash
# Tạo VM bằng Vagrant (như tài liệu Ansible)
vagrant init ubuntu/jammy64
vagrant up
vagrant ssh

# Hoặc dùng cloud VM (AWS EC2, GCP, Azure)
# Hoặc WSL2 (Windows)
```

### Bước 2: Command line cơ bản

```bash
# Navigation
pwd                    # Print working directory
ls -la                 # List chi tiết (hidden files)
cd /path/to/dir
cd ~                   # Home
cd -                   # Previous dir
cd ..                  # Parent

# Help
man ls                 # Manual page
ls --help              # Quick help
tldr ls                 # Simplified manual (cài: apt install tldr)
which ls                # Tìm binary
type ls                 # Alias/function/binary

# File operations
cp source dest
mv source dest
rm file
rm -rf directory       # -r recursive, -f force
mkdir -p dir/subdir    # -p create parent
touch file             # Tạo file rỗng / update timestamp

# View file
cat file
less file              # Phân trang, q để thoát
more file
head -20 file
tail -20 file
tail -f file           # Follow log
```

### Bước 3: SSH

```bash
# Tạo keypair
ssh-keygen -t ed25519 -C "[email protected]"
# Key mặc định: ~/.ssh/id_ed25519, ~/.ssh/id_ed25519.pub

# Copy key lên server
ssh-copy-id user@server
# Hoặc
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Kết nối
ssh user@server
ssh -p 2222 user@server
ssh -i ~/.ssh/key.pem user@server

# SSH config (~/.ssh/config)
Host myserver
    HostName 192.168.1.100
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes

# Sau đó chỉ cần:
ssh myserver

# SSH agent (không cần nhập passphrase mỗi lần)
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519

# Tunneling
ssh -L 8080:localhost:80 user@server    # Local forward
ssh -R 8080:localhost:80 user@server    # Remote forward
ssh -D 1080 user@server                 # SOCKS proxy

# Copy files
scp file.txt user@server:/tmp/
scp -r directory/ user@server:/tmp/
rsync -avz --progress src/ user@server:/dest/

# SSH hardening (server)
/etc/ssh/sshd_config:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2

sudo systemctl restart sshd
```

### Bước 4: Bash shell essentials

```bash
# Wildcards
*           # Any string
?           # Any single char
[abc]       # a, b, hoặc c
[!abc]      # Không phải a, b, c
{a,b,c}     # Brace expansion
*.txt       # Tất cả file .txt

# Quoting
echo "Hello $USER"           # Variable expansion
echo 'Hello $USER'           # Literal
echo "Path: $(pwd)"          # Command substitution
echo "Today: `date`"         # Cũ

# Redirect
command > file               # Stdout to file (overwrite)
command >> file              # Append
command 2> error.log         # Stderr
command &> all.log           # Both
command < input.txt          # Stdin
command1 | command2          # Pipe
command &                    # Background
command1; command2           # Sequential
command1 && command2         # If success
command1 || command2         # If fail

# History
history
!!                           # Last command
!n                           # Command số n
!$                           # Last argument of previous
Ctrl+R                       # Reverse search
```

---

<a id="p1"></a>
## P1. Filesystem & Files

### Bước 1: Filesystem hierarchy

```
/                   # Root
├── bin/            # Essential binaries
├── sbin/           # System binaries
├── etc/            # Configuration
├── home/           # User homes
├── root/           # Root home
├── var/            # Variable data
│   ├── log/        # Logs
│   ├── lib/        # State info
│   └── www/        # Web root
├── tmp/            # Temp files
├── usr/            # User system resources
│   ├── local/      # Local installs
│   └── bin/        # User binaries
├── opt/            # Optional software
├── dev/            # Device files
├── proc/           # Process info (virtual)
├── sys/            # System info (virtual)
├── mnt/            # Mount points
└── media/          # Removable media
```

### Bước 2: Phân quyền

```bash
# Quyền: r (4), w (2), x (1)
# Owner (u), Group (g), Other (o), All (a)

ls -la file
# -rwxr-xr-- 1 user group 1234 Jan 1 12:00 file
#  ^^^^^^^^^   ^^^^  ^^^^^
#  permissions owner group

# Symbolic mode
chmod u+x file          # Owner +x
chmod g+w file          # Group +w
chmod o-r file          # Other -r
chmod a=r file          # All = read only
chmod u=rwx,g=rx,o=r file
chmod -R u+x directory  # Recursive

# Numeric mode
chmod 755 file          # rwxr-xr-x
chmod 644 file          # rw-r--r--
chmod 700 dir           # rwx------ (private)
chmod 777 file          # rwxrwxrwx (avoid!)

# Special permissions
chmod u+s file          # SUID (4000) - run as owner
chmod g+s dir           # SGID (2000) - inherit group
chmod +t dir            # Sticky bit (1000) - only owner can delete

# Common permission patterns
chmod 600 file          # Owner read/write only (ssh key)
chmod 644 file          # Standard file
chmod 755 dir           # Standard dir
chmod 700 dir           # Private dir
```

### Bước 3: Ownership

```bash
chown user:group file
chown -R user:group dir
chown user file         # Change owner only
chown :group file       # Change group only
chgrp group file        # Change group only

# Tìm file theo owner
find / -user username
find / -group groupname
```

### Bước 4: Find & locate

```bash
# find - real-time search
find / -name "*.log"                          # Tên file
find / -type f -size +100M                    # File > 100MB
find / -type d -name "node_modules"            # Directory
find / -mtime -7                              # Modified 7 ngày trước
find / -atime +30                             # Accessed 30+ ngày trước
find / -user root -perm -4000                 # SUID files
find / -name "*.conf" -exec grep -l "password" {} \;
find / -type f -empty                         # File rỗng
find / -maxdepth 2 -name "*.yaml"

# locate - indexed search (nhanh hơn)
sudo updatedb
locate nginx.conf
locate -i "*.jpg"      # Case insensitive
locate -c "*.log"      # Count

# xargs - xử lý kết quả find
find . -name "*.log" | xargs rm
find . -name "*.log" -exec rm {} \;
find . -name "*.log" -exec rm {} +    # Tối ưu hơn
```

### Bước 5: Archive & compress

```bash
# tar
tar -cvf archive.tar files/         # Create
tar -xvf archive.tar               # Extract
tar -tvf archive.tar               # List
tar -czvf archive.tar.gz files/    # gzip
tar -cjvf archive.tar.bz2 files/   # bzip2
tar -cJvf archive.tar.xz files/    # xz (best compression)

# Extract specific file
tar -xvf archive.tar path/to/file
tar -xvzf archive.tar.gz -C /target/dir

# gzip/gunzip
gzip file           # → file.gz
gunzip file.gz
gzip -k file        # Keep original
gzip -9 file        # Max compression

# zip/unzip
zip -r archive.zip directory/
unzip archive.zip
unzip -l archive.zip    # List
```

### Bước 6: Disk usage

```bash
df -h                # Filesystem usage
df -i                # Inodes
du -sh directory     # Size của directory
du -h --max-depth=1 # Top level sizes
du -ah | sort -h | tail -20    # Top 20 files lớn nhất

# Xem file lớn
find / -type f -size +100M -exec ls -lh {} \; | awk '{print $5, $9}' | sort -h

# ncdu - interactive
sudo apt install ncdu
ncdu /

# Tree view
tree -L 2 -h
```

---

<a id="p2"></a>
## P2. User & Group

### Bước 1: Quản lý user

```bash
# Tạo user
useradd username
useradd -m -s /bin/bash -G sudo,docker username
# -m: tạo home
# -s: shell
# -G: groups phụ

# Ubuntu có adduser (thân thiện hơn)
adduser username

# Sửa user
usermod -aG docker username    # Thêm vào group
usermod -s /bin/zsh username   # Đổi shell
usermod -L username           # Lock
usermod -U username           # Unlock
usermod -d /new/home -m username  # Đổi home

# Xoá user
userdel username
userdel -r username    # Xoá cả home

# Đặt password
passwd username
passwd -e username    # Force expire
```

### Bước 2: Quản lý group

```bash
groupadd developers
groupmod -n newname oldname
groupdel developers

# Thêm user vào group
usermod -aG developers username

# Xem groups của user
groups username
id username

# Group administrator
gpasswd -A username developers
gpasswd -a username developers    # Thêm
gpasswd -d username developers    # Xoá
```

### Bước 3: sudo

```bash
# Thêm user vào sudo group
usermod -aG sudo username

# /etc/sudoers (dùng visudo)
visudo

# Pattern trong sudoers
root    ALL=(ALL:ALL) ALL
%sudo   ALL=(ALL:ALL) ALL
deploy  ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
deploy  ALL=(ALL) NOPASSWD: /opt/deploy/*.sh

# Custom file
echo "deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx" \
  > /etc/sudoers.d/deploy
```

### Bước 4: PAM (Pluggable Authentication Modules)

```bash
# /etc/pam.d/
# Login limits
/etc/security/limits.conf
*    hard    nofile    65535
*    soft    nofile    32768

# Password policy
/etc/security/pwquality.conf
minlen = 12
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1

# Failed login lockout
/etc/pam.d/common-auth
auth required pam_faillock.so preauth deny=5 unlock_time=900
auth required pam_faillock.so authfail deny=5 unlock_time=900
```

---

<a id="p3"></a>
## P3. Process & Service

### Bước 1: Process

```bash
# Xem process
ps aux                    # Tất cả process
ps aux | grep nginx
ps -ef --forest           # Tree view
ps -u username            # Theo user

# Top/htop
top                       # Real-time
htop                      # Cải tiến (cài: apt install htop)
btop                      # Đẹp hơn (cài từ github)

# Kill process
kill PID                  # SIGTERM (graceful)
kill -9 PID               # SIGKILL (force)
kill -HUP PID             # Reload config (Nginx, Apache)
killall nginx             # Kill theo tên
pkill -f "pattern"        # Kill theo pattern

# Process priority
nice -n 10 command        # Lower priority
renice -n 5 -p PID        # Đổi priority

# Background jobs
command &
jobs                     # List
fg %1                     # Bring to foreground
Ctrl+Z                    # Suspend
bg %1                     # Resume background

# Daemon - chạy không phụ thuộc terminal
nohup command &
disown
```

### Bước 2: systemd

```bash
# Unit file location
/etc/systemd/system/      # Custom
/lib/systemd/system/      # Distribution
/usr/lib/systemd/system/  # Vendor

# Quản lý service
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl enable nginx   # Start on boot
systemctl disable nginx
systemctl mask nginx     # Không cho start
systemctl unmask nginx

# Xem logs
journalctl -u nginx
journalctl -u nginx -f              # Follow
journalctl -u nginx --since "1h ago"
journalctl -u nginx --since today
journalctl -u nginx --since "2025-01-01" --until "2025-01-02"
journalctl -p err                  # Errors only
journalctl --vacuum-time=7d        # Cleanup

# System info
systemctl list-units --type=service
systemctl list-units --state=running
systemctl list-unit-files

# Targets (runlevels)
systemctl get-default
systemctl set-default multi-user.target
systemctl isolate rescue.target   # Single user mode
```

### Bước 3: Viết service unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env

# Resource limits
LimitNOFILE=65535
LimitNPROC=65535
MemoryMax=1G
CPUQuota=200%

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp

# Logging
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now myapp
systemctl status myapp
```

### Bước 4: Signals

```bash
# Signals thường gặp
SIGHUP (1)    - Reload config
SIGINT (2)    - Interrupt (Ctrl+C)
SIGQUIT (3)   - Quit
SIGKILL (9)   - Force kill (không catch được)
SIGTERM (15)  - Graceful stop (default)
SIGSTOP (19)  - Pause (không catch được)
SIGCONT (18)  - Resume
SIGUSR1 (10)  - User-defined 1
SIGUSR2 (12)  - User-defined 2

# Gửi signal
kill -SIGTERM PID
kill -USR1 PID
```

---

<a id="p4"></a>
## P4. Networking

### Bước 1: Network interfaces

```bash
# ip command (modern)
ip addr                    # Show all interfaces
ip addr show eth0          # Specific interface
ip -4 addr                 # IPv4 only
ip link                    # Link layer
ip route                   # Routing table
ip route get 8.8.8.8       # Route to specific IP
ip neigh                   # ARP table
ip -s link show eth0       # Stats

# Legacy (deprecated nhưng vẫn dùng)
ifconfig
route -n
arp -n

# NetworkManager (Desktop)
nmcli device status
nmcli connection show
nmcli connection up "Wired connection 1"
```

### Bước 2: Cấu hình IP (Netplan - Ubuntu 18+)

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

```bash
sudo netplan apply
sudo netplan try        # Auto rollback nếu fail
```

### Bước 3: DNS

```bash
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
search example.com
domain example.com

# Test DNS
nslookup google.com
dig google.com
dig +short google.com      # Chỉ IP
dig @8.8.8.8 google.com   # Specific server
dig -x 8.8.8.8            # Reverse lookup
dig +trace google.com      # Trace DNS chain

# /etc/hosts (override DNS cho local)
127.0.0.1 localhost
192.168.1.10 web.local
```

### Bước 4: Network tools

```bash
# Test connectivity
ping -c 4 google.com
ping6 ipv6.google.com
traceroute google.com
mtr google.com                # Better traceroute

# Port check
nc -zv hostname 80             # TCP
nc -zuv hostname 53            # UDP
ss -tlnp                       # Listening TCP ports
ss -ulnp                       # Listening UDP
ss -tunap                      # All connections

# Curl
curl -I https://example.com              # Headers
curl -L https://example.com              # Follow redirect
curl -o file.zip https://example.com     # Download
curl -u user:pass https://example.com    # Auth
curl -X POST -d "data" https://api.com   # POST
curl -H "Content-Type: application/json" \
     -d '{"key":"value"}' https://api.com

# Wget
wget https://example.com/file.zip
wget -c https://example.com/file.zip     # Continue
wget -r -np https://example.com/         # Recursive

# Bandwidth
iftop                           # Live bandwidth
nethogs                         # Per-process
iperf3 -s                       # Server
iperf3 -c server                # Client
```

### Bước 5: Firewall (iptables/nftables/ufw/firewalld)

```bash
# UFW (Ubuntu)
sudo ufw enable
sudo ufw disable
sudo ufw status verbose
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.1.0/24 to any port 3306
sudo ufw deny 23
sudo ufw delete allow 80

# iptables
sudo iptables -L                    # List
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables-save > /etc/iptables.rules

# nftables (modern)
nft list ruleset
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; }
nft add rule inet filter input tcp dport 22 accept

# firewalld (CentOS/RHEL)
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

### Bước 6: Network troubleshooting

```bash
# Checklist
1. Link up?           ip link
2. IP configured?     ip addr
3. Default route?     ip route
4. DNS working?       dig/nslookup
5. Host reachable?    ping
6. Port open?         nc/telnet
7. Service running?   ss/netstat
8. Firewall?          iptables/ufw
9. App log?           journalctl/app log

# tcpdump - packet capture
sudo tcpdump -i eth0
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 host 192.168.1.10
sudo tcpdump -i eth0 -w capture.pcap    # Save
sudo tcpdump -r capture.pcap            # Read
sudo tcpdump -i eth0 -A port 80         # ASCII content

# Wireshark (GUI)
sudo apt install wireshark
wireshark capture.pcap
```

---

<a id="p5"></a>
## P5. Storage

### Bước 1: Disk & partition

```bash
# List disks
lsblk
lsblk -f                       # Show filesystem
fdisk -l
parted -l

# Partition
sudo fdisk /dev/sdb
# n: new, p: primary, w: write, d: delete, t: type

# GPT partitioning
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
```

### Bước 2: Filesystem

```bash
# Format
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.xfs /dev/sdb1
sudo mkfs.btrfs /dev/sdb1

# Mount
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data

# Unmount
sudo umount /mnt/data

# Auto-mount (/etc/fstab)
/dev/sdb1  /mnt/data  ext4  defaults,nofail  0  2
UUID=xxx   /mnt/data  ext4  defaults,nofail  0  2
```

### Bước 3: LVM (Logical Volume Manager)

```bash
# Physical Volume
sudo pvcreate /dev/sdb /dev/sdc
sudo pvdisplay

# Volume Group
sudo vgcreate vg_data /dev/sdb /dev/sdc
sudo vgdisplay

# Logical Volume
sudo lvcreate -L 100G -n lv_data vg_data
sudo lvcreate -l 100%FREE -n lv_data vg_data    # Use all
sudo lvdisplay

# Format & mount
sudo mkfs.ext4 /dev/vg_data/lv_data
sudo mount /dev/vg_data/lv_data /mnt/data

# Extend
sudo lvextend -L +50G /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data    # ext4
sudo xfs_growfs /mnt/data              # xfs

# Snapshot
sudo lvcreate -L 10G -s -n snap_data /dev/vg_data/lv_data

# Remove
sudo lvremove /dev/vg_data/lv_data
sudo vgremove vg_data
sudo pvremove /dev/sdb
```

### Bước 4: RAID

```bash
# Software RAID (mdadm)
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
sudo mdadm --detail /dev/md0
sudo mkfs.ext4 /dev/md0
sudo mdadm --examine --scan | sudo tee -a /etc/mdadm/mdadm.conf

# Save config
sudo update-initramfs -u

# /etc/fstab
/dev/md0  /mnt/raid  ext4  defaults  0  2

# Levels:
# 0: Striping (fast, no redundancy)
# 1: Mirroring (redundancy, 50% capacity)
# 5: Striping + parity (1 disk fail tolerance)
# 6: Double parity
# 10: 1+0 (mirror of stripes)
```

### Bước 5: Swap

```bash
# Tạo swap file
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# Swap on partition
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2

# Tối ưu swappiness
sudo sysctl vm.swappiness=10
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf

# Xem
free -h
swapon --show
```

### Bước 6: NFS & SAMBA

```bash
# NFS Server
sudo apt install nfs-kernel-server
echo "/shared 192.168.1.0/24(rw,sync,no_subtree_check)" | sudo tee /etc/exports
sudo exportfs -a
sudo systemctl enable --now nfs-server

# NFS Client
sudo apt install nfs-common
sudo mount 192.168.1.10:/shared /mnt/nfs

# /etc/fstab
192.168.1.10:/shared  /mnt/nfs  nfs  defaults  0  0

# Samba
sudo apt install samba
sudo smbpasswd -a username
# /etc/samba/smb.conf
[shared]
   path = /shared
   browseable = yes
   read only = no
   valid users = username

sudo systemctl restart smbd
```

---

<a id="p6"></a>
## P6. Package Management

### Bước 1: APT (Debian/Ubuntu)

```bash
# Update
sudo apt update
sudo apt upgrade
sudo apt full-upgrade        # Có thể xoá package

# Search
apt search nginx
apt show nginx
apt list --installed

# Install/Remove
sudo apt install nginx
sudo apt install -y nginx php-fpm
sudo apt remove nginx
sudo apt purge nginx         # Xoá cả config
sudo apt autoremove          # Xoá dependencies không cần

# Repository
# /etc/apt/sources.list
# /etc/apt/sources.list.d/

# GPG key
wget -qO - https://example.com/key.gpg | sudo apt-key add -    # Deprecated
# Cách mới:
wget -qO - https://example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/example.gpg

# Held package
sudo apt-mark hold nginx
sudo apt-mark unhold nginx
```

### Bước 2: YUM/DNF (RHEL/CentOS)

```bash
# DNF (modern)
sudo dnf check-update
sudo dnf update
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf search nginx
sudo dnf info nginx
sudo dnf list installed
sudo dnf autoremove

# Repository
sudo dnf config-manager --add-repo https://example.com/repo.repo
sudo dnf repolist

# Groups
sudo dnf group install "Development Tools"
sudo dnf group list

# EPEL
sudo dnf install epel-release
```

### Bước 3: Build from source

```bash
# Cài build tools
sudo apt install build-essential
# hoặc
sudo dnf groupinstall "Development Tools"

# Download source
wget https://example.com/app.tar.gz
tar xzf app.tar.gz
cd app

# Build (thường là autotools hoặc cmake)
./configure --prefix=/usr/local
make
sudo make install

# CMake
mkdir build && cd build
cmake ..
make
sudo make install

# Clean
make clean
make distclean
```

### Bước 4: Snap / Flatpak

```bash
# Snap
sudo snap install code
sudo snap list
sudo snap refresh
sudo snap remove code

# Flatpak
sudo flatpak install flathub org.gimp.GIMP
flatpak list
flatpak update
```

---

<a id="p7"></a>
## P7. Security

### Bước 1: SELinux

```bash
# Status
getenforce                # Enforcing | Permissive | Disabled
sestatus

# Tạm thời
sudo setenforce 0         # Permissive
sudo setenforce 1         # Enforcing

# Vĩnh viễn
# /etc/selinux/config
SELINUX=enforcing

# Quản lý
sudo yum install setools-console
seinfo
sesearch --allow --target httpd_t
sealert -a /var/log/audit/audit.log

# Đổi context
sudo chcon -t httpd_sys_content_t /var/www/html/index.html
sudo restorecon -Rv /var/www/html

# Boolean
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
```

### Bước 2: AppArmor

```bash
# Status
sudo aa-status

# Profiles
ls /etc/apparmor.d/
ls /var/lib/apparmor/

# Disable cho 1 app
sudo aa-complain /usr/bin/nginx
sudo aa-enforce /usr/bin/nginx

# Disable hoàn toàn
sudo systemctl disable --now apparmor
```

### Bước 3: Audit

```bash
# Cài
sudo apt install auditd
sudo systemctl enable --now auditd

# Rule
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
sudo auditctl -w /var/log/auth.log -p wa -k auth_log

# Quy tắc persistent
# /etc/audit/rules.d/audit.rules
-w /etc/passwd -p wa -k passwd_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /var/log/auth.log -p wa -k auth_log

# Search
sudo ausearch -k passwd_changes
sudo ausearch -m USER_LOGIN
sudo aureport --summary
sudo aureport --failed
```

### Bước 4: SSH hardening

```bash
# /etc/ssh/sshd_config
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server

# Limit
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Allow specific users
AllowGroups sshusers
AllowUsers deploy admin

# Port
Port 2222

# Banner
Banner /etc/issue.net
```

```bash
sudo sshd -t               # Test config
sudo systemctl restart sshd
```

### Bước 5: Fail2ban

```bash
# Cài
sudo apt install fail2ban

# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 24h

# Commands
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

### Bước 6: SSL/TLS

```bash
# Generate self-signed
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/ssl/private.key \
  -out /etc/ssl/certificate.crt \
  -subj "/CN=example.com"

# Generate CSR
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.csr

# Let's Encrypt (certbot)
sudo apt install certbot
sudo certbot certonly --standalone -d example.com
sudo certbot certonly --nginx -d example.com
sudo certbot renew --dry-run

# Auto-renew
echo "0 0 * * 0 certbot renew" | sudo crontab -
```

---

<a id="p8"></a>
## P8. Performance Tuning

### Bước 1: System info & monitoring

```bash
# CPU info
lscpu
nproc
cat /proc/cpuinfo

# Memory
free -h
cat /proc/meminfo

# Disk I/O
iostat -xz 1
iotop                          # Per-process I/O
sudo blktrace /dev/sda         # Detailed

# Network
iftop
nethogs
sar -n DEV 1                   # System Activity Reporter
nicstat                        # NIC stats

# System load
uptime
w
top
htop
```

### Bước 2: Sysctl tuning

```bash
# /etc/sysctl.conf

# Network
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Memory
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50

# File handles
fs.file-max = 2097152
fs.nr_open = 1048576

# Apply
sudo sysctl -p
sudo sysctl --system        # Apply all configs
```

### Bước 3: ulimit

```bash
# Temporary
ulimit -n 65535              # Open files
ulimit -u 65535              # Processes
ulimit -s unlimited          # Stack

# Permanent - /etc/security/limits.conf
*    hard    nofile    65535
*    soft    nofile    32768
*    hard    nproc     65535
*    soft    nproc     32768
*    hard    memlock   unlimited

# Systemd service
[Service]
LimitNOFILE=65535
LimitNPROC=65535
```

### Bước 4: Disk I/O tuning

```bash
# I/O scheduler
cat /sys/block/sda/queue/scheduler
echo mq-deadline | sudo tee /sys/block/sda/queue/scheduler

# Read-ahead
sudo blockdev --setra 4096 /dev/sda
cat /sys/block/sda/queue/read_ahead_kb

# I/O stats
iostat -dx 1
iotop -o
```

### Bước 5: Profiling

```bash
# strace - system call trace
strace -p PID
strace -e trace=network -p PID
strace -c -p PID                  # Summary

# perf - performance counter
sudo apt install linux-tools-generic
perf top
perf record -g -p PID
perf report

# bpftrace
sudo apt install bpftrace
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_open { printf("%s %s\n", comm, str(args->filename)); }'

# flamegraph
git clone https://github.com/brendangregg/FlameGraph
perf record -F 99 -p PID -g -- sleep 30
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg
```

---

<a id="p9"></a>
## P9. Scripting & Automation

### Bước 1: Bash script cơ bản

```bash
#!/usr/bin/env bash
set -euo pipefail        # Exit on error, undefined var, pipe fail
IFS=$'\n\t'              # Safe word splitting

# Variables
NAME="World"
readonly PI=3.14

# Arrays
FRUITS=("apple" "banana" "cherry")
echo "${FRUITS[0]}"
echo "${FRUITS[@]}"
echo "${#FRUITS[@]}"

# Associative arrays (Bash 4+)
declare -A USER
USER[name]="John"
USER[age]=30
echo "${USER[name]}"

# Functions
greet() {
    local name="${1:-World}"
    echo "Hello, $name!"
}
greet "Alice"

# Conditionals
if [[ -f "$file" ]]; then
    echo "File exists"
elif [[ -d "$file" ]]; then
    echo "Directory"
else
    echo "Not found"
fi

# Loops
for item in "${FRUITS[@]}"; do
    echo "$item"
done

for i in {1..10}; do
    echo "$i"
done

for i in $(seq 1 10); do
    echo "$i"
done

while read -r line; do
    echo "$line"
done < file.txt

# Case
case "$1" in
    start)
        systemctl start myapp
        ;;
    stop)
        systemctl stop myapp
        ;;
    *)
        echo "Usage: $0 {start|stop}"
        exit 1
        ;;
esac
```

### Bước 2: Script thực tế

```bash
#!/usr/bin/env bash
# backup.sh - Backup directories to S3
set -euo pipefail

readonly BACKUP_DIRS=("/var/www" "/etc/nginx" "/opt/app")
readonly S3_BUCKET="s3://mybucket/backups/$(date +%Y%m%d)"
readonly LOG_FILE="/var/log/backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

main() {
    log "Starting backup"
    
    for dir in "${BACKUP_DIRS[@]}"; do
        if [[ ! -d "$dir" ]]; then
            log "WARNING: $dir not found, skipping"
            continue
        fi
        
        local basename=$(basename "$dir")
        local archive="/tmp/${basename}-$(date +%Y%m%d).tar.gz"
        
        log "Archiving $dir"
        tar czf "$archive" -C "$(dirname "$dir")" "$basename"
        
        log "Uploading to S3"
        aws s3 cp "$archive" "$S3_BUCKET/"
        
        rm "$archive"
    done
    
    log "Backup completed"
}

main "$@"
```

### Bước 3: Cron

```bash
# Crontab format
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
# │ │ │ │ │
# * * * * * command

# Edit
crontab -e

# List
crontab -l

# Examples
0 2 * * * /opt/backup.sh                    # Mỗi ngày 2h sáng
*/5 * * * * /opt/check.sh                   # Mỗi 5 phút
0 0 * * 0 /opt/weekly.sh                    # Chủ nhật 0h
@reboot /opt/startup.sh                     # Khi boot
@hourly /opt/hourly.sh                      # Mỗi giờ

# Redirect output
0 2 * * * /opt/backup.sh >> /var/log/cron.log 2>&1

# System cron
# /etc/cron.d/
# /etc/cron.daily/
# /etc/cron.hourly/
# /etc/cron.weekly/
# /etc/cron.monthly/

# Anacron (cho laptop)
# /etc/anacrontab
```

### Bước 4: Systemd timer (thay cron)

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/opt/backup.sh

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers
```

---

<a id="p10"></a>
## P10. Troubleshooting

### Bước 1: System không boot

```bash
# Boot vào single-user / recovery
# Trong GRUB: e, thêm "single" hoặc "init=/bin/bash"

# Remount root read-write
mount -o remount,rw /

# Check fs
fsck /dev/sda1

# Repair fstab nếu sai
nano /etc/fstab

# Reset password root
passwd

# Reinstall GRUB
grub-install /dev/sda
update-grub
```

### Bước 2: High CPU load

```bash
# Xem top processes
top -c
htop

# Process chiếm nhiều CPU
ps aux --sort=-%cpu | head

# Trace process
strace -p PID
perf top -p PID

# Check load average
uptime
# load average: 0.5, 1.0, 2.0 = OK nếu < số cores
```

### Bước 3: Memory issues

```bash
# Xem memory
free -h
cat /proc/meminfo

# Top memory processes
ps aux --sort=-%mem | head

# OOM killer logs
dmesg | grep -i "out of memory"
journalctl -k | grep -i oom

# Xem chi tiết process
pmap PID
cat /proc/PID/smaps | head -30

# Clear cache
sync
echo 3 > /proc/sys/vm/drop_caches    # Free page cache
```

### Bước 4: Disk full

```bash
# Tìm chỗ chiếm nhiều
df -h
du -sh /* | sort -h | tail

# Top files lớn
find / -type f -size +100M -exec ls -lh {} \;
find /var/log -type f -name "*.log" -size +100M

# Xoá log cũ
sudo journalctl --vacuum-time=7d
sudo find /var/log -type f -name "*.gz" -delete

# Xoá package cache
sudo apt clean
sudo apt autoremove

# Truncate file đang mở (không cần restart service)
sudo truncate -s 0 /var/log/app.log
```

### Bước 5: Network down

```bash
# Checklist (theo OSI model)
# L1 - Physical: cable, NIC
ip link show
sudo ethtool eth0

# L2 - Data Link: ARP, switch
ip neigh
arp -n

# L3 - Network: IP, route
ip addr
ip route
ping gateway

# L4 - Transport: TCP/UDP, port
ss -tlnp
nc -zv server 80

# L7 - Application: HTTP, DNS
curl -v http://server
dig server

# DNS check
cat /etc/resolv.conf
dig @8.8.8.8 server

# Interface restart
sudo ip link set eth0 down
sudo ip link set eth0 up
sudo systemctl restart networking
```

### Bước 6: Service không start

```bash
# Status chi tiết
systemctl status nginx -l

# Logs
journalctl -u nginx -n 50 --no-pager
journalctl -u nginx --since "10 min ago"

# Test config
nginx -t
apache2ctl configtest

# Manual run
sudo -u www-data nginx -g "daemon off;"
sudo /usr/sbin/sshd -D -d

# Permission check
ls -la /var/log/app.log
ls -la /etc/myapp/

# Port conflict
sudo ss -tlnp | grep :80
sudo lsof -i :80
```

### Bước 7: Recovery mode

```bash
# Boot vào rescue target
systemctl rescue

# Hoặc emergency (chỉ root shell)
systemctl emergency

# Từ GRUB: thêm "systemd.unit=rescue.target"
```

---

## 🎯 Bài tập P0-P10

1. Cài Vagrant box Ubuntu, cấu hình SSH key, sudo cho user mới
2. Tìm và xoá 5 file lớn nhất trong /var
3. Tạo LVM volume, format ext4, mount persistent
4. Cấu hình firewall cho phép chỉ SSH từ IP cụ thể
5. Viết bash script backup 3 directory ra tar.gz, log ra file
6. Cron job chạy script mỗi ngày 2h sáng
7. Debug một service không start, fix và restart thành công

---

> **💡 Tip cuối**: Linux không khó - chỉ cần thực hành hàng ngày. Đọc `man`, dùng `tmux` để có môi trường làm việc tốt hơn. Luôn backup trước khi thay đổi config quan trọng.

---

*Tạo bởi tài liệu học Linux System Administration - Chúc bạn thành công! 🚀*
