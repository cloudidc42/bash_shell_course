# Part 35: System Automation & Configuration Management
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 35.1 Systemd Service Management

```bash
#!/bin/bash
# systemd_manager.sh - Manage systemd services

# ─── Service Status Check ────────────────────────────────────
check_services() {
    local services=("$@")
    
    echo "=== Service Status ==="
    for svc in "${services[@]}"; do
        local status
        status=$(systemctl is-active "$svc" 2>/dev/null)
        local enabled
        enabled=$(systemctl is-enabled "$svc" 2>/dev/null)
        
        case "$status" in
            active)   printf "  ✓ %-30s active/%-10s\n" "$svc" "$enabled" ;;
            inactive) printf "  - %-30s inactive/%-10s\n" "$svc" "$enabled" ;;
            failed)   printf "  ✗ %-30s FAILED/%-10s\n" "$svc" "$enabled" ;;
            *)        printf "  ? %-30s unknown/%-10s\n" "$svc" "$enabled" ;;
        esac
    done
}

# ─── Create Systemd Unit File ────────────────────────────────
create_service() {
    local name=$1
    local description=$2
    local exec_start=$3
    local user=${4:-root}
    local restart_policy=${5:-on-failure}
    
    local unit_file="/etc/systemd/system/${name}.service"
    
    cat > "$unit_file" << EOF
[Unit]
Description=${description}
After=network.target
Wants=network.target

[Service]
Type=simple
User=${user}
ExecStart=${exec_start}
Restart=${restart_policy}
RestartSec=5s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=${name}

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/var/lib/${name}

[Install]
WantedBy=multi-user.target
EOF
    
    systemctl daemon-reload
    echo "Created service: $unit_file"
}

# ─── Create Timer (Cron replacement) ──────────────────────────
create_timer() {
    local name=$1
    local on_calendar=$2
    local service_name=${3:-$name}
    
    cat > "/etc/systemd/system/${name}.timer" << EOF
[Unit]
Description=Timer for ${name}

[Timer]
OnCalendar=${on_calendar}
Persistent=true
RandomizedDelaySec=60

[Install]
WantedBy=timers.target
EOF
    
    systemctl daemon-reload
    systemctl enable --now "${name}.timer"
    echo "Timer created and enabled: $name (${on_calendar})"
}

# ─── Service Watchdog ────────────────────────────────────────
service_watchdog() {
    local services=("$@")
    local check_interval=60
    
    while true; do
        for svc in "${services[@]}"; do
            if ! systemctl is-active --quiet "$svc"; then
                echo "$(date): $svc is not running, restarting..."
                systemctl restart "$svc"
                
                sleep 5
                if systemctl is-active --quiet "$svc"; then
                    echo "$(date): $svc restarted successfully"
                else
                    echo "$(date): Failed to restart $svc!" >&2
                fi
            fi
        done
        
        sleep "$check_interval"
    done
}

# ─── Journal Log Viewer ───────────────────────────────────────
show_service_logs() {
    local service=$1
    local lines=${2:-50}
    local since=${3:-"1 hour ago"}
    
    journalctl -u "$service" \
        --since "$since" \
        -n "$lines" \
        --no-pager \
        -o short-precise
}
```

---

## 35.2 Configuration File Management

```bash
#!/bin/bash
# config_manager.sh - Manage configuration files

# ─── INI File Parser ───────────────────────────────────────────
parse_ini() {
    local file=$1
    local -n result=$2
    
    local section=""
    
    while IFS= read -r line; do
        [[ "$line" =~ ^[[:space:]]*(#|;|$) ]] && continue
        
        if [[ "$line" =~ ^\[([^\]]+)\] ]]; then
            section="${BASH_REMATCH[1]}"
            continue
        fi
        
        if [[ "$line" =~ ^[[:space:]]*([^=]+)[[:space:]]*=[[:space:]]*(.*) ]]; then
            local key="${BASH_REMATCH[1]// /}"
            local value="${BASH_REMATCH[2]}"
            value="${value%%[[:space:]]*#*}"
            value="${value%%[[:space:]]*;*}"
            value="${value%\"${value##*[! $'\t']}\"}"            
            local full_key="${section:+${section}.}${key}"
            result["$full_key"]="$value"
        fi
    done < "$file"
}

# ─── Template Renderer ─────────────────────────────────────────
render_config() {
    local template=$1
    local output=$2
    local -n vars=$3
    
    local sed_script=""
    for key in "${!vars[@]}"; do
        local value="${vars[$key]}"
        value="${value//\//\\/}"
        value="${value//&/\\&}"
        sed_script+="s/\\${${key}}/${value}/g;"
        sed_script+="s/{{${key}}}/${value}/g;"
    done
    
    sed "$sed_script" "$template" > "$output"
    echo "Rendered: $template → $output"
}

# ─── Config Diff and Apply ──────────────────────────────────
apply_config_changes() {
    local current=$1
    local desired=$2
    
    echo "=== Config Changes ==="
    diff --unified=2 "$current" "$desired" | \
        grep -E "^[+-]" | \
        grep -v "^---\|^+++" | \
        head -50
    
    echo ""
    read -r -p "Apply changes? [y/N] " confirm
    
    if [[ "$confirm" =~ ^[Yy]$ ]]; then
        cp "$current" "${current}.$(date +%Y%m%d_%H%M%S).bak"
        cp "$desired" "$current"
        echo "Applied."
    fi
}
```

