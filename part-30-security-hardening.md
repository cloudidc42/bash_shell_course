# Part 30: Security Hardening Scripts
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 30.1 System Hardening

```bash
#!/bin/bash
# system_hardening.sh - Linux security hardening

set -euo pipefail

log() { echo "[$(date '+%H:%M:%S')] $*"; }
applied() { echo -e "\033[32m✓\033[0m $*"; }
skipped() { echo -e "\033[33m-\033[0m $*"; }

# ─── System Updates ───────────────────────────────────────────
update_system() {
    log "Updating system packages..."
    
    if command -v apt-get &>/dev/null; then
        apt-get update -q
        apt-get upgrade -y -q
        apt-get dist-upgrade -y -q
        apt-get autoremove -y -q
        applied "System updated (apt)"
    elif command -v dnf &>/dev/null; then
        dnf update -y -q
        applied "System updated (dnf)"
    fi
}

# ─── User Account Security ────────────────────────────────────
harden_accounts() {
    log "Hardening user accounts..."
    
    # Disable root login
    passwd -l root
    applied "Root account locked"
    
    # Set password policy
    if [[ -f /etc/security/pwquality.conf ]]; then
        cat >> /etc/security/pwquality.conf << 'EOF'
minlen = 14
minclass = 4
maxrepeat = 3
EOF
        applied "Password policy configured"
    fi
    
    # Set password aging
    cat > /etc/login.defs_hardened << 'EOF'
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14
LOGIN_RETRIES   5
LOGIN_TIMEOUT   60
EOF
    
    # Lock accounts with no password
    awk -F: '($2 == "" ) {print $1}' /etc/shadow | while read -r user; do
        passwd -l "$user"
        applied "Locked passwordless account: $user"
    done
}

# ─── SSH Hardening ────────────────────────────────────────────
harden_ssh() {
    log "Hardening SSH..."
    
    local sshd_config="/etc/ssh/sshd_config"
    
    # Backup
    cp "$sshd_config" "${sshd_config}.bak"
    
    # Apply settings
    declare -A ssh_settings=(
        ["PermitRootLogin"]="no"
        ["PasswordAuthentication"]="no"
        ["PermitEmptyPasswords"]="no"
        ["X11Forwarding"]="no"
        ["MaxAuthTries"]="3"
        ["LoginGraceTime"]="60"
        ["ClientAliveInterval"]="300"
        ["ClientAliveCountMax"]="2"
        ["Protocol"]="2"
        ["AllowAgentForwarding"]="no"
        ["AllowTcpForwarding"]="no"
        ["UsePAM"]="yes"
        ["UsePrivilegeSeparation"]="sandbox"
    )
    
    for key in "${!ssh_settings[@]}"; do
        local value="${ssh_settings[$key]}"
        
        if grep -q "^${key}" "$sshd_config"; then
            sed -i "s/^${key}.*/${key} ${value}/" "$sshd_config"
        else
            echo "${key} ${value}" >> "$sshd_config"
        fi
        applied "SSH: $key = $value"
    done
    
    # Restart SSH
    systemctl restart sshd
}

# ─── Kernel Parameters ────────────────────────────────────────
harden_kernel() {
    log "Hardening kernel parameters..."
    
    cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
# IP Forwarding
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Ignore source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log martians
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# ASLR
kernel.randomize_va_space = 2

# Restrict core dumps
fs.suid_dumpable = 0
kernel.core_uses_pid = 1

# Restrict ptrace
kernel.yama.ptrace_scope = 1

# IPv6 privacy
net.ipv6.conf.all.use_tempaddr = 2
EOF
    
    sysctl -p /etc/sysctl.d/99-hardening.conf
    applied "Kernel parameters hardened"
}

# ─── Firewall Setup (UFW) ─────────────────────────────────────
setup_firewall() {
    log "Configuring firewall..."
    
    if ! command -v ufw &>/dev/null; then
        apt-get install -y ufw
    fi
    
    # Reset
    ufw --force reset
    
    # Default policies
    ufw default deny incoming
    ufw default allow outgoing
    
    # Allow SSH
    ufw allow ssh
    
    # Rate limit SSH
    ufw limit ssh
    
    ufw --force enable
    applied "Firewall configured"
    ufw status
}

# ─── Disable unused services ──────────────────────────────────
disable_services() {
    log "Disabling unnecessary services..."
    
    local services_to_disable=(
        "avahi-daemon"
        "cups"
        "rpcbind"
        "nfs-server"
        "bluetooth"
        "wifi-handler"
    )
    
    for service in "${services_to_disable[@]}"; do
        if systemctl is-active "$service" &>/dev/null; then
            systemctl stop "$service"
            systemctl disable "$service"
            applied "Disabled: $service"
        else
            skipped "$service (not active)"
        fi
    done
}

# ─── File System Security ─────────────────────────────────────
harden_filesystem() {
    log "Hardening filesystem..."
    
    # Set immutable on critical files
    local protected_files=(
        /etc/passwd
        /etc/shadow
        /etc/gshadow
        /etc/sudoers
    )
    
    for f in "${protected_files[@]}"; do
        [[ -f "$f" ]] || continue
        chattr +i "$f"
        applied "Protected: $f"
    done
    
    # Fix world-writable files
    find / -xdev -perm -002 -not -path "/tmp/*" -not -path "/proc/*" \
        -not -path "/sys/*" -not -path "/dev/*" -type f 2>/dev/null | \
        while read -r f; do
            chmod o-w "$f"
            applied "Fixed world-writable: $f"
        done
}

# ─── Audit System ─────────────────────────────────────────────
setup_auditd() {
    log "Configuring audit daemon..."
    
    apt-get install -y auditd 2>/dev/null || yum install -y audit
    
    cat > /etc/audit/rules.d/hardening.rules << 'EOF'
# Delete all existing rules
-D

# Buffer size
-b 8192

# Failure mode: 1=printk, 2=panic
-f 1

# Audit file access to sensitive files
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/ssh/sshd_config -p wa -k sshd

# Audit login events
-w /var/log/faillog -p wa -k logins
-w /var/log/lastlog -p wa -k logins

# Audit privileged commands
-a always,exit -F path=/usr/bin/sudo -F perm=x -k privileged
-a always,exit -F path=/usr/bin/su -F perm=x -k privileged

# Audit network configuration changes
-a always,exit -F arch=b64 -S sethostname -S setdomainname -k network
-w /etc/hosts -p wa -k network
-w /etc/network -p wa -k network

# Audit cron
-w /etc/cron.allow -p wa -k cron
-w /etc/cron.d -p wa -k cron
-w /etc/crontab -p wa -k cron

# Audit kernel modules
-w /sbin/insmod -p x -k modules
-w /sbin/rmmod -p x -k modules
-w /sbin/modprobe -p x -k modules
EOF
    
    systemctl restart auditd
    applied "Auditd configured"
}

# ─── Main ─────────────────────────────────────────────────────
main() {
    [[ $EUID -ne 0 ]] && echo "Must run as root" && exit 1
    
    echo "=== System Hardening ==="
    echo ""
    
    update_system
    harden_accounts
    harden_ssh
    harden_kernel
    setup_firewall
    disable_services
    harden_filesystem
    setup_auditd
    
    echo ""
    echo "=== Hardening Complete ==="
    echo "Please review changes and test system functionality"
}

main
```

