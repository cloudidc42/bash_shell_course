# Part 45: System Administration Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 45.1 User Management Automation

```bash
#!/bin/bash
# user_management.sh - Automated user lifecycle management

# ─── User Creation ─────────────────────────────────────────────
create_user() {
    local username=$1
    local full_name=${2:-$username}
    local groups=${3:-}
    local shell=${4:-/bin/bash}
    local home_base=${5:-/home}

    if id "$username" &>/dev/null; then
        echo "User already exists: $username" >&2
        return 1
    fi

    local useradd_args=(
        --create-home
        --home-dir "${home_base}/${username}"
        --shell "$shell"
        --comment "$full_name"
    )

    [[ -n "$groups" ]] && useradd_args+=(--groups "$groups")

    useradd "${useradd_args[@]}" "$username" || return 1

    mkdir -p "${home_base}/${username}/.ssh"
    chmod 700 "${home_base}/${username}/.ssh"
    chown "${username}:${username}" "${home_base}/${username}/.ssh"

    echo "Created user: $username (home: ${home_base}/${username})"
}

provision_user() {
    local username=$1
    local ssh_key=${2:-}
    local sudo_access=${3:-false}
    local password_disabled=${4:-true}

    create_user "$username" || return 1

    if [[ -n "$ssh_key" ]]; then
        local auth_keys="/home/${username}/.ssh/authorized_keys"
        echo "$ssh_key" >> "$auth_keys"
        chmod 600 "$auth_keys"
        chown "${username}:${username}" "$auth_keys"
        echo "Added SSH key for: $username"
    fi

    if $sudo_access; then
        echo "${username} ALL=(ALL) NOPASSWD:ALL" > "/etc/sudoers.d/${username}"
        chmod 440 "/etc/sudoers.d/${username}"
        echo "Granted sudo access: $username"
    fi

    if $password_disabled; then
        passwd -l "$username"
        echo "Locked password for: $username"
    fi
}

deactivate_user() {
    local username=$1
    local archive=${2:-true}

    if ! id "$username" &>/dev/null; then
        echo "User not found: $username" >&2
        return 1
    fi

    pkill -u "$username" 2>/dev/null || true

    usermod -s /sbin/nologin -L "$username"

    if $archive; then
        local archive_dir="/var/backups/users"
        mkdir -p "$archive_dir"
        local archive_file="${archive_dir}/${username}_$(date +%Y%m%d).tar.gz"
        tar -czf "$archive_file" -C /home "$username" 2>/dev/null
        echo "Archived home to: $archive_file"
    fi

    echo "Deactivated user: $username"
}

bulk_create_users() {
    local csv_file=$1

    while IFS=',' read -r username full_name groups ssh_key; do
        [[ "$username" == username ]] && continue
        [[ -z "$username" ]] && continue

        echo "Provisioning: $username"
        provision_user "$username" "$ssh_key" false false
    done < "$csv_file"
}

audit_users() {
    echo "=== User Audit Report ==="
    echo "Date: $(date)"
    echo ""

    echo "--- Users with login shell ---"
    getent passwd | awk -F: '$7 !~ /nologin|false/ && $3 >= 1000 {printf "%-15s %-30s %s\n", $1, $5, $7}'

    echo ""
    echo "--- Users with sudo access ---"
    getent group sudo wheel 2>/dev/null | awk -F: '{print $4}' | tr ',' '\n' | sort -u

    echo ""
    echo "--- Users with empty passwords ---"
    awk -F: '($2 == "" || $2 == "!") && $3 >= 1000 {print $1}' /etc/shadow 2>/dev/null

    echo ""
    echo "--- Recently logged in users ---"
    last -n 20 | awk 'NF > 1 && $1 != "wtmp" {print}'
}
```

---

## 45.2 Package Management Automation

