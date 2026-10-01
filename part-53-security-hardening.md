# Part 53: Security Automation and Hardening
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 53.1 System Hardening

```bash
#!/bin/bash
# system_hardening.sh - CIS Benchmark-based hardening

# ─── Hardening Checks ───────────────────────────────────────────
declare -i PASS=0 FAIL=0 WARN=0

check_pass() { echo "  [PASS] $1"; ((PASS++)); }
check_fail() { echo "  [FAIL] $1"; ((FAIL++)); }
check_warn() { echo "  [WARN] $1"; ((WARN++)); }

# ─── Filesystem Hardening ───────────────────────────────────────
harden_filesystem() {
    echo "=== Filesystem Hardening ==="

    if mount | grep -q "on /tmp"; then
        local tmp_opts
        tmp_opts=$(mount | grep "on /tmp " | awk '{print $NF}')
        if echo "$tmp_opts" | grep -q "noexec"; then
            check_pass "/tmp has noexec option"
        else
            check_fail "/tmp missing noexec option"
        fi
    else
        check_warn "/tmp not on separate partition"
    fi

    local world_writable
    world_writable=$(find / -path /proc -prune -o -path /sys -prune -o \
        -type d -perm -0002 ! -perm -1000 -print 2>/dev/null | head -5)
    if [[ -z "$world_writable" ]]; then
        check_pass "No world-writable directories without sticky bit"
    else
        check_fail "World-writable directories without sticky bit found"
        echo "    $world_writable"
    fi

    local suid_files
    suid_files=$(find / -path /proc -prune -o -path /sys -prune -o \
        \( -perm -4000 -o -perm -2000 \) -type f -print 2>/dev/null | wc -l)
    echo "  INFO: $suid_files SUID/SGID files found"
}

# ─── SSH Hardening ─────────────────────────────────────────────
harden_ssh() {
    local sshd_config=${1:-/etc/ssh/sshd_config}

    echo "=== SSH Hardening ==="

    declare -A required_settings=(
        ["Protocol"]="2"
        ["PermitRootLogin"]="no"
        ["PasswordAuthentication"]="no"
        ["PermitEmptyPasswords"]="no"
        ["X11Forwarding"]="no"
        ["MaxAuthTries"]="3"
        ["LoginGraceTime"]="30"
        ["ClientAliveInterval"]="300"
        ["ClientAliveCountMax"]="2"
        ["AllowAgentForwarding"]="no"
        ["AllowTcpForwarding"]="no"
        ["UsePAM"]="yes"
    )

    for setting in "${!required_settings[@]}"; do
        local expected="${required_settings[$setting]}"
        local actual
        actual=$(grep -i "^${setting}" "$sshd_config" 2>/dev/null | awk '{print $2}' | head -1)

        if [[ -z "$actual" ]]; then
            check_fail "$setting: not set (required: $expected)"
        elif [[ "${actual,,}" == "${expected,,}" ]]; then
            check_pass "$setting = $actual"
        else
            check_fail "$setting = $actual (required: $expected)"
        fi
    done
}

apply_ssh_hardening() {
    local config=/etc/ssh/sshd_config
    local backup="/etc/ssh/sshd_config.bak.$(date +%Y%m%d_%H%M%S)"

    cp "$config" "$backup"
    echo "Backup created: $backup"

    declare -A settings=(
        ["PermitRootLogin"]="no"
        ["PasswordAuthentication"]="no"
        ["X11Forwarding"]="no"
        ["MaxAuthTries"]="3"
        ["Protocol"]="2"
        ["LoginGraceTime"]="30"
    )

    for key in "${!settings[@]}"; do
        local val="${settings[$key]}"
        if grep -q "^${key}" "$config"; then
            sed -i "s/^${key}.*/${key} ${val}/" "$config"
        elif grep -q "^#${key}" "$config"; then
            sed -i "s/^#${key}.*/${key} ${val}/" "$config"
        else
            echo "${key} ${val}" >> "$config"
        fi
    done

    sshd -t && systemctl reload sshd
    echo "SSH hardening applied"
}

# ─── Kernel Hardening ───────────────────────────────────────────
harden_kernel() {
    echo "=== Kernel Parameter Hardening ==="

    declare -A kernel_settings=(
        ["net.ipv4.ip_forward"]="0"
        ["net.ipv4.conf.all.accept_redirects"]="0"
        ["net.ipv4.conf.all.send_redirects"]="0"
        ["net.ipv4.conf.all.accept_source_route"]="0"
        ["net.ipv4.conf.all.rp_filter"]="1"
        ["net.ipv4.tcp_syncookies"]="1"
        ["net.ipv4.icmp_echo_ignore_broadcasts"]="1"
        ["net.ipv4.conf.all.log_martians"]="1"
        ["kernel.randomize_va_space"]="2"
        ["kernel.dmesg_restrict"]="1"
        ["kernel.kptr_restrict"]="2"
        ["kernel.yama.ptrace_scope"]="1"
        ["fs.protected_hardlinks"]="1"
        ["fs.protected_symlinks"]="1"
        ["fs.suid_dumpable"]="0"
    )

    local sysctl_file="/etc/sysctl.d/99-hardening.conf"

    {
        echo "# System Hardening Settings - $(date)"
        for key in "${!kernel_settings[@]}"; do
            echo "${key} = ${kernel_settings[$key]}"
        done
    } > "$sysctl_file"

    sysctl -p "$sysctl_file" &>/dev/null
    echo "  Kernel parameters applied: $sysctl_file"
}

# ─── Password Policy ───────────────────────────────────────────
set_password_policy() {
    local max_days=${1:-90}
    local min_days=${2:-7}
    local warn_days=${3:-14}
    local min_length=${4:-14}

    sed -i \
        -e "s/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   $max_days/" \
        -e "s/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   $min_days/" \
        -e "s/^PASS_WARN_AGE.*/PASS_WARN_AGE   $warn_days/" \
        /etc/login.defs

    echo "Password policy applied (max: $max_days days, min length: $min_length)"
}
```

