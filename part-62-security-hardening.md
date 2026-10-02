# Part 62: Security Hardening and Secure Scripting
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 62.1 Secure Script Foundations

```bash
#!/bin/bash
# secure_base.sh - Secure scripting foundations

# ─── Strict Mode ──────────────────────────────────────────────────
set -euo pipefail
IFS=$'\n\t'

# ─── Secure Temp Files ──────────────────────────────────────────────
umask 077

secure_temp() {
    local suffix=${1:-.tmp}
    mktemp "/tmp/secure_XXXXXX${suffix}"
}

secure_temp_dir() {
    mktemp -d "/tmp/secure_XXXXXX"
}

# ─── Input Validation ───────────────────────────────────────────────
validate_integer() {
    local val=$1 name=${2:-value}
    [[ "$val" =~ ^-?[0-9]+$ ]] || { echo "ERROR: $name must be integer, got: $val" >&2; return 1; }
}

validate_positive_integer() {
    local val=$1 name=${2:-value}
    [[ "$val" =~ ^[1-9][0-9]*$ ]] || { echo "ERROR: $name must be positive integer" >&2; return 1; }
}

validate_alphanumeric() {
    local val=$1 name=${2:-value}
    [[ "$val" =~ ^[a-zA-Z0-9_-]+$ ]] || { echo "ERROR: $name contains invalid chars" >&2; return 1; }
}

validate_path() {
    local path=$1 name=${2:-path}
    [[ "$path" == *".."* ]] && { echo "ERROR: $name contains path traversal" >&2; return 1; }
    [[ "$path" =~ ^[a-zA-Z0-9_./:@-]+$ ]] || { echo "ERROR: $name contains unsafe chars" >&2; return 1; }
    return 0
}

validate_hostname() {
    local host=$1
    [[ "$host" =~ ^[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$ ]] || \
        { echo "ERROR: Invalid hostname: $host" >&2; return 1; }
}

validate_ip() {
    local ip=$1
    local IFS='.'
    read -r -a octets <<< "$ip"
    (( ${#octets[@]} == 4 )) || return 1
    for octet in "${octets[@]}"; do
        [[ "$octet" =~ ^[0-9]+$ ]] && (( octet >= 0 && octet <= 255 )) || return 1
    done
}

validate_email() {
    local email=$1
    [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

# ─── Safe Command Execution ───────────────────────────────────────────
safe_exec() {
    local cmd=("$@")
    "${cmd[@]}"
}

sanitize_filename() {
    local name=$1
    echo "${name//[^a-zA-Z0-9._-]/_}" | cut -c1-255
}
```

---

## 62.2 Secrets Management

```bash
#!/bin/bash
# secrets.sh - Secure secrets handling

# ─── Environment Variable Security ───────────────────────────────────────
secret_get() {
    local name=$1
    local val="${!name:-}"

    if [[ -z "$val" ]]; then
        echo "ERROR: Required secret $name is not set" >&2
        return 1
    fi

    echo "$val"
}

secret_from_file() {
    local file=$1 name=${2:-}

    if [[ ! -f "$file" ]]; then
        echo "ERROR: Secret file not found: $file" >&2
        return 1
    fi

    local perms
    perms=$(stat -c '%a' "$file" 2>/dev/null || stat -f '%A' "$file" 2>/dev/null)
    if [[ "$perms" != "600" && "$perms" != "400" ]]; then
        echo "WARNING: Secret file $file has permissive permissions: $perms" >&2
    fi

    if [[ -n "$name" ]]; then
        grep "^${name}=" "$file" | cut -d= -f2- | head -1
    else
        cat "$file"
    fi
}

mask_secret() {
    local secret=$1 text=$2
    echo "${text//$secret/****}"
}

# ─── Credential Store ─────────────────────────────────────────────────
CRED_DIR="${CRED_DIR:-${HOME}/.credentials}"

cred_store() {
    local name=$1 value=$2

    mkdir -p "$CRED_DIR"
    chmod 700 "$CRED_DIR"

    local file="${CRED_DIR}/${name}"
    printf '%s' "$value" > "$file"
    chmod 600 "$file"
}

cred_get() {
    local name=$1
    local file="${CRED_DIR}/${name}"

    [[ -f "$file" ]] || { echo "ERROR: Credential '$name' not found" >&2; return 1; }
    cat "$file"
}

cred_delete() {
    local name=$1
    local file="${CRED_DIR}/${name}"

    [[ -f "$file" ]] && shred -u "$file" 2>/dev/null || rm -f "$file"
}

# ─── Password Generation ───────────────────────────────────────────────
generate_password() {
    local length=${1:-32}
    local charset=${2:-'A-Za-z0-9!@#$%^&*'}

    LC_ALL=C tr -dc "$charset" < /dev/urandom | head -c "$length"
    echo
}

generate_token() {
    local bytes=${1:-32}
    head -c "$bytes" /dev/urandom | base64 | tr -d '\n/+=' | head -c $(( bytes * 4 / 3 ))
    echo
}

wipe_var() {
    local var_name=$1
    printf -v "$var_name" '%*s' "${#!var_name}" '' 2>/dev/null || true
    unset "$var_name"
}
```

