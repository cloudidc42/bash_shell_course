# Part 39: Security Hardening & Audit Automation
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 39.1 System Security Audit

```bash
#!/bin/bash
# security_audit.sh - Comprehensive system security audit

set -euo pipefail

readonly AUDIT_DIR="/var/log/security_audit/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$AUDIT_DIR"

SCORE=0
TOTAL=0
FINDINGS=()

check_pass() {
    local name=$1
    echo "  ✓ $name"
    (( SCORE++ ))
    (( TOTAL++ ))
}

check_fail() {
    local name=$1
    local detail=${2:-}
    echo "  ✗ $name ${detail:+(${detail})}"
    FINDINGS+=("$name: $detail")
    (( TOTAL++ ))
}

check_warn() {
    local name=$1
    local detail=${2:-}
    echo "  ⚠ $name ${detail:+(${detail})}"
    FINDINGS+=("WARN: $name: $detail")
    (( SCORE++ ))
    (( TOTAL++ ))
}

audit_users() {
    echo "=== User Security ==="
    
    local empty_pw
    empty_pw=$(awk -F: '($2 == "" || $2 == "!") && $3 >= 1000 {print $1}' /etc/shadow 2>/dev/null | head -5)
    if [[ -z "$empty_pw" ]]; then
        check_pass "No users with empty passwords"
    else
        check_fail "Users with empty passwords" "$empty_pw"
    fi
    
    if grep -q "^PermitRootLogin no" /etc/ssh/sshd_config 2>/dev/null; then
        check_pass "SSH root login disabled"
    else
        check_fail "SSH root login enabled"
    fi
    
    local sudo_timeout
    sudo_timeout=$(grep "timestamp_timeout" /etc/sudoers 2>/dev/null | grep -oP '\d+' | head -1)
    if [[ -n "$sudo_timeout" ]] && (( sudo_timeout <= 5 )); then
        check_pass "Sudo timeout ≤5 min ($sudo_timeout)"
    else
        check_warn "Sudo timeout not restricted" "${sudo_timeout:-default}"
    fi
    
    local uid0_users
    uid0_users=$(awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd)
    if [[ -z "$uid0_users" ]]; then
        check_pass "No non-root UID 0 accounts"
    else
        check_fail "Non-root UID 0 accounts" "$uid0_users"
    fi
    
    if grep -q "^PASS_MAX_DAYS.*[1-9]" /etc/login.defs 2>/dev/null; then
        check_pass "Password max age configured"
    else
        check_warn "Password aging not configured"
    fi
}

audit_network() {
    echo ""
    echo "=== Network Security ==="
    
    local listening
    listening=$(ss -tlnp 2>/dev/null | awk 'NR>1 {print $4}' | \
        grep -oP ':\K\d+' | sort -un)
    echo "  Listening ports: $(echo "$listening" | tr '\n' ' ')"
    
    local ip_forward
    ip_forward=$(cat /proc/sys/net/ipv4/ip_forward)
    if [[ "$ip_forward" == "0" ]]; then
        check_pass "IP forwarding disabled"
    else
        check_warn "IP forwarding enabled"
    fi
    
    if command -v ufw &>/dev/null; then
        if ufw status 2>/dev/null | grep -q "Status: active"; then
            check_pass "UFW firewall active"
        else
            check_fail "UFW firewall not active"
        fi
    elif command -v firewall-cmd &>/dev/null; then
        if firewall-cmd --state 2>/dev/null | grep -q "running"; then
            check_pass "firewalld active"
        else
            check_fail "firewalld not running"
        fi
    else
        local rules
        rules=$(iptables -L INPUT 2>/dev/null | grep -c "ACCEPT\|DROP\|REJECT" || true)
        if (( rules > 2 )); then
            check_pass "iptables rules configured ($rules rules)"
        else
            check_fail "No firewall configured"
        fi
    fi
    
    if grep -qE "^Protocol[[:space:]]+2" /etc/ssh/sshd_config 2>/dev/null || \
       ! grep -qE "^Protocol[[:space:]]+1" /etc/ssh/sshd_config 2>/dev/null; then
        check_pass "SSH protocol 2 only"
    else
        check_fail "SSH protocol 1 allowed"
    fi
}

audit_filesystem() {
    echo ""
    echo "=== Filesystem Security ==="
    
    local ww_files
    ww_files=$(find / -xdev -type f -perm -o+w 2>/dev/null \
        ! -path "/proc/*" ! -path "/sys/*" ! -path "/dev/*" | head -5)
    if [[ -z "$ww_files" ]]; then
        check_pass "No world-writable files"
    else
        check_fail "World-writable files found" "$(echo "$ww_files" | head -3 | tr '\n' ' ')"
    fi
    
    local suid_files
    suid_files=$(find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null \
        ! -path "/proc/*" | sort)
    echo "  SUID/SGID files: $(echo "$suid_files" | wc -l)"
    
    if mount | grep -q "on /tmp.*noexec"; then
        check_pass "/tmp mounted noexec"
    else
        check_warn "/tmp not mounted noexec"
    fi
    
    if [[ $(stat -c "%a" /etc/cron.d 2>/dev/null) == "750" ]] || \
       [[ $(stat -c "%a" /etc/cron.d 2>/dev/null) == "700" ]]; then
        check_pass "Cron directory permissions secure"
    else
        check_warn "Cron directory permissions: $(stat -c '%a' /etc/cron.d 2>/dev/null)"
    fi
}

audit_services() {
    echo ""
    echo "=== Service Security ==="
    
    local unnecessary_services=(telnet ftp rsh rlogin rexec)
    
    for svc in "${unnecessary_services[@]}"; do
        if systemctl is-active --quiet "$svc" 2>/dev/null; then
            check_fail "Insecure service running" "$svc"
        else
            check_pass "Service not running: $svc"
        fi
    done
    
    if grep -q "^PasswordAuthentication no" /etc/ssh/sshd_config 2>/dev/null; then
        check_pass "SSH password auth disabled"
    else
        check_warn "SSH password authentication allowed"
    fi
    
    if grep -q "^PermitEmptyPasswords no" /etc/ssh/sshd_config 2>/dev/null; then
        check_pass "SSH empty passwords denied"
    else
        check_warn "SSH empty passwords not explicitly denied"
    fi
}

generate_report() {
    local report_file="${AUDIT_DIR}/security_report.txt"
    
    {
        echo "========================================"
        echo "  Security Audit Report"
        echo "  Host: $(hostname)"
        echo "  Date: $(date)"
        echo "========================================"
        echo ""
        echo "Score: ${SCORE}/${TOTAL} ($(( SCORE * 100 / TOTAL ))%)"
        echo ""
        
        if (( ${#FINDINGS[@]} > 0 )); then
            echo "FINDINGS:"
            for finding in "${FINDINGS[@]}"; do
                echo "  - $finding"
            done
        else
            echo "No critical findings."
        fi
    } | tee "$report_file"
    
    echo ""
    echo "Full report: $report_file"
}

audit_users
audit_network
audit_filesystem
audit_services
generate_report
```

