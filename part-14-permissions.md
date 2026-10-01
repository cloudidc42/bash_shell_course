# Part 14: Permissions & Ownership Advanced
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 14.1 Linux Permission Model

```bash
# Format: [type][owner][group][others]
# d rwx rwx rwx
# | |   |   └── other permissions
# | |   └────── group permissions
# | └────────── owner permissions
# └──────────── file type

# File types:
# - = regular file
# d = directory
# l = symbolic link
# b = block device
# c = character device
# p = named pipe (FIFO)
# s = socket

ls -la
# -rw-r--r-- 1 user group 1234 Jan 1 12:00 file.txt
# drwxr-xr-x 2 user group 4096 Jan 1 12:00 directory/
# lrwxrwxrwx 1 user group   10 Jan 1 12:00 link -> target

# Octal representation
# r = 4, w = 2, x = 1
# rwx = 7, rw- = 6, r-x = 5, r-- = 4, etc.
# 755 = rwxr-xr-x
# 644 = rw-r--r--
# 600 = rw-------
# 700 = rwx------

stat -c "%a %n" file.txt    # show octal permission
stat file.txt               # full stat info

# ─── chmod ────────────────────────────────────────────────────
chmod 755 file           # octal mode
chmod u+x file           # add execute to owner
chmod g-w file           # remove write from group
chmod o=r file           # set other to read-only
chmod a+r file           # add read to all
chmod u+x,g-w,o-r file   # multiple changes
chmod -R 755 directory   # recursive

# Common permissions
chmod 644 file.txt       # rw-r--r-- (regular files)
chmod 755 script.sh      # rwxr-xr-x (executables)
chmod 600 private.key    # rw------- (private keys)
chmod 700 ~/private/     # rwx------ (private dirs)
chmod 777 shared/        # rwxrwxrwx (world-writable, avoid!)
chmod 1777 /tmp          # with sticky bit

# ─── chown / chgrp ────────────────────────────────────────────
chown user file              # change owner
chown user:group file        # change owner and group
chown :group file            # change group only
chgrp group file             # change group
chown -R user:group dir/     # recursive
sudo chown root:root file    # change to root
```

---

## 14.2 Special Permissions

```bash
# ─── Setuid (SUID) - 4000 ─────────────────────────────────────
# Executable runs as file owner, not caller
# rws (x becomes s for owner)
chmod 4755 /usr/bin/sudo     # typical sudo setup
chmod u+s executable         # add suid
chmod u-s executable         # remove suid

ls -la /usr/bin/passwd       # -rwsr-xr-x (setuid)
ls -la /usr/bin/sudo         # -rwsr-xr-x (setuid)

# Find setuid files (security audit)
find / -perm -4000 -type f 2>/dev/null

# ─── Setgid (SGID) - 2000 ─────────────────────────────────────
# On files: runs as file's group
# On dirs: new files inherit parent's group
chmod 2775 shared_dir/       # typical shared dir
chmod g+s shared_dir/

ls -la shared_dir/           # drwxrwsr-x

# Find setgid files
find / -perm -2000 -type f 2>/dev/null

# ─── Sticky Bit - 1000 ────────────────────────────────────────
# On dirs: only file owner can delete/rename (even if others have write)
chmod 1777 /tmp              # typical /tmp setup
chmod +t public_dir/

ls -la /tmp                  # drwxrwxrwt
ls -la /var/tmp              # drwxrwxrwt

# ─── Combined Special Permissions ─────────────────────────────
chmod 6755 file              # setuid + setgid
chmod 7777 dir               # all specials (dangerous!)

# ─── Lowercase s vs uppercase S ───────────────────────────────
# lowercase s = special bit + execute bit set
# uppercase S = special bit set but execute NOT set
# -rwSr--r-- = setuid set, but no execute (unusual/dangerous)
```

---

## 14.3 umask

```bash
# umask: ลบสิทธิ์ออกจาก default permission ใหม่
# default file: 666 (no execute)
# default dir:  777

# umask 022:
# files: 666 - 022 = 644 (rw-r--r--)
# dirs:  777 - 022 = 755 (rwxr-xr-x)

umask              # show current (e.g., 0022)
umask 027          # set new umask
umask -S           # symbolic display (u=rwx,g=rx,o=)

# Common umask values
# 022 = 644/755 (default, world-readable)
# 027 = 640/750 (no world access)
# 077 = 600/700 (private)
# 002 = 664/775 (group-writable)

# Persistent umask in .bashrc
echo "umask 022" >> ~/.bashrc

# Temporary umask change
(
    umask 077
    touch private_file      # created as 600
    mkdir private_dir       # created as 700
)
# original umask restored after subshell
```