---

## 62.3 File System Security

```bash
#!/bin/bash
# fs_security.sh - Filesystem security checks

# ─── Permission Auditing ──────────────────────────────────────────────
autit_world_writable() {
    local dir=${1:-/}

    find "$dir" -xdev \( -perm -002 \) -not -type l 2>/dev/null | while read -r f; do
        local owner; owner=$(stat -c '%U' "$f")
        printf "WORLD-WRITABLE: %s (owner: %s)\n" "$f" "$owner"
    done
}

audit_suid_files() {
    local dir=${1:-/}

    find "$dir" -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | while read -r f; do
        local owner; owner=$(stat -c '%U:%G %a' "$f")
        printf "SUID/SGID: %s (%s)\n" "$f" "$owner"
    done
}

audit_unowned_files() {
    local dir=${1:-/}

    find "$dir" -xdev \( -nouser -o -nogroup \) 2>/dev/null | while read -r f; do
        printf "UNOWNED: %s\n" "$f"
    done
}

check_sticky_dirs() {
    for dir in /tmp /var/tmp; do
        local perms; perms=$(stat -c '%a' "$dir" 2>/dev/null)
        if [[ "${perms: -1}" != "t" ]] && (( perms & 1000 )); then
            echo "WARNING: $dir missing sticky bit (perms: $perms)"
        fi
    done
}

# ─── File Integrity ─────────────────────────────────────────────────
create_integrity_db() {
    local dir=$1 db_file=${2:-/var/lib/integrity.db}

    find "$dir" -type f -exec sha256sum {} \; 2>/dev/null | sort > "$db_file"
    echo "Created integrity DB: $(wc -l < "$db_file") files"
}

verify_integrity() {
    local db_file=${1:-/var/lib/integrity.db}

    local ok=0 modified=0 missing=0

    while IFS='  ' read -r expected_hash file_path; do
        if [[ ! -f "$file_path" ]]; then
            echo "MISSING: $file_path"
            (( missing++ ))
            continue
        fi

        local actual_hash; actual_hash=$(sha256sum "$file_path" | cut -d' ' -f1)
        if [[ "$actual_hash" != "$expected_hash" ]]; then
            echo "MODIFIED: $file_path"
            (( modified++ ))
        else
            (( ok++ ))
        fi
    done < "$db_file"

    echo ""
    echo "Integrity Check: OK=$ok  MODIFIED=$modified  MISSING=$missing"
    (( modified == 0 && missing == 0 ))
}

# ─── Secure Delete ──────────────────────────────────────────────────
secure_delete() {
    local file=$1

    if command -v shred &>/dev/null; then
        shred -vzu --iterations=3 "$file" 2>/dev/null
    else
        local size; size=$(stat -c '%s' "$file" 2>/dev/null || echo 4096)
        dd if=/dev/urandom of="$file" bs=1 count="$size" conv=notrunc 2>/dev/null
        dd if=/dev/urandom of="$file" bs=1 count="$size" conv=notrunc 2>/dev/null
        dd if=/dev/zero   of="$file" bs=1 count="$size" conv=notrunc 2>/dev/null
        rm -f "$file"
    fi
}
```

---

## 62.4 Network Security

