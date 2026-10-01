# Part 29: Backup & Disaster Recovery
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 29.1 Backup Strategies

```bash
# ─── Backup Types ─────────────────────────────────────────────
# Full backup: copy everything
# Incremental: only changed since last backup
# Differential: changed since last FULL backup
# Snapshot: point-in-time using hardlinks or LVM

# ─── rsync-based backup ───────────────────────────────────────
# Full backup
rsync -av --progress /source/ /backup/full/

# Incremental with hardlinks (similar to Time Machine)
rsync -av --link-dest=/backup/latest /source/ /backup/"$(date +%Y%m%d_%H%M%S)"/
ln -sfn /backup/"$(date +%Y%m%d_%H%M%S)" /backup/latest

# Remote backup
rsync -avz -e "ssh -i ~/.ssh/backup_key" /source/ backup@remote:/backup/

# Exclude patterns
rsync -av \
    --exclude='*.tmp' \
    --exclude='.git/' \
    --exclude='node_modules/' \
    --exclude='*.log' \
    /source/ /backup/
```

---

## 29.2 Complete Backup System

```bash
#!/bin/bash
# backup.sh - Production backup system

set -euo pipefail

# ─── Configuration ────────────────────────────────────────────
BACKUP_ROOT="/backup"
SOURCE_DIRS=("/etc" "/home" "/var/www" "/opt/myapp")
REMOTE_HOST="${BACKUP_REMOTE:-}"
REMOTE_PATH="${BACKUP_REMOTE_PATH:-/backup}"
RETENTION_DAILY=7
RETENTION_WEEKLY=4
RETENTION_MONTHLY=6
LOG_FILE="/var/log/backup.log"
NOTIFY_EMAIL="${NOTIFY_EMAIL:-}"
ENCRYPTION_KEY="${BACKUP_KEY:-}"

# ─── Logging ──────────────────────────────────────────────────
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
die() { log "ERROR: $*"; send_notification "FAILED" "$*"; exit 1; }

send_notification() {
    local status=$1
    local message=$2
    
    [[ -z "$NOTIFY_EMAIL" ]] && return
    
    echo "$message" | mail -s "[Backup $status] $(hostname) - $(date +%Y-%m-%d)" "$NOTIFY_EMAIL" 2>/dev/null || true
}

# ─── Directories ──────────────────────────────────────────────
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DATE_DAY=$(date +%Y%m%d)
DATE_WEEK=$(date +%Y_W%V)
DATE_MONTH=$(date +%Y%m)

BACKUP_DIR="$BACKUP_ROOT/daily/$DATE_DAY"
WORK_DIR="$BACKUP_ROOT/.working"

mkdir -p "$BACKUP_DIR" "$WORK_DIR"

# ─── Core backup function ─────────────────────────────────────
backup_files() {
    local name=$1
    local source=$2
    local dest="$BACKUP_DIR/${name}.tar.gz"
    
    log "Backing up $name: $source → $dest"
    
    local start_time
    start_time=$(date +%s)
    
    # Create archive
    if [[ -n "$ENCRYPTION_KEY" ]]; then
        tar -czf - "$source" 2>/dev/null | \
            gpg --symmetric --cipher-algo AES256 --passphrase "$ENCRYPTION_KEY" \
            --batch --yes -o "$dest.gpg"
        dest="${dest}.gpg"
    else
        tar -czf "$dest" "$source" 2>/dev/null
    fi
    
    local end_time
    end_time=$(date +%s)
    local size
    size=$(du -sh "$dest" | cut -f1)
    local duration=$(( end_time - start_time ))
    
    log "  Done: $size in ${duration}s"
    
    # Checksum
    sha256sum "$dest" >> "$BACKUP_DIR/checksums.sha256"
}

# ─── Database backups ─────────────────────────────────────────
backup_postgres() {
    local db=$1
    local dest="$BACKUP_DIR/db_${db}.sql.gz"
    
    log "Backing up PostgreSQL database: $db"
    
    pg_dump "$db" 2>/dev/null | gzip > "$dest"
    
    local size
    size=$(du -sh "$dest" | cut -f1)
    log "  PostgreSQL $db: $size"
}

backup_mysql() {
    local db=$1
    local dest="$BACKUP_DIR/db_${db}.sql.gz"
    
    log "Backing up MySQL database: $db"
    
    mysqldump "$db" 2>/dev/null | gzip > "$dest"
    
    local size
    size=$(du -sh "$dest" | cut -f1)
    log "  MySQL $db: $size"
}

# ─── Retention management ─────────────────────────────────────
apply_retention() {
    log "Applying retention policy..."
    
    # Daily: keep last N days
    find "$BACKUP_ROOT/daily" -maxdepth 1 -type d -name "[0-9]*" | \
        sort -r | tail -n "+$(( RETENTION_DAILY + 1 ))" | \
        xargs -r rm -rf
    
    # Weekly: keep last N weeks
    find "$BACKUP_ROOT/weekly" -maxdepth 1 -type d -name "*.W*" 2>/dev/null | \
        sort -r | tail -n "+$(( RETENTION_WEEKLY + 1 ))" | \
        xargs -r rm -rf
    
    # Monthly: keep last N months
    find "$BACKUP_ROOT/monthly" -maxdepth 1 -type d -name "[0-9]*" 2>/dev/null | \
        sort -r | tail -n "+$(( RETENTION_MONTHLY + 1 ))" | \
        xargs -r rm -rf
    
    log "Retention applied"
}

# Create weekly/monthly symlinks
create_time_snapshots() {
    # Weekly snapshot (every Sunday)
    if [[ "$(date +%u)" == "7" ]]; then
        local weekly_dir="$BACKUP_ROOT/weekly/$DATE_WEEK"
        mkdir -p "$BACKUP_ROOT/weekly"
        cp -al "$BACKUP_DIR" "$weekly_dir" 2>/dev/null || \
            rsync -a "$BACKUP_DIR/" "$weekly_dir/"
        log "Created weekly snapshot: $DATE_WEEK"
    fi
    
    # Monthly snapshot (first day of month)
    if [[ "$(date +%d)" == "01" ]]; then
        local monthly_dir="$BACKUP_ROOT/monthly/$DATE_MONTH"
        mkdir -p "$BACKUP_ROOT/monthly"
        cp -al "$BACKUP_DIR" "$monthly_dir" 2>/dev/null || \
            rsync -a "$BACKUP_DIR/" "$monthly_dir/"
        log "Created monthly snapshot: $DATE_MONTH"
    fi
}

# ─── Remote sync ──────────────────────────────────────────────
sync_to_remote() {
    [[ -z "$REMOTE_HOST" ]] && return
    
    log "Syncing to remote: $REMOTE_HOST:$REMOTE_PATH"
    
    rsync -az \
        --delete \
        --exclude='.working' \
        -e "ssh -o ConnectTimeout=30 -i ~/.ssh/backup_key" \
        "$BACKUP_ROOT/" \
        "${REMOTE_HOST}:${REMOTE_PATH}/" || \
        die "Remote sync failed"
    
    log "Remote sync complete"
}

# ─── Verify backup ────────────────────────────────────────────
verify_backup() {
    log "Verifying backup integrity..."
    
    local errors=0
    
    # Check checksums
    if [[ -f "$BACKUP_DIR/checksums.sha256" ]]; then
        cd "$BACKUP_DIR" && sha256sum -c checksums.sha256 2>/dev/null | \
            grep -v "OK" | while read -r line; do
                log "  CHECKSUM FAIL: $line"
                (( errors++ ))
            done
    fi
    
    # Check archives can be listed (not corrupted)
    while IFS= read -r archive; do
        if ! tar -tzf "$archive" &>/dev/null 2>&1; then
            log "  ARCHIVE FAIL: $archive"
            (( errors++ ))
        fi
    done < <(find "$BACKUP_DIR" -name "*.tar.gz")
    
    if (( errors > 0 )); then
        die "Backup verification failed: $errors errors"
    fi
    
    log "Backup verified OK"
}

# ─── Main ─────────────────────────────────────────────────────
main() {
    local start_time
    start_time=$(date +%s)
    
    log "=== Backup Started: $TIMESTAMP ==="
    
    # Backup files
    for dir in "${SOURCE_DIRS[@]}"; do
        [[ -d "$dir" ]] || { log "Skip (not found): $dir"; continue; }
        local name
        name=$(echo "$dir" | tr '/' '_' | sed 's/^_//')
        backup_files "$name" "$dir"
    done
    
    # Backup databases
    # backup_postgres myapp
    # backup_mysql myapp
    
    # Verify
    verify_backup
    
    # Snapshots and retention
    create_time_snapshots
    apply_retention
    
    # Remote sync
    sync_to_remote
    
    local end_time
    end_time=$(date +%s)
    local total_size
    total_size=$(du -sh "$BACKUP_DIR" | cut -f1)
    local duration=$(( end_time - start_time ))
    
    log "=== Backup Complete: ${total_size} in ${duration}s ==="
    
    send_notification "SUCCESS" "Backup complete: $total_size in ${duration}s"
}

main
```