---

## 39.2 SSH Hardening Automation

```bash
#!/bin/bash
# ssh_hardening.sh

harden_ssh() {
    local sshd_config="/etc/ssh/sshd_config"
    local backup="${sshd_config}.$(date +%Y%m%d).bak"
    
    cp "$sshd_config" "$backup"
    echo "Backed up: $backup"
    
    declare -A settings=(
        [Protocol]="2"
        [PermitRootLogin]="no"
        [PasswordAuthentication]="no"
        [PermitEmptyPasswords]="no"
        [ChallengeResponseAuthentication]="no"
        [UsePAM]="yes"
        [X11Forwarding]="no"
        [PrintMotd]="no"
        [MaxAuthTries]="3"
        [LoginGraceTime]="30"
        [ClientAliveInterval]="300"
        [ClientAliveCountMax]="2"
        [AllowAgentForwarding]="no"
        [AllowTcpForwarding]="no"
    )
    
    for key in "${!settings[@]}"; do
        local value="${settings[$key]}"
        
        if grep -qE "^${key}[[:space:]]" "$sshd_config"; then
            sed -i "s/^${key}[[:space:]].*/${key} ${value}/" "$sshd_config"
        elif grep -qE "^#${key}[[:space:]]" "$sshd_config"; then
            sed -i "s/^#${key}[[:space:]].*/${key} ${value}/" "$sshd_config"
        else
            echo "${key} ${value}" >> "$sshd_config"
        fi
    done
    
    if sshd -t 2>/dev/null; then
        echo "SSH config valid"
        systemctl reload sshd
        echo "SSH reloaded"
    else
        echo "SSH config invalid! Restoring backup..." >&2
        cp "$backup" "$sshd_config"
        return 1
    fi
}

setup_ssh_keys() {
    local username=$1
    local public_key=$2
    
    local home_dir
    home_dir=$(getent passwd "$username" | cut -d: -f6)
    
    local ssh_dir="${home_dir}/.ssh"
    local auth_keys="${ssh_dir}/authorized_keys"
    
    mkdir -p "$ssh_dir"
    chmod 700 "$ssh_dir"
    
    if ! grep -qF "$public_key" "$auth_keys" 2>/dev/null; then
        echo "$public_key" >> "$auth_keys"
    fi
    
    chmod 600 "$auth_keys"
    chown -R "${username}:${username}" "$ssh_dir"
    
    echo "SSH key configured for $username"
}
```