```bash
#!/bin/bash
# net_security.sh - Network security functions

# ─── Firewall Management ──────────────────────────────────────────────
fw_allow_port() {
    local port=$1 proto=${2:-tcp}
    validate_positive_integer "$port" "port" || return 1

    if command -v ufw &>/dev/null; then
        ufw allow "${port}/${proto}" comment "script-added"
    elif command -v firewall-cmd &>/dev/null; then
        firewall-cmd --permanent --add-port="${port}/${proto}"
        firewall-cmd --reload
    else
        iptables -A INPUT -p "$proto" --dport "$port" -j ACCEPT
    fi
}

fw_block_ip() {
    local ip=$1
    validate_ip "$ip" || return 1

    iptables -I INPUT 1 -s "$ip" -j DROP
    echo "Blocked: $ip"
}

fw_unblock_ip() {
    local ip=$1
    iptables -D INPUT -s "$ip" -j DROP 2>/dev/null || true
    echo "Unblocked: $ip"
}

fw_list_blocked() {
    iptables -L INPUT -n | grep DROP | awk '{print $4}'
}

# ─── SSL/TLS Verification ─────────────────────────────────────────────
verify_ssl_cert() {
    local host=$1 port=${2:-443}

    local cert_info
    cert_info=$(echo | openssl s_client -servername "$host" -connect "${host}:${port}" 2>/dev/null | \
        openssl x509 -noout -dates -subject -issuer 2>/dev/null)

    if [[ -z "$cert_info" ]]; then
        echo "ERROR: Cannot retrieve certificate for ${host}:${port}"
        return 1
    fi

    echo "$cert_info"

    local expiry_date
    expiry_date=$(echo "$cert_info" | grep 'notAfter' | cut -d= -f2)
    local expiry_epoch; expiry_epoch=$(date -d "$expiry_date" +%s 2>/dev/null || \
                                      date -j -f "%b %d %T %Y %Z" "$expiry_date" +%s 2>/dev/null)
    local now_epoch; now_epoch=$(date +%s)
    local days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

    echo "Days until expiry: $days_left"
    (( days_left < 30 )) && echo "WARNING: Certificate expires soon!" >&2

    return 0
}

check_tls_version() {
    local host=$1 port=${2:-443}

    for version in tls1 tls1_1 tls1_2 tls1_3; do
        if echo | openssl s_client "-${version}" -connect "${host}:${port}" 2>&1 | grep -q "Cipher"; then
            echo "SUPPORTED: $version"
        else
            echo "disabled:  $version"
        fi
    done
}
```

---

## 62.5 Audit Logging

```bash
#!/bin/bash
# audit_log.sh - Security audit logging

AUDIT_LOG="${AUDIT_LOG:-/var/log/script_audit.log}"
AUDIT_USER="${SUDO_USER:-${USER:-unknown}}"
AUDIT_HOST="${HOSTNAME:-$(hostname)}"

audit_log() {
    local event_type=$1 message=$2 result=${3:-ok}

    local ts; ts=$(date '+%Y-%m-%dT%H:%M:%S%z')
    local pid=$$
    local ppid=$PPID
    local caller="${BASH_SOURCE[1]:-unknown}:${BASH_LINENO[0]:-0}"

    printf '%s host=%s user=%s pid=%d ppid=%d type=%s result=%s caller=%s msg=%s\n' \
        "$ts" "$AUDIT_HOST" "$AUDIT_USER" "$pid" "$ppid" \
        "$event_type" "$result" "$caller" "$message" \
        >> "$AUDIT_LOG"
}

audit_command() {
    local cmd=("$@")
    local cmd_str="${cmd[*]}"

    audit_log "EXEC" "cmd=$cmd_str" "start"
    "${cmd[@]}"
    local exit_code=$?
    audit_log "EXEC" "cmd=$cmd_str" "exit=$exit_code"
    return $exit_code
}

audit_file_access() {
    local action=$1 file=$2
    audit_log "FILE_$action" "path=$file"
}

audit_auth() {
    local action=$1 target=${2:-} result=${3:-ok}
    audit_log "AUTH_$action" "target=$target" "$result"
}

verify_script_hash() {
    local script=$1 expected_hash=$2

    local actual_hash; actual_hash=$(sha256sum "$script" | cut -d' ' -f1)
    if [[ "$actual_hash" != "$expected_hash" ]]; then
        audit_log "INTEGRITY" "script=$script" "TAMPERED"
        echo "CRITICAL: Script $script has been tampered with!" >&2
        return 1
    fi
    return 0
}
```

---

## 62.6 Exercises

### Exercise 1: Input Sanitizer Library
สร้าง library ที่:
- Validate all common input types
- SQL injection prevention
- Shell injection prevention
- Path traversal prevention

### Exercise 2: Secrets Vault
สร้าง vault ที่:
- Encrypted storage with GPG
- TOTP-based access
- Audit trail
- Auto-expiry

### Exercise 3: Security Scanner
สร้าง scanner ที่:
- World-writable files
- SUID/SGID binaries
- Weak file permissions
- Exposed credentials in scripts

---

## สรุป Part 62

✅ Strict mode (set -euo pipefail) and secure umask (077)
✅ Input validation: integer, alphanumeric, path traversal, hostname, IP, email
✅ Secrets management: env vars, file-based (permission check), credential store
✅ Password/token generation from /dev/urandom, variable wiping
✅ Filesystem auditing: world-writable, SUID/SGID, unowned files, sticky dirs
✅ SHA256 integrity database create and verify
✅ Secure file deletion with shred / 3-pass overwrite fallback
✅ Firewall management (ufw/firewalld/iptables abstraction)
✅ SSL/TLS certificate verification, expiry days, TLS version probe
✅ Structured audit logging with timestamp, user, pid, caller site

---

**→ Part 63: System Administration Automation**
