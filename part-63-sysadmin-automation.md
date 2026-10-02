# Part 63: System Administration Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 63.1 User and Group Management

```bash
#!/bin/bash
# user_mgmt.sh - User and group management automation

# ─── User Operations ────────────────────────────────────────────────
user_create() {
    local username=$1
    local -A opts=(
        [shell]="/bin/bash"
        [home]="/home/$username"
        [groups]=""
        [comment]=""
        [password]=""
    )

    while [[ $# -gt 1 ]]; do
        case "$2" in
            --shell=*)   opts[shell]="${2#--shell=}" ;;
            --home=*)    opts[home]="${2#--home=}" ;;
            --groups=*)  opts[groups]="${2#--groups=}" ;;
            --comment=*) opts[comment]="${2#--comment=}" ;;
        esac
        shift
    done

    if id "$username" &>/dev/null; then
        echo "User $username already exists, skipping"
        return 0
    fi

    local useradd_args=(
        --create-home
        --home-dir "${opts[home]}"
        --shell "${opts[shell]}"
    )

    [[ -n "${opts[comment]}" ]] && useradd_args+=(--comment "${opts[comment]}")
    [[ -n "${opts[groups]}" ]]  && useradd_args+=(--groups "${opts[groups]}")

    useradd "${useradd_args[@]}" "$username"
    echo "Created user: $username"

    if [[ -n "${opts[password]}" ]]; then
        echo "${username}:${opts[password]}" | chpasswd
    fi
}

user_delete() {
    local username=$1 remove_home=${2:-false}

    id "$username" &>/dev/null || { echo "User $username not found"; return 1; }

    if $remove_home; then
        userdel --remove "$username"
    else
        userdel "$username"
    fi

    echo "Deleted user: $username"
}

user_lock() {
    local username=$1
    usermod --lock "$username"
    echo "Locked: $username"
}

user_unlock() {
    local username=$1
    usermod --unlock "$username"
    echo "Unlocked: $username"
}

user_add_ssh_key() {
    local username=$1 public_key=$2

    local home; home=$(getent passwd "$username" | cut -d: -f6)
    local ssh_dir="${home}/.ssh"
    local authorized_keys="${ssh_dir}/authorized_keys"

    mkdir -p "$ssh_dir"
    chmod 700 "$ssh_dir"
    chown "$username:$username" "$ssh_dir"

    if grep -qF "$public_key" "$authorized_keys" 2>/dev/null; then
        echo "SSH key already present for $username"
        return 0
    fi

    echo "$public_key" >> "$authorized_keys"
    chmod 600 "$authorized_keys"
    chown "$username:$username" "$authorized_keys"
    echo "Added SSH key for $username"
}

# ─── Group Operations ───────────────────────────────────────────────
group_create() {
    local groupname=$1 gid=${2:-}

    getent group "$groupname" &>/dev/null && { echo "Group $groupname exists"; return 0; }

    local args=(--force)
    [[ -n "$gid" ]] && args+=(--gid "$gid")

    groupadd "${args[@]}" "$groupname"
    echo "Created group: $groupname"
}

group_add_member() {
    local username=$1 groupname=$2

    usermod --append --groups "$groupname" "$username"
    echo "Added $username to $groupname"
}

# ─── Bulk User Provisioning ───────────────────────────────────────────
provision_users_from_file() {
    local file=$1

    while IFS=: read -r username groups shell comment; do
        [[ "$username" =~ ^# ]] && continue
        [[ -z "$username" ]] && continue

        user_create "$username" \
            --groups="$groups" \
            --shell="${shell:-/bin/bash}" \
            --comment="$comment"
    done < "$file"
}
```

---

## 63.2 Package Management

