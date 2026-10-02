# Part 65: Backup and Disaster Recovery
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 65.1 Backup Framework

```bash
#!/bin/bash
# backup_framework.sh - Comprehensive backup system

BACKUP_BASE="${BACKUP_BASE:-/var/backups}"
BACKUP_RETENTION_DAYS="${BACKUP_RETENTION_DAYS:-30}"
BACKUP_LOG="${BACKUP_LOG:-/var/log/backup.log}"

backup_log() {
    local level=$1 msg=$2
    printf '%s [%s] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$level" "$msg" | tee -a "$BACKUP_LOG"
}

# ─── File Backup ──────────────────────────────────────────────────
backup_files() {
    local name=$1 source_path=$2
    local dest_dir="${BACKUP_BASE}/${name}/$(date +%Y/%m/%d)"
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)
    local archive="${dest_dir}/${name}_${timestamp}.tar.gz"

    mkdir -p "$dest_dir"

    backup_log "INFO" "Starting file backup: $name ($source_path)"

    local start_time; start_time=$(date +%s)

    tar -czf "$archive" \
        --exclude='*.tmp' \
        --exclude='*.log' \
        --exclude='.cache' \
        "$source_path" 2>/dev/null

    local exit_code=$?
    local end_time; end_time=$(date +%s)
    local duration=$(( end_time - start_time ))
    local size; size=$(du -sh "$archive" 2>/dev/null | cut -f1)

    if (( exit_code == 0 )); then
        sha256sum "$archive" > "${archive}.sha256"
        backup_log "INFO" "Backup complete: $archive ($size, ${duration}s)"
        echo "$archive"
    else
        backup_log "ERROR" "Backup failed: $name (exit $exit_code)"
        rm -f "$archive"
        return 1
    fi
}

# ─── Incremental Backup ───────────────────────────────────────────────
backup_incremental() {
    local name=$1 source_path=$2
    local state_dir="${BACKUP_BASE}/${name}/.state"
    local dest_dir="${BACKUP_BASE}/${name}/incremental/$(date +%Y%m%d)"
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)
    local archive="${dest_dir}/${name}_incr_${timestamp}.tar.gz"

    mkdir -p "$state_dir" "$dest_dir"

    local snapshot_file="${state_dir}/snapshot"

    backup_log "INFO" "Starting incremental backup: $name"

    tar -czf "$archive" \
        --listed-incremental="$snapshot_file" \
        "$source_path" 2>/dev/null

    local exit_code=$?

    if (( exit_code == 0 )); then
        sha256sum "$archive" > "${archive}.sha256"
        backup_log "INFO" "Incremental backup complete: $archive"
        echo "$archive"
    else
        backup_log "ERROR" "Incremental backup failed: $name"
        return 1
    fi
}

# ─── Rsync Backup ───────────────────────────────────────────────────
backup_rsync() {
    local name=$1 source=$2 dest=${3:-${BACKUP_BASE}/${name}}
    local rsync_log="${BACKUP_LOG}.rsync"
    local link_dest=""

    mkdir -p "$dest"

    local prev; prev=$(ls -dt "${dest}"/????-??-??_?????? 2>/dev/null | head -1)
    [[ -n "$prev" ]] && link_dest="--link-dest=$prev"

    local this_backup="${dest}/$(date +%Y-%m-%d_%H%M%S)"
    mkdir -p "$this_backup"

    backup_log "INFO" "Starting rsync backup: $name"

    rsync -a --delete \
        --exclude='*.tmp' \
        --exclude='.cache/' \
        --log-file="$rsync_log" \
        $link_dest \
        "$source/" \
        "$this_backup/"

    local exit_code=$?

    if (( exit_code == 0 )); then
        backup_log "INFO" "Rsync backup complete: $this_backup"
        echo "$this_backup"
    else
        backup_log "ERROR" "Rsync backup failed: $name (exit $exit_code)"
        return 1
    fi
}

# ─── Retention Cleanup ───────────────────────────────────────────────
backup_cleanup_old() {
    local name=$1 days=${2:-$BACKUP_RETENTION_DAYS}

    backup_log "INFO" "Cleaning backups older than $days days for: $name"

    find "${BACKUP_BASE}/${name}" \
        -maxdepth 4 \
        -name "*.tar.gz" \
        -mtime "+${days}" \
        -print \
        -delete 2>/dev/null | while read -r file; do
            rm -f "${file}.sha256"
            backup_log "INFO" "Removed old backup: $file"
        done

    backup_log "INFO" "Cleanup complete for: $name"
}
```

