# Part 58: Database Operations and Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 58.1 MySQL/MariaDB Operations

```bash
#!/bin/bash
# mysql_ops.sh - MySQL/MariaDB automation

MYSQL_HOST="${MYSQL_HOST:-localhost}"
MYSQL_PORT="${MYSQL_PORT:-3306}"
MYSQL_USER="${MYSQL_USER:-root}"
MYSQL_PASS="${MYSQL_PASS:-}"
MYSQL_DB="${MYSQL_DB:-}"
BACKUP_DIR="${BACKUP_DIR:-/var/backups/mysql}"

# ─── Connection Helper ───────────────────────────────────────────
mysql_exec() {
    local query=$1 db=${2:-$MYSQL_DB}

    local args=(
        "--host=$MYSQL_HOST"
        "--port=$MYSQL_PORT"
        "--user=$MYSQL_USER"
        "--batch"
        "--silent"
    )

    [[ -n "$MYSQL_PASS" ]] && args+=("--password=$MYSQL_PASS")
    [[ -n "$db" ]] && args+=("$db")

    echo "$query" | mysql "${args[@]}" 2>/dev/null
}

mysql_exec_file() {
    local sql_file=$1 db=${2:-$MYSQL_DB}

    local args=(
        "--host=$MYSQL_HOST"
        "--port=$MYSQL_PORT"
        "--user=$MYSQL_USER"
    )

    [[ -n "$MYSQL_PASS" ]] && args+=("--password=$MYSQL_PASS")
    [[ -n "$db" ]] && args+=("$db")

    mysql "${args[@]}" < "$sql_file"
}

mysql_check_connection() {
    mysql_exec "SELECT 1" &>/dev/null && echo "OK" || echo "FAIL"
}

# ─── Database Management ─────────────────────────────────────────
mysql_create_database() {
    local db_name=$1 charset=${2:-utf8mb4} collation=${3:-utf8mb4_unicode_ci}

    mysql_exec "CREATE DATABASE IF NOT EXISTS \`${db_name}\`
        CHARACTER SET ${charset} COLLATE ${collation};"
    echo "Created database: $db_name"
}

mysql_create_user() {
    local username=$1 password=$2 host=${3:-localhost}

    mysql_exec "CREATE USER IF NOT EXISTS '${username}'@'${host}'
        IDENTIFIED BY '${password}';"
    echo "Created user: ${username}@${host}"
}

mysql_grant_privileges() {
    local username=$1 db=$2 host=${3:-localhost} privs=${4:-ALL PRIVILEGES}

    mysql_exec "GRANT ${privs} ON \`${db}\`.* TO '${username}'@'${host}'; FLUSH PRIVILEGES;"
    echo "Granted $privs on $db to ${username}@${host}"
}

mysql_list_databases() {
    mysql_exec "SHOW DATABASES;" | grep -v "^Database$"
}

mysql_list_tables() {
    local db=${1:-$MYSQL_DB}
    mysql_exec "SHOW TABLES;" "$db" | grep -v "^Tables_in"
}

mysql_table_size() {
    local db=${1:-$MYSQL_DB}

    mysql_exec "SELECT
        table_name AS 'Table',
        ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size_MB',
        table_rows AS 'Rows'
    FROM information_schema.TABLES
    WHERE table_schema = '${db}'
    ORDER BY (data_length + index_length) DESC;" "information_schema"
}

# ─── Backup and Restore ──────────────────────────────────────────
mysql_backup_database() {
    local db=${1:-$MYSQL_DB}
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$BACKUP_DIR"
    local backup_file="$BACKUP_DIR/${db}_${timestamp}.sql.gz"

    local dump_args=(
        "--host=$MYSQL_HOST"
        "--port=$MYSQL_PORT"
        "--user=$MYSQL_USER"
        "--single-transaction"
        "--routines"
        "--triggers"
        "--events"
        "--hex-blob"
    )
    [[ -n "$MYSQL_PASS" ]] && dump_args+=("--password=$MYSQL_PASS")

    mysqldump "${dump_args[@]}" "$db" | gzip > "$backup_file"

    local size; size=$(du -sh "$backup_file" | cut -f1)
    echo "Backup: $backup_file ($size)"
    echo "$backup_file"
}

mysql_restore_database() {
    local backup_file=$1 db=${2:-$MYSQL_DB}

    [[ ! -f "$backup_file" ]] && { echo "Backup not found: $backup_file"; return 1; }

    mysql_create_database "$db"

    local restore_args=(
        "--host=$MYSQL_HOST" "--port=$MYSQL_PORT"
        "--user=$MYSQL_USER" "$db"
    )
    [[ -n "$MYSQL_PASS" ]] && restore_args+=("--password=$MYSQL_PASS")

    if [[ "$backup_file" == *.gz ]]; then
        zcat "$backup_file" | mysql "${restore_args[@]}"
    else
        mysql "${restore_args[@]}" < "$backup_file"
    fi
    echo "Restore complete"
}

# ─── Maintenance ──────────────────────────────────────────────────
mysql_slow_queries() {
    local threshold_seconds=${1:-1} limit=${2:-20}

    mysql_exec "SELECT
        query_time, lock_time, rows_sent, rows_examined,
        SUBSTRING(sql_text, 1, 100) AS query
    FROM mysql.slow_log
    WHERE query_time >= '${threshold_seconds}'
    ORDER BY query_time DESC
    LIMIT ${limit};"
}

mysql_kill_long_queries() {
    local threshold_seconds=${1:-300}

    mysql_exec "SELECT id FROM information_schema.processlist
        WHERE command != 'Sleep'
        AND time > ${threshold_seconds}
        AND user != 'root';" | while read -r pid; do
            [[ "$pid" == "id" ]] && continue
            mysql_exec "KILL ${pid};"
            echo "Killed query PID: $pid"
        done
}
```