---

## 14.4 Access Control Lists (ACL)

```bash
# ACL ให้ granular permission มากกว่า owner/group/other

# Install
apt-get install acl
# Mount filesystem with ACL support: add acl to /etc/fstab
# /dev/sda1 /  ext4  defaults,acl  0 1

# ─── getfacl ──────────────────────────────────────────────────
getfacl file.txt
# file: file.txt
# owner: user
# group: group
# user::rw-
# group::r--
# other::r--

getfacl -R directory/    # recursive

# ─── setfacl ──────────────────────────────────────────────────
# Add permission for specific user
setfacl -m u:alice:rw file.txt
setfacl -m u:bob:r file.txt
setfacl -m u:charlie:--- file.txt  # no access

# Add for specific group
setfacl -m g:developers:rw file.txt

# Default ACL for directory (new files inherit)
setfacl -d -m u:alice:rw shared_dir/
setfacl -d -m g:team:r shared_dir/

# Remove specific ACL entry
setfacl -x u:alice file.txt

# Remove all ACL entries (keep standard permissions)
setfacl -b file.txt

# Recursive
setfacl -R -m u:alice:rx directory/

# Copy ACL from one file to another
getfacl source.txt | setfacl --set-file=- destination.txt

# View ACL in ls (+ indicator)
ls -la file.txt    # -rw-rw-r--+ (+ means has ACL)

# ─── ACL Scripts ──────────────────────────────────────────────
# Backup ACLs
getfacl -R /path/to/dir > acl_backup.txt

# Restore ACLs
setfacl --restore=acl_backup.txt
```

---

## 14.5 File Attributes (chattr/lsattr)

```bash
# File attributes beyond normal permissions
# Works on ext2/ext3/ext4/xfs filesystems

lsattr file.txt        # list attributes
lsattr -R directory/   # recursive

# ─── Attributes ───────────────────────────────────────────────
# a = append only (can append, not modify/delete)
# i = immutable (cannot modify/delete/rename, even root!)
# e = extent format (usually set by default)
# j = journal (data journaling)
# d = no dump (exclude from backup)
# A = no atime update
# D = synchronous directory update
# S = synchronous file update
# T = top of directory hierarchy

# ─── chattr usage ─────────────────────────────────────────────
chattr +i important_file    # make immutable
chattr -i important_file    # remove immutable
chattr +a log_file          # append-only
chattr -a log_file          # remove append-only

# Protection examples
sudo chattr +i /etc/passwd          # protect passwd file
sudo chattr +i /etc/sudoers         # protect sudoers
sudo chattr +a /var/log/auth.log    # append-only log

# Check
lsattr /etc/passwd      # ----i--------e-- /etc/passwd

# ─── Extended Attributes (xattr) ──────────────────────────────
# Store metadata as key-value pairs

# Install xattr tools
apt-get install attr

# Set/get extended attributes
setfattr -n user.description -v "My important file" file.txt
getfattr -n user.description file.txt
getfattr -d file.txt          # list all xattrs

# Remove
setfattr -x user.description file.txt

# Common namespaces:
# user.*     = user-defined (normal users)
# system.*   = system-defined (kernel/OS)
# security.* = security (SELinux labels)
# trusted.*  = trusted (root only)
```

---

## 14.6 sudo & Privilege Escalation (Legitimate)

