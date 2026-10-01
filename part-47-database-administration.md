# Part 47: Database Administration Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 47.1 MySQL/MariaDB Administration

```bash
#!/bin/bash
# mysql_admin.sh - MySQL/MariaDB administration automation

MYSQL_ADMIN_USER="${MYSQL_ADMIN_USER:-root}"
MYSQL_ADMIN_PASS="${MYSQL_ADMIN_PASS:-}"
MYSQL_HOST="${MYSQL_HOST:-localhost}"
MYSQL_PORT="${MYSQL_PORT:-3306}"

mysql_exec() {
    local query=$1
    local database=${2:-}
    local format=${3:---batch --skip-column-names}

    local mysql_args=(
        --host="$MYSQL_HOST"
        --port="$MYSQL_PORT"
        --user="$MYSQL_ADMIN_USER"
        --silent
    )

    [[ -n "$MYSQL_ADMIN_PASS" ]] && mysql_args+=(--password="$MYSQL_ADMIN_PASS")
    [[ -n "$database" ]] && mysql_args+=(--database="$database")

    echo "$query" | mysql "${mysql_args[@]}" $format
}

# ─── User Management ───────────────────────────────────────────
mysql_create_user() {
    local username=$1
    local password=$2
    local host=${3:-localhost}
    local databases=${4:-}

    mysql_exec "CREATE USER IF NOT EXISTS '${username}'@'${host}' IDENTIFIED BY '${password}';"
    echo "Created MySQL user: ${username}@${host}"

    if [[ -n "$databases" ]]; then
        IFS=',' read -ra dbs <<< "$databases"
        for db in "${dbs[@]}"; do
            mysql_exec "GRANT ALL PRIVILEGES ON \`${db}\`.* TO '${username}'@'${host}';"
            echo "Granted access to: $db"
        done
        mysql_exec "FLUSH PRIVILEGES;"
    fi
}

mysql_drop_user() {
    local username=$1
    local host=${2:-localhost}
    mysql_exec "DROP USER IF EXISTS '${username}'@'${host}';"
    mysql_exec "FLUSH PRIVILEGES;"
    echo "Dropped MySQL user: ${username}@${host}"
}

mysql_list_users() {
    mysql_exec "SELECT User, Host, authentication_string != '' AS has_password
                FROM mysql.user
                ORDER BY User, Host;" \
        "" "--table"
}

# ─── Database Management ───────────────────────────────────────
mysql_create_db() {
    local dbname=$1
    local charset=${2:-utf8mb4}
    local collation=${3:-utf8mb4_unicode_ci}

    mysql_exec "CREATE DATABASE IF NOT EXISTS \`${dbname}\`
                CHARACTER SET ${charset}
                COLLATE ${collation};"
    echo "Created database: $dbname"
}

mysql_drop_db() {
    local dbname=$1
    local confirm=${2:-false}

    if ! $confirm; then
        read -r -p "Drop database '$dbname'? [y/N] " response
        [[  "${response,,}" != "y" ]] && { echo "Aborted"; return 0; }
    fi

    mysql_exec "DROP DATABASE IF EXISTS \`${dbname}\`;"
    echo "Dropped database: $dbname"
}

mysql_copy_db() {
    local src=$1
    local dst=$2

    echo "Copying $src → $dst..."
    mysql_create_db "$dst"
    mysqldump --host="$MYSQL_HOST" --user="$MYSQL_ADMIN_USER" \
        ${MYSQL_ADMIN_PASS:+--password="$MYSQL_ADMIN_PASS"} \
        --single-transaction --routines --triggers --events \
        "$src" | mysql --host="$MYSQL_HOST" --user="$MYSQL_ADMIN_USER" \
        ${MYSQL_ADMIN_PASS:+--password="$MYSQL_ADMIN_PASS"} "$dst"
    echo "Copy complete"
}

# ─── Performance Analysis ──────────────────────────────────────
mysql_slow_queries() {
    local threshold=${1:-1}
    local limit=${2:-20}

    mysql_exec "SELECT
        ROUND(timer_wait/1e12, 3) AS exec_time_s,
        sql_text
      FROM performance_schema.events_statements_history_long
      WHERE timer_wait/1e12 > ${threshold}
      ORDER BY timer_wait DESC
      LIMIT ${limit};" "" "--table"
}

mysql_table_sizes() {
    local database=$1
    local limit=${2:-20}

    mysql_exec "SELECT
        table_name,
        ROUND(data_length/1024/1024, 2) AS data_mb,
        ROUND(index_length/1024/1024, 2) AS index_mb,
        ROUND((data_length+index_length)/1024/1024, 2) AS total_mb,
        table_rows
      FROM information_schema.tables
      WHERE table_schema = '${database}'
      ORDER BY total_mb DESC
      LIMIT ${limit};" "" "--table"
}

mysql_index_usage() {
    local database=$1

    mysql_exec "SELECT
        t.TABLE_SCHEMA,
        t.TABLE_NAME,
        s.INDEX_NAME,
        s.COLUMN_NAME,
        s.CARDINALITY
      FROM information_schema.tables t
      JOIN information_schema.statistics s
        ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
        AND t.TABLE_NAME = s.TABLE_NAME
      WHERE t.TABLE_SCHEMA = '${database}'
        AND s.INDEX_NAME != 'PRIMARY'
      ORDER BY s.CARDINALITY DESC;" "" "--table"
}

mysql_replication_status() {
    echo "=== Replication Status ==="
    mysql_exec "SHOW SLAVE STATUS\G" 2>/dev/null || \
    mysql_exec "SHOW REPLICA STATUS\G" 2>/dev/null || \
        echo "Not a replica"
}

# ─── Backup with Verification ──────────────────────────────────
mysql_backup_verified() {
    local database=$1
    local backup_dir=${2:-/var/backups/mysql}
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="${backup_dir}/${database}_${timestamp}.sql.gz"

    mkdir -p "$backup_dir"

    echo "Backing up: $database → $backup_file"
    mysqldump \
        --host="$MYSQL_HOST" \
        --user="$MYSQL_ADMIN_USER" \
        ${MYSQL_ADMIN_PASS:+--password="$MYSQL_ADMIN_PASS"} \
        --single-transaction \
        --routines \
        --triggers \
        --events \
        --master-data=2 \
        "$database" | gzip > "$backup_file"

    local checksum_file="${backup_file}.sha256"
    sha256sum "$backup_file" > "$checksum_file"

    echo "Backup: $backup_file ($(du -sh "$backup_file" | cut -f1))"
    echo "Checksum: $checksum_file"

    zcat "$backup_file" | head -20 | grep -q "MySQL dump" && \
        echo "Verification: OK" || echo "Verification: FAILED"
}
```