```bash
#!/bin/bash
# package_manager.sh - Cross-distro package management

detect_pkg_manager() {
    if command -v apt-get &>/dev/null; then
        echo "apt"
    elif command -v yum &>/dev/null; then
        echo "yum"
    elif command -v dnf &>/dev/null; then
        echo "dnf"
    elif command -v zypper &>/dev/null; then
        echo "zypper"
    elif command -v pacman &>/dev/null; then
        echo "pacman"
    elif command -v brew &>/dev/null; then
        echo "brew"
    else
        echo "unknown"
        return 1
    fi
}

PKG_MANAGER=$(detect_pkg_manager)

pkg_install() {
    local packages=("$@")
    echo "Installing: ${packages[*]}"

    case "$PKG_MANAGER" in
        apt)    apt-get install -y "${packages[@]}" ;;
        yum)    yum install -y "${packages[@]}" ;;
        dnf)    dnf install -y "${packages[@]}" ;;
        zypper) zypper install -y "${packages[@]}" ;;
        pacman) pacman -S --noconfirm "${packages[@]}" ;;
        brew)   brew install "${packages[@]}" ;;
        *)      echo "Unknown package manager" >&2; return 1 ;;
    esac
}

pkg_remove() {
    local packages=("$@")
    case "$PKG_MANAGER" in
        apt)    apt-get remove -y "${packages[@]}" ;;
        yum)    yum remove -y "${packages[@]}" ;;
        dnf)    dnf remove -y "${packages[@]}" ;;
        zypper) zypper remove -y "${packages[@]}" ;;
        pacman) pacman -R --noconfirm "${packages[@]}" ;;
        brew)   brew uninstall "${packages[@]}" ;;
        *)      echo "Unknown package manager" >&2; return 1 ;;
    esac
}

pkg_update() {
    echo "Updating package lists..."
    case "$PKG_MANAGER" in
        apt)    apt-get update && apt-get upgrade -y ;;
        yum)    yum update -y ;;
        dnf)    dnf upgrade -y ;;
        zypper) zypper update -y ;;
        pacman) pacman -Syu --noconfirm ;;
        brew)   brew update && brew upgrade ;;
        *)      echo "Unknown package manager" >&2; return 1 ;;
    esac
}

pkg_is_installed() {
    local pkg=$1
    case "$PKG_MANAGER" in
        apt)    dpkg -l "$pkg" &>/dev/null ;;
        yum|dnf) rpm -q "$pkg" &>/dev/null ;;
        zypper) zypper se --installed-only "$pkg" &>/dev/null ;;
        pacman) pacman -Q "$pkg" &>/dev/null ;;
        brew)   brew list "$pkg" &>/dev/null ;;
        *)      command -v "$pkg" &>/dev/null ;;
    esac
}

ensure_packages() {
    local packages=("$@")
    local missing=()

    for pkg in "${packages[@]}"; do
        pkg_is_installed "$pkg" || missing+=("$pkg")
    done

    if (( ${#missing[@]} > 0 )); then
        echo "Installing missing packages: ${missing[*]}"
        pkg_install "${missing[@]}"
    else
        echo "All packages already installed"
    fi
}

pkg_list_installed() {
    case "$PKG_MANAGER" in
        apt)    dpkg --get-selections | grep -v deinstall | awk '{print $1}' ;;
        yum|dnf) rpm -qa --queryformat '%{name}\n' | sort ;;
        pacman) pacman -Qq ;;
        brew)   brew list ;;
        *)      echo "Not supported" >&2; return 1 ;;
    esac
}

pkg_security_update() {
    echo "Applying security updates only..."
    case "$PKG_MANAGER" in
        apt)
            apt-get update
            apt-get upgrade -y --only-upgrade \
                $(apt-get -s upgrade 2>/dev/null | grep "^Inst" | grep security | awk '{print $2}')
            ;;
        yum)    yum update -y --security ;;
        dnf)    dnf upgrade -y --security ;;
        *)      echo "Security-only update not supported, running full update"; pkg_update ;;
    esac
}
```

---

## 45.3 Service Management

