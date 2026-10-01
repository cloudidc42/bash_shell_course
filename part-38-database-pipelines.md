# Part 38: Database Operations & Data Pipelines
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 38.1 MySQL/MariaDB Automation

```bash
#!/bin/bash
# mysql_automation.sh - MySQL database operations

# ─── Connection Config ─────────────────────────────────────────
declare -A DB_CONFIG=(
    [host]="localhost"
    [port]="3306"
    [user]="root"
    [password]=""
    [database]=""
    [socket]=""
)

mysql_exec() {
    local query=$1
    local database=${2:-${DB_CONFIG[database]}}
    
    local args=(
        -h "${DB_CONFIG[host]}"
        -P "${DB_CONFIG[port]}"
        -u "${DB_CONFIG[user]}"
        --batch
        --silent
    )
    
    [[ -n "${DB_CONFIG[password]}" ]] && args+=(-p"${DB_CONFIG[password]}")
    [[ -n "${DB_CONFIG[socket]}" ]]   && args+=(--socket="${DB_CONFIG[socket]}")
    [[ -n "$database" ]]              && args+=("$database")
    
    echo "$query" | mysql "${args[@]}"
}

# ─── Database Management ───────────────────────────────────────
create_database() {
    local dbname=$1
    local charset=${2:-utf8mb4}
    local collation=${3:-utf8mb4_unicode_ci}
    
    mysql_exec "CREATE DATABASE IF NOT EXISTS \`${dbname}\`
        CHARACTER SET ${charset}
        COLLATE ${collation};"
    echo "Database created: $dbname"
}

list_databases() {
    mysql_exec "SHOW DATABASES;" | grep -v "Database\|information_schema\|performance_schema\|sys"
}

list_tables() {
    local database=$1
    mysql_exec "SHOW TABLES;" "$database"
}

# ─── Backup & Restore ──────────────────────────────────────────
backup_database() {
    local database=$1
    local output_dir=${2:-/var/backups/mysql}
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="${output_dir}/${database}_${timestamp}.sql.gz"
    
    mkdir -p "$output_dir"
    
    local dump_args=(
        -h "${DB_CONFIG[host]}"
        -u "${DB_CONFIG[user]}"
        --single-transaction
        --routines
        --triggers
        --events
        --hex-blob
        --set-gtid-purged=OFF
        "$database"
    )
    
    [[ -n "${DB_CONFIG[password]}" ]] && dump_args=(-p"${DB_CONFIG[password]}" "${dump_args[@]}")
    
    mysqldump "${dump_args[@]}" | gzip -9 > "$backup_file"
    
    local size
    size=$(du -sh "$backup_file" | cut -f1)
    echo "Backup: $backup_file ($size)"
}

restore_database() {
    local backup_file=$1
    local database=$2
    
    create_database "$database"
    
    echo "Restoring $backup_file → $database"
    
    if [[ "$backup_file" == *.gz ]]; then
        zcat "$backup_file" | mysql_exec "" "$database" < /dev/stdin
    else
        mysql_exec "" "$database" < "$backup_file"
    fi
    
    echo "Restore complete"
}

# ─── Automated Backup with Rotation ───────────────────────────
backup_all_databases() {
    local output_dir=${1:-/var/backups/mysql}
    local retention_days=${2:-30}
    
    local start_time=$SECONDS
    local success=0 failed=0
    
    while IFS= read -r db; do
        if backup_database "$db" "$output_dir"; then
            (( success++ ))
        else
            echo "Failed: $db" >&2
            (( failed++ ))
        fi
    done < <(list_databases)
    
    # Rotate old backups
    find "$output_dir" -name "*.sql.gz" -mtime +"$retention_days" -delete
    
    local duration=$(( SECONDS - start_time ))
    echo "Backup done: $success success, $failed failed in ${duration}s"
}

# ─── Query Performance Analysis ────────────────────────────────
show_slow_queries() {
    local threshold=${1:-1}
    
    mysql_exec "
        SELECT
            query_time,
            lock_time,
            rows_sent,
            rows_examined,
            LEFT(sql_text, 100) AS query_preview
        FROM mysql.slow_log
        WHERE query_time >= '$threshold'
        ORDER BY query_time DESC
        LIMIT 20;
    "
}

table_stats() {
    local database=$1
    
    mysql_exec "
        SELECT
            TABLE_NAME,
            TABLE_ROWS,
            ROUND(DATA_LENGTH/1024/1024, 2) AS data_mb,
            ROUND(INDEX_LENGTH/1024/1024, 2) AS index_mb,
            ROUND((DATA_LENGTH+INDEX_LENGTH)/1024/1024, 2) AS total_mb
        FROM information_schema.TABLES
        WHERE TABLE_SCHEMA = '$database'
        ORDER BY total_mb DESC;
    "
}
```