---

## 39.3 Log Monitoring & Intrusion Detection

```bash
#!/bin/bash
# log_monitor.sh

detect_brute_force() {
    local log_file=${1:-/var/log/auth.log}
    local threshold=${2:-10}
    local window=${3:-300}
    
    declare -A failed_attempts
    declare -A first_seen
    
    tail -F "$log_file" | while IFS= read -r line; do
        if [[ "$line" =~ Failed\ password.*from\ ([0-9.]+) ]]; then
            local ip="${BASH_REMATCH[1]}"
            local now=$SECONDS
            
            [[ -z "${first_seen[$ip]:-}" ]] && first_seen[$ip]=$now
            
            if (( now - first_seen[$ip] > window )); then
                failed_attempts[$ip]=0
                first_seen[$ip]=$now
            fi
            
            (( failed_attempts[$ip]++ ))
            
            if (( failed_attempts[$ip] >= threshold )); then
                echo "$(date): BRUTE FORCE from $ip (${failed_attempts[$ip]} attempts)"
                
                if command -v fail2ban-client &>/dev/null; then
                    fail2ban-client set sshd banip "$ip"
                else
                    iptables -A INPUT -s "$ip" -j DROP
                    echo "Blocked: $ip"
                fi
                
                failed_attempts[$ip]=0
            fi
        fi
        
        if [[ "$line" =~ Accepted\ password.*from\ ([0-9.]+)\ port ]]; then
            echo "$(date): LOGIN from ${BASH_REMATCH[1]}"
        fi
        
        if [[ "$line" =~ sudo:.*COMMAND ]]; then
            echo "$(date): SUDO: $line"
        fi
    done
}

create_baseline() {
    local paths=("$@")
    local baseline_file="/var/lib/security/baseline.sha256"
    
    mkdir -p "$(dirname "$baseline_file")"
    
    echo "Creating integrity baseline..."
    
    for path in "${paths[@]}"; do
        find "$path" -type f 2>/dev/null | \
            xargs sha256sum 2>/dev/null
    done > "$baseline_file"
    
    local count
    count=$(wc -l < "$baseline_file")
    echo "Baseline: $count files → $baseline_file"
}

check_integrity() {
    local baseline_file="/var/lib/security/baseline.sha256"
    
    if [[ ! -f "$baseline_file" ]]; then
        echo "No baseline found. Run create_baseline first." >&2
        return 1
    fi
    
    echo "=== File Integrity Check: $(date) ==="
    
    local changed=0 missing=0
    
    while IFS='  ' read -r expected_hash filepath; do
        if [[ ! -f "$filepath" ]]; then
            echo "MISSING: $filepath"
            (( missing++ ))
            continue
        fi
        
        local current_hash
        current_hash=$(sha256sum "$filepath" 2>/dev/null | cut -d' ' -f1)
        
        if [[ "$current_hash" != "$expected_hash" ]]; then
            echo "CHANGED: $filepath"
            (( changed++ ))
        fi
    done < "$baseline_file"
    
    echo ""
    echo "Summary: $changed changed, $missing missing"
    
    (( changed + missing > 0 )) && return 1 || return 0
}

failed_login_summary() {
    local hours=${1:-24}
    local log_file=${2:-/var/log/auth.log}
    
    echo "=== Failed Logins (last ${hours}h) ==="
    echo ""
    
    grep "Failed password" "$log_file" | \
        grep -oP 'from \K[0-9.]+' | \
        sort | uniq -c | sort -rn | \
        head -20 | \
        awk '{printf "  %-6d %s\n", $1, $2}'
}
```

---

## 39.4 Firewall Automation