---

## 29.3 Restore Procedures

```bash
#!/bin/bash
# restore.sh - Restore from backup

set -euo pipefail

BACKUP_ROOT="/backup"

list_backups() {
    echo "Available backups:"
    echo ""
    echo "Daily:"
    ls -1 "$BACKUP_ROOT/daily/" 2>/dev/null | sort -r | head -10
    echo ""
    echo "Weekly:"
    ls -1 "$BACKUP_ROOT/weekly/" 2>/dev/null | sort -r | head -5
    echo ""
    echo "Monthly:"
    ls -1 "$BACKUP_ROOT/monthly/" 2>/dev/null | sort -r | head -6
}

restore_files() {
    local backup_date=$1
    local target_file=$2
    local restore_to=${3:-.}
    
    local backup_dir
    backup_dir=$(find "$BACKUP_ROOT" -type d -name "$backup_date*" | head -1)
    
    [[ -z "$backup_dir" ]] && die "Backup not found: $backup_date"
    
    local archive
    archive=$(find "$backup_dir" -name "${target_file}*.tar.gz" | head -1)
    
    [[ -z "$archive" ]] && die "Archive not found: $target_file"
    
    echo "Restoring from: $archive"
    echo "Destination: $restore_to"
    
    read -rp "Continue? [y/N] " confirm
    [[ "$confirm" =~ ^[Yy] ]] || exit 0
    
    mkdir -p "$restore_to"
    tar -xzf "$archive" -C "$restore_to"
    
    echo "Restore complete"
}

restore_database() {
    local backup_date=$1
    local db_name=$2
    local target_db=${3:-$db_name}
    
    local backup_dir
    backup_dir=$(find "$BACKUP_ROOT" -type d -name "$backup_date*" | head -1)
    
    local dump_file
    dump_file=$(find "$backup_dir" -name "db_${db_name}.sql.gz" | head -1)
    
    [[ -z "$dump_file" ]] && die "Database backup not found"
    
    echo "Restoring $db_name → $target_db from $dump_file"
    
    read -rp "This will overwrite $target_db. Continue? [y/N] " confirm
    [[ "$confirm" =~ ^[Yy] ]] || exit 0
    
    zcat "$dump_file" | psql "$target_db"
    echo "Database restore complete"
}

# ─── Test restore ─────────────────────────────────────────────
test_restore() {
    local backup_date=${1:-$(ls "$BACKUP_ROOT/daily" | sort -r | head -1)}
    local test_dir="/tmp/restore_test_$$"
    
    echo "Testing restore: $backup_date → $test_dir"
    
    mkdir -p "$test_dir"
    
    while IFS= read -r archive; do
        local name
        name=$(basename "$archive" .tar.gz)
        echo "Testing $name..."
        
        if tar -tzf "$archive" &>/dev/null; then
            echo "  ✓ Archive valid"
        else
            echo "  ✗ Archive corrupted: $archive"
        fi
    done < <(find "$BACKUP_ROOT/daily/$backup_date" -name "*.tar.gz")
    
    rm -rf "$test_dir"
    echo "Restore test complete"
}

case "${1:-list}" in
    list)     list_backups ;;
    restore)  restore_files "$2" "$3" "${4:-/tmp/restore}" ;;
    db)       restore_database "$2" "$3" "${4:-}" ;;
    test)     test_restore "${2:-}" ;;
    *)        echo "Usage: $0 [list|restore DATE FILE [DEST]|db DATE NAME|test [DATE]]" ;;
esac
```