```bash
# ─── sudo basics ──────────────────────────────────────────────
sudo command                    # run as root
sudo -u user command            # run as specific user
sudo -g group command           # run as specific group
sudo -i                         # root login shell
sudo -s                         # root shell with current env
sudo -l                         # list allowed commands
sudo -l -U user                 # list for specific user
sudo -v                         # refresh sudo timeout
sudo -k                         # invalidate sudo cache

# Run script as root
sudo bash script.sh
sudo bash -c "command1; command2"

# Environment with sudo
sudo env VAR=value command
sudo -E command                 # preserve environment

# ─── /etc/sudoers ─────────────────────────────────────────────
# Always edit with visudo (validates syntax)
sudo visudo

# Syntax: user ALL=(run_as) commands
# user   = username or %group
# ALL    = hostname (any host)
# (user) = run as this user
# cmd    = command(s) allowed

# Examples:
alice   ALL=(ALL) ALL              # full sudo for alice
bob     ALL=(ALL) NOPASSWD: ALL   # no password required
%admin  ALL=(ALL) ALL             # group admin has full sudo
deploy  ALL=(www-data) /usr/bin/systemctl restart nginx  # specific

# Include files
#includedir /etc/sudoers.d/

# /etc/sudoers.d/myapp (must be 440)
cat > /etc/sudoers.d/backup_operator << 'EOF'
backupuser ALL=(root) NOPASSWD: /usr/bin/rsync, /usr/bin/tar
EOF
chmod 440 /etc/sudoers.d/backup_operator

# ─── Capabilities (fine-grained) ──────────────────────────────
# Give specific capabilities to executable without full setuid

# Install
apt-get install libcap2-bin

# List capabilities
getcap /usr/bin/ping
# /usr/bin/ping = cap_net_raw+ep

# Set capabilities
sudo setcap cap_net_raw+ep myapp        # raw network access
sudo setcap cap_net_bind_service+ep app  # bind port < 1024
sudo setcap cap_dac_read_search+ep app  # bypass file read permissions

# Remove capabilities
sudo setcap -r myapp

# Common capabilities
# cap_net_raw       - raw sockets (ping)
# cap_net_bind_service - bind port < 1024
# cap_setuid/gid    - change uid/gid
# cap_sys_admin     - various admin operations
# cap_dac_override  - bypass file permissions
# cap_chown         - change file ownership
```

---

## 14.7 SELinux & AppArmor Basics

```bash
# ─── SELinux (Red Hat/CentOS/Fedora) ──────────────────────────
# Security contexts: user:role:type:level

# Check status
getenforce              # Enforcing / Permissive / Disabled
sestatus                # detailed status
sestatus -b             # boolean values

# Modes
setenforce 1            # Enforcing (enforces policy)
setenforce 0            # Permissive (logs only)

# View context
ls -Z file              # file context
ps -Z                   # process context
id -Z                   # user context

# Change context
chcon -t httpd_sys_content_t /var/www/html/
restorecon -R /var/www/html/   # restore to policy default

# Manage booleans
getsebool -a | grep http
setsebool -P httpd_can_network_connect on
setsebool -P allow_user_exec_content on

# Audit2allow (fix denials)
ausearch -m AVC -ts recent | audit2allow -m mymodule > mymodule.te
checkmodule -M -m -o mymodule.mod mymodule.te
semodule_package -o mymodule.pp -m mymodule.mod
semodule -i mymodule.pp

# ─── AppArmor (Ubuntu/Debian) ─────────────────────────────────
aa-status               # show status
aa-enforce /path/to/profile   # enforce mode
aa-complain /path/to/profile  # complain mode (log only)
aa-disable /path/to/profile   # disable profile

# Load profile
apparmor_parser -r /etc/apparmor.d/usr.bin.myapp

# Profile location
ls /etc/apparmor.d/

# Simple profile
cat > /etc/apparmor.d/usr.bin.myapp << 'EOF'
#include <tunables/global>

/usr/bin/myapp {
  #include <abstractions/base>
  
  /etc/myapp/** r,
  /var/log/myapp/ rw,
  /var/log/myapp/*.log rw,
  /tmp/ rw,
  /tmp/myapp* rw,
  
  deny /etc/shadow r,
  deny /proc/*/mem rw,
}
EOF
```

---

## 14.8 Permission Audit Script