```bash
#!/bin/bash
# pkg_mgmt.sh - Cross-distro package management

# ─── Package Manager Detection ────────────────────────────────────────────
detect_pkg_manager() {
    if command -v apt-get &>/dev/null; then
        echo "apt"
    elif command -v dnf &>/dev/null; then
        echo "dnf"
    elif command -v yum &>/dev/null; then
        echo "yum"
    elif command -v zypper &>/dev/null; then
        echo "zypper"
    elif command -v apk &>/dev/null; then
        echo "apk"
    elif command -v pacman &>/dev/null; then
        echo "pacman"
    else
        echo "unknown"
        return 1
    fi
}

PKG_MANAGER=$(detect_pkg_manager)

# ─── Package Operations ───────────────────────────────────────────────
pkg_update_index() {
    case "$PKG_MANAGER" in
        apt)    apt-get update -qq ;;
        dnf)    dnf makecache -q ;;
        yum)    yum makecache -q ;;
        zypper) zypper refresh -q ;;
        apk)    apk update -q ;;
        pacman) pacman -Sy --noconfirm ;;
    esac
}

pkg_install() {
    local packages=("$@")

    local to_install=()
    for pkg in "${packages[@]}"; do
        if ! pkg_is_installed "$pkg"; then
            to_install+=("$pkg")
        fi
    done

    (( ${#to_install[@]} == 0 )) && { echo "All packages already installed"; return 0; }

    echo "Installing: ${to_install[*]}"

    case "$PKG_MANAGER" in
        apt)    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq "${to_install[@]}" ;;
        dnf)    dnf install -y -q "${to_install[@]}" ;;
        yum)    yum install -y -q "${to_install[@]}" ;;
        zypper) zypper install -y -q "${to_install[@]}" ;;
        apk)    apk add -q "${to_install[@]}" ;;
        pacman) pacman -S --noconfirm "${to_install[@]}" ;;
    esac
}

pkg_remove() {
    local packages=("$@")

    case "$PKG_MANAGER" in
        apt)    apt-get remove -y -qq "${packages[@]}" ;;
        dnf)    dnf remove -y -q "${packages[@]}" ;;
        yum)    yum remove -y -q "${packages[@]}" ;;
        apk)    apk del -q "${packages[@]}" ;;
        pacman) pacman -R --noconfirm "${packages[@]}" ;;
    esac
}

pkg_is_installed() {
    local pkg=$1

    case "$PKG_MANAGER" in
        apt)    dpkg -l "$pkg" 2>/dev/null | grep -q '^ii' ;;
        dnf|yum) rpm -q "$pkg" &>/dev/null ;;
        apk)    apk info -eq "$pkg" &>/dev/null ;;
        pacman) pacman -Q "$pkg" &>/dev/null ;;
        *)      command -v "$pkg" &>/dev/null ;;
    esac
}

pkg_upgrade_all() {
    case "$PKG_MANAGER" in
        apt)    DEBIAN_FRONTEND=noninteractive apt-get upgrade -y -qq ;;
        dnf)    dnf upgrade -y -q ;;
        yum)    yum update -y -q ;;
        apk)    apk upgrade -q ;;
        pacman) pacman -Su --noconfirm ;;
    esac
}

pkg_list_upgradeable() {
    case "$PKG_MANAGER" in
        apt)    apt list --upgradable 2>/dev/null | grep -v Listing ;;
        dnf)    dnf check-update -q ;;
        yum)    yum check-update -q ;;
        apk)    apk version -l '<' 2>/dev/null ;;
    esac
}
```

---

## 63.3 System Configuration