---

## 65.2 Database Backup

```bash
#!/bin/bash
# db_backup.sh - Database backup with rotation

BACKUP_ENCRYPT_KEY="${BACKUP_ENCRYPT_KEY:-}"

# ─── MySQL Full Backup ──────────────────────────────────────────────
mysql_full_backup() {
    local name=$1 db=${2:-all}
    local dest="${BACKUP_BASE}/mysql/${name}"
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$dest"

    local archive="${dest}/mysql_${db}_${timestamp}.sql.gz"

    backup_log "INFO" "MySQL backup: $db"

    local mysqldump_args=(
        --single-transaction
        --routines
        --triggers
        --events
        --flush-logs
        --host="${MYSQL_HOST:-localhost}"
        --user="${MYSQL_USER:-root}"
    )

    [[ -n "${MYSQL_PASSWORD:-}" ]] && mysqldump_args+=("--password=${MYSQL_PASSWORD}")

    if [[ "$db" == "all" ]]; then
        mysqldump_args+=(--all-databases)
    else
        mysqldump_args+=("$db")
    fi

    mysqldump "${mysqldump_args[@]}" 2>/dev/null | gzip -9 > "$archive"

    local exit_code=${PIPESTATUS[0]}

    if (( exit_code == 0 )); then
        sha256sum "$archive" > "${archive}.sha256"
        local size; size=$(du -sh "$archive" | cut -f1)
        backup_log "INFO" "MySQL backup complete: $archive ($size)"

        if [[ -n "$BACKUP_ENCRYPT_KEY" ]]; then
            openssl enc -aes-256-cbc -pbkdf2 -pass "pass:${BACKUP_ENCRYPT_KEY}" \
                -in "$archive" -out "${archive}.enc"
            rm -f "$archive"
            backup_log "INFO" "Backup encrypted: ${archive}.enc"
        fi
    else
        backup_log "ERROR" "MySQL backup failed: $db"
        rm -f "$archive"
        return 1
    fi
}

mysql_restore_backup() {
    local archive=$1 target_db=${2:-}

    backup_log "INFO" "Restoring MySQL from: $archive"

    local restore_cmd="mysql --host=${MYSQL_HOST:-localhost} --user=${MYSQL_USER:-root}"
    [[ -n "${MYSQL_PASSWORD:-}" ]] && restore_cmd+=" --password=${MYSQL_PASSWORD}"
    [[ -n "$target_db" ]] && restore_cmd+=" $target_db"

    if [[ "$archive" =~ \.enc$ ]]; then
        if [[ -z "$BACKUP_ENCRYPT_KEY" ]]; then
            backup_log "ERROR" "Encrypted backup but BACKUP_ENCRYPT_KEY not set"
            return 1
        fi
        openssl enc -d -aes-256-cbc -pbkdf2 -pass "pass:${BACKUP_ENCRYPT_KEY}" \
            -in "$archive" | zcat | eval "$restore_cmd"
    elif [[ "$archive" =~ \.gz$ ]]; then
        zcat "$archive" | eval "$restore_cmd"
    else
        eval "$restore_cmd" < "$archive"
    fi

    backup_log "INFO" "MySQL restore complete"
}

# ─── PostgreSQL Backup ───────────────────────────────────────────────
pg_full_backup() {
    local name=$1 db=$2
    local dest="${BACKUP_BASE}/postgres/${name}"
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$dest"
    local archive="${dest}/pg_${db}_${timestamp}.dump"

    backup_log "INFO" "PostgreSQL backup: $db"

    PGPASSWORD="${PGPASSWORD:-}" pg_dump \
        --host="${PGHOST:-localhost}" \
        --port="${PGPORT:-5432}" \
        --username="${PGUSER:-postgres}" \
        --format=custom \
        --compress=9 \
        --file="$archive" \
        "$db" 2>/dev/null

    local exit_code=$?

    if (( exit_code == 0 )); then
        sha256sum "$archive" > "${archive}.sha256"
        local size; size=$(du -sh "$archive" | cut -f1)
        backup_log "INFO" "PostgreSQL backup complete: $archive ($size)"
    else
        backup_log "ERROR" "PostgreSQL backup failed: $db"
        rm -f "$archive"
        return 1
    fi
}
```

