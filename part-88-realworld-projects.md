# Part 88: Real-World Shell Script Projects
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 88.1 Server Health Dashboard

```bash
#!/bin/bash
# health_dashboard.sh - Real-time server health monitoring dashboard

set -euo pipefail

DASHBOARD_INTERVAL="${DASHBOARD_INTERVAL:-5}"
ALERT_CPU_THRESHOLD="${ALERT_CPU_THRESHOLD:-80}"
ALERT_MEM_THRESHOLD="${ALERT_MEM_THRESHOLD:-85}"
ALERT_DISK_THRESHOLD="${ALERT_DISK_THRESHOLD:-90}"
ALERT_WEBHOOK="${ALERT_WEBHOOK:-}"

# ─── Collectors ───────────────────────────────────────────────
collect_cpu() {
    awk '/^cpu / {
        idle=$5; total=$2+$3+$4+$5+$6+$7+$8
        printf "%.1f", (1 - idle/total) * 100
    }' /proc/stat
}

collect_mem() {
    awk '
    /MemTotal/ {total=$2}
    /MemAvailable/ {avail=$2}
    END {printf "%.1f", (1 - avail/total) * 100}
    ' /proc/meminfo
}

collect_disk() {
    df -h / | awk 'NR==2 {gsub(/%/,"",$5); print $5}'
}

collect_load() {
    awk '{print $1, $2, $3}' /proc/loadavg
}

collect_top_procs() {
    ps aux --no-header --sort=-%cpu | head -5 | \
        awk '{printf "  %-20s CPU:%-6s MEM:%-6s\n", $11, $3, $4}'
}

collect_network() {
    local iface=${1:-$(ip route show default 2>/dev/null | awk '{print $5}' | head -1)}
    [[ -z "$iface" ]] && { echo "N/A"; return; }

    local rx1 tx1 rx2 tx2
    rx1=$(cat /sys/class/net/$iface/statistics/rx_bytes 2>/dev/null || echo 0)
    tx1=$(cat /sys/class/net/$iface/statistics/tx_bytes 2>/dev/null || echo 0)
    sleep 1
    rx2=$(cat /sys/class/net/$iface/statistics/rx_bytes 2>/dev/null || echo 0)
    tx2=$(cat /sys/class/net/$iface/statistics/tx_bytes 2>/dev/null || echo 0)

    local rx_kbs=$(( (rx2 - rx1) / 1024 ))
    local tx_kbs=$(( (tx2 - tx1) / 1024 ))
    echo "RX: ${rx_kbs} KB/s  TX: ${tx_kbs} KB/s  (${iface})"
}

# ─── Alerting ─────────────────────────────────────────────────
send_alert() {
    local level=$1 message=$2
    echo "[ALERT-$level] $message" >&2

    [[ -n "$ALERT_WEBHOOK" ]] && \
        curl -s -X POST "$ALERT_WEBHOOK" \
             -H "Content-Type: application/json" \
             -d "{\"text\":\"[$level] $message\"}" > /dev/null 2>&1 || true
}

check_thresholds() {
    local cpu=$1 mem=$2 disk=$3

    (( $(echo "$cpu > $ALERT_CPU_THRESHOLD" | awk '{print ($1>$3)}') )) && \
        send_alert "WARN" "CPU at ${cpu}% (threshold: ${ALERT_CPU_THRESHOLD}%)"
    (( $(echo "$mem > $ALERT_MEM_THRESHOLD" | awk '{print ($1>$3)}') )) && \
        send_alert "WARN" "Memory at ${mem}% (threshold: ${ALERT_MEM_THRESHOLD}%)"
    (( disk > ALERT_DISK_THRESHOLD )) && \
        send_alert "CRIT" "Disk at ${disk}% (threshold: ${ALERT_DISK_THRESHOLD}%)"
}

# ─── Dashboard render ─────────────────────────────────────────
draw_bar() {
    local pct=${1%.*} width=${2:-40}
    local filled=$(( pct * width / 100 ))
    local empty=$(( width - filled ))
    printf '['
    printf '%*s' "$filled" '' | tr ' ' '#'
    printf '%*s' "$empty" '' | tr ' ' '-'
    printf '] %3d%%' "$pct"
}

run_dashboard() {
    while true; do
        clear
        echo "╔══════════════════════════════════════════════════════╗"
        printf  "║  Server Health Dashboard  %-28s║\n" "$(date '+%Y-%m-%d %H:%M:%S')"
        echo "╠══════════════════════════════════════════════════════╣"

        local cpu mem disk
        cpu=$(collect_cpu)
        mem=$(collect_mem)
        disk=$(collect_disk)
        local load; load=$(collect_load)

        printf "║  CPU  "; draw_bar "$cpu"; echo "         ║"
        printf "║  MEM  "; draw_bar "$mem"; echo "         ║"
        printf "║  DISK "; draw_bar "$disk"; echo "         ║"
        echo "║                                                      ║"
        printf  "║  Load avg: %-41s║\n" "$load"

        echo "╠══════════════════════════════════════════════════════╣"
        echo "║  Top Processes (by CPU)                              ║"
        collect_top_procs | while read -r line; do
            printf "║  %-52s║\n" "$line"
        done
        echo "╚══════════════════════════════════════════════════════╝"

        check_thresholds "$cpu" "$mem" "$disk"
        sleep "$DASHBOARD_INTERVAL"
    done
}
```