---

## 35.3 Package Management Automation

```bash
#!/bin/bash
# package_manager.sh - Automated package management

detect_pkg_manager() {
    if command -v apt-get &>/dev/null; then
        echo "apt"
    elif command -v dnf &>/dev/null; then
        echo "dnf"
    elif command -v yum &>/dev/null; then
        echo "yum"
    elif command -v pacman &>/dev/null; then
        echo "pacman"
    elif command -v zypper &>/dev/null; then
        echo "zypper"
    else
        echo "unknown"
    fi
}

install_package() {
    local package=$1
    local pkg_manager
    pkg_manager=$(detect_pkg_manager)
    
    echo "Installing $package via $pkg_manager..."
    
    case "$pkg_manager" in
        apt)    apt-get install -y -q "$package" ;;
        dnf)    dnf install -y "$package" ;;
        yum)    yum install -y "$package" ;;
        pacman) pacman -S --noconfirm "$package" ;;
        zypper) zypper install -y "$package" ;;
        *)      echo "Unknown package manager" >&2; return 1 ;;
    esac
}

ensure_tools() {
    local tools=("$@")
    local missing=()
    
    for tool in "${tools[@]}"; do
        if ! command -v "$tool" &>/dev/null; then
            missing+=("$tool")
        fi
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "Missing tools: ${missing[*]}"
        echo "Installing..."
        for tool in "${missing[@]}"; do
            install_package "$tool" || echo "Failed to install: $tool"
        done
    fi
}

auto_update() {
    local log_file="/var/log/auto_update.log"
    local pkg_manager
    pkg_manager=$(detect_pkg_manager)
    
    {
        echo "=== Auto Update: $(date) ==="
        
        case "$pkg_manager" in
            apt)
                apt-get update -q
                apt-get upgrade -y -q
                apt-get autoremove -y -q
                apt-get autoclean -q
                ;;
            dnf)
                dnf update -y
                dnf autoremove -y
                ;;
            yum)
                yum update -y
                ;;
        esac
        
        echo "Update complete: $(date)"
    } >> "$log_file" 2>&1
}
```

---

## 35.4 User & Permission Management

```bash
#!/bin/bash
# user_management.sh

create_users_from_csv() {
    local csv_file=$1
    
    while IFS=, read -r username fullname group shell; do
        [[ "$username" == "username" ]] && continue
        [[ -z "$username" ]] && continue
        
        shell="${shell:-/bin/bash}"
        
        if id "$username" &>/dev/null; then
            echo "User already exists: $username"
            continue
        fi
        
        if [[ -n "$group" ]] && ! getent group "$group" &>/dev/null; then
            groupadd "$group"
        fi
        
        useradd \
            --create-home \
            --comment "$fullname" \
            --shell "$shell" \
            ${group:+--gid "$group"} \
            "$username"
        
        local password
        password=$(openssl rand -base64 12)
        echo "${username}:${password}" | chpasswd
        chage -d 0 "$username"
        
        echo "Created user: $username (temp pass: $password)"
    done < "$csv_file"
}

audit_sudo_access() {
    echo "=== Sudo Access Audit ==="
    echo ""
    
    echo "Users with sudo access:"
    getent group sudo 2>/dev/null | cut -d: -f4 | tr , '\n' | \
        while read -r user; do
            echo "  $user (group: sudo)"
        done
    
    getent group wheel 2>/dev/null | cut -d: -f4 | tr , '\n' | \
        while read -r user; do
            echo "  $user (group: wheel)"
        done
    
    echo ""
    echo "Direct sudoers entries:"
    grep -v "^#\|^$\|^Defaults\|^%\|^root" /etc/sudoers 2>/dev/null | \
        grep -v "^$" | sed 's/^/  /'
}
```

---

## 35.5 Cron & Scheduled Task Management

