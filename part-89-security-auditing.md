# Part 89: Shell Scripting for Security Auditing
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 89.1 System Hardening Audit

```bash
#!/bin/bash
# security_audit.sh - System security audit and hardening checks

set -euo pipefail

AUDIT_REPORT="${AUDIT_REPORT:-/tmp/security_audit_$(date +%Y%m%d).txt}"
declare -i AUDIT_PASS=0 AUDIT_WARN=0 AUDIT_FAIL=0

audit_log() {
    local level=$1; shift
    local msg="[$level] $*"
    echo "$msg" | tee -a "$AUDIT_REPORT"
    case "$level" in
        PASS) (( AUDIT_PASS++ )) ;;
        WARN) (( AUDIT_WARN++ )) ;;
        FAIL) (( AUDIT_FAIL++ )) ;;
    esac
}

# ─── User and Authentication Checks ──────────────────────────
audit_users() {
    echo "=== User and Authentication ===" | tee -a "$AUDIT_REPORT"

    # Root login via SSH
    if grep -qP '^\s*PermitRootLogin\s+yes' /etc/ssh/sshd_config 2>/dev/null; then
        audit_log "FAIL" "SSH PermitRootLogin is enabled"
    else
        audit_log "PASS" "SSH PermitRootLogin is disabled"
    fi

    # Password auth via SSH
    if grep -qP '^\s*PasswordAuthentication\s+yes' /etc/ssh/sshd_config 2>/dev/null; then
        audit_log "WARN" "SSH PasswordAuthentication is enabled"
    else
        audit_log "PASS" "SSH PasswordAuthentication is disabled"
    fi

    # Users with empty passwords
    local empty_pw; empty_pw=$(awk -F: '$2==""' /etc/shadow 2>/dev/null | cut -d: -f1)
    if [[ -n "$empty_pw" ]]; then
        audit_log "FAIL" "Users with empty passwords: $empty_pw"
    else
        audit_log "PASS" "No users with empty passwords"
    fi

    # Users with UID 0 (other than root)
    local uid0_users; uid0_users=$(awk -F: '$3==0 && $1!="root" {print $1}' /etc/passwd 2>/dev/null)
    if [[ -n "$uid0_users" ]]; then
        audit_log "FAIL" "Non-root users with UID 0: $uid0_users"
    else
        audit_log "PASS" "No non-root UID 0 users"
    fi

    # Password max age
    local max_age; max_age=$(awk -F: '$5!="" && $5+0 > 90 {print $1, $5}' /etc/shadow 2>/dev/null || true)
    [[ -n "$max_age" ]] && audit_log "WARN" "Password max age > 90 days for: $(echo $max_age | tr '\n' ', ')"
}

# ─── File System Checks ───────────────────────────────────────
audit_filesystem() {
    echo "=== File System Security ===" | tee -a "$AUDIT_REPORT"

    # World-writable files not in /tmp
    local ww_files; ww_files=$(find / -xdev -type f -perm -0002 \
        ! -path '/tmp/*' ! -path '/var/tmp/*' 2>/dev/null | head -10)
    if [[ -n "$ww_files" ]]; then
        audit_log "WARN" "World-writable files found (first 10):"
        echo "$ww_files" >> "$AUDIT_REPORT"
    else
        audit_log "PASS" "No suspicious world-writable files"
    fi

    # SUID/SGID files
    local suid_count; suid_count=$(find / -xdev \( -perm -4000 -o -perm -2000 \) \
        -type f 2>/dev/null | wc -l)
    if (( suid_count > 20 )); then
        audit_log "WARN" "Found $suid_count SUID/SGID files (expected <20)"
    else
        audit_log "PASS" "SUID/SGID file count OK ($suid_count)"
    fi

    # /etc/passwd and /etc/shadow permissions
    local passwd_perm; passwd_perm=$(stat -c '%a' /etc/passwd 2>/dev/null || echo 0)
    [[ "$passwd_perm" == "644" ]] && audit_log "PASS" "/etc/passwd permissions OK (644)" || \
        audit_log "FAIL" "/etc/passwd permissions: $passwd_perm (expected 644)"

    local shadow_perm; shadow_perm=$(stat -c '%a' /etc/shadow 2>/dev/null || echo 0)
    [[ "$shadow_perm" =~ ^(640|000)$ ]] && audit_log "PASS" "/etc/shadow permissions OK" || \
        audit_log "WARN" "/etc/shadow permissions: $shadow_perm (expected 640 or 000)"
}

# ─── Network Security Checks ─────────────────────────────────
audit_network() {
    echo "=== Network Security ===" | tee -a "$AUDIT_REPORT"

    # Open ports
    local open_ports; open_ports=$(ss -tlnp 2>/dev/null | awk 'NR>1{print $4}' | \
        grep -oP ':\K[0-9]+' | sort -nu | tr '\n' ' ')
    audit_log "INFO" "Open TCP ports: $open_ports"

    # IP forwarding
    local ipfwd; ipfwd=$(sysctl -n net.ipv4.ip_forward 2>/dev/null || echo 0)
    if [[ "$ipfwd" == "1" ]]; then
        audit_log "WARN" "IP forwarding is enabled"
    else
        audit_log "PASS" "IP forwarding is disabled"
    fi

    # Firewall active
    if systemctl is-active --quiet ufw 2>/dev/null || \
       systemctl is-active --quiet firewalld 2>/dev/null || \
       iptables -L -n 2>/dev/null | grep -q ACCEPT; then
        audit_log "PASS" "Firewall appears to be active"
    else
        audit_log "WARN" "No active firewall detected"
    fi
}

# ─── Services Audit ───────────────────────────────────────────
audit_services() {
    echo "=== Running Services ===" | tee -a "$AUDIT_REPORT"

    local risky_services=(telnet rsh rlogin tftp finger rexec)
    for svc in "${risky_services[@]}"; do
        if systemctl is-active --quiet "$svc" 2>/dev/null; then
            audit_log "FAIL" "Insecure service running: $svc"
        fi
    done

    # Check SSH version
    local ssh_ver; ssh_ver=$(ssh -V 2>&1 | head -1)
    audit_log "INFO" "SSH version: $ssh_ver"
}

audit_summary() {
    echo "" | tee -a "$AUDIT_REPORT"
    echo "=== Audit Summary ===" | tee -a "$AUDIT_REPORT"
    printf "PASS: %d  WARN: %d  FAIL: %d\n" "$AUDIT_PASS" "$AUDIT_WARN" "$AUDIT_FAIL" | \
        tee -a "$AUDIT_REPORT"
    echo "Full report: $AUDIT_REPORT"
    (( AUDIT_FAIL == 0 ))
}

run_full_audit() {
    echo "Security Audit - $(date)" > "$AUDIT_REPORT"
    echo "Host: $(hostname)" >> "$AUDIT_REPORT"
    echo "" >> "$AUDIT_REPORT"

    audit_users
    audit_filesystem
    audit_network
    audit_services
    audit_summary
}
```