---

## 65.3 Restore Procedures

```bash
#!/bin/bash
# restore.sh - Disaster recovery restore procedures

restore_files() {
    local archive=$1 dest=${2:-/}

    backup_log "INFO" "Restoring files from: $archive to $dest"

    local checksum_file="${archive}.sha256"
    if [[ -f "$checksum_file" ]]; then
        if ! sha256sum --check "$checksum_file" 2>/dev/null; then
            backup_log "ERROR" "Checksum verification FAILED for $archive"
            return 1
        fi
        backup_log "INFO" "Checksum verified OK"
    fi

    tar -xzf "$archive" -C "$dest" 2>/dev/null
    local exit_code=$?

    if (( exit_code == 0 )); then
        backup_log "INFO" "Restore complete: $archive -> $dest"
    else
        backup_log "ERROR" "Restore failed from: $archive (exit $exit_code)"
        return 1
    fi
}

find_backup_at_time() {
    local name=$1 target_time=$2

    local target_epoch; target_epoch=$(date -d "$target_time" +%s 2>/dev/null || \
                                      date -j -f "%Y-%m-%d %H:%M:%S" "$target_time" +%s 2>/dev/null)

    local best_archive="" best_diff=999999999

    while IFS= read -r archive; do
        local mtime; mtime=$(stat -c '%Y' "$archive" 2>/dev/null || \
                            stat -f '%m' "$archive" 2>/dev/null)
        local diff=$(( target_epoch - mtime ))

        if (( diff >= 0 && diff < best_diff )); then
            best_diff=$diff
            best_archive="$archive"
        fi
    done < <(find "${BACKUP_BASE}/${name}" -name "*.tar.gz" -o -name "*.dump" 2>/dev/null | sort)

    echo "$best_archive"
}

verify_backup() {
    local archive=$1

    echo "Verifying: $archive"

    local checksum_file="${archive}.sha256"
    if [[ -f "$checksum_file" ]]; then
        sha256sum --check "$checksum_file" 2>/dev/null && \
            echo "  [OK] Checksum valid" || echo "  [FAIL] Checksum mismatch"
    else
        echo "  [WARN] No checksum file"
    fi

    if [[ "$archive" =~ \.tar\.gz$ ]]; then
        tar -tzf "$archive" > /dev/null 2>&1 && \
            echo "  [OK] Archive integrity" || echo "  [FAIL] Archive corrupted"
    elif [[ "$archive" =~ \.dump$ ]]; then
        pg_restore --list "$archive" > /dev/null 2>&1 && \
            echo "  [OK] PostgreSQL dump valid" || echo "  [FAIL] PostgreSQL dump corrupted"
    fi

    local size; size=$(stat -c '%s' "$archive" 2>/dev/null)
    if (( size > 0 )); then
        echo "  [OK] File size: $(du -sh "$archive" | cut -f1)"
    else
        echo "  [FAIL] Empty archive"
        return 1
    fi
}

backup_status_report() {
    echo "=== Backup Status Report ==="
    echo "Date: $(date)"
    echo ""

    for name_dir in "${BACKUP_BASE}"/*/; do
        local name; name=$(basename "$name_dir")
        [[ "$name" == "mysql" || "$name" == "postgres" ]] && continue

        local latest; latest=$(find "$name_dir" -name "*.tar.gz" -o -name "*.dump" 2>/dev/null | \
            sort | tail -1)

        if [[ -n "$latest" ]]; then
            local mtime; mtime=$(stat -c '%Y' "$latest" 2>/dev/null)
            local age_hours=$(( ( $(date +%s) - mtime ) / 3600 ))
            local size; size=$(du -sh "$latest" | cut -f1)
            local status="OK"
            (( age_hours > 25 )) && status="STALE"

            printf "  %-20s %-6s  Age: %dh  Size: %s\n" "$name" "[$status]" "$age_hours" "$size"
        else
            printf "  %-20s [MISSING]\n" "$name"
        fi
    done
}
```