```bash
#!/bin/bash
# firewall_manager.sh

apply_baseline_firewall() {
    local allow_ssh_from=${1:-any}
    local http_ports=(80 443)
    
    echo "Applying baseline firewall rules..."
    
    iptables -F
    iptables -X
    iptables -Z
    
    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -P OUTPUT ACCEPT
    
    iptables -A INPUT -i lo -j ACCEPT
    iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
    iptables -A INPUT -p icmp --icmp-type echo-request -m limit \
        --limit 1/s --limit-burst 5 -j ACCEPT
    
    if [[ "$allow_ssh_from" == "any" ]]; then
        iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    else
        iptables -A INPUT -p tcp --dport 22 -s "$allow_ssh_from" -j ACCEPT
    fi
    
    for port in "${http_ports[@]}"; do
        iptables -A INPUT -p tcp --dport "$port" -j ACCEPT
    done
    
    iptables -A INPUT -p tcp --syn -m limit --limit 25/s --limit-burst 50 -j ACCEPT
    iptables -A INPUT -p tcp --syn -j DROP
    
    iptables -A INPUT -m limit --limit 5/min -j LOG \
        --log-prefix "iptables-dropped: " --log-level 4
    
    echo "Firewall rules applied"
    iptables -L INPUT --line-numbers -n
}

block_ip() {
    local ip=$1
    local reason=${2:-"Manual block"}
    
    iptables -I INPUT -s "$ip" -j DROP
    iptables -I OUTPUT -d "$ip" -j DROP
    
    echo "$(date): Blocked $ip - $reason" >> /var/log/blocked_ips.log
    echo "Blocked: $ip"
}

unblock_ip() {
    local ip=$1
    iptables -D INPUT -s "$ip" -j DROP 2>/dev/null && echo "Unblocked INPUT: $ip"
    iptables -D OUTPUT -d "$ip" -j DROP 2>/dev/null && echo "Unblocked OUTPUT: $ip"
}

list_blocked() {
    echo "Blocked IPs:"
    iptables -L INPUT -n | grep DROP | awk '{print $4}' | \
        grep -v "^0\." | sort -u
}

save_rules() {
    if command -v iptables-save &>/dev/null; then
        iptables-save > /etc/iptables/rules.v4
        echo "Rules saved to /etc/iptables/rules.v4"
    fi
}
```

---

## 39.5 Secrets & Credential Management

```bash
#!/bin/bash
# secrets_manager.sh

vault_get_secret() {
    local path=$1
    local field=${2:-value}
    
    curl -s \
        -H "X-Vault-Token: ${VAULT_TOKEN:?}" \
        "${VAULT_ADDR:?}/v1/${path}" | \
        jq -r ".data.${field}"
}

vault_set_secret() {
    local path=$1
    local key=$2
    local value=$3
    
    curl -s \
        -H "X-Vault-Token: ${VAULT_TOKEN:?}" \
        -X POST \
        -d "{\"data\":{\"${key}\":\"${value}\"}}" \
        "${VAULT_ADDR}/v1/${path}"
}

encrypt_secret() {
    local plaintext=$1
    local recipient=$2
    echo "$plaintext" | gpg --encrypt --recipient "$recipient" \
        --trust-model always --armor 2>/dev/null
}

decrypt_secret() {
    local encrypted=$1
    echo "$encrypted" | gpg --decrypt --quiet 2>/dev/null
}

load_secrets() {
    local secrets_file=${1:-.secrets.gpg}
    if [[ ! -f "$secrets_file" ]]; then
        echo "Secrets file not found: $secrets_file" >&2
        return 1
    fi
    while IFS='=' read -r key value; do
        [[ "$key" =~ ^[A-Za-z_] ]] || continue
        export "${key}=${value}"
    done < <(gpg --decrypt --quiet "$secrets_file" 2>/dev/null)
}

scan_secrets() {
    local directory=${1:-.}
    local output_file="${2:-/tmp/secret_scan_$(date +%Y%m%d).txt}"
    
    echo "=== Secret Scanner ===" | tee "$output_file"
    echo "Scanning: $directory" | tee -a "$output_file"
    echo "" | tee -a "$output_file"
    
    declare -A patterns=(
        [AWS_KEY]='AKIA[0-9A-Z]{16}'
        [PRIVATE_KEY]='-----BEGIN.*PRIVATE KEY-----'
        [PASSWORD_VAR]='(password|passwd|pwd)\s*=\s*["\047][^"]+["\047]'
        [API_KEY]='(api_key|apikey|api-key)\s*=\s*["\047][^\047"]{8,}["\047]'
        [TOKEN]='(token|secret)\s*=\s*["\047][^\047"]{8,}["\047]'
        [DB_URL]='(mysql|postgresql|mongodb)://[^:]+:[^@]+@'
    )
    
    local found=0
    
    for pattern_name in "${!patterns[@]}"; do
        local pattern="${patterns[$pattern_name]}"
        
        local matches
        matches=$(grep -rIn -P "$pattern" "$directory" \
            --include="*.{sh,py,js,ts,go,rb,php,yaml,yml,env,conf}" \
            --exclude-dir=".git" \
            2>/dev/null | head -5)
        
        if [[ -n "$matches" ]]; then
            echo "[${pattern_name}] Potential secrets found:" | tee -a "$output_file"
            echo "$matches" | tee -a "$output_file"
            echo "" | tee -a "$output_file"
            (( found++ ))
        fi
    done
    
    if (( found == 0 )); then
        echo "No secrets detected." | tee -a "$output_file"
    else
        echo "Found $found potential secret types!" | tee -a "$output_file"
    fi
    
    echo ""
    echo "Report: $output_file"
    (( found > 0 )) && return 1 || return 0
}
```