---

## 58.2 PostgreSQL Operations

```bash
#!/bin/bash
# pg_ops.sh - PostgreSQL automation

PG_HOST="${PG_HOST:-localhost}"
PG_PORT="${PG_PORT:-5432}"
PG_USER="${PG_USER:-postgres}"
PG_PASS="${PG_PASS:-}"
PG_DB="${PG_DB:-postgres}"
PG_BACKUP_DIR="${PG_BACKUP_DIR:-/var/backups/postgresql}"

# ─── Connection Helper ───────────────────────────────────────────
pg_exec() {
    local query=$1 db=${2:-$PG_DB}

    PGPASSWORD="$PG_PASS" psql \
        --host="$PG_HOST" --port="$PG_PORT" --username="$PG_USER" \
        --dbname="$db" --no-align --tuples-only \
        --command="$query" 2>/dev/null
}

pg_exec_file() {
    local sql_file=$1 db=${2:-$PG_DB}

    PGPASSWORD="$PG_PASS" psql \
        --host="$PG_HOST" --port="$PG_PORT" --username="$PG_USER" \
        --dbname="$db" --file="$sql_file" 2>/dev/null
}

pg_check_connection() {
    pg_exec "SELECT 1;" &>/dev/null && echo "OK" || echo "FAIL"
}

# ─── Database Management ─────────────────────────────────────────
pg_create_database() {
    local db_name=$1 owner=${2:-$PG_USER}
    pg_exec "CREATE DATABASE \"${db_name}\" OWNER \"${owner}\";" "postgres"
    echo "Created database: $db_name"
}

pg_create_user() {
    local username=$1 password=$2
    pg_exec "CREATE USER \"${username}\" WITH PASSWORD '${password}';" "postgres"
    echo "Created user: $username"
}

pg_grant_privileges() {
    local username=$1 db=$2
    pg_exec "GRANT CONNECT ON DATABASE \"${db}\" TO \"${username}\";" "postgres"
    pg_exec "GRANT USAGE ON SCHEMA public TO \"${username}\";" "$db"
    pg_exec "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"${username}\";" "$db"
    echo "Granted privileges to $username on $db"
}

pg_table_sizes() {
    local db=${1:-$PG_DB}

    pg_exec "SELECT
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total,
        pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
        pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS index_size
    FROM pg_tables
    WHERE schemaname = 'public'
    ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;" "$db"
}

# ─── Backup and Restore ──────────────────────────────────────────
pg_backup_database() {
    local db=${1:-$PG_DB}
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$PG_BACKUP_DIR"
    local backup_file="$PG_BACKUP_DIR/${db}_${timestamp}.dump"

    PGPASSWORD="$PG_PASS" pg_dump \
        --host="$PG_HOST" --port="$PG_PORT" --username="$PG_USER" \
        --format=custom --compress=9 --file="$backup_file" "$db"

    local size; size=$(du -sh "$backup_file" | cut -f1)
    echo "Backup: $backup_file ($size)"
    echo "$backup_file"
}

pg_restore_database() {
    local backup_file=$1 db=${2:-$PG_DB}

    [[ ! -f "$backup_file" ]] && { echo "Backup not found: $backup_file"; return 1; }

    pg_create_database "$db" 2>/dev/null || true

    PGPASSWORD="$PG_PASS" pg_restore \
        --host="$PG_HOST" --port="$PG_PORT" --username="$PG_USER" \
        --dbname="$db" --no-owner --no-privileges "$backup_file"

    echo "Restore complete: $db"
}

# ─── Performance Analysis ────────────────────────────────────────
pg_slow_queries() {
    local min_ms=${1:-1000}

    pg_exec "SELECT
        round(total_exec_time::numeric, 2) AS total_ms,
        calls,
        round(mean_exec_time::numeric, 2) AS mean_ms,
        substring(query, 1, 80) AS query
    FROM pg_stat_statements
    WHERE mean_exec_time > ${min_ms}
    ORDER BY mean_exec_time DESC
    LIMIT 20;"
}

pg_blocking_queries() {
    pg_exec "SELECT
        blocked.pid AS blocked_pid,
        blocking.pid AS blocking_pid,
        substring(blocked.query, 1, 60) AS blocked_query,
        substring(blocking.query, 1, 60) AS blocking_query
    FROM pg_stat_activity blocked
    JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
    WHERE blocked.state = 'active';"
}

pg_vacuum_stats() {
    local db=${1:-$PG_DB}

    pg_exec "SELECT schemaname, tablename, n_dead_tup, n_live_tup,
        last_vacuum, last_autovacuum
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 1000
    ORDER BY n_dead_tup DESC
    LIMIT 20;" "$db"
}
```