---

## 65.4 Remote Backup

```bash
#!/bin/bash
# remote_backup.sh - Remote backup via SSH/rsync/S3

backup_to_remote() {
    local name=$1 source=$2 remote_host=$3 remote_path=$4
    local remote_user=${5:-backup}

    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)
    local remote_dest="${remote_path}/${name}/${timestamp}"

    backup_log "INFO" "Syncing to ${remote_host}:${remote_dest}"

    ssh "${remote_user}@${remote_host}" "mkdir -p '${remote_dest}'" 2>/dev/null

    rsync -az --delete \
        --rsh="ssh -o StrictHostKeyChecking=no -o BatchMode=yes" \
        "$source/" \
        "${remote_user}@${remote_host}:${remote_dest}/"

    local exit_code=$?

    if (( exit_code == 0 )); then
        backup_log "INFO" "Remote backup complete: ${remote_host}:${remote_dest}"
    else
        backup_log "ERROR" "Remote backup failed: $name (exit $exit_code)"
        return 1
    fi
}

backup_to_s3() {
    local name=$1 source_archive=$2 bucket=$3
    local s3_key="${name}/$(date +%Y/%m/%d)/$(basename "$source_archive")"

    backup_log "INFO" "Uploading to s3://${bucket}/${s3_key}"

    if command -v aws &>/dev/null; then
        aws s3 cp "$source_archive" "s3://${bucket}/${s3_key}" \
            --storage-class STANDARD_IA \
            --metadata "backup-name=${name},created=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
        local exit_code=$?
    else
        backup_log "ERROR" "AWS CLI not available"
        return 1
    fi

    if (( exit_code == 0 )); then
        backup_log "INFO" "S3 upload complete: s3://${bucket}/${s3_key}"
    else
        backup_log "ERROR" "S3 upload failed: $name"
        return 1
    fi
}

restore_from_s3() {
    local bucket=$1 s3_key=$2 local_dest=$3

    backup_log "INFO" "Downloading from s3://${bucket}/${s3_key}"
    aws s3 cp "s3://${bucket}/${s3_key}" "$local_dest"
    backup_log "INFO" "S3 download complete: $local_dest"
}

list_s3_backups() {
    local bucket=$1 name=${2:-}
    local prefix="${name:+${name}/}"

    aws s3 ls "s3://${bucket}/${prefix}" --recursive | \
        awk '{print $1, $2, $3, $4}' | sort -rk1
}
```

---

## 65.5 Disaster Recovery Runbook