```bash
#!/bin/bash
# service_manager.sh - Systemd service automation

svc_start()   { systemctl start   "$1"; }
svc_stop()    { systemctl stop    "$1"; }
svc_restart() { systemctl restart "$1"; }
svc_reload()  { systemctl reload  "$1" 2>/dev/null || systemctl restart "$1"; }
svc_enable()  { systemctl enable  "$1"; }
svc_disable() { systemctl disable "$1"; }
svc_status()  { systemctl is-active "$1" 2>/dev/null; }
svc_enabled() { systemctl is-enabled "$1" &>/dev/null; }

svc_ensure_running() {
    local service=$1
    local max_attempts=${2:-3}

    for (( i=1; i<=max_attempts; i++ )); do
        if [[ "$(svc_status "$service")" == "active" ]]; then
            echo "$service is running"
            return 0
        fi
        echo "Starting $service (attempt $i/${max_attempts})..."
        svc_start "$service"
        sleep 2
    done

    echo "Failed to start $service after $max_attempts attempts" >&2
    return 1
}

svc_wait_ready() {
    local service=$1
    local timeout=${2:-30}
    local check_cmd=${3:-}
    local start=$SECONDS

    while (( SECONDS - start < timeout )); do
        if [[ "$(svc_status "$service")" == "active" ]]; then
            if [[ -n "$check_cmd" ]]; then
                eval "$check_cmd" &>/dev/null && return 0
            else
                return 0
            fi
        fi
        sleep 1
    done

    echo "Timeout waiting for $service to be ready" >&2
    return 1
}

create_service() {
    local name=$1
    local exec_start=$2
    local description=${3:-$name}
    local user=${4:-root}
    local restart_policy=${5:-on-failure}

    cat > "/etc/systemd/system/${name}.service" << EOF
[Unit]
Description=${description}
After=network.target

[Service]
Type=simple
User=${user}
ExecStart=${exec_start}
Restart=${restart_policy}
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    echo "Created service: ${name}.service"
}

svc_health_report() {
    local services=("$@")
    local failed=0

    printf "%-30s %-10s %-10s\n" "SERVICE" "STATUS" "ENABLED"
    printf "%-30s %-10s %-10s\n" "-------" "------" "-------"

    for svc in "${services[@]}"; do
        local status enabled
        status=$(svc_status "$svc" 2>/dev/null || echo "unknown")
        svc_enabled "$svc" && enabled="yes" || enabled="no"

        printf "%-30s %-10s %-10s\n" "$svc" "$status" "$enabled"
        [[ "$status" != "active" ]] && (( failed++ ))
    done

    echo ""
    echo "Failed/inactive: $failed / ${#services[@]}"
    return $failed
}
```

---

## 45.4 Disk and Storage Management

```bash
#!/bin/bash
# storage_manager.sh

# ─── Disk Usage Analysis ───────────────────────────────────────
disk_usage_report() {
    local threshold=${1:-80}

    echo "=== Disk Usage Report ==="
    echo "Date: $(date)"
    echo ""
    echo "Filesystem Usage:"
    df -h | awk 'NR>1 {
        gsub(/%/, "", $5)
        printf "%-30s %6s / %-6s  (%s%%)", $6, $3, $2, $5
        if ($5+0 >= '"$threshold"') print "  [WARNING]"
        else print ""
    }'

    echo ""
    echo "Top 10 largest directories (/):"
    du -sh /* 2>/dev/null | sort -rh | head -10

    echo ""
    echo "Large files (>100MB):"
    find / -xdev -type f -size +100M -printf "%s\t%p\n" 2>/dev/null | \
        sort -rn | head -10 | awk '{printf "%.1f MB\t%s\n", $1/1048576, $2}'
}

disk_cleanup() {
    local target_dir=${1:-/var/log}
    local days_old=${2:-30}
    local dry_run=${3:-true}

    echo "Cleaning files older than $days_old days in $target_dir"

    if $dry_run; then
        echo "[DRY RUN] Files to be removed:"
        find "$target_dir" -type f -mtime +"$days_old" -print
    else
        local count
        count=$(find "$target_dir" -type f -mtime +"$days_old" | wc -l)
        find "$target_dir" -type f -mtime +"$days_old" -delete
        echo "Removed $count files"
    fi
}

log_rotation_setup() {
    local log_file=$1
    local max_size=${2:-100M}
    local keep_count=${3:-7}
    local compress=${4:-true}

    local logrotate_conf="/etc/logrotate.d/$(basename "$log_file" .log)"

    cat > "$logrotate_conf" << EOF
${log_file} {
    size ${max_size}
    rotate ${keep_count}
    missingok
    notifempty
    $(${compress} && echo compress || echo '')
    delaycompress
    copytruncate
    dateext
}
EOF
    echo "Created logrotate config: $logrotate_conf"
}

# ─── LVM Management ────────────────────────────────────────────
lvm_extend_volume() {
    local vg=$1
    local lv=$2
    local size=$3
    local filesystem=${4:-ext4}

    echo "Extending /dev/${vg}/${lv} by ${size}..."

    lvextend -L "+${size}" "/dev/${vg}/${lv}" || return 1

    case "$filesystem" in
        ext4|ext3) resize2fs "/dev/${vg}/${lv}" ;;
        xfs)       xfs_growfs "/dev/${vg}/${lv}" ;;
        btrfs)     btrfs filesystem resize max "/dev/${vg}/${lv}" ;;
        *)
            echo "Unknown filesystem: $filesystem" >&2
            return 1
            ;;
    esac

    echo "Extended successfully"
    df -h "/dev/${vg}/${lv}"
}

# ─── Inode Exhaustion Check ────────────────────────────────────
check_inodes() {
    local threshold=${1:-80}

    echo "=== Inode Usage Check ==="
    df -i | awk 'NR>1 {
        gsub(/%/, "", $5)
        if ($5+0 >= '"$threshold"')
            printf "WARNING: %s inode usage at %s%%\n", $6, $5
    }'
}
```