```bash
#!/bin/bash
# sysconfig.sh - System configuration management

# ─── Sysctl Management ───────────────────────────────────────────────
sysctl_set() {
    local key=$1 value=$2 persist=${3:-true}

    sysctl -w "${key}=${value}"

    if $persist; then
        local conf_file="/etc/sysctl.d/99-custom.conf"
        mkdir -p "$(dirname "$conf_file")"

        if grep -q "^${key}" "$conf_file" 2>/dev/null; then
            sed -i "s|^${key}.*|${key} = ${value}|" "$conf_file"
        else
            echo "${key} = ${value}" >> "$conf_file"
        fi
    fi

    echo "Set $key = $value"
}

apply_sysctl_profile() {
    local profile=$1

    declare -A profiles=(
        [web_server]="net.core.somaxconn=65535 net.ipv4.tcp_max_syn_backlog=65535 net.ipv4.tcp_fin_timeout=15"
        [database]="vm.swappiness=10 vm.dirty_ratio=15 vm.dirty_background_ratio=5"
        [security]="net.ipv4.conf.all.rp_filter=1 net.ipv4.conf.all.accept_redirects=0 net.ipv4.tcp_syncookies=1"
    )

    local settings="${profiles[$profile]:-}"
    [[ -z "$settings" ]] && { echo "Unknown profile: $profile"; return 1; }

    for setting in $settings; do
        local key="${setting%%=*}"
        local value="${setting#*=}"
        sysctl_set "$key" "$value"
    done
}

# ─── Limits Configuration ─────────────────────────────────────────────
set_ulimit() {
    local user=$1 limit_type=$2 value=$3

    local conf_file="/etc/security/limits.d/${user}.conf"

    printf '%s soft %s %s\n%s hard %s %s\n' \
        "$user" "$limit_type" "$value" \
        "$user" "$limit_type" "$value" >> "$conf_file"

    echo "Set $limit_type=$value for $user"
}

# ─── Hostname and Network Config ─────────────────────────────────────────
set_hostname() {
    local new_hostname=$1

    hostnamectl set-hostname "$new_hostname" 2>/dev/null || \
        echo "$new_hostname" > /etc/hostname

    local old_hostname; old_hostname=$(hostname)
    sed -i "s/\b${old_hostname}\b/$new_hostname/g" /etc/hosts

    echo "Hostname set to: $new_hostname"
}

configure_dns() {
    local -a nameservers=("$@")

    if systemctl is-active systemd-resolved &>/dev/null; then
        local conf="/etc/systemd/resolved.conf"
        local dns_line="DNS=$(IFS=' '; echo "${nameservers[*]}")"

        if grep -q "^DNS=" "$conf"; then
            sed -i "s|^DNS=.*|$dns_line|" "$conf"
        else
            echo "$dns_line" >> "$conf"
        fi
        systemctl restart systemd-resolved
    else
        local tmp; tmp=$(mktemp)
        grep -v '^nameserver' /etc/resolv.conf > "$tmp"
        for ns in "${nameservers[@]}"; do
            echo "nameserver $ns" >> "$tmp"
        done
        mv "$tmp" /etc/resolv.conf
    fi

    echo "DNS configured: ${nameservers[*]}"
}
```

---

## 63.4 Scheduled Jobs Management

```bash
#!/bin/bash
# cron_mgmt.sh - Cron and systemd timer management

# ─── Cron Operations ────────────────────────────────────────────────
cron_add() {
    local user=$1 schedule=$2 command=$3 comment=${4:-}
    local entry=""

    [[ -n "$comment" ]] && entry="# $comment\n"
    entry+="${schedule} ${command}"

    if crontab -l -u "$user" 2>/dev/null | grep -qF "$command"; then
        echo "Cron job already exists for $user: $command"
        return 0
    fi

    local tmp; tmp=$(mktemp)
    crontab -l -u "$user" 2>/dev/null > "$tmp" || true
    printf '%b\n' "$entry" >> "$tmp"
    crontab -u "$user" "$tmp"
    rm -f "$tmp"

    echo "Added cron job for $user"
}

cron_remove() {
    local user=$1 pattern=$2

    local tmp; tmp=$(mktemp)
    crontab -l -u "$user" 2>/dev/null | grep -v "$pattern" > "$tmp" || true
    crontab -u "$user" "$tmp"
    rm -f "$tmp"

    echo "Removed cron jobs matching: $pattern"
}

cron_list() {
    local user=${1:-$(whoami)}
    crontab -l -u "$user" 2>/dev/null || echo "No crontab for $user"
}

# ─── Systemd Timer ───────────────────────────────────────────────────
create_systemd_timer() {
    local name=$1 description=$2 exec_start=$3 on_calendar=$4
    local user=${5:-root}

    local unit_dir="/etc/systemd/system"
    [[ "$user" != "root" ]] && unit_dir="/home/${user}/.config/systemd/user"
    mkdir -p "$unit_dir"

    printf '[Unit]\nDescription=%s\nAfter=network.target\n\n[Service]\nType=oneshot\nExecStart=%s\nUser=%s\n' \
        "$description" "$exec_start" "$user" > "${unit_dir}/${name}.service"

    printf '[Unit]\nDescription=%s timer\n\n[Timer]\nOnCalendar=%s\nPersistent=true\n\n[Install]\nWantedBy=timers.target\n' \
        "$description" "$on_calendar" > "${unit_dir}/${name}.timer"

    systemctl daemon-reload
    systemctl enable --now "${name}.timer"
    echo "Created and enabled timer: ${name}.timer"
}

list_timers() {
    systemctl list-timers --all --no-pager 2>/dev/null || \
        echo "systemd not available"
}
```