---

## 38.2 PostgreSQL Automation

```bash
#!/bin/bash
# postgres_automation.sh

PGHOST="${PGHOST:-localhost}"
PGPORT="${PGPORT:-5432}"
PGUSER="${PGUSER:-postgres}"
PGPASSWORD="${PGPASSWORD:-}"
PGDATABASE="${PGDATABASE:-postgres}"

export PGPASSWORD

pg_exec() {
    local query=$1
    local database=${2:-$PGDATABASE}
    
    psql -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" -d "$database" \
        -t -A -F $'\t' \
        -c "$query" 2>/dev/null
}

# ─── Database Operations ───────────────────────────────────────
pg_create_db() {
    local dbname=$1
    local owner=${2:-$PGUSER}
    
    psql -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" \
        -c "CREATE DATABASE \"${dbname}\" OWNER \"${owner}\";" 2>/dev/null
}

pg_backup() {
    local database=$1
    local output_dir=${2:-/var/backups/postgresql}
    local format=${3:-c}
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    
    mkdir -p "$output_dir"
    local backup_file="${output_dir}/${database}_${timestamp}.pgdump"
    
    pg_dump \
        -h "$PGHOST" \
        -p "$PGPORT" \
        -U "$PGUSER" \
        -Fc \
        --compress=9 \
        -d "$database" \
        -f "$backup_file"
    
    echo "Backup: $backup_file"
}

pg_restore_db() {
    local backup_file=$1
    local database=$2
    
    pg_create_db "$database"
    
    pg_restore \
        -h "$PGHOST" \
        -p "$PGPORT" \
        -U "$PGUSER" \
        -d "$database" \
        --clean \
        --if-exists \
        "$backup_file"
}

# ─── Connection Monitoring ─────────────────────────────────────
pg_connections() {
    pg_exec "
        SELECT
            datname AS database,
            count(*) AS connections,
            count(*) FILTER (WHERE state = 'active') AS active,
            count(*) FILTER (WHERE state = 'idle') AS idle,
            count(*) FILTER (WHERE wait_event_type = 'Lock') AS waiting
        FROM pg_stat_activity
        WHERE pid <> pg_backend_pid()
        GROUP BY datname
        ORDER BY connections DESC;
    "
}

pg_long_queries() {
    local threshold_secs=${1:-30}
    
    pg_exec "
        SELECT
            pid,
            usename,
            datname,
            EXTRACT(EPOCH FROM (now() - query_start))::int AS duration_secs,
            LEFT(query, 80) AS query_preview
        FROM pg_stat_activity
        WHERE state = 'active'
          AND query_start < now() - interval '$threshold_secs seconds'
          AND query NOT LIKE '%pg_stat_activity%'
        ORDER BY duration_secs DESC;
    "
}

pg_bloat_check() {
    local database=$1
    
    pg_exec "
        SELECT
            schemaname,
            tablename,
            ROUND(CAST(n_dead_tup AS numeric) / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
            n_dead_tup,
            n_live_tup,
            last_autovacuum
        FROM pg_stat_user_tables
        WHERE n_dead_tup > 1000
        ORDER BY dead_pct DESC
        LIMIT 20;
    " "$database"
}
```

---

## 38.3 SQLite Operations

```bash
#!/bin/bash
# sqlite_operations.sh

sqlite_exec() {
    local db_file=$1
    local query=$2
    
    sqlite3 -separator $'\t' "$db_file" "$query"
}

sqlite_init_schema() {
    local db_file=$1
    local schema_file=$2
    
    sqlite3 "$db_file" < "$schema_file"
    echo "Schema applied to $db_file"
}

sqlite_import_csv() {
    local db_file=$1
    local csv_file=$2
    local table_name=$3
    
    sqlite3 "$db_file" << EOF
.mode csv
.import $csv_file $table_name
.quit
EOF
    echo "Imported: $csv_file → $table_name"
}

sqlite_export_json() {
    local db_file=$1
    local table=$2
    local output_file=$3
    
    sqlite3 "$db_file" << EOF > "$output_file"
.mode json
SELECT * FROM $table;
.quit
EOF
    echo "Exported: $table → $output_file"
}

sqlite_stats() {
    local db_file=$1
    
    echo "=== SQLite Stats: $db_file ==="
    echo ""
    
    local size
    size=$(du -sh "$db_file" 2>/dev/null | cut -f1)
    echo "File size: $size"
    
    echo ""
    echo "Tables:"
    sqlite3 "$db_file" ".tables" | tr ' ' '\n' | sort | \
        while read -r table; do
            [[ -z "$table" ]] && continue
            local count
            count=$(sqlite3 "$db_file" "SELECT COUNT(*) FROM \"$table\";")
            printf "  %-30s %s rows\n" "$table" "$count"
        done
}
```