---

## 45.5 Backup Automation

```bash
#!/bin/bash
# backup_automation.sh - Enterprise backup system

BACKUP_ROOT="${BACKUP_ROOT:-/var/backups}"
BACKUP_RETENTION_DAYS="${BACKUP_RETENTION_DAYS:-30}"
BACKUP_LOG="${BACKUP_ROOT}/backup.log"

backup_log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$BACKUP_LOG"; }

create_backup() {
    local name=$1
    local source=$2
    local dest_dir="${BACKUP_ROOT}/${name}"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="${dest_dir}/${name}_${timestamp}.tar.gz"

    mkdir -p "$dest_dir"

    backup_log "Starting backup: $name ($source)"

    local start=$SECONDS
    tar --create \
        --gzip \
        --file="$backup_file" \
        --exclude='*.tmp' \
        --exclude='*.log' \
        --exclude='.git' \
        "$source" 2>/dev/null

    local exit_code=$?
    local duration=$(( SECONDS - start ))
    local size
    size=$(du -sh "$backup_file" 2>/dev/null | cut -f1)

    if (( exit_code == 0 )); then
        backup_log "SUCCESS: $backup_file ($size, ${duration}s)"
    else
        backup_log "FAILED: $name (exit=$exit_code)"
        rm -f "$backup_file"
        return 1
    fi

    echo "$backup_file"
}

incremental_backup() {
    local name=$1
    local source=$2
    local dest_dir="${BACKUP_ROOT}/${name}"
    local snapshot_file="${dest_dir}/.snapshot"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$dest_dir"

    local backup_file="${dest_dir}/${name}_incr_${timestamp}.tar.gz"

    if [[ -f "$snapshot_file" ]]; then
        tar --create \
            --gzip \
            --newer-mtime="$(cat "$snapshot_file")" \
            --file="$backup_file" \
            "$source" 2>/dev/null
    else
        create_backup "$name" "$source"
    fi

    date > "$snapshot_file"
    backup_log "Incremental backup: $backup_file"
}

verify_backup() {
    local backup_file=$1

    backup_log "Verifying: $backup_file"

    if tar -tzf "$backup_file" &>/dev/null; then
        local file_count
        file_count=$(tar -tzf "$backup_file" | wc -l)
        backup_log "Verified: $file_count files"
        return 0
    else
        backup_log "CORRUPT: $backup_file"
        return 1
    fi
}

cleanup_old_backups() {
    local name=$1
    local retention=${2:-$BACKUP_RETENTION_DAYS}
    local dest_dir="${BACKUP_ROOT}/${name}"

    local removed=0
    while IFS= read -r old_backup; do
        rm -f "$old_backup"
        backup_log "Removed old backup: $old_backup"
        (( removed++ ))
    done < <(find "$dest_dir" -name "*.tar.gz" -mtime +"$retention" -type f)

    [[ $removed -gt 0 ]] && backup_log "Cleaned $removed old backups from $name"
}

sync_to_remote() {
    local source=$1
    local remote=$2
    local ssh_key=${3:-}

    local rsync_args=(
        --archive
        --compress
        --delete
        --progress
        --stats
        --timeout=60
    )

    [[ -n "$ssh_key" ]] && rsync_args+=(--rsh="ssh -i $ssh_key -o StrictHostKeyChecking=no")

    backup_log "Syncing to remote: $remote"
    rsync "${rsync_args[@]}" "$source" "$remote" 2>&1 | tee -a "$BACKUP_LOG"
}

backup_mysql_all() {
    local backup_dir="${BACKUP_ROOT}/mysql"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$backup_dir"

    mysql --batch --skip-column-names -e "SHOW DATABASES;" | \
    grep -Ev "^(information_schema|performance_schema|sys)$" | \
    while IFS= read -r db; do
        local backup_file="${backup_dir}/${db}_${timestamp}.sql.gz"
        backup_log "Backing up MySQL: $db"
        mysqldump \
            --single-transaction \
            --routines \
            --triggers \
            --events \
            "$db" | gzip > "$backup_file"
        backup_log "Done: $backup_file ($(du -sh "$backup_file" | cut -f1))"
    done
}
```