---

## 47.2 PostgreSQL Administration

```bash
#!/bin/bash
# postgres_admin.sh - PostgreSQL administration automation

PGHOST="${PGHOST:-localhost}"
PGPORT="${PGPORT:-5432}"
PGUSER="${PGUSER:-postgres}"
PGPASSWORD="${PGPASSWORD:-}"
export PGPASSWORD

pg_exec() {
    local query=$1
    local database=${2:-postgres}
    local format=${3:---tuples-only --no-align}

    psql \
        --host="$PGHOST" \
        --port="$PGPORT" \
        --username="$PGUSER" \
        --dbname="$database" \
        $format \
        --command="$query"
}

# ─── User/Role Management ──────────────────────────────────────
pg_create_role() {
    local rolename=$1
    local password=$2
    local superuser=${3:-false}
    local create_db=${4:-false}

    local role_opts="LOGIN PASSWORD '${password}'"
    $superuser && role_opts+=" SUPERUSER"
    $create_db && role_opts+=" CREATEDB"

    pg_exec "DO \$\$
    BEGIN
      IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = '${rolename}') THEN
        CREATE ROLE ${rolename} ${role_opts};
      END IF;
    END\$\$;"
    echo "Created role: $rolename"
}

pg_grant_schema() {
    local role=$1
    local database=$2
    local schema=${3:-public}

    pg_exec "GRANT CONNECT ON DATABASE \"${database}\" TO ${role};" postgres
    pg_exec "GRANT USAGE ON SCHEMA ${schema} TO ${role};" "$database"
    pg_exec "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA ${schema} TO ${role};" "$database"
    pg_exec "ALTER DEFAULT PRIVILEGES IN SCHEMA ${schema} GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO ${role};" "$database"
    echo "Granted schema access: $role on $database.$schema"
}

# ─── Performance Monitoring ────────────────────────────────────
pg_active_queries() {
    local min_duration=${1:-5}

    pg_exec "SELECT
        pid,
        now() - query_start AS duration,
        state,
        left(query, 80) AS query
      FROM pg_stat_activity
      WHERE state != 'idle'
        AND query_start < now() - interval '${min_duration} seconds'
      ORDER BY duration DESC;" "" "--expanded"
}

pg_table_bloat() {
    local database=$1

    pg_exec "SELECT
        schemaname,
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
        pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
        pg_size_pretty(
          pg_total_relation_size(schemaname||'.'||tablename) -
          pg_relation_size(schemaname||'.'||tablename)
        ) AS index_size
      FROM pg_tables
      WHERE schemaname = 'public'
      ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
      LIMIT 20;" "$database" "--expanded"
}

pg_lock_monitor() {
    pg_exec "SELECT
        pid,
        usename,
        pg_blocking_pids(pid) AS blocked_by,
        query,
        state
      FROM pg_stat_activity
      WHERE cardinality(pg_blocking_pids(pid)) > 0;" "" "--expanded"
}

pg_vacuum_stats() {
    local database=$1

    pg_exec "SELECT
        schemaname,
        relname,
        n_live_tup,
        n_dead_tup,
        last_vacuum,
        last_autovacuum,
        last_analyze
      FROM pg_stat_user_tables
      ORDER BY n_dead_tup DESC
      LIMIT 20;" "$database" "--expanded"
}

# ─── Connection Pool Analysis ──────────────────────────────────
pg_connection_stats() {
    pg_exec "SELECT
        datname AS database,
        count(*) AS connections,
        count(*) FILTER (WHERE state = 'active') AS active,
        count(*) FILTER (WHERE state = 'idle') AS idle,
        count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx
      FROM pg_stat_activity
      WHERE datname IS NOT NULL
      GROUP BY datname
      ORDER BY connections DESC;" "" "--expanded"
}

pg_kill_idle_connections() {
    local database=$1
    local idle_minutes=${2:-30}

    pg_exec "SELECT pg_terminate_backend(pid)
      FROM pg_stat_activity
      WHERE datname = '${database}'
        AND state = 'idle'
        AND query_start < now() - interval '${idle_minutes} minutes';"
    echo "Killed idle connections in: $database"
}
```