---

## 38.4 Data Pipeline Framework

```bash
#!/bin/bash
# data_pipeline.sh - ETL pipeline framework

declare -A PIPELINE_STAGES=()
declare -a PIPELINE_ORDER=()

pipeline_register() {
    local name=$1
    local func=$2
    PIPELINE_STAGES["$name"]="$func"
    PIPELINE_ORDER+=("$name")
}

pipeline_run() {
    local pipeline_name=$1
    local input_file=$2
    local output_dir=${3:-/tmp/pipeline_output}
    local start_time=$SECONDS
    
    mkdir -p "$output_dir"
    
    local current_input="$input_file"
    local stage_num=0
    
    echo "=== Pipeline: $pipeline_name ==="
    echo "Input: $input_file"
    echo ""
    
    for stage_name in "${PIPELINE_ORDER[@]}"; do
        (( stage_num++ ))
        local func="${PIPELINE_STAGES[$stage_name]}"
        local stage_output="${output_dir}/stage_${stage_num}_${stage_name}.dat"
        
        echo "Stage $stage_num: $stage_name"
        local stage_start=$SECONDS
        
        if "$func" "$current_input" "$stage_output"; then
            local stage_duration=$(( SECONDS - stage_start ))
            local rows
            rows=$(wc -l < "$stage_output" 2>/dev/null || echo "?")
            echo "  ✓ ${stage_duration}s, ${rows} records"
            current_input="$stage_output"
        else
            echo "  ✗ FAILED at stage: $stage_name" >&2
            return 1
        fi
    done
    
    cp "$current_input" "${output_dir}/final_output.dat"
    
    local total_duration=$(( SECONDS - start_time ))
    echo ""
    echo "Pipeline complete: ${total_duration}s"
    echo "Output: ${output_dir}/final_output.dat"
}

stage_filter_empty() {
    local input=$1
    local output=$2
    grep -v '^[[:space:]]*$' "$input" > "$output"
}

stage_deduplicate() {
    local input=$1
    local output=$2
    sort -u "$input" > "$output"
}

stage_normalize() {
    local input=$1
    local output=$2
    awk '{print tolower($0)}' "$input" > "$output"
}

stage_add_timestamp() {
    local input=$1
    local output=$2
    local ts
    ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    awk -v ts="$ts" '{print $0 "\t" ts}' "$input" > "$output"
}

process_csv_pipeline() {
    local input_csv=$1
    local output_csv=$2
    local delimiter=${3:-,}
    
    local headers
    headers=$(head -1 "$input_csv")
    
    tail -n +2 "$input_csv" | \
    awk -F"$delimiter" '
    NF > 0 && $1 != "" {
        for (i=1; i<=NF; i++) {
            gsub(/^[[:space:]]+|[[:space:]]+$/, "", $i)
        }
        print
    }' OFS="$delimiter" | \
    sort -t"$delimiter" -k1 | \
    { echo "$headers"; cat; } > "$output_csv"
    
    echo "Processed: $(wc -l < "$output_csv") rows"
}

stream_process() {
    local input_stream=$1
    local processor=$2
    local output_stream=${3:-/dev/stdout}
    local batch_size=${4:-100}
    
    local batch=()
    local count=0
    
    while IFS= read -r line; do
        batch+=("$line")
        (( count++ ))
        
        if (( count % batch_size == 0 )); then
            printf '%s\n' "${batch[@]}" | "$processor" >> "$output_stream"
            batch=()
            echo "Processed: $count records" >&2
        fi
    done < "$input_stream"
    
    if (( ${#batch[@]} > 0 )); then
        printf '%s\n' "${batch[@]}" | "$processor" >> "$output_stream"
    fi
    
    echo "Total: $count records" >&2
}
```

---

## 38.5 Redis Operations