---

## 30.2 Security Audit Script

```bash
#!/bin/bash
# security_audit.sh - Security audit and reporting

REPORT_FILE="/tmp/security_audit_$(date +%Y%m%d_%H%M%S).txt"

section() { echo ""; echo "═══ $1 ═══"; }
pass() { echo "  ✓ $1"; }
fail() { echo "  ✗ $1"; }
warn() { echo "  ! $1"; }
info() { echo "  - $1"; }

{
    echo "Security Audit Report"
    echo "Host: $(hostname)"
    echo "Date: $(date)"
    echo "User: $(id)"
    
    # ─── SSH Config ───────────────────────────────────────────
    section "SSH Configuration"
    sshd_config=/etc/ssh/sshd_config
    
    check_ssh() {
        local setting=$1 expected=$2
        local actual
        actual=$(grep -i "^$setting" "$sshd_config" 2>/dev/null | awk '{print $2}')
        
        if [[ "$actual" == "$expected" ]]; then
            pass "$setting: $actual"
        else
            fail "$setting: '$actual' (expected: $expected)"
        fi
    }
    
    check_ssh "PermitRootLogin" "no"
    check_ssh "PasswordAuthentication" "no"
    check_ssh "X11Forwarding" "no"
    check_ssh "MaxAuthTries" "3"
    
    # ─── Firewall Status ──────────────────────────────────────
    section "Firewall Status"
    
    if ufw status 2>/dev/null | grep -q "Status: active"; then
        pass "UFW: active"
        ufw status | grep -v "^$" | tail -n +3 | sed 's/^/  /'
    elif iptables -L 2>/dev/null | grep -q "Chain INPUT"; then
        warn "iptables: configured (check rules manually)"
    else
        fail "No firewall detected"
    fi
    
    # ─── Updates ──────────────────────────────────────────────
    section "Security Updates"
    
    if command -v apt-get &>/dev/null; then
        apt-get -s upgrade 2>/dev/null | grep -c "^Inst" | \
            xargs -I{} bash -c '[[ {} -eq 0 ]] && echo "  ✓ System up to date" || echo "  ✗ {} updates available"'
    fi
    
    # ─── User Accounts ────────────────────────────────────────
    section "User Accounts"
    
    # Empty passwords
    while IFS=: read -r user pass _; do
        [[ -z "$pass" ]] && fail "Empty password: $user"
    done < /etc/shadow 2>/dev/null
    
    # UID 0 accounts (besides root)
    awk -F: '($3 == 0) {print $1}' /etc/passwd | \
        grep -v "^root$" | while read -r user; do
            fail "UID 0 account (not root): $user"
        done
    
    # ─── SUID/SGID ────────────────────────────────────────────
    section "SUID/SGID Files"
    
    suid_count=$(find / -perm -4000 -type f 2>/dev/null | wc -l)
    info "SUID files: $suid_count"
    find / -perm -4000 -type f 2>/dev/null | head -10 | sed 's/^/  /'
    
    # ─── Open Ports ───────────────────────────────────────────
    section "Open Ports"
    ss -tulpn | grep LISTEN | sed 's/^/  /'
    
    # ─── Failed Logins ────────────────────────────────────────
    section "Recent Failed Logins (last 24h)"
    
    journalctl -u ssh --since "24 hours ago" 2>/dev/null | \
        grep "Failed password" | \
        awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | \
        head -10 | sed 's/^/  /'
    
} | tee "$REPORT_FILE"

echo ""
echo "Report saved: $REPORT_FILE"
```