---

## 45.6 Network Diagnostics

```bash
#!/bin/bash
# network_diagnostics.sh

network_health_check() {
    local targets=("${@:-8.8.8.8 1.1.1.1 google.com}")
    local issues=0

    echo "=== Network Health Check ==="

    echo ""
    echo "Interface Status:"
    ip -brief link show | awk '{printf "  %-15s %s\n", $1, $2}'

    echo ""
    echo "IP Addresses:"
    ip -brief addr show | awk 'NF > 2 {printf "  %-15s %s\n", $1, $3}'

    echo ""
    echo "Default Routes:"
    ip route show default | awk '{printf "  via %s dev %s\n", $3, $5}'

    echo ""
    echo "Connectivity Tests:"
    for target in "${targets[@]}"; do
        if ping -c 1 -W 2 "$target" &>/dev/null; then
            printf "  %-20s OK\n" "$target"
        else
            printf "  %-20s FAILED\n" "$target"
            (( issues++ ))
        fi
    done

    echo ""
    echo "DNS Resolution:"
    for domain in google.com github.com; do
        local ip
        ip=$(dig +short "$domain" 2>/dev/null | head -1)
        if [[ -n "$ip" ]]; then
            printf "  %-20s -> %s\n" "$domain" "$ip"
        else
            printf "  %-20s FAILED\n" "$domain"
            (( issues++ ))
        fi
    done

    echo ""
    echo "Open Ports (listening):"
    ss -tlnp | awk 'NR>1 {printf "  %-25s %s\n", $4, $6}' | head -20

    echo ""
    echo "Issues found: $issues"
    return $issues
}

bandwidth_test() {
    local server=${1:-iperf3.he.net}
    local port=${2:-5201}
    local duration=${3:-10}

    if command -v iperf3 &>/dev/null; then
        echo "Testing bandwidth to $server..."
        iperf3 -c "$server" -p "$port" -t "$duration"
    elif command -v curl &>/dev/null; then
        echo "Estimating download speed..."
        local start=$SECONDS
        local bytes
        bytes=$(curl -s "https://speed.hetzner.de/100MB.bin" | wc -c)
        local duration_actual=$(( SECONDS - start ))
        local mbps=$(( bytes / duration_actual / 125000 ))
        echo "Download: ~${mbps} Mbps"
    fi
}

port_scan_local() {
    local host=${1:-localhost}
    local start_port=${2:-1}
    local end_port=${3:-1024}
    local timeout=${4:-1}

    echo "Scanning $host ports $start_port-$end_port..."

    for (( port=start_port; port<=end_port; port++ )); do
        (echo >/dev/tcp/"$host"/"$port") 2>/dev/null && \
            echo "  OPEN: $port"
    done
}

trace_route() {
    local target=$1
    local max_hops=${2:-30}

    if command -v traceroute &>/dev/null; then
        traceroute -m "$max_hops" "$target"
    elif command -v tracepath &>/dev/null; then
        tracepath -m "$max_hops" "$target"
    else
        echo "No traceroute tool available" >&2
    fi
}
```

---

## 45.7 Exercises

### Exercise 1: Automated Server Provisioning
สร้าง script ที่:
- รับ server spec จาก YAML
- ติดตั้ง packages, users, services
- Verify provisioning สำเร็จ
- Generate provision report

### Exercise 2: Centralized Backup System
สร้าง backup system ที่:
- รับ config จาก file
- Backup หลาย sources
- Sync ไป S3/remote
- Email alert เมื่อ fail

### Exercise 3: Service Health Dashboard
สร้าง dashboard ที่:
- Monitor services (systemd, ports, processes)
- Auto-restart services ที่ down
- Send alerts via Slack/email
- Log history

---

## สรุป Part 45

✅ User lifecycle automation (create/provision/deactivate)
✅ Bulk user provisioning from CSV
✅ Cross-distro package management
✅ Service management with health checks
✅ Disk usage analysis and cleanup
✅ LVM extension automation
✅ Enterprise backup system (full/incremental/verify)
✅ Network diagnostics suite
✅ Remote backup sync with rsync

---

**→ Part 46: Container and Kubernetes Automation**