```bash
#!/bin/bash
# dr_runbook.sh - Automated disaster recovery procedures

DR_STATE_DIR="${DR_STATE_DIR:-/var/run/dr}"
DR_LOG="${DR_LOG:-/var/log/dr.log}"

dr_log() {
    printf '%s [DR] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "$DR_LOG"
}

dr_precheck() {
    dr_log "Running pre-checks..."

    local checks_passed=true

    local avail_gb; avail_gb=$(df -BG / | awk 'NR==2 {print $4}' | tr -d 'G')
    if (( avail_gb < 10 )); then
        dr_log "ERROR: Insufficient disk space (${avail_gb}GB available, need 10GB)"
        checks_passed=false
    fi

    if ! ping -c 1 -W 3 8.8.8.8 &>/dev/null; then
        dr_log "WARNING: No external network connectivity"
    fi

    if [[ ! -d "$BACKUP_BASE" ]]; then
        dr_log "ERROR: Backup directory not accessible: $BACKUP_BASE"
        checks_passed=false
    fi

    $checks_passed && dr_log "Pre-checks passed" || { dr_log "Pre-checks FAILED"; return 1; }
}

dr_restore_service() {
    local service_name=$1

    dr_log "Restoring service: $service_name"

    systemctl stop "$service_name" 2>/dev/null || true

    local latest_backup; latest_backup=$(find "${BACKUP_BASE}/${service_name}" \
        -name "*.tar.gz" 2>/dev/null | sort | tail -1)

    if [[ -z "$latest_backup" ]]; then
        dr_log "ERROR: No backup found for $service_name"
        return 1
    fi

    dr_log "Using backup: $latest_backup"
    restore_files "$latest_backup" "/"

    systemctl start "$service_name" 2>/dev/null
    sleep 3

    if systemctl is-active "$service_name" &>/dev/null; then
        dr_log "Service $service_name restored and running"
    else
        dr_log "ERROR: Service $service_name failed to start after restore"
        return 1
    fi
}

dr_full_recovery() {
    local recovery_timestamp=${1:-$(date +%Y%m%d_%H%M%S)}

    mkdir -p "$DR_STATE_DIR"

    dr_log "=== DISASTER RECOVERY STARTED ==="
    dr_log "Recovery ID: $recovery_timestamp"

    dr_precheck || return 1

    local services=(nginx mysql postgresql redis)

    for service in "${services[@]}"; do
        dr_log "Recovering: $service"
        dr_restore_service "$service" && \
            echo "$service=OK" >> "${DR_STATE_DIR}/recovery_${recovery_timestamp}.status" || \
            echo "$service=FAIL" >> "${DR_STATE_DIR}/recovery_${recovery_timestamp}.status"
    done

    dr_log "=== DISASTER RECOVERY COMPLETE ==="
    dr_log "Status file: ${DR_STATE_DIR}/recovery_${recovery_timestamp}.status"

    cat "${DR_STATE_DIR}/recovery_${recovery_timestamp}.status"
}
```

---

## 65.6 Exercises

### Exercise 1: Automated Backup System
สร้าง system ที่:
- Schedule multiple backup jobs
- Full + incremental strategy
- Email notification on failure
- Dashboard with backup health

### Exercise 2: DR Test Automation
สร้าง automation ที่:
- Spin up test environment
- Restore latest backup
- Run smoke tests
- Generate DR test report

### Exercise 3: Backup Cost Optimizer
สร้าง tool ที่:
- Analyze backup sizes and frequency
- Recommend retention policy
- Calculate storage costs
- Auto-tier old backups to cold storage

---

## สรุป Part 65

✅ Full file backup with tar + SHA256 checksum
✅ Incremental backup with tar --listed-incremental
✅ Rsync backup with hard-link dedup (space efficient)
✅ Retention cleanup by age
✅ MySQL full backup (mysqldump + gzip + optional AES encryption)
✅ PostgreSQL custom-format backup
✅ Restore with checksum verification
✅ Point-in-time recovery via closest backup before target time
✅ Remote backup via SSH+rsync and AWS S3
✅ DR runbook: pre-check, service restore, full system recovery with status file

---

**→ Part 66: CI/CD Pipeline Automation**