---

## 30.3 Exercises

### Exercise 1: CIS Benchmark Checker
Implement checks for CIS benchmarks:
- Level 1 and Level 2
- Score system (pass/fail/warning)
- HTML report
- Remediation guide

### Exercise 2: Intrusion Detection
สร้าง basic IDS:
- Watch for suspicious processes
- Monitor privileged commands
- Alert on new SUID files
- Track login failures

### Exercise 3: Security Policy Enforcer
สร้าง policy enforcement:
- Check configurations
- Fix violations automatically
- Report compliance score
- Schedule regular audits

---

## สรุป Part 30 (และ Intermediate Level)

✅ System hardening (users, SSH, kernel)  
✅ Firewall configuration (UFW)  
✅ Disabling unnecessary services  
✅ File system protection (chattr +i)  
✅ Audit daemon (auditd) configuration  
✅ Security audit script  
✅ SSH hardening parameters  
✅ Kernel parameter hardening (sysctl)  

---

## สรุป Level 2: Intermediate (Parts 21-30)

| Part | Topic |
|------|-------|
| 21 | Advanced awk |
| 22 | Advanced sed |
| 23 | JSON & Data Formats |
| 24 | Database Operations |
| 25 | Docker & Kubernetes |
| 26 | SSH Automation |
| 27 | Git Automation |
| 28 | Log Analysis |
| 29 | Backup & Recovery |
| 30 | Security Hardening |

**→ Part 31: Shell Internals & Advanced Bash Features (Advanced Level)**