---

## 89.2 Log Analysis for Security Events

```bash
#!/bin/bash
# security_log_analysis.sh - Security event detection in system logs

SECURE_LOG="${SECURE_LOG:-/var/log/auth.log}"
[[ -f /var/log/secure ]] && SECURE_LOG="/var/log/secure"

detect_failed_logins() {
    local log=${1:-$SECURE_LOG} threshold=${2:-5}
    echo "=== Failed Login Attempts ==="

    grep -a "Failed password\|authentication failure" "$log" 2>/dev/null | \
        grep -oP 'from \K[0-9.]+' | sort | uniq -c | sort -rn | \
        awk -v thresh="$threshold" '$1 >= thresh {
            printf "  %-15s %d failures\n", $2, $1
        }'
}

detect_successful_logins() {
    local log=${1:-$SECURE_LOG}
    echo "=== Successful Logins ==="

    grep -a "Accepted\|session opened" "$log" 2>/dev/null | \
        grep -oP 'for \K\S+' | sort | uniq -c | sort -rn | head -10 | \
        awk '{printf "  %-20s %d times\n", $2, $1}'
}

detect_sudo_usage() {
    local log=${1:-$SECURE_LOG}
    echo "=== Sudo Usage ==="

    grep -a "sudo:" "$log" 2>/dev/null | grep "COMMAND" | \
        awk '{for(i=1;i<=NF;i++) if($i~/COMMAND=/) {cmd=substr($0,index($0,$i)); break}; print $5, cmd}' | \
        head -20
}

detect_new_users() {
    local log=${1:-$SECURE_LOG}
    echo "=== New User Creation ==="

    grep -a "useradd\|adduser" "$log" 2>/dev/null | \
        grep -v "^#" | tail -20
}

detect_privilege_escalation() {
    local log=${1:-$SECURE_LOG}
    echo "=== Privilege Escalation Events ==="

    grep -aE "(su|sudo|pkexec|doas).*root" "$log" 2>/dev/null | \
        awk '{print $1, $2, $3, $0}' | tail -20
}

detect_port_scan() {
    local log="${1:-/var/log/syslog}"
    echo "=== Possible Port Scans ==="

    grep -a "SYN_RECV\|Nmap\|REJECT" "$log" 2>/dev/null | \
        grep -oP 'SRC=\K[0-9.]+' | sort | uniq -c | sort -rn | \
        awk '$1 > 100 {printf "  %-15s %d connection attempts\n", $2, $1}'
}

generate_security_report() {
    local log=${1:-$SECURE_LOG}
    echo "=== Security Log Report: $(date) ==="
    echo "Log file: $log"
    echo ""

    detect_failed_logins "$log"
    echo ""
    detect_successful_logins "$log"
    echo ""
    detect_sudo_usage "$log"
    echo ""
    detect_new_users "$log"
    echo ""
    detect_privilege_escalation "$log"
}
```