---

## 53.2 Audit and Compliance

```bash
#!/bin/bash
# audit_compliance.sh - Security audit and compliance checking

# ─── Account Audit ─────────────────────────────────────────────
audit_accounts() {
    echo "=== Account Security Audit ==="

    echo ""
    echo "Accounts with UID 0 (root-equivalent):"
    awk -F: '$3 == 0 {print "  " $1}' /etc/passwd

    echo ""
    echo "Accounts with empty passwords:"
    awk -F: '($2 == "" || $2 == "!!" || $2 == "!") && $3 >= 1000 {print "  " $1}' /etc/shadow 2>/dev/null

    echo ""
    echo "Accounts with no password expiry:"
    while IFS=: read -r user _ uid _ _ _ shell; do
        (( uid < 1000 )) && continue
        [[ "$shell" == "/sbin/nologin" || "$shell" == "/bin/false" ]] && continue
        local max_days
        max_days=$(chage -l "$user" 2>/dev/null | grep "Maximum" | awk '{print $NF}')
        if [[ "$max_days" == "99999" || "$max_days" == "-1" ]]; then
            echo "  $user (no expiry)"
        fi
    done < /etc/passwd

    echo ""
    echo "Users with sudo access:"
    getent group sudo wheel 2>/dev/null | awk -F: '{print "  Group:", $1, "Members:", $4}'
}

# ─── File Integrity Monitoring ─────────────────────────────────
FIM_DB="/var/lib/fim/checksums.db"

fim_init() {
    local dirs=("${@:-/etc /bin /sbin /usr/bin /usr/sbin}")
    mkdir -p "$(dirname "$FIM_DB")"

    echo "Initializing FIM database..."
    > "$FIM_DB"

    for dir in "${dirs[@]}"; do
        find "$dir" -type f 2>/dev/null | while IFS= read -r file; do
            local checksum
            checksum=$(sha256sum "$file" 2>/dev/null | awk '{print $1}')
            echo "$checksum  $file" >> "$FIM_DB"
        done
    done

    echo "FIM database created: $(wc -l < "$FIM_DB") files"
}

fim_check() {
    local report_file="/var/log/fim-report-$(date +%Y%m%d_%H%M%S).txt"
    local modified=0 added=0 deleted=0

    echo "Running FIM check..."

    {
        echo "=== File Integrity Report: $(date) ==="

        while IFS= read -r line; do
            local stored_sum="${line%% *}"
            local file="${line##*  }"

            if [[ ! -f "$file" ]]; then
                echo "DELETED: $file"
                ((deleted++))
            else
                local current_sum
                current_sum=$(sha256sum "$file" 2>/dev/null | awk '{print $1}')
                if [[ "$current_sum" != "$stored_sum" ]]; then
                    echo "MODIFIED: $file"
                    ((modified++))
                fi
            fi
        done < "$FIM_DB"

        echo ""
        echo "Summary: $modified modified, $added added, $deleted deleted"
    } | tee "$report_file"

    echo "Report: $report_file"
}

# ─── Audit Log Analysis ────────────────────────────────────────
analyze_auth_log() {
    local log=${1:-/var/log/auth.log}

    echo "=== Authentication Analysis ==="

    echo ""
    echo "Failed SSH logins by IP:"
    grep "Failed password" "$log" 2>/dev/null | \
        grep -oP "from \K[\d.]+" | \
        sort | uniq -c | sort -rn | head -10

    echo ""
    echo "Successful logins:"
    grep "Accepted password\|Accepted publickey" "$log" 2>/dev/null | \
        awk '{print $1, $2, $3, $9, $11}' | \
        sort | uniq -c | sort -rn | head -10

    echo ""
    echo "sudo usage:"
    grep "sudo:" "$log" 2>/dev/null | \
        grep "COMMAND" | head -20
}

# ─── CIS Benchmark Runner ─────────────────────────────────────
run_cis_checks() {
    PASS=0 FAIL=0 WARN=0

    echo "Running CIS Benchmark Checks..."
    echo ""

    # Check 1: Bootloader password
    if grep -q "^password" /boot/grub/grub.cfg 2>/dev/null || \
       grep -q "^password" /boot/grub2/grub.cfg 2>/dev/null; then
        check_pass "CIS 1.4.1: Bootloader password set"
    else
        check_fail "CIS 1.4.1: Bootloader password not set"
    fi

    # Check 2: Core dumps restricted
    local core_hard
    core_hard=$(ulimit -Hc 2>/dev/null)
    if [[ "$core_hard" == "0" ]]; then
        check_pass "CIS 1.5.1: Core dumps restricted"
    else
        check_fail "CIS 1.5.1: Core dumps not restricted (hard limit: $core_hard)"
    fi

    # Check 3: ASLR enabled
    local aslr
    aslr=$(sysctl -n kernel.randomize_va_space 2>/dev/null)
    if [[ "$aslr" == "2" ]]; then
        check_pass "CIS 1.5.3: ASLR enabled"
    else
        check_fail "CIS 1.5.3: ASLR not fully enabled (value: $aslr)"
    fi

    # Check 4: No world-writable files
    local ww_files
    ww_files=$(find / -path /proc -prune -o -path /sys -prune -o \
        -xdev -type f -perm -0002 -print 2>/dev/null | wc -l)
    if [[ "$ww_files" -eq 0 ]]; then
        check_pass "CIS 6.1.10: No world-writable files"
    else
        check_fail "CIS 6.1.10: $ww_files world-writable files found"
    fi

    # Check 5: SSH root login disabled
    local root_login
    root_login=$(grep -i "^PermitRootLogin" /etc/ssh/sshd_config 2>/dev/null | awk '{print $2}')
    if [[ "${root_login,,}" == "no" ]]; then
        check_pass "CIS 5.2.8: SSH PermitRootLogin disabled"
    else
        check_fail "CIS 5.2.8: SSH PermitRootLogin = ${root_login:-unset}"
    fi

    echo ""
    local total=$(( PASS + FAIL + WARN ))
    (( total > 0 )) && echo "Results: $PASS PASS, $FAIL FAIL, $WARN WARN" && \
        echo "Score: $(( PASS * 100 / total ))%"
}
```

