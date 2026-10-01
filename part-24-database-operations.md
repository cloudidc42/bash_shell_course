# Part 24: Database Operations from Shell
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 24.1 PostgreSQL from Shell

```bash
# ─── Connection ───────────────────────────────────────────────
# Using psql
psql -h localhost -p 5432 -U myuser -d mydb
psql "postgresql://user:pass@host:5432/dbname"
psql $DATABASE_URL

# Environment variables (avoid passwords in command line)
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=mydb
export PGUSER=myuser
export PGPASSWORD=mypassword    # less secure

# Better: .pgpass file
echo "localhost:5432:mydb:myuser:mypassword" >> ~/.pgpass
chmod 600 ~/.pgpass

# ─── Execute SQL ──────────────────────────────────────────────
# Single command
psql -c "SELECT version();"
psql -c "SELECT COUNT(*) FROM users WHERE active = true;"

# Multiple commands
psql << 'SQL'
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
SQL

# Execute SQL file
psql < migration.sql
psql -f migration.sql

# ─── Non-interactive options ──────────────────────────────────
psql -t          # tuples only (no headers)
psql -A          # unaligned output
psql -q          # quiet (no messages)
psql -F','       # field separator
psql -P format=csv  # CSV output
psql -P pager=off   # disable pager

# ─── psql in scripts ──────────────────────────────────────────
query_db() {
    local query=$1
    psql -t -A -F',' << SQL
$query
SQL
}

# Get user count
count=$(query_db "SELECT COUNT(*) FROM users")
echo "Users: $count"

# Fetch rows
while IFS=, read -r id name email; do
    echo "Processing: $name ($email)"
done < <(query_db "SELECT id, name, email FROM users LIMIT 100")

# ─── Export data ──────────────────────────────────────────────
# CSV export
psql -c "\COPY users TO 'users.csv' WITH CSV HEADER;"

# Using COPY command in script
psql << 'SQL'
COPY (
    SELECT id, name, email, created_at 
    FROM users 
    WHERE active = true
) TO '/tmp/active_users.csv' WITH CSV HEADER;
SQL

# SQL query to CSV
psql -t -A -F',' -c "SELECT * FROM orders WHERE date > '2024-01-01'" > orders.csv

# ─── Import data ──────────────────────────────────────────────
psql -c "\COPY users FROM 'users.csv' WITH CSV HEADER;"

# ─── Database management ──────────────────────────────────────
# Create/drop database
createdb mydb
dropdb mydb

# Backup
pg_dump mydb > backup.sql
pg_dump -Fc mydb > backup.dump      # custom format (faster)
pg_dump -t users mydb > users.sql   # single table
pg_dumpall > all_databases.sql      # all databases

# Restore
psql mydb < backup.sql
pg_restore -d mydb backup.dump
pg_restore -t users -d mydb backup.dump

# ─── Monitoring queries ───────────────────────────────────────
# Long running queries
psql -c "
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';"

# Table sizes
psql -c "
SELECT relname AS table, 
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables 
ORDER BY pg_total_relation_size(relid) DESC 
LIMIT 20;"
```

---

## 24.2 MySQL/MariaDB from Shell