---

## 89.3 File Integrity Monitor

```bash
#!/bin/bash
# fim.sh - File Integrity Monitoring

FIM_DB="${FIM_DB:-/var/lib/fim/baseline.db}"
FIM_DIRS="${FIM_DIRS:-/etc /usr/bin /usr/sbin /bin /sbin}"
FIM_ALERT_CMD="${FIM_ALERT_CMD:-echo}"

mkdir -p "$(dirname "$FIM_DB")"

fim_init() {
    sqlite3 "$FIM_DB" "
        CREATE TABLE IF NOT EXISTS files (
            path TEXT PRIMARY KEY,
            sha256 TEXT NOT NULL,
            size INTEGER,
            mtime INTEGER,
            uid INTEGER,
            gid INTEGER,
            mode TEXT,
            last_checked INTEGER
        );
    "
    echo "FIM database initialized: $FIM_DB"
}

fim_baseline() {
    fim_init

    echo "Building baseline..."
    local count=0

    for dir in $FIM_DIRS; do
        [[ -d "$dir" ]] || continue
        find "$dir" -type f 2>/dev/null | while read -r file; do
            local sha256; sha256=$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1) || continue
            local stat_out; stat_out=$(stat -c '%s %Y %u %g %a' "$file" 2>/dev/null) || continue
            local size mtime uid gid mode
            read -r size mtime uid gid mode <<< "$stat_out"

            sqlite3 "$FIM_DB" "
                INSERT OR REPLACE INTO files
                    (path, sha256, size, mtime, uid, gid, mode, last_checked)
                VALUES
                    ('$(echo "$file" | sed "s/'/''/g")', '$sha256', $size, $mtime, $uid, $gid, '$mode', $(date +%s));
            "
            (( count++ ))
        done
    done

    echo "Baseline: $count files indexed"
}

fim_check() {
    local violations=0

    echo "Checking file integrity..."

    sqlite3 "$FIM_DB" "SELECT path, sha256 FROM files;" | \
    while IFS='|' read -r path expected_sha; do
        [[ -f "$path" ]] || {
            $FIM_ALERT_CMD "FIM ALERT: DELETED $path"
            continue
        }

        local actual_sha; actual_sha=$(sha256sum "$path" 2>/dev/null | cut -d' ' -f1) || continue

        if [[ "$actual_sha" != "$expected_sha" ]]; then
            $FIM_ALERT_CMD "FIM ALERT: MODIFIED $path"
        fi
    done

    # Check for new files not in baseline
    for dir in $FIM_DIRS; do
        [[ -d "$dir" ]] || continue
        find "$dir" -type f 2>/dev/null | while read -r file; do
            local in_db; in_db=$(sqlite3 "$FIM_DB" \
                "SELECT COUNT(*) FROM files WHERE path='$(echo "$file" | sed "s/'/''/g")';" 2>/dev/null || echo 0)
            if [[ "$in_db" == "0" ]]; then
                $FIM_ALERT_CMD "FIM ALERT: NEW FILE $file"
            fi
        done
    done

    echo "FIM check complete"
}
```

---

## 89.4 Exercises

### Exercise 1: CIS Benchmark Checks
Implement checks for CIS Linux Benchmark Level 1:
- Kernel parameters (sysctl)
- Mount options (noexec, nosuid)
- Cron file permissions
- Unused services disabled

### Exercise 2: Intrusion Detection
สร้าง simple IDS ที่:
- Monitor auth.log in real-time
- Alert after 3 failures from same IP
- Auto-block with iptables after 10 failures
- Daily summary report

### Exercise 3: SSL Certificate Scanner
สร้าง scanner ที่:
- Check certificates on a list of hosts
- Flag expiring within 30 days
- Flag weak ciphers (SSLv3, TLS 1.0)
- Export report to CSV

---

## สรุป Part 89

✅ audit_users: SSH PermitRootLogin, empty passwords, UID 0 users, password age
┅ audit_filesystem: world-writable files, SUID/SGID count, /etc/passwd /etc/shadow permissions
┅ audit_network: open ports, IP forwarding, firewall detection
┅ audit_services: risky service check (telnet, rsh, etc.)
┅ Security log analysis: failed logins, sudo usage, new user creation, privilege escalation
┅ detect_port_scan: high SYN_RECV counts from single IP
┅ FIM: SQLite-backed baseline, sha256 comparison, new file detection

---

**→ Part 90: Shell Script Mastery — Capstone and Advanced Topics**