---

## 29.4 Disaster Recovery Plan Script

```bash
#!/bin/bash
# dr_test.sh - Test disaster recovery procedures

echo "=== Disaster Recovery Test ==="
echo "Date: $(date)"
echo ""

# ─── Check backup integrity ───────────────────────────────────
check_backups() {
    echo "1. Checking backup files..."
    local ok=0 failed=0
    
    while IFS= read -r archive; do
        if tar -tzf "$archive" &>/dev/null; then
            (( ok++ ))
        else
            echo "  FAIL: $archive"
            (( failed++ ))
        fi
    done < <(find /backup/daily -name "*.tar.gz" -mtime -1)
    
    echo "  OK: $ok, Failed: $failed"
}

# ─── Verify checksums ─────────────────────────────────────────
verify_checksums() {
    echo "2. Verifying checksums..."
    local latest
    latest=$(ls -t /backup/daily | head -1)
    
    if [[ -f "/backup/daily/$latest/checksums.sha256" ]]; then
        cd "/backup/daily/$latest"
        sha256sum -c checksums.sha256 --quiet 2>&1 | \
            grep -v "OK" || echo "  All checksums OK"
    else
        echo "  No checksums file found"
    fi
}

# ─── Estimate restore time ────────────────────────────────────
estimate_restore() {
    echo "3. Estimating restore time..."
    local total_size
    total_size=$(du -sh /backup/daily | head -1 | cut -f1)
    echo "  Latest backup size: $total_size"
    
    # Test extract speed
    local test_archive
    test_archive=$(find /backup/daily -name "*.tar.gz" | head -1)
    if [[ -n "$test_archive" ]]; then
        local start
        start=$(date +%s%N)
        tar -tzf "$test_archive" &>/dev/null
        local end
        end=$(date +%s%N)
        echo "  Archive list time: $(( (end-start)/1000000 ))ms"
    fi
}

# ─── Check remote backup ──────────────────────────────────────
check_remote() {
    echo "4. Checking remote backup..."
    local remote="${BACKUP_REMOTE:-}"
    
    if [[ -z "$remote" ]]; then
        echo "  No remote configured"
        return
    fi
    
    if ssh -o ConnectTimeout=10 "$remote" "ls /backup/daily | tail -1"; then
        echo "  Remote backup accessible"
    else
        echo "  FAIL: Cannot access remote backup"
    fi
}

check_backups
verify_checksums
estimate_restore
check_remote

echo ""
echo "DR test complete: $(date)"
```

---

## 29.5 Exercises

### Exercise 1: S3 Backup Integration
สร้าง script ที่ backup ไปยัง AWS S3:
- Incremental backup
- Lifecycle rules
- Cost estimation
- Restore from S3

### Exercise 2: Encrypted Backup System
สร้าง encrypted backup:
- GPG asymmetric encryption
- Key rotation
- Backup key management
- Verify without decrypting

### Exercise 3: RTO/RPO Measurement
สร้าง tool ที่วัด:
- Recovery Time Objective (RTO)
- Recovery Point Objective (RPO)
- Test restore periodically
- Report to dashboard

---

## สรุป Part 29

✅ Backup strategies (full, incremental, differential)  
✅ rsync-based backup with hardlinks  
✅ Complete backup system script  
✅ Database backups (PostgreSQL, MySQL)  
✅ Retention policies (daily/weekly/monthly)  
✅ Remote sync  
✅ Backup verification  
✅ Restore procedures  
✅ DR test automation  

---

**→ Part 30: Security Hardening Scripts**