---

## 47.3 Database Migration Framework

```bash
#!/bin/bash
# db_migrations.sh - Database migration system

MIGRATIONS_DIR="${MIGRATIONS_DIR:-./migrations}"
MIGRATIONS_TABLE="${MIGRATIONS_TABLE:-schema_migrations}"

migration_init() {
    local db_type=$1
    local database=$2

    case "$db_type" in
        mysql)
            mysql_exec "CREATE TABLE IF NOT EXISTS \`${MIGRATIONS_TABLE}\` (
                id INT AUTO_INCREMENT PRIMARY KEY,
                version VARCHAR(255) NOT NULL UNIQUE,
                applied_at DATETIME DEFAULT CURRENT_TIMESTAMP
            );" "$database"
            ;;
        postgres)
            pg_exec "CREATE TABLE IF NOT EXISTS ${MIGRATIONS_TABLE} (
                id SERIAL PRIMARY KEY,
                version VARCHAR(255) NOT NULL UNIQUE,
                applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );" "$database"
            ;;
    esac
    echo "Initialized migration table: $MIGRATIONS_TABLE"
}

get_applied_migrations() {
    local db_type=$1
    local database=$2

    case "$db_type" in
        mysql)    mysql_exec "SELECT version FROM \`${MIGRATIONS_TABLE}\` ORDER BY version;" "$database" ;;
        postgres) pg_exec "SELECT version FROM ${MIGRATIONS_TABLE} ORDER BY version;" "$database" ;;
    esac
}

run_migration() {
    local db_type=$1
    local database=$2
    local version=$3
    local sql_file=$4

    echo "Applying migration: $version"

    case "$db_type" in
        mysql)
            mysql --host="$MYSQL_HOST" --user="$MYSQL_ADMIN_USER" \
                ${MYSQL_ADMIN_PASS:+--password="$MYSQL_ADMIN_PASS"} \
                "$database" < "$sql_file" || return 1

            mysql_exec "INSERT INTO \`${MIGRATIONS_TABLE}\` (version) VALUES ('${version}');" "$database"
            ;;
        postgres)
            psql --host="$PGHOST" --username="$PGUSER" --dbname="$database" \
                --file="$sql_file" || return 1

            pg_exec "INSERT INTO ${MIGRATIONS_TABLE} (version) VALUES ('${version}');" "$database"
            ;;
    esac

    echo "Applied: $version"
}

migrate() {
    local db_type=$1
    local database=$2
    local target=${3:-latest}

    migration_init "$db_type" "$database"

    declare -A applied=()
    while IFS= read -r version; do
        applied["$version"]=1
    done < <(get_applied_migrations "$db_type" "$database")

    local pending=0

    find "$MIGRATIONS_DIR" -name "*.sql" | sort | while IFS= read -r migration_file; do
        local version
        version=$(basename "$migration_file" .sql)

        [[ -v "applied[$version]" ]] && continue

        [[  "$target" != "latest" && "$version" > "$target" ]] && break

        run_migration "$db_type" "$database" "$version" "$migration_file" || {
            echo "Migration failed: $version" >&2
            exit 1
        }
        (( pending++ ))
    done

    echo "Migration complete"
}

rollback_migration() {
    local db_type=$1
    local database=$2
    local version=$3

    local rollback_file="${MIGRATIONS_DIR}/${version}.down.sql"

    [[ ! -f "$rollback_file" ]] && { echo "No rollback file: $rollback_file" >&2; return 1; }

    echo "Rolling back: $version"

    case "$db_type" in
        mysql)
            mysql --host="$MYSQL_HOST" --user="$MYSQL_ADMIN_USER" \
                ${MYSQL_ADMIN_PASS:+--password="$MYSQL_ADMIN_PASS"} \
                "$database" < "$rollback_file" || return 1
            mysql_exec "DELETE FROM \`${MIGRATIONS_TABLE}\` WHERE version='${version}';" "$database"
            ;;
        postgres)
            psql --host="$PGHOST" --username="$PGUSER" --dbname="$database" \
                --file="$rollback_file" || return 1
            pg_exec "DELETE FROM ${MIGRATIONS_TABLE} WHERE version='${version}';" "$database"
            ;;
    esac

    echo "Rolled back: $version"
}

create_migration() {
    local name=$1
    local timestamp
    timestamp=$(date +%Y%m%d%H%M%S)
    local version="${timestamp}_${name}"

    mkdir -p "$MIGRATIONS_DIR"

    cat > "${MIGRATIONS_DIR}/${version}.sql" << EOF
-- Migration: ${name}
-- Created: $(date -u +%Y-%m-%dT%H:%M:%SZ)

-- Write your UP migration here

EOF

    cat > "${MIGRATIONS_DIR}/${version}.down.sql" << EOF
-- Rollback: ${name}

-- Write your DOWN migration here

EOF

    echo "Created migration: $version"
}
```