---

## 88.2 Deployment Automation Script

```bash
#!/bin/bash
# deploy.sh - Full deployment automation

set -euo pipefail

DEPLOY_ENV="${DEPLOY_ENV:-staging}"
DEPLOY_APP="${DEPLOY_APP:-myapp}"
DEPLOY_REPO="${DEPLOY_REPO:-}"
DEPLOY_BRANCH="${DEPLOY_BRANCH:-main}"
DEPLOY_DIR="/opt/${DEPLOY_APP}"
DEPLOY_LOG="/var/log/${DEPLOY_APP}-deploy.log"
DEPLOY_USER="${DEPLOY_USER:-deploy}"

deploy_log() {
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] $*" | tee -a "$DEPLOY_LOG"
}

deploy_notify() {
    local status=$1 message=$2
    deploy_log "NOTIFY [$status]: $message"
}

deploy_pre_checks() {
    deploy_log "Running pre-deployment checks..."

    local free_kb; free_kb=$(df "$DEPLOY_DIR" | awk 'NR==2{print $4}')
    (( free_kb < 524288 )) && { deploy_log "ERROR: Less than 512MB free disk space"; return 1; }

    for cmd in git rsync systemctl; do
        command -v "$cmd" &>/dev/null || { deploy_log "ERROR: $cmd not found"; return 1; }
    done

    deploy_log "Pre-checks passed"
}

deploy_fetch_code() {
    local release_dir="${DEPLOY_DIR}/releases/$(date +%Y%m%d%H%M%S)"
    mkdir -p "$release_dir"

    deploy_log "Fetching code from $DEPLOY_REPO branch $DEPLOY_BRANCH"
    git clone --depth 1 --branch "$DEPLOY_BRANCH" "$DEPLOY_REPO" "$release_dir"

    echo "$release_dir"
}

deploy_install_deps() {
    local release_dir=$1
    deploy_log "Installing dependencies in $release_dir"

    if [[ -f "${release_dir}/package.json" ]]; then
        (cd "$release_dir" && npm ci --production)
    elif [[ -f "${release_dir}/requirements.txt" ]]; then
        pip install -r "${release_dir}/requirements.txt" -q
    elif [[ -f "${release_dir}/go.mod" ]]; then
        (cd "$release_dir" && go build ./...)
    fi
}

deploy_run_migrations() {
    local release_dir=$1
    deploy_log "Running database migrations..."

    [[ -f "${release_dir}/migrate.sh" ]] && \
        DEPLOY_ENV="$DEPLOY_ENV" bash "${release_dir}/migrate.sh"
}

deploy_switch() {
    local release_dir=$1
    local current="${DEPLOY_DIR}/current"

    deploy_log "Switching current -> $release_dir"
    ln -sfn "$release_dir" "${current}.new"
    mv -f "${current}.new" "$current"
}

deploy_restart_service() {
    local service="${DEPLOY_APP}-${DEPLOY_ENV}"
    deploy_log "Restarting service: $service"

    if systemctl is-active --quiet "$service" 2>/dev/null; then
        systemctl reload-or-restart "$service"
    else
        deploy_log "Service $service not found, skipping restart"
    fi
}

deploy_health_check() {
    local url=${1:-"http://localhost:8080/health"}
    local retries=${2:-10}

    deploy_log "Health check: $url"
    for (( i=1; i<=retries; i++ )); do
        local code; code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$url" 2>/dev/null || echo 0)
        if [[ "$code" == "200" ]]; then
            deploy_log "Health check passed"
            return 0
        fi
        deploy_log "Health check attempt $i/$retries failed (HTTP $code)"
        sleep 3
    done

    deploy_log "ERROR: Health check failed after $retries attempts"
    return 1
}

deploy_cleanup_old() {
    local keep=${1:-5}
    deploy_log "Cleaning up old releases (keeping $keep)"
    ls -dt "${DEPLOY_DIR}/releases/"*/ 2>/dev/null | \
        tail -n +$(( keep + 1 )) | xargs rm -rf
}

deploy_rollback() {
    local releases_dir="${DEPLOY_DIR}/releases"
    local prev; prev=$(ls -dt "${releases_dir}/"*/ 2>/dev/null | sed -n '2p')

    [[ -z "$prev" ]] && { deploy_log "No previous release to rollback to"; return 1; }

    deploy_log "Rolling back to: $prev"
    deploy_switch "${prev%/}"
    deploy_restart_service
    deploy_notify "ROLLBACK" "Rolled back to ${prev}"
}

deploy_full() {
    deploy_log "=== Starting deployment to $DEPLOY_ENV ==="
    deploy_notify "START" "Deployment started for $DEPLOY_APP to $DEPLOY_ENV"

    deploy_pre_checks || { deploy_notify "FAIL" "Pre-checks failed"; return 1; }

    local release_dir; release_dir=$(deploy_fetch_code)
    deploy_install_deps "$release_dir"
    deploy_run_migrations "$release_dir"
    deploy_switch "$release_dir"
    deploy_restart_service

    if ! deploy_health_check; then
        deploy_log "Deployment failed, rolling back..."
        deploy_notify "FAIL" "Health check failed, rolling back"
        deploy_rollback
        return 1
    fi

    deploy_cleanup_old
    deploy_notify "SUCCESS" "Deployment complete: $release_dir"
    deploy_log "=== Deployment complete ==="
}
```