```bash
#!/bin/bash
# redis_operations.sh

REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_AUTH="${REDIS_AUTH:-}"

redis_cmd() {
    local args=(-h "$REDIS_HOST" -p "$REDIS_PORT")
    [[ -n "$REDIS_AUTH" ]] && args+=(-a "$REDIS_AUTH")
    redis-cli "${args[@]}" "$@"
}

redis_set_json() {
    local key=$1
    local json=$2
    local ttl=${3:-3600}
    redis_cmd SET "$key" "$json" EX "$ttl"
}

redis_get_json() {
    local key=$1
    redis_cmd GET "$key"
}

redis_delete_pattern() {
    local pattern=$1
    local count=0
    while IFS= read -r key; do
        redis_cmd DEL "$key" > /dev/null
        (( count++ ))
    done < <(redis_cmd --scan --pattern "$pattern")
    echo "Deleted $count keys matching: $pattern"
}

redis_rate_limit() {
    local key=$1
    local limit=$2
    local window=${3:-60}
    
    local result
    result=$(redis_cmd EVAL "
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local current = redis.call('incr', key)
        if current == 1 then
            redis.call('expire', key, window)
        end
        if current > limit then
            return 0
        end
        return 1
    " 1 "ratelimit:$key" "$limit" "$window")
    
    [[ "$result" == "1" ]]
}

redis_enqueue() {
    local queue=$1
    local message=$2
    redis_cmd RPUSH "queue:$queue" "$message"
}

redis_dequeue() {
    local queue=$1
    local timeout=${2:-0}
    if (( timeout > 0 )); then
        redis_cmd BLPOP "queue:$queue" "$timeout" | tail -1
    else
        redis_cmd LPOP "queue:$queue"
    fi
}

redis_queue_length() {
    local queue=$1
    redis_cmd LLEN "queue:$queue"
}

redis_publish() {
    local channel=$1
    local message=$2
    redis_cmd PUBLISH "$channel" "$message"
}

redis_info() {
    local section=${1:-all}
    if [[ "$section" == "all" ]]; then
        redis_cmd INFO | grep -E "^(redis_version|used_memory_human|connected_clients|total_commands_processed|keyspace_hits|keyspace_misses):"
    else
        redis_cmd INFO "$section"
    fi
}
```

---

## 38.6 Data Transformation & Reporting

```bash
#!/bin/bash
# data_transform.sh

aggregate_csv() {
    local input=$1
    local group_col=${2:-1}
    local value_col=${3:-2}
    local delimiter=${4:-,}
    
    awk -F"$delimiter" -v gc="$group_col" -v vc="$value_col" '
    NR > 1 {
        group = $gc
        value = $vc + 0
        sum[group] += value
        count[group]++
        if (value > max[group] || !(group in max)) max[group] = value
        if (value < min[group] || !(group in min)) min[group] = value
    }
    END {
        printf "%-30s %10s %10s %10s %10s\n", "Group", "Count", "Sum", "Min", "Max"
        printf "%-30s %10s %10s %10s %10s\n", "-----", "-----", "---", "---", "---"
        for (g in sum) {
            printf "%-30s %10d %10.2f %10.2f %10.2f\n",
                g, count[g], sum[g], min[g], max[g]
        }
    }' "$input"
}

ascii_bar_chart() {
    local input=$1
    local max_width=${2:-50}
    
    local max_val
    max_val=$(awk '{print $2}' "$input" | sort -n | tail -1)
    
    awk -v max_val="$max_val" -v max_width="$max_width" '
    {
        label = $1
        value = $2
        bar_len = int(value / max_val * max_width)
        bar = sprintf("%*s", bar_len, "")
        gsub(/ /, "█", bar)
        printf "%-20s │%s %g\n", label, bar, value
    }' "$input"
}

validate_data_pipeline() {
    local input=$1
    local rules_file=$2
    local valid_output=$3
    local invalid_output=$4
    
    local valid_count=0
    local invalid_count=0
    
    while IFS=$'\t' read -r -a fields; do
        local valid=true
        local errors=()
        
        while IFS='|' read -r field_idx rule value message; do
            local field_val="${fields[$((field_idx-1))]}"
            
            case "$rule" in
                required)
                    [[ -z "$field_val" ]] && { valid=false; errors+=("$message"); }
                    ;;
                regex)
                    [[ ! "$field_val" =~ $value ]] && { valid=false; errors+=("$message"); }
                    ;;
                range)
                    local min max
                    IFS=':' read -r min max <<< "$value"
                    (( $(echo "$field_val < $min || $field_val > $max" | bc -l) )) && \
                        { valid=false; errors+=("$message"); }
                    ;;
                enum)
                    IFS=',' read -ra options <<< "$value"
                    local found=false
                    for opt in "${options[@]}"; do
                        [[ "$field_val" == "$opt" ]] && found=true
                    done
                    $found || { valid=false; errors+=("$message"); }
                    ;;
            esac
        done < "$rules_file"
        
        if $valid; then
            printf '%s\n' "${fields[@]}" | paste -s -d$'\t' >> "$valid_output"
            (( valid_count++ ))
        else
            local error_str
            error_str=$(IFS=';'; echo "${errors[*]}")
            printf '%s\t%s\n' "$(printf '%s\t' "${fields[@]}")" "$error_str" >> "$invalid_output"
            (( invalid_count++ ))
        fi
    done < "$input"
    
    echo "Valid: $valid_count, Invalid: $invalid_count"
}
```