---

## 39.6 Compliance Checking

```bash
#!/bin/bash
# compliance_check.sh - CIS benchmark checks

cis_level1_checks() {
    echo "=== CIS Level 1 Benchmark ==="
    echo ""
    
    local pass=0 fail=0 warn=0
    
    _pass() { echo "  [PASS] $1"; (( pass++ )); }
    _fail() { echo "  [FAIL] $1"; (( fail++ )); }
    _warn() { echo "  [WARN] $1"; (( warn++ )); }
    
    if ! modprobe -n -v cramfs 2>/dev/null | grep -q "^install /bin/true"; then
        _warn "cramfs not disabled in modprobe"
    else
        _pass "cramfs disabled"
    fi
    
    if command -v aide &>/dev/null || command -v aidecheck &>/dev/null; then
        _pass "AIDE installed"
    else
        _fail "AIDE not installed"
    fi
    
    if grep -qE "^\*\s+hard\s+core\s+0" /etc/security/limits.conf 2>/dev/null; then
        _pass "Core dumps restricted"
    else
        _warn "Core dumps not restricted in limits.conf"
    fi
    
    if systemctl is-active --quiet ntpd 2>/dev/null || \
       systemctl is-active --quiet chronyd 2>/dev/null || \
       systemctl is-active --quiet systemd-timesyncd 2>/dev/null; then
        _pass "Time synchronization active"
    else
        _fail "No time synchronization service"
    fi
    
    local ip_forward
    ip_forward=$(sysctl -n net.ipv4.ip_forward 2>/dev/null || echo "1")
    [[ "$ip_forward" == "0" ]] && _pass "IP forwarding disabled" || _fail "IP forwarding enabled"
    
    if command -v auditd &>/dev/null && systemctl is-active --quiet auditd; then
        _pass "auditd installed and running"
    else
        _fail "auditd not installed or not running"
    fi
    
    if grep -qE "^LogLevel (INFO|VERBOSE)" /etc/ssh/sshd_config 2>/dev/null; then
        _pass "SSH LogLevel is INFO or VERBOSE"
    else
        _warn "SSH LogLevel not set"
    fi
    
    local passwd_perm
    passwd_perm=$(stat -c "%a" /etc/passwd 2>/dev/null)
    [[ "$passwd_perm" == "644" ]] && _pass "/etc/passwd permissions 644" || \
        _fail "/etc/passwd permissions: $passwd_perm (expected 644)"
    
    local shadow_perm
    shadow_perm=$(stat -c "%a" /etc/shadow 2>/dev/null)
    if [[ "$shadow_perm" == "640" ]] || [[ "$shadow_perm" == "000" ]]; then
        _pass "/etc/shadow permissions secure ($shadow_perm)"
    else
        _fail "/etc/shadow permissions: $shadow_perm (expected 640 or 000)"
    fi
    
    echo ""
    echo "Results: $pass passed, $fail failed, $warn warnings"
    
    (( fail > 0 )) && return 1 || return 0
}
```

---

## 39.7 Exercises

### Exercise 1: Automated Security Baseline
สร้าง script ที่:
- Run all audit checks
- Compare against policy
- Generate HTML report
- Email results

### Exercise 2: Intrusion Detection System
สร้าง lightweight IDS:
- Monitor auth.log in real-time
- Pattern-based detection
- Auto-block with iptables
- Alert via Slack

### Exercise 3: Secrets Rotation
สร้าง automated rotation:
- Rotate database passwords
- Update application configs
- Reload services gracefully
- Log all rotations

---

## สรุป Part 39

✅ System security audit framework  
✅ SSH hardening automation  
✅ Brute force detection  
✅ File integrity monitoring  
✅ Firewall management  
✅ Secret scanning  
✅ GPG credential management  
✅ CIS benchmark checking  

---

**→ Part 40: Monitoring & Observability**