---

## 88.3 Backup and Recovery System

```bash
#!/bin/bash
# backup_system.sh - Comprehensive backup and recovery automation

set -euo pipefail

BACKUP_ROOT="${BACKUP_ROOT:-/backups}"
BACKUP_RETENTION_DAYS="${BACKUP_RETENTION_DAYS:-30}"
BACKUP_ENCRYPT="${BACKUP_ENCRYPT:-false}"
BACKUP_GPG_KEY="${BACKUP_GPG_KEY:-}"
BACKUP_S3_BUCKET="${BACKUP_S3_BUCKET:-}"

backup_log() {
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] BACKUP: $*"
}

backup_files() {
    local name=$1 src_dir=$2 dest_dir=${3:-$BACKUP_ROOT}
    local ts; ts=$(date +%Y%m%d_%H%M%S)
    local archive="${dest_dir}/${name}_${ts}.tar.gz"

    mkdir -p "$dest_dir"
    backup_log "Archiving $src_dir -> $archive"

    tar czf "$archive" --exclude='*.log' --exclude='*.tmp' \
        -C "$(dirname "$src_dir")" "$(basename "$src_dir")" 2>/dev/null

    local size; size=$(du -sh "$archive" | cut -f1)
    backup_log "Created: $archive ($size)"

    if [[ "$BACKUP_ENCRYPT" == "true" && -n "$BACKUP_GPG_KEY" ]]; then
        gpg --batch --yes --recipient "$BACKUP_GPG_KEY" \
            --encrypt-files "$archive" 2>/dev/null
        rm -f "$archive"
        archive="${archive}.gpg"
        backup_log "Encrypted: $archive"
    fi

    echo "$archive"
}

backup_database() {
    local name=$1 db_type=${2:-postgres} dest_dir=${3:-$BACKUP_ROOT}
    local ts; ts=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$dest_dir"

    case "$db_type" in
        postgres)
            local dump="${dest_dir}/${name}_${ts}.sql.gz"
            PGPASSWORD="${DB_PASS:-}" pg_dump -h "${DB_HOST:-localhost}" \
                -U "${DB_USER:-postgres}" "${DB_NAME:-$name}" | \
                gzip > "$dump"
            backup_log "PostgreSQL dump: $dump"
            echo "$dump"
            ;;
        mysql)
            local dump="${dest_dir}/${name}_${ts}.sql.gz"
            mysqldump -h "${DB_HOST:-localhost}" -u "${DB_USER:-root}" \
                -p"${DB_PASS:-}" "${DB_NAME:-$name}" | gzip > "$dump"
            backup_log "MySQL dump: $dump"
            echo "$dump"
            ;;
        sqlite)
            local src="${DB_PATH:-/data/${name}.db}"
            local dump="${dest_dir}/${name}_${ts}.db.gz"
            sqlite3 "$src" ".backup /tmp/${name}_backup.db" && \
                gzip -c "/tmp/${name}_backup.db" > "$dump" && \
                rm -f "/tmp/${name}_backup.db"
            backup_log "SQLite backup: $dump"
            echo "$dump"
            ;;
    esac
}

backup_upload_s3() {
    local file=$1
    [[ -z "$BACKUP_S3_BUCKET" ]] && return 0

    backup_log "Uploading to S3: $BACKUP_S3_BUCKET"
    aws s3 cp "$file" "s3://${BACKUP_S3_BUCKET}/$(basename "$file")" \
        --storage-class STANDARD_IA 2>/dev/null
    backup_log "Uploaded: $(basename "$file")"
}

backup_verify() {
    local archive=$1

    backup_log "Verifying: $archive"
    if [[ "$archive" =~ \.tar\.gz$ ]]; then
        tar tzf "$archive" > /dev/null && backup_log "Verify OK: $archive"
    elif [[ "$archive" =~ \.sql\.gz$ ]]; then
        gzip -t "$archive" && backup_log "Verify OK: $archive"
    fi
}

backup_cleanup_old() {
    local dir=${1:-$BACKUP_ROOT}
    backup_log "Cleaning backups older than $BACKUP_RETENTION_DAYS days in $dir"
    find "$dir" \( -name "*.tar.gz" -o -name "*.sql.gz" -o -name "*.db.gz" \) | \
        while read -r f; do
            local age=$(( ( $(date +%s) - $(stat -c '%Y' "$f") ) / 86400 ))
            (( age > BACKUP_RETENTION_DAYS )) && rm -f "$f" && backup_log "Deleted: $f"
        done
}

backup_restore_files() {
    local archive=$1 dest_dir=$2
    mkdir -p "$dest_dir"
    backup_log "Restoring $archive -> $dest_dir"

    if [[ "$archive" =~ \.gpg$ ]]; then
        local decrypted="${archive%.gpg}"
        gpg --batch --yes --output "$decrypted" --decrypt "$archive"
        tar xzf "$decrypted" -C "$dest_dir"
        rm -f "$decrypted"
    else
        tar xzf "$archive" -C "$dest_dir"
    fi

    backup_log "Restore complete: $dest_dir"
}
```

---

## 88.4 Exercises

### Exercise 1: Full Deployment Pipeline
Combine Parts 80 + 88:
- Git tag detection
- Build + test
- Deploy to staging
- Smoke test
- Promote to production

### Exercise 2: Multi-Server Backup
สร้างระบบ backup ที่:
- SSH to multiple servers
- Backup databases + app dirs
- Centralize to S3
- Verify + alert on failure

### Exercise 3: Dashboard Extension
ขยาย health_dashboard ด้วย:
- Service health checks (HTTP endpoints)
- Database connection count
- Error rate from logs
- Alert history panel

---

## สรุป Part 88

✅ Health dashboard: CPU/MEM/DISK bar charts with threshold alerting
┅ collect_cpu/mem/disk/load/network: /proc-based collectors
┅ draw_bar: ASCII progress bar renderer
┅ deploy_full: pre-check → fetch → deps → migrate → switch → restart → health check
┅ deploy_rollback: previous release symlink swap
┅ backup_files: tar.gz with optional GPG encryption
┅ backup_database: postgres/mysql/sqlite dump with gzip
┅ backup_upload_s3, backup_verify, backup_cleanup_old, backup_restore_files

---

**→ Part 89: Shell Scripting for Security Auditing**