```bash
#!/bin/bash
# cron_manager.sh

add_cron_job() {
    local schedule=$1
    local command=$2
    local user=${3:-root}
    local comment=${4:-""}
    
    local field_count
    field_count=$(echo "$schedule" | awk '{print NF}')
    if (( field_count != 5 )); then
        echo "Invalid cron schedule: $schedule" >&2
        return 1
    fi
    
    local current_cron
    current_cron=$(crontab -u "$user" -l 2>/dev/null || true)
    
    if echo "$current_cron" | grep -qF "$command"; then
        echo "Job already exists"
        return 0
    fi
    
    {
        echo "$current_cron"
        [[ -n "$comment" ]] && echo "# $comment"
        echo "$schedule $command"
    } | crontab -u "$user" -
    
    echo "Added cron job for $user: $schedule $command"
}

list_all_crons() {
    echo "=== System Cron Jobs ==="
    echo ""
    
    [[ -f /etc/crontab ]] && {
        echo "=== /etc/crontab ==="
        grep -v "^#\|^$" /etc/crontab
        echo ""
    }
    
    for f in /etc/cron.d/*; do
        [[ -f "$f" ]] || continue
        echo "=== $f ==="
        grep -v "^#\|^$" "$f"
        echo ""
    done
    
    echo "=== User Crontabs ==="
    for user in $(cut -d: -f1 /etc/passwd); do
        local cron
        cron=$(crontab -u "$user" -l 2>/dev/null | grep -v "^#\|^$")
        if [[ -n "$cron" ]]; then
            echo "--- $user ---"
            echo "$cron"
        fi
    done
}

monitor_cron_execution() {
    local job_name=$1
    local max_duration=${2:-3600}
    
    local start_time=$SECONDS
    local lock_file="/tmp/cron_lock_${job_name}"
    
    if [[ -f "$lock_file" ]]; then
        local lock_pid
        lock_pid=$(cat "$lock_file")
        if kill -0 "$lock_pid" 2>/dev/null; then
            echo "Job $job_name already running (PID $lock_pid)" >&2
            exit 1
        fi
        rm -f "$lock_file"
    fi
    
    echo $$ > "$lock_file"
    trap "rm -f $lock_file" EXIT
    
    echo "$(date): Starting $job_name"
    
    "$@" &
    local job_pid=$!
    
    while kill -0 "$job_pid" 2>/dev/null; do
        local elapsed=$(( SECONDS - start_time ))
        if (( elapsed > max_duration )); then
            echo "$(date): $job_name exceeded max duration, killing..."
            kill -TERM "$job_pid"
            sleep 5
            kill -KILL "$job_pid" 2>/dev/null
            break
        fi
        sleep 10
    done
    
    wait "$job_pid"
    local exit_code=$?
    local duration=$(( SECONDS - start_time ))
    
    echo "$(date): $job_name completed in ${duration}s (exit: $exit_code)"
    return "$exit_code"
}
```

---

## 35.6 Environment & Deployment

```bash
#!/bin/bash
# deployment.sh

deploy_blue_green() {
    local app_name=$1
    local new_version=$2
    local deploy_dir="/opt/${app_name}"
    
    echo "=== Blue/Green Deployment: $app_name v$new_version ==="
    
    local current_link="${deploy_dir}/current"
    local current_slot
    current_slot=$(readlink "$current_link" 2>/dev/null | xargs basename)
    
    local next_slot
    if [[ "$current_slot" == "blue" ]]; then
        next_slot="green"
    else
        next_slot="blue"
    fi
    
    local next_dir="${deploy_dir}/${next_slot}"
    
    echo "Current: ${current_slot:-none} → Next: $next_slot"
    
    mkdir -p "$next_dir"
    tar -xzf "${app_name}-${new_version}.tar.gz" -C "$next_dir"
    
    systemctl start "${app_name}-${next_slot}"
    sleep 5
    
    if ! systemctl is-active --quiet "${app_name}-${next_slot}"; then
        echo "Service failed to start" >&2
        systemctl stop "${app_name}-${next_slot}"
        return 1
    fi
    
    ln -sfn "${deploy_dir}/${next_slot}" "${deploy_dir}/current.new"
    mv -f "${deploy_dir}/current.new" "$current_link"
    
    [[ -n "$current_slot" ]] && systemctl stop "${app_name}-${current_slot}" || true
    
    echo "Deployment complete: now running $next_slot"
}

manage_env() {
    local env_file=$1
    local action=$2
    local key=$3
    local value=$4
    
    case "$action" in
        get)    grep -E "^${key}=" "$env_file" | cut -d= -f2- | head -1 ;;
        set)
            if grep -q "^${key}=" "$env_file" 2>/dev/null; then
                sed -i "s/^${key}=.*/${key}=${value}/" "$env_file"
            else
                echo "${key}=${value}" >> "$env_file"
            fi
            ;;
        delete) sed -i "/^${key}=/d" "$env_file" ;;
        list)   grep -v "^#\|^$" "$env_file" | sort ;;
    esac
}
```

---

## 35.7 Exercises

### Exercise 1: Service Dashboard
สร้าง dashboard แสดงสถานะ:
- All systemd services
- CPU/memory per service
- Restart count
- Auto-refresh every 5s

### Exercise 2: Configuration Drift Detector
สร้าง tool ตรวจหา config drift:
- Compare current vs baseline
- Report differences
- Alert on changes
- Auto-remediate minor drift

### Exercise 3: Zero-Downtime Deployer
สร้าง deployment system:
- Blue/green switching
- Health checks
- Automatic rollback on failure
- Slack notification

---

## สรุป Part 35

✅ Systemd service management  
✅ Service creation & timers  
✅ INI config file parsing  
✅ Template config renderer  
✅ Package management (universal)  
✅ User & permission management  
✅ Cron job automation & monitoring  
✅ Blue/green deployment  

---

**→ Part 36: Containers & Kubernetes Automation**