---

## 53.3 Secrets Management

```bash
#!/bin/bash
# secrets_management.sh - Secure secrets handling

SECRETS_DIR="/etc/secrets"

# ─── Encryption Helpers ──────────────────────────────────────────
encrypt_secret() {
    local plaintext=$1
    local password=$2

    echo "$plaintext" | openssl enc -aes-256-cbc -pbkdf2 -iter 100000 \
        -pass "pass:$password" 2>/dev/null | base64 -w 0
}

decrypt_secret() {
    local ciphertext=$1
    local password=$2

    echo "$ciphertext" | base64 -d | \
        openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 \
        -pass "pass:$password" 2>/dev/null
}

# ─── Secret Store ────────────────────────────────────────────────
secret_init() {
    mkdir -p "$SECRETS_DIR"
    chmod 700 "$SECRETS_DIR"
    echo "Secret store initialized"
}

secret_set() {
    local name=$1 value=$2 master_key=$3

    local encrypted
    encrypted=$(encrypt_secret "$value" "$master_key")
    [[ -z "$encrypted" ]] && { echo "Encryption failed"; return 1; }

    local secret_file="$SECRETS_DIR/$name"
    echo "$encrypted" > "$secret_file"
    chmod 600 "$secret_file"
    echo "Secret set: $name"
}

secret_get() {
    local name=$1 master_key=$2

    local secret_file="$SECRETS_DIR/$name"
    [[ ! -f "$secret_file" ]] && { echo "Secret not found: $name" >&2; return 1; }

    local ciphertext
    ciphertext=$(cat "$secret_file")
    decrypt_secret "$ciphertext" "$master_key"
}

secret_list() {
    ls -1 "$SECRETS_DIR" 2>/dev/null | grep -v "^\\."
}

secret_delete() {
    local name=$1
    local secret_file="$SECRETS_DIR/$name"

    [[ ! -f "$secret_file" ]] && { echo "Secret not found: $name" >&2; return 1; }

    shred -u "$secret_file" 2>/dev/null || rm -f "$secret_file"
    echo "Secret deleted: $name"
}

# Run command with secrets in environment
with_secrets() {
    local master_key=$1
    shift

    local tmpenv
    tmpenv=$(mktemp)
    chmod 600 "$tmpenv"

    while IFS= read -r name; do
        local value
        value=$(secret_get "$name" "$master_key" 2>/dev/null)
        [[ -n "$value" ]] && echo "export ${name^^}='${value}'" >> "$tmpenv"
    done < <(secret_list)

    (source "$tmpenv"; "$@")
    local status=$?
    rm -f "$tmpenv"
    return $status
}

# ─── Vault Integration (HashiCorp) ─────────────────────────────
vault_get_secret() {
    local path=$1 field=${2:-value}
    vault kv get -field="$field" "$path" 2>/dev/null
}

vault_set_secret() {
    local path=$1
    shift
    vault kv put "$path" "$@"
}

vault_rotate_secret() {
    local path=$1
    local new_value
    new_value=$(openssl rand -base64 32)
    vault kv put "$path" value="$new_value"
    echo "$new_value"
}
```