```bash
# ─── Connection ───────────────────────────────────────────────
mysql -h localhost -P 3306 -u myuser -pmypassword mydb
mysql --user=myuser --password=mypassword --host=localhost mydb

# ~/.my.cnf for credentials
cat > ~/.my.cnf << 'EOF'
[client]
host = localhost
user = myuser
password = mypassword
database = mydb
EOF
chmod 600 ~/.my.cnf

# Now can connect without credentials
mysql mydb

# ─── Execute SQL ──────────────────────────────────────────────
mysql -e "SELECT VERSION();"
mysql -e "SHOW DATABASES;"
mysql mydb -e "SELECT COUNT(*) FROM users;"

# Non-interactive
mysql -B           # batch mode (tab-separated)
mysql -s           # silent
mysql -N           # no column headers

# ─── In scripts ───────────────────────────────────────────────
run_query() {
    local query=$1
    mysql -BNe "$query" 2>/dev/null
}

# Get value
active_users=$(run_query "SELECT COUNT(*) FROM users WHERE active=1")

# Process rows
while IFS=$'\t' read -r id name email; do
    echo "User: $name <$email>"
done < <(run_query "SELECT id, name, email FROM users LIMIT 10")

# Run SQL file
mysql mydb < schema.sql

# Heredoc
mysql mydb << 'SQL'
START TRANSACTION;
INSERT INTO events (type, data) VALUES ('deploy', '{"version":"1.0"}');
UPDATE settings SET last_deploy = NOW();
COMMIT;
SQL

# ─── Import/Export ────────────────────────────────────────────
# Dump
mysqldump mydb > backup.sql
mysqldump mydb users orders > tables.sql     # specific tables
mysqldump --all-databases > all_dbs.sql
mysqldump -d mydb > schema_only.sql          # structure only

# Restore
mysql mydb < backup.sql
mysql < all_dbs.sql

# CSV export
mysql -B -e "SELECT * FROM users" mydb | \
    sed 's/\t/,/g' > users.csv

# Or use INTO OUTFILE (server must have permissions)
mysql -e "
SELECT id, name, email INTO OUTFILE '/tmp/users.csv'
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"'
LINES TERMINATED BY '\n'
FROM users;" mydb
```

---

## 24.3 SQLite from Shell

```bash
# ─── SQLite basics ────────────────────────────────────────────
# Perfect for scripts, no server needed
sqlite3 mydb.db

# Execute SQL
sqlite3 mydb.db "SELECT * FROM users;"
sqlite3 mydb.db < query.sql

# In scripts
sqlite3 mydb.db << 'SQL'
CREATE TABLE IF NOT EXISTS config (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
SQL

# ─── Shell wrapper for SQLite ─────────────────────────────────
DB_FILE="${HOME}/.myapp/data.db"

# Initialize database
init_db() {
    mkdir -p "$(dirname "$DB_FILE")"
    sqlite3 "$DB_FILE" << 'SQL'
CREATE TABLE IF NOT EXISTS settings (
    key TEXT PRIMARY KEY,
    value TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT NOT NULL,
    data TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_events_type ON events(type);
CREATE INDEX IF NOT EXISTS idx_events_created ON events(created_at);
SQL
}

# Set config value
set_setting() {
    local key=$1
    local value=$2
    sqlite3 "$DB_FILE" "
        INSERT INTO settings (key, value, updated_at)
        VALUES ('$key', '$value', CURRENT_TIMESTAMP)
        ON CONFLICT(key) DO UPDATE SET
            value = excluded.value,
            updated_at = excluded.updated_at;"
}

# Get config value
get_setting() {
    local key=$1
    local default=${2:-}
    local value
    value=$(sqlite3 "$DB_FILE" "SELECT value FROM settings WHERE key='$key';")
    echo "${value:-$default}"
}

# Log event
log_event() {
    local type=$1
    local data=${2:-{}}
    sqlite3 "$DB_FILE" "
        INSERT INTO events (type, data) 
        VALUES ('$type', '$(echo "$data" | sed "s/'/''/g")');"
}

# Query recent events
get_recent_events() {
    local type=${1:-}
    local limit=${2:-10}
    
    local where=""
    [[ -n "$type" ]] && where="WHERE type='$type'"
    
    sqlite3 -separator '|' "$DB_FILE" "
        SELECT id, type, data, created_at 
        FROM events $where
        ORDER BY created_at DESC 
        LIMIT $limit;"
}

# Usage
init_db
set_setting "api_url" "https://api.example.com"
set_setting "timeout" "30"
log_event "startup" '{"version":"1.0","pid":"'"$$"'"}'

echo "API URL: $(get_setting api_url)"
echo "Recent events:"
get_recent_events | while IFS='|' read -r id type data ts; do
    echo "  [$ts] $type: $data"
done

# ─── SQLite for CSV storage ───────────────────────────────────
import_csv_to_sqlite() {
    local csv_file=$1
    local db_file=$2
    local table_name=$3
    
    sqlite3 "$db_file" << SQL
.mode csv
.import $csv_file $table_name
SQL
}

export_sqlite_to_csv() {
    local db_file=$1
    local table_name=$2
    local csv_file=$3
    
    sqlite3 "$db_file" << SQL
.headers on
.mode csv
.output $csv_file
SELECT * FROM $table_name;
.output stdout
SQL
}
```