```bash
#!/bin/bash
# permission_audit.sh - Security permission audit

set -euo pipefail

RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
NC='\033[0m'

REPORT_FILE="/tmp/perm_audit_$(date +%Y%m%d_%H%M%S).txt"

log() { echo "$1" | tee -a "$REPORT_FILE"; }
warn() { echo -e "${YELLOW}[WARN]${NC} $1" | tee -a "$REPORT_FILE"; }
alert() { echo -e "${RED}[ALERT]${NC} $1" | tee -a "$REPORT_FILE"; }
ok() { echo -e "${GREEN}[OK]${NC} $1" | tee -a "$REPORT_FILE"; }

log "Permission Security Audit - $(date)"
log "============================================"
log ""

# ─── Check SUID files ─────────────────────────────────────────
log "=== SUID Files ==="
known_suid=(
    /usr/bin/sudo /usr/bin/passwd /usr/bin/su /usr/bin/ping
    /usr/bin/mount /usr/bin/umount /bin/su /bin/ping
    /usr/lib/openssh/ssh-keysign
)

while IFS= read -r file; do
    known=false
    for k in "${known_suid[@]}"; do
        [[ "$file" == "$k" ]] && known=true && break
    done
    if $known; then
        ok "$file"
    else
        alert "UNEXPECTED SUID: $file"
    fi
done < <(find / -perm -4000 -type f 2>/dev/null)

log ""

# ─── Check world-writable files ───────────────────────────────
log "=== World-Writable Files (excluding /tmp, /proc) ==="
while IFS= read -r file; do
    alert "World-writable: $file"
done < <(find / \
    -not -path "/tmp/*" \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -not -path "/dev/*" \
    -perm -002 -type f 2>/dev/null)

log ""

# ─── Check SSH keys ───────────────────────────────────────────
log "=== SSH Key Permissions ==="
while IFS= read -r key; do
    perm=$(stat -c "%a" "$key")
    if [[ "$perm" != "600" && "$perm" != "400" ]]; then
        alert "Insecure SSH key permission ($perm): $key"
    else
        ok "$key ($perm)"
    fi
done < <(find /home -name "id_*" -not -name "*.pub" 2>/dev/null)
find /root/.ssh -name "id_*" -not -name "*.pub" 2>/dev/null | while read -r key; do
    perm=$(stat -c "%a" "$key")
    if [[ "$perm" != "600" && "$perm" != "400" ]]; then
        alert "Insecure SSH key permission ($perm): $key"
    else
        ok "$key ($perm)"
    fi
done

log ""

# ─── Check /etc permissions ───────────────────────────────────
log "=== Critical File Permissions ==="
declare -A expected_perms=(
    ["/etc/passwd"]="644"
    ["/etc/shadow"]="000"
    ["/etc/sudoers"]="440"
    ["/etc/hosts"]="644"
    ["/etc/fstab"]="644"
    ["/etc/crontab"]="600"
    ["/etc/ssh/sshd_config"]="600"
)

for file in "${!expected_perms[@]}"; do
    if [[ -f "$file" ]]; then
        actual=$(stat -c "%a" "$file")
        expected="${expected_perms[$file]}"
        if [[ "$actual" != "$expected" ]]; then
            warn "$file: expected $expected, got $actual"
        else
            ok "$file: $actual"
        fi
    fi
done

log ""

# ─── Check home directories ───────────────────────────────────
log "=== Home Directory Permissions ==="
while IFS=: read -r user _ uid _ _ home _; do
    (( uid >= 1000 && uid < 65534 )) || continue
    [[ -d "$home" ]] || continue
    
    perm=$(stat -c "%a" "$home")
    if [[ "$perm" =~ ^[67] ]]; then
        # 7xx or 6xx - owner has access (ok)
        if [[ "$perm" =~ ..[^0] ]]; then
            warn "Other users can access $home ($perm)"
        else
            ok "$home ($perm)"
        fi
    else
        alert "Owner cannot access $home ($perm)"
    fi
done < /etc/passwd

log ""
log "Report saved to: $REPORT_FILE"
```

---

## 14.9 Exercises

### Exercise 1: Permission Hardener
สร้าง script ที่ hardened server:
- Fix insecure permissions
- Remove unnecessary SUID
- Set proper umask
- Document changes made

### Exercise 2: ACL Manager
สร้าง wrapper ที่ manage ACLs:
- Show current ACLs ด้วย format ที่อ่านง่าย
- Add/remove user/group access
- Apply templates (read-only, developer, admin)
- Backup/restore ACLs

### Exercise 3: File Protection System
สร้าง script ที่:
- Mark files as immutable (+i)
- Track protected files ใน database
- Allow temporary unprotect with logging
- Auto-reprotect after timeout

---

## สรุป Part 14

✅ Permission model (rwx, octal, symbolic)  
✅ Special permissions (SUID, SGID, sticky bit)  
✅ umask calculation  
✅ ACL with getfacl/setfacl  
✅ File attributes with chattr/lsattr  
✅ sudo configuration  
✅ Linux capabilities  
✅ SELinux/AppArmor basics  
✅ Permission audit script  

---

**→ Part 15: Package Management**