---

## 38.7 Complete ETL Example

```bash
#!/bin/bash
# etl_example.sh - Complete Extract-Transform-Load pipeline

set -euo pipefail

readonly WORK_DIR="/tmp/etl_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$WORK_DIR"
trap "rm -rf $WORK_DIR" EXIT

extract_data() {
    local source_db=$1
    local output_file="${WORK_DIR}/raw_data.tsv"
    
    echo "[Extract] Querying source database..."
    
    mysql_exec "
        SELECT
            order_id,
            customer_id,
            DATE(created_at) AS order_date,
            total_amount,
            status
        FROM orders
        WHERE created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
          AND status NOT IN ('cancelled', 'pending')
        ORDER BY created_at;
    " "$source_db" | sed 's/\t/,/g' > "$output_file"
    
    local count
    count=$(wc -l < "$output_file")
    echo "[Extract] Extracted $count rows"
    echo "$output_file"
}

transform_data() {
    local input_file=$1
    local output_file="${WORK_DIR}/transformed.tsv"
    
    echo "[Transform] Processing data..."
    
    awk -F, '
    NR == 1 { next }
    {
        order_id = $1
        customer_id = $2
        order_date = $3
        amount = $4 + 0
        status = $5
        
        if (status == "completed" || status == "shipped") status = "fulfilled"
        
        tax = amount * 0.10
        total_with_tax = amount + tax
        
        split(order_date, d, "-")
        month = d[1] "-" d[2]
        
        printf "%s\t%s\t%s\t%.2f\t%.2f\t%.2f\t%s\n",
            order_id, customer_id, order_date,
            amount, tax, total_with_tax, month
    }' "$input_file" > "$output_file"
    
    echo "[Transform] Transformed $(wc -l < "$output_file") rows"
    echo "$output_file"
}

load_data() {
    local input_file=$1
    local target_db=$2
    local target_table="order_summary"
    
    echo "[Load] Loading to $target_db.$target_table..."
    
    mysql_exec "
        CREATE TABLE IF NOT EXISTS ${target_table} (
            order_id BIGINT PRIMARY KEY,
            customer_id BIGINT,
            order_date DATE,
            amount DECIMAL(10,2),
            tax DECIMAL(10,2),
            total_with_tax DECIMAL(10,2),
            month_year VARCHAR(7),
            loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
    " "$target_db"
    
    mysql_exec "
        LOAD DATA LOCAL INFILE '${input_file}'
        INTO TABLE ${target_table}
        FIELDS TERMINATED BY '\t'
        LINES TERMINATED BY '\n'
        (order_id, customer_id, order_date, amount, tax, total_with_tax, month_year);
    " "$target_db"
    
    local count
    count=$(mysql_exec "SELECT ROW_COUNT();" "$target_db")
    echo "[Load] Loaded $count rows"
}

run_etl() {
    local source_db="${1:?Source DB required}"
    local target_db="${2:?Target DB required}"
    
    echo "=== ETL Pipeline: $(date) ==="
    local total_start=$SECONDS
    
    local raw_data transformed_data
    raw_data=$(extract_data "$source_db")
    transformed_data=$(transform_data "$raw_data")
    load_data "$transformed_data" "$target_db"
    
    echo ""
    echo "ETL complete: $(( SECONDS - total_start ))s"
}

run_etl "${@}"
```

---

## 38.8 Exercises

### Exercise 1: Database Monitoring Dashboard
สร้าง dashboard แสดง:
- Connection counts per database
- Slow queries in real-time
- Table sizes trending
- Lock contention

### Exercise 2: Incremental Data Sync
สร้าง tool sync ข้อมูลแบบ incremental:
- Track last sync timestamp
- Extract only new/changed records
- Conflict resolution strategy
- Resume on failure

### Exercise 3: Data Quality Reporter
สร้าง data quality checks:
- Null percentages per column
- Duplicate detection
- Value distribution stats
- Schema drift detection

---

## สรุป Part 38

✅ MySQL/MariaDB automation  
✅ PostgreSQL operations  
✅ SQLite operations  
✅ Data pipeline framework  
✅ Redis cache & queue ops  
✅ CSV aggregation & stats  
✅ ASCII data visualization  
✅ Complete ETL example  

---

**→ Part 39: Security Hardening & Audit Automation**