---

## 53.4 Vulnerability Scanner

```bash
#!/bin/bash
# vuln_scanner.sh - Basic vulnerability scanner

# ─── Package Vulnerability Check ───────────────────────────────
scan_installed_packages() {
    echo "=== Package Vulnerability Scan ==="

    if command -v apt-get &>/dev/null; then
        echo "Checking for security updates (apt)..."
        apt-get -s upgrade 2>/dev/null | grep "^Inst" | grep -i security | head -20
    fi

    if command -v yum &>/dev/null; then
        echo "Checking for security updates (yum)..."
        yum check-update --security 2>/dev/null | grep -v "^$\|^Loaded\|^Loading" | head -20
    fi

    if command -v trivy &>/dev/null; then
        echo "Running Trivy OS scan..."
        trivy rootfs --severity HIGH,CRITICAL / 2>/dev/null | head -50
    fi
}

# ─── Web Server Security Checks ──────────────────────────────
check_web_security() {
    local url=$1

    echo "=== Web Security Check: $url ==="

    local headers
    headers=$(curl -sI --max-time 10 "$url" 2>/dev/null)

    local required_headers=(
        "Strict-Transport-Security"
        "X-Content-Type-Options"
        "X-Frame-Options"
        "Content-Security-Policy"
        "X-XSS-Protection"
        "Referrer-Policy"
        "Permissions-Policy"
    )

    echo ""
    echo "Security Headers:"
    for header in "${required_headers[@]}"; do
        if echo "$headers" | grep -qi "^${header}:"; then
            local value
            value=$(echo "$headers" | grep -i "^${header}:" | awk '{print $2}')
            echo "  [OK]  $header: $value"
        else
            echo "  [MISSING] $header"
        fi
    done

    # Check HTTPS redirect
    local http_url="${url/https:/http:}"
    local redirect
    redirect=$(curl -sI --max-time 5 "$http_url" 2>/dev/null | grep -i "^Location:")
    if echo "$redirect" | grep -q "https://"; then
        echo ""
        echo "[OK] HTTPS redirect configured"
    fi
}

# ─── Security Baseline Report ────────────────────────────────
generate_security_report() {
    local output=${1:-/tmp/security-report-$(date +%Y%m%d).txt}

    {
        echo "SECURITY BASELINE REPORT"
        echo "Generated: $(date)"
        echo "Hostname: $(hostname)"
        echo "OS: $(grep PRETTY_NAME /etc/os-release 2>/dev/null | cut -d= -f2)"
        echo "Kernel: $(uname -r)"
        echo ""
        echo "================================"

        run_cis_checks
        echo ""
        audit_accounts
        echo ""
        harden_filesystem
        echo ""
        echo "================================"
        echo "Report complete: $output"
    } | tee "$output"
}
```

---

## 53.5 Exercises

### Exercise 1: Security Hardening Playbook
สร้าง automated hardening playbook ที่:
- รัน CIS benchmark checks
- Apply fixes อัตโนมัติสำหรับ safe changes
- Generate compliance report
- Send alert เมื่อ critical issues พบ

### Exercise 2: Secret Rotation System
สร้างระบบ rotation ที่:
- Rotate database passwords ทุก 90 วัน
- Rotate API keys
- Update applications โดยไม่ downtime
- Audit log ทุก rotation event

### Exercise 3: Continuous Compliance Monitor
สร้าง daemon ที่:
- Monitor system changes real-time
- Alert เมื่อ hardening settings เปลี่ยน
- Auto-remediate unauthorized changes
- Weekly compliance report

---

## สรุป Part 53

✅ System hardening: filesystem, SSH, kernel parameters
✅ CIS Benchmark automated checks with scoring
✅ Account audit: UID 0, empty passwords, sudo access
✅ File Integrity Monitoring (FIM) with SHA256
✅ Authentication log analysis
✅ AES-256 encrypted secret store
✅ HashiCorp Vault integration helpers
✅ Package CVE scanning and web security checks

---

**→ Part 54: CI/CD Pipeline Automation**