---

## 63.5 System Health Dashboard

```bash
#!/bin/bash
# sys_health.sh - System health overview

system_health_report() {
    local output_format=${1:-text}

    local hostname; hostname=$(hostname -f)
    local uptime_info; uptime_info=$(uptime -p 2>/dev/null || uptime)
    local kernel; kernel=$(uname -r)
    local os_info; os_info=$(cat /etc/os-release 2>/dev/null | grep PRETTY_NAME | cut -d'"' -f2)

    local cpu_cores; cpu_cores=$(nproc)
    local load_avg; load_avg=$(awk '{print $1, $2, $3}' /proc/loadavg)

    local mem_total mem_avail mem_used mem_pct
    mem_total=$(awk '/^MemTotal/ {print $2}' /proc/meminfo)
    mem_avail=$(awk '/^MemAvailable/ {print $2}' /proc/meminfo)
    mem_used=$(( mem_total - mem_avail ))
    mem_pct=$(( mem_used * 100 / mem_total ))

    local disk_info; disk_info=$(df -h / | awk 'NR==2{print $3"/"$2, $5}')
    local failed_services; failed_services=$(systemctl --failed --no-legend 2>/dev/null | wc -l || echo 0)

    if [[ "$output_format" == "json" ]]; then
        jq -n \
            --arg host "$hostname" \
            --arg uptime "$uptime_info" \
            --arg kernel "$kernel" \
            --arg os "$os_info" \
            --argjson cpu_cores "$cpu_cores" \
            --arg load "$load_avg" \
            --argjson mem_pct "$mem_pct" \
            --arg disk "$disk_info" \
            --argjson failed_svcs "$failed_services" \
            '{hostname:$host,uptime:$uptime,kernel:$kernel,os:$os,
              cpu:{cores:$cpu_cores,load:$load},
              memory:{used_pct:$mem_pct},disk:$disk,failed_services:$failed_svcs}'
    else
        echo "========================================="
        printf "Host:    %s\n" "$hostname"
        printf "OS:      %s\n" "$os_info"
        printf "Kernel:  %s\n" "$kernel"
        printf "Uptime:  %s\n" "$uptime_info"
        echo "-----------------------------------------"
        printf "CPU:     %d cores  Load: %s\n" "$cpu_cores" "$load_avg"
        printf "Memory:  %d%% used\n" "$mem_pct"
        printf "Disk(/): %s\n" "$disk_info"
        printf "Failed services: %d\n" "$failed_services"
        echo "========================================="
    fi
}

watch_system() {
    local interval=${1:-5}

    while true; do
        clear
        system_health_report text
        echo ""
        echo "Refreshing every ${interval}s... (Ctrl+C to stop)"
        sleep "$interval"
    done
}
```

---

## 63.6 Exercises

### Exercise 1: User Provisioning System
สร้าง system ที่:
- Read users from LDAP/CSV
- Create accounts with SSH keys
- Enforce password policies
- Generate provisioning reports

### Exercise 2: Package Baseline Manager
สร้าง tool ที่:
- Capture installed packages baseline
- Detect unauthorized installs
- Auto-remove unwanted packages
- Update reporting

### Exercise 3: System Compliance Checker
สร้าง checker ที่:
- CIS benchmark checks
- Custom compliance rules
- HTML report generation
- Remediation scripts

---

## สรุป Part 63

✅ User/group CRUD with idempotent create, lock/unlock, SSH key injection
✅ Bulk provisioning from file
✅ Cross-distro package manager detection and unified install/remove/upgrade
✅ sysctl management with persistence and profile presets (web/db/security)
✅ ulimits configuration via /etc/security/limits.d
✅ Hostname and DNS configuration (systemd-resolved + resolv.conf)
✅ Cron job add/remove with dedup check
✅ Systemd service+timer unit generation and enable
✅ System health dashboard in text and JSON format

---

**→ Part 64: Networking and Service Discovery**