---

## 58.3 Redis Operations

```bash
#!/bin/bash
# redis_ops.sh - Redis automation

REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_PASS="${REDIS_PASS:-}"
REDIS_DB="${REDIS_DB:-0}"

# ─── Connection Helper ───────────────────────────────────────────
redis_exec() {
    local cmd=$1; shift

    local args=("--host" "$REDIS_HOST" "--port" "$REDIS_PORT" "--no-auth-warning")
    [[ -n "$REDIS_PASS" ]] && args+=("-a" "$REDIS_PASS")
    [[ "$REDIS_DB" != "0" ]] && args+=("-n" "$REDIS_DB")

    redis-cli "${args[@]}" "$cmd" "$@" 2>/dev/null
}

redis_ping() { [[ "$(redis_exec PING)" == "PONG" ]]; }

# ─── Key Management ───────────────────────────────────────────────
redis_scan_keys() {
    local pattern=${1:-*} count=${2:-100}
    local cursor=0

    while true; do
        local result
        result=$(redis_exec SCAN "$cursor" MATCH "$pattern" COUNT "$count")
        cursor=$(echo "$result" | head -1)
        echo "$result" | tail -n +2 | grep -v '^$'
        [[ "$cursor" == "0" ]] && break
    done
}

redis_expire_bulk() {
    local pattern=$1 ttl_seconds=$2
    local count=0

    while IFS= read -r key; do
        redis_exec EXPIRE "$key" "$ttl_seconds" &>/dev/null
        (( count++ ))
    done < <(redis_scan_keys "$pattern")

    echo "Set TTL ${ttl_seconds}s on $count keys matching: $pattern"
}

redis_delete_pattern() {
    local pattern=$1
    local count=0

    while IFS= read -r key; do
        redis_exec DEL "$key" &>/dev/null
        (( count++ ))
    done < <(redis_scan_keys "$pattern")

    echo "Deleted $count keys matching: $pattern"
}

# ─── Monitoring ───────────────────────────────────────────────────
redis_memory_stats() {
    redis_exec INFO memory | awk -F: '
    /used_memory_human/      { printf "Used:  %s", $2 }
    /used_memory_peak_human/ { printf "Peak:  %s", $2 }
    /mem_fragmentation_ratio/{ printf "Frag:  %s", $2 }
    /maxmemory_human/        { printf "Max:   %s", $2 }
    '
}

redis_slowlog_get() {
    local count=${1:-10}
    redis_exec SLOWLOG GET "$count"
}

# ─── Backup ───────────────────────────────────────────────────────
redis_bgsave() {
    redis_exec BGSAVE
    local timeout=60 elapsed=0

    while (( elapsed < timeout )); do
        local in_progress
        in_progress=$(redis_exec INFO persistence | grep 'rdb_bgsave_in_progress' | \
            cut -d: -f2 | tr -d '[:space:]')
        [[ "$in_progress" == "0" ]] && { echo "Save complete"; return 0; }
        sleep 2; (( elapsed += 2 ))
    done

    echo "Save timeout"; return 1
}

redis_backup_rdb() {
    local backup_dir=${1:-/var/backups/redis}
    local rdb_path
    rdb_path="$(redis_exec CONFIG GET dir | tail -1)/$(redis_exec CONFIG GET dbfilename | tail -1)"

    mkdir -p "$backup_dir"
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="$backup_dir/dump_${timestamp}.rdb"

    redis_bgsave
    cp "$rdb_path" "$backup_file"
    echo "Redis backup: $backup_file"
    echo "$backup_file"
}
```