---

## 24.4 Redis from Shell

```bash
# ─── redis-cli ────────────────────────────────────────────────
redis-cli ping              # PONG
redis-cli set key value
redis-cli get key
redis-cli del key
redis-cli exists key        # 1 or 0
redis-cli keys pattern*     # list keys (careful in production!)
redis-cli scan 0            # safe iteration

# With auth
redis-cli -a "password" get key
export REDISCLI_AUTH=password
redis-cli get key

# ─── Data types ───────────────────────────────────────────────
# String
redis-cli set name "Alice"
redis-cli get name
redis-cli incr counter
redis-cli expire name 3600   # TTL in seconds
redis-cli ttl name

# Hash
redis-cli hset user:1 name "Alice" age 30 email "alice@ex.com"
redis-cli hget user:1 name
redis-cli hgetall user:1
redis-cli hmget user:1 name email

# List
redis-cli lpush queue "task1" "task2"
redis-cli rpop queue
redis-cli llen queue
redis-cli lrange queue 0 -1  # all items

# Set
redis-cli sadd tags "linux" "bash" "shell"
redis-cli smembers tags
redis-cli sismember tags "bash"   # 1 or 0

# Sorted Set
redis-cli zadd scores 100 "Alice" 85 "Bob" 92 "Carol"
redis-cli zrange scores 0 -1 withscores
redis-cli zrevrank scores "Alice"

# ─── Shell scripts with Redis ─────────────────────────────────
REDIS="redis-cli"

redis_get()  { $REDIS get "$1"; }
redis_set()  { $REDIS set "$1" "$2" ${3:+EX $3}; }
redis_del()  { $REDIS del "$1"; }
redis_incr() { $REDIS incr "$1"; }
redis_push() { $REDIS lpush "$1" "${@:2}"; }
redis_pop()  { $REDIS rpop "$1"; }

# Rate limiting
rate_limit() {
    local key="ratelimit:$1"
    local limit=$2
    local window=$3
    
    local count
    count=$($REDIS incr "$key")
    
    if [[ "$count" -eq 1 ]]; then
        $REDIS expire "$key" "$window"
    fi
    
    (( count <= limit ))  # returns 0 (success) if within limit
}

# Simple job queue
enqueue_job() {
    local queue=$1
    local job_data=$2
    
    local job_id
    job_id=$($REDIS incr "job:counter")
    
    $REDIS hset "job:$job_id" \
        status "pending" \
        data "$job_data" \
        created_at "$(date -Iseconds)"
    
    $REDIS lpush "$queue" "$job_id"
    echo "$job_id"
}

process_job() {
    local queue=$1
    
    local job_id
    job_id=$($REDIS rpop "$queue")
    [[ -z "$job_id" ]] && return 1
    
    $REDIS hset "job:$job_id" status "processing"
    
    local data
    data=$($REDIS hget "job:$job_id" data)
    
    # Process...
    echo "Processing job $job_id: $data"
    
    $REDIS hset "job:$job_id" status "completed" completed_at "$(date -Iseconds)"
}
```

---

## 24.5 Database Migration Script