---

## 47.4 Database Health Monitoring

```bash
#!/bin/bash
# db_health.sh - Database health monitoring

mysql_health_check() {
    local checks_passed=0
    local checks_failed=0

    check_pass() { echo "  [OK] $1"; (( checks_passed++ )); }
    check_fail() { echo "  [FAIL] $1" >&2; (( checks_failed++ )); }

    echo "=== MySQL Health Check ==="

    if mysql_exec "SELECT 1;" &>/dev/null; then
        check_pass "Connection"
    else
        check_fail "Connection - cannot connect"
        return 1
    fi

    local uptime
    uptime=$(mysql_exec "SHOW STATUS LIKE 'Uptime';" | awk '{print $2}')
    check_pass "Uptime: ${uptime}s"

    local max_conn curr_conn
    max_conn=$(mysql_exec "SHOW VARIABLES LIKE 'max_connections';" | awk '{print $2}')
    curr_conn=$(mysql_exec "SHOW STATUS LIKE 'Threads_connected';" | awk '{print $2}')
    local conn_pct=$(( curr_conn * 100 / max_conn ))

    if (( conn_pct < 80 )); then
        check_pass "Connections: ${curr_conn}/${max_conn} (${conn_pct}%)"
    else
        check_fail "Connections: ${curr_conn}/${max_conn} (${conn_pct}%)"
    fi

    local replication_running
    replication_running=$(mysql_exec "SHOW SLAVE STATUS\G" 2>/dev/null | \
        grep "Slave_IO_Running: Yes" | wc -l)
    if (( replication_running > 0 )); then
        local lag
        lag=$(mysql_exec "SHOW SLAVE STATUS\G" 2>/dev/null | \
            grep "Seconds_Behind_Master" | awk '{print $2}')
        (( ${lag:-0} < 60 )) && \
            check_pass "Replication: running (lag=${lag}s)" || \
            check_fail "Replication: high lag (${lag}s)"
    fi

    echo ""
    echo "Passed: $checks_passed | Failed: $checks_failed"
    (( checks_failed == 0 ))
}

pg_health_check() {
    local checks_passed=0
    local checks_failed=0

    check_pass() { echo "  [OK] $1"; (( checks_passed++ )); }
    check_fail() { echo "  [FAIL] $1" >&2; (( checks_failed++ )); }

    echo "=== PostgreSQL Health Check ==="

    if pg_exec "SELECT 1;" &>/dev/null; then
        check_pass "Connection"
    else
        check_fail "Connection"
        return 1
    fi

    local version
    version=$(pg_exec "SELECT version();" | head -1)
    check_pass "Version: ${version%%,*}"

    local db_size
    db_size=$(pg_exec "SELECT pg_size_pretty(pg_database_size(current_database()));")
    check_pass "DB Size: $db_size"

    local active_conn max_conn
    active_conn=$(pg_exec "SELECT count(*) FROM pg_stat_activity WHERE state != 'idle';")
    max_conn=$(pg_exec "SELECT setting FROM pg_settings WHERE name = 'max_connections';")
    local conn_pct=$(( ${active_conn:-0} * 100 / ${max_conn:-100} ))

    if (( conn_pct < 80 )); then
        check_pass "Active connections: ${active_conn}/${max_conn} (${conn_pct}%)"
    else
        check_fail "Active connections: ${active_conn}/${max_conn} (${conn_pct}%)"
    fi

    local long_txn
    long_txn=$(pg_exec "SELECT count(*) FROM pg_stat_activity
        WHERE state = 'idle in transaction'
          AND query_start < now() - interval '5 minutes';")
    if (( ${long_txn:-0} == 0 )); then
        check_pass "No long-running idle transactions"
    else
        check_fail "Long idle transactions: $long_txn"
    fi

    echo ""
    echo "Passed: $checks_passed | Failed: $checks_failed"
    (( checks_failed == 0 ))
}
```

---

## 47.5 Exercises

### Exercise 1: Multi-DB Backup Orchestrator
สร้าง orchestrator ที่:
- Backup MySQL + PostgreSQL + Redis
- Upload ไป S3 / remote storage
- Verify checksums
- Alert on failure

### Exercise 2: Database Performance Analyzer
สร้าง analyzer ที่:
- Collect slow queries
- Generate HTML report
- Compare กับ baseline
- Recommend indexes

### Exercise 3: Schema Drift Detector
สร้าง detector ที่:
- Compare schema across environments
- Detect missing tables/columns/indexes
- Generate migration SQL
- CI/CD integration

---

## สรุป Part 47

✅ MySQL user/database lifecycle management
✅ MySQL performance analysis (slow queries, table sizes)
✅ MySQL backup with verification
✅ PostgreSQL role/permission management
✅ PostgreSQL performance monitoring (bloat, locks, vacuums)
✅ Database migration framework (up/down/rollback)
✅ Migration file generation
✅ MySQL and PostgreSQL health check suites

---

**→ Part 48: Log Management and Analysis**