---

## 58.4 Database Migration Framework

```bash
#!/bin/bash
# db_migrate.sh - Database migration management

MIGRATIONS_DIR="${MIGRATIONS_DIR:-./migrations}"
MIGRATION_TABLE="${MIGRATION_TABLE:-schema_migrations}"

# ─── Migration Tracking ─────────────────────────────────────────
migration_init() {
    local db_type=${1:-mysql}

    case "$db_type" in
        mysql)
            mysql_exec "CREATE TABLE IF NOT EXISTS ${MIGRATION_TABLE} (
                version VARCHAR(255) PRIMARY KEY,
                applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );" ;;
        postgres)
            pg_exec "CREATE TABLE IF NOT EXISTS ${MIGRATION_TABLE} (
                version VARCHAR(255) PRIMARY KEY,
                applied_at TIMESTAMPTZ DEFAULT NOW()
            );" ;;
    esac
    echo "Migration table initialized"
}

migration_is_applied() {
    local version=$1 db_type=${2:-mysql}
    local count

    case "$db_type" in
        mysql)    count=$(mysql_exec "SELECT COUNT(*) FROM ${MIGRATION_TABLE} WHERE version='${version}';") ;;
        postgres) count=$(pg_exec "SELECT COUNT(*) FROM ${MIGRATION_TABLE} WHERE version='${version}';") ;;
    esac

    (( count > 0 ))
}

migration_record() {
    local version=$1 db_type=${2:-mysql}

    case "$db_type" in
        mysql)    mysql_exec "INSERT INTO ${MIGRATION_TABLE} (version) VALUES ('${version}');" ;;
        postgres) pg_exec "INSERT INTO ${MIGRATION_TABLE} (version) VALUES ('${version}');" ;;
    esac
}

migration_run_pending() {
    local db_type=${1:-mysql}
    local applied=0 failed=0

    migration_init "$db_type"

    for migration_file in $(ls "$MIGRATIONS_DIR"/*.sql 2>/dev/null | sort); do
        local version; version=$(basename "$migration_file" .sql)

        migration_is_applied "$version" "$db_type" && continue

        echo "Applying: $version"
        local success=false

        case "$db_type" in
            mysql)    mysql_exec_file "$migration_file" && success=true ;;
            postgres) pg_exec_file "$migration_file" && success=true ;;
        esac

        if $success; then
            migration_record "$version" "$db_type"
            (( applied++ ))
            echo "  Applied: $version"
        else
            (( failed++ ))
            echo "  FAILED: $version"
            break
        fi
    done

    echo "Migrations: $applied applied, $failed failed"
    return $(( failed > 0 ))
}

migration_status() {
    local db_type=${1:-mysql}
    echo "=== Migration Status ==="

    for migration_file in $(ls "$MIGRATIONS_DIR"/*.sql 2>/dev/null | sort); do
        local version; version=$(basename "$migration_file" .sql)
        migration_is_applied "$version" "$db_type" && \
            printf "  [APPLIED] %s\n" "$version" || \
            printf "  [PENDING] %s\n" "$version"
    done
}
```

---

## 58.5 Exercises

### Exercise 1: Automated Backup System
สร้าง backup system ที่:
- Backup MySQL + PostgreSQL + Redis
- Compress และ encrypt backups
- Upload to S3/remote storage
- Verify backup integrity

### Exercise 2: Database Health Monitor
สร้าง monitor ที่:
- Check connection health
- Track slow queries
- Alert on deadlocks
- Monitor replication lag

### Exercise 3: Migration Framework
สร้าง migration system ที่:
- Track applied migrations
- Support rollback
- Handle concurrent deployments
- Generate migration diffs

---

## สรุป Part 58

✅ MySQL: exec helper, CRUD databases/users, backup/restore with mysqldump
✅ MySQL: table size analysis, slow query log, kill long queries
✅ PostgreSQL: psql wrapper, privilege grants, pg_dump/pg_restore
✅ PostgreSQL: slow queries, blocking queries, vacuum stats
✅ Redis: SCAN-based key iteration, bulk TTL/delete
✅ Redis: memory stats, slowlog, BGSAVE with wait, RDB backup
✅ Migration framework: version tracking, pending run, status report

---

**→ Part 59: Log Analysis and Event Processing**