```bash
#!/bin/bash
# migrate.sh - Database migration manager

set -euo pipefail

DB_URL="${DATABASE_URL:?DATABASE_URL required}"
MIGRATIONS_DIR="./migrations"
TRACKING_TABLE="_migrations"

# Initialize migration tracking table
init_tracking() {
    psql "$DB_URL" << 'SQL'
CREATE TABLE IF NOT EXISTS _migrations (
    id SERIAL PRIMARY KEY,
    filename TEXT UNIQUE NOT NULL,
    applied_at TIMESTAMP DEFAULT NOW(),
    checksum TEXT
);
SQL
}

# Check if migration applied
is_applied() {
    local filename=$1
    local result
    result=$(psql -t -A "$DB_URL" -c "SELECT COUNT(*) FROM $TRACKING_TABLE WHERE filename='$filename';")
    [[ "$result" -gt 0 ]]
}

# Apply migration
apply_migration() {
    local file=$1
    local filename
    filename=$(basename "$file")
    local checksum
    checksum=$(md5sum "$file" | cut -d' ' -f1)
    
    echo "Applying: $filename"
    
    psql "$DB_URL" << SQL
BEGIN;
$(cat "$file")
INSERT INTO $TRACKING_TABLE (filename, checksum) VALUES ('$filename', '$checksum');
COMMIT;
SQL
    
    echo "✓ Applied: $filename"
}

# Run pending migrations
migrate_up() {
    init_tracking
    
    local applied=0
    
    for migration_file in "$MIGRATIONS_DIR"/*.sql; do
        [[ -f "$migration_file" ]] || continue
        
        local filename
        filename=$(basename "$migration_file")
        
        if is_applied "$filename"; then
            echo "  Skip: $filename (already applied)"
        else
            apply_migration "$migration_file"
            (( applied++ ))
        fi
    done
    
    echo ""
    echo "Applied $applied migrations"
}

# Show migration status
migrate_status() {
    init_tracking
    
    echo "Migration Status:"
    echo "─────────────────────────────────────────"
    printf "%-40s %-10s %-20s\n" "Migration" "Status" "Applied At"
    echo "─────────────────────────────────────────"
    
    for migration_file in "$MIGRATIONS_DIR"/*.sql; do
        [[ -f "$migration_file" ]] || continue
        local filename
        filename=$(basename "$migration_file")
        
        local result
        result=$(psql -t -A "$DB_URL" -c "
            SELECT applied_at FROM $TRACKING_TABLE WHERE filename='$filename';
        ")
        
        if [[ -n "$result" ]]; then
            printf "%-40s %-10s %-20s\n" "$filename" "Applied" "$result"
        else
            printf "%-40s %-10s\n" "$filename" "Pending"
        fi
    done
}

case "${1:-}" in
    up)     migrate_up ;;
    status) migrate_status ;;
    *)      echo "Usage: $0 [up|status]"; exit 1 ;;
esac
```

---

## 24.6 Exercises

### Exercise 1: PostgreSQL Backup Automation
สร้าง script ที่:
- Daily incremental backups
- Weekly full backups
- Upload to S3/GCS
- Alert ถ้า backup fails

### Exercise 2: SQLite Config Store
สร้าง configuration system:
- Store app settings ใน SQLite
- Version history
- Import/export as JSON
- Sync ระหว่าง environments

### Exercise 3: Database Health Monitor
สร้าง monitor ที่:
- Check connection
- Monitor query performance
- Track table sizes
- Alert on issues

---

## สรุป Part 24

✅ PostgreSQL (psql, pg_dump, pg_restore)  
✅ MySQL/MariaDB (mysql, mysqldump)  
✅ SQLite (sqlite3 ใน scripts)  
✅ Redis (redis-cli, rate limiting, job queue)  
✅ Database migration script  
✅ Secure credential handling  

---

**→ Part 25: Shell Scripting for DevOps - Docker & Kubernetes**
