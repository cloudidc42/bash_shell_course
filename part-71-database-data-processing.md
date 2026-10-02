# Part 71: Database Operations and Data Processing
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 71.1 Database Query Wrappers

```bash
#!/bin/bash
# db_query.sh - Unified database query interface

set -euo pipefail

# ─── MySQL ────────────────────────────────────────────────────────────────
mysql_query() {
    local db=${1:-} sql=$2
    local format=${3:-table}

    local mysql_args=(
        --host="${MYSQL_HOST:-localhost}"
        --user="${MYSQL_USER:-root}"
        --silent
    )
    [[ -n "${MYSQL_PASSWORD:-}" ]] && mysql_args+=("--password=${MYSQL_PASSWORD}")
    [[ -n "$db" ]] && mysql_args+=("$db")

    case "$format" in
        json)  mysql_args+=(--json) ;;
        csv)   mysql_args+=(--batch --fields-terminated-by=',' --lines-terminated-by='\n') ;;
        table) mysql_args+=(--table) ;;
        raw)   mysql_args+=(--batch) ;;
    esac

    echo "$sql" | mysql "${mysql_args[@]}" 2>/dev/null
}

mysql_exec_file() {
    local db=$1 sql_file=$2

    mysql \
        --host="${MYSQL_HOST:-localhost}" \
        --user="${MYSQL_USER:-root}" \
        ${MYSQL_PASSWORD:+"--password=${MYSQL_PASSWORD}"} \
        "$db" < "$sql_file"
}

mysql_table_exists() {
    local db=$1 table=$2
    local count; count=$(mysql_query "information_schema" \
        "SELECT COUNT(*) FROM tables WHERE table_schema='$db' AND table_name='$table'" raw)
    (( count > 0 ))
}

mysql_row_count() {
    local db=$1 table=$2
    mysql_query "$db" "SELECT COUNT(*) FROM \`${table}\`" raw
}

# ─── PostgreSQL ──────────────────────────────────────────────────────────
pg_query() {
    local db=${1:-postgres} sql=$2
    local format=${3:-table}

    local psql_args=(
        --host="${PGHOST:-localhost}"
        --port="${PGPORT:-5432}"
        --username="${PGUSER:-postgres}"
        --dbname="$db"
        --no-password
        --tuples-only
    )

    case "$format" in
        table) psql_args+=(--no-align); unset 'psql_args[5]' ;;
        csv)   psql_args+=(--csv) ;;
        json)  sql="SELECT row_to_json(t) FROM (${sql}) t" ;;
    esac

    PGPASSWORD="${PGPASSWORD:-}" psql "${psql_args[@]}" --command="$sql" 2>/dev/null
}

pg_exec_file() {
    local db=$1 sql_file=$2
    PGPASSWORD="${PGPASSWORD:-}" psql \
        --host="${PGHOST:-localhost}" \
        --username="${PGUSER:-postgres}" \
        --dbname="$db" \
        --no-password \
        --file="$sql_file" 2>/dev/null
}

# ─── SQLite ─────────────────────────────────────────────────────────────────
SQLITE_DB="${SQLITE_DB:-/tmp/app.db}"

sqlite_query() {
    local sql=$1 db=${2:-$SQLITE_DB}
    sqlite3 -separator '|' "$db" "$sql" 2>/dev/null
}

sqlite_json() {
    local sql=$1 db=${2:-$SQLITE_DB}
    sqlite3 -json "$db" "$sql" 2>/dev/null
}

sqlite_csv() {
    local sql=$1 db=${2:-$SQLITE_DB}
    sqlite3 -csv -header "$db" "$sql" 2>/dev/null
}

sqlite_table_info() {
    local table=$1 db=${2:-$SQLITE_DB}
    sqlite3 "$db" "PRAGMA table_info('$table')" 2>/dev/null
}

sqlite_kv_set() {
    local key=$1 value=$2 db=${3:-$SQLITE_DB}

    sqlite3 "$db" "
        CREATE TABLE IF NOT EXISTS kv (key TEXT PRIMARY KEY, value TEXT, updated_at INTEGER);
        INSERT OR REPLACE INTO kv VALUES ('$key', '$value', strftime('%s','now'));
    " 2>/dev/null
}

sqlite_kv_get() {
    local key=$1 db=${2:-$SQLITE_DB}
    sqlite3 "$db" "SELECT value FROM kv WHERE key='$key'" 2>/dev/null
}

sqlite_kv_delete() {
    local key=$1 db=${2:-$SQLITE_DB}
    sqlite3 "$db" "DELETE FROM kv WHERE key='$key'" 2>/dev/null
}
```

---

## 71.2 Schema Migration System

```bash
#!/bin/bash
# migrations.sh - Database schema migration runner

MIGRATION_DIR="${MIGRATION_DIR:-./migrations}"
MIGRATION_TABLE="${MIGRATION_TABLE:-schema_migrations}"

migrations_init() {
    local db_type=${1:-sqlite}

    case "$db_type" in
        sqlite)
            sqlite3 "$SQLITE_DB" "
                CREATE TABLE IF NOT EXISTS $MIGRATION_TABLE (
                    version TEXT PRIMARY KEY,
                    applied_at INTEGER DEFAULT (strftime('%s','now'))
                );
            "
            ;;
        mysql)
            mysql_query "${MYSQL_DB:-app}" "
                CREATE TABLE IF NOT EXISTS $MIGRATION_TABLE (
                    version VARCHAR(255) PRIMARY KEY,
                    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
            " raw
            ;;
        postgres)
            pg_query "${PGDATABASE:-app}" "
                CREATE TABLE IF NOT EXISTS $MIGRATION_TABLE (
                    version VARCHAR(255) PRIMARY KEY,
                    applied_at TIMESTAMP DEFAULT NOW()
                );
            "
            ;;
    esac

    echo "Migration table ready: $MIGRATION_TABLE"
}

migration_is_applied() {
    local version=$1 db_type=${2:-sqlite}

    local count=0
    case "$db_type" in
        sqlite) count=$(sqlite3 "$SQLITE_DB" "SELECT COUNT(*) FROM $MIGRATION_TABLE WHERE version='$version'" 2>/dev/null) ;;
        mysql)  count=$(mysql_query "${MYSQL_DB:-app}" "SELECT COUNT(*) FROM $MIGRATION_TABLE WHERE version='$version'" raw) ;;
    esac

    (( ${count:-0} > 0 ))
}

migration_record() {
    local version=$1 db_type=${2:-sqlite}

    case "$db_type" in
        sqlite) sqlite3 "$SQLITE_DB" "INSERT INTO $MIGRATION_TABLE (version) VALUES ('$version')" ;;
        mysql)  mysql_query "${MYSQL_DB:-app}" "INSERT INTO $MIGRATION_TABLE (version) VALUES ('$version')" raw ;;
    esac
}

run_migrations() {
    local db_type=${1:-sqlite}
    local applied=0 skipped=0 failed=0

    migrations_init "$db_type"

    echo "Running migrations from: $MIGRATION_DIR"

    for migration_file in $(ls "$MIGRATION_DIR"/*.sql 2>/dev/null | sort); do
        local version; version=$(basename "$migration_file" .sql)

        if migration_is_applied "$version" "$db_type"; then
            (( skipped++ ))
            continue
        fi

        echo "  Applying: $version"

        local exit_code=0
        case "$db_type" in
            sqlite) sqlite3 "$SQLITE_DB" < "$migration_file" 2>/dev/null || exit_code=$? ;;
            mysql)  mysql_exec_file "${MYSQL_DB:-app}" "$migration_file" || exit_code=$? ;;
            postgres) pg_exec_file "${PGDATABASE:-app}" "$migration_file" || exit_code=$? ;;
        esac

        if (( exit_code == 0 )); then
            migration_record "$version" "$db_type"
            (( applied++ ))
            echo "    [OK] $version"
        else
            (( failed++ ))
            echo "    [FAIL] $version"
            break
        fi
    done

    echo "Migrations: applied=$applied skipped=$skipped failed=$failed"
    (( failed == 0 ))
}
```

---

## 71.3 CSV Processing

```bash
#!/bin/bash
# csv_proc.sh - CSV data processing with awk

csv_header() {
    local file=$1
    head -1 "$file"
}

csv_row_count() {
    local file=$1
    awk 'END{print NR-1}' "$file"
}

csv_col_index() {
    local file=$1 col_name=$2
    head -1 "$file" | tr ',' '\n' | awk -v name="$col_name" '{if($0==name){print NR; exit}}'
}

csv_select() {
    local file=$1; shift
    local cols=("$@")

    local col_nums=()
    for col in "${cols[@]}"; do
        col_nums+=("$(csv_col_index "$file" "$col")")
    done

    local field_spec; field_spec=$(IFS=,; echo "\$${col_nums[*]}" | sed 's/ /$,/g; s/,/,\$/g')

    awk -F',' "OFS=',' {print ${field_spec//\$/\$}}" "$file"
}

csv_filter() {
    local file=$1 col=$2 op=$3 value=$4
    local col_idx; col_idx=$(csv_col_index "$file" "$col")

    awk -F',' -v col="$col_idx" -v op="$op" -v val="$value" '
    NR==1 { print; next }
    {
        v = $col
        gsub(/"/, "", v)
        if (op == "=" && v == val) print
        else if (op == "!=" && v != val) print
        else if (op == ">" && v+0 > val+0) print
        else if (op == "<" && v+0 < val+0) print
        else if (op == "contains" && index(v, val) > 0) print
    }' "$file"
}

csv_aggregate() {
    local file=$1 group_col=$2 agg_col=$3 func=${4:-count}
    local group_idx; group_idx=$(csv_col_index "$file" "$group_col")
    local agg_idx; agg_idx=$(csv_col_index "$file" "$agg_col")

    awk -F',' -v gi="$group_idx" -v ai="$agg_idx" -v func="$func" '
    NR==1 { next }
    {
        g = $gi; gsub(/"/, "", g)
        v = $ai + 0
        if (func == "count") cnt[g]++
        else if (func == "sum") sum[g] += v
        else if (func == "max") { if (!(g in max) || v > max[g]) max[g]=v }
        else if (func == "min") { if (!(g in min) || v < min[g]) min[g]=v }
    }
    END {
        for (g in cnt) if (func=="count") print g","cnt[g]
        for (g in sum) if (func=="sum") print g","sum[g]
        for (g in max) if (func=="max") print g","max[g]
        for (g in min) if (func=="min") print g","min[g]
    }' "$file" | sort
}

csv_join() {
    local file1=$1 file2=$2 key_col=$3
    local key1; key1=$(csv_col_index "$file1" "$key_col")
    local key2; key2=$(csv_col_index "$file2" "$key_col")

    awk -F',' -v k1="$key1" -v k2="$key2" '
    NR==FNR {
        key = $k2; gsub(/"/, "", key)
        row = $0; sub($k2 "(,|$)", "", row)
        lookup[key] = row
        next
    }
    FNR==1 { print; next }
    {
        key = $k1; gsub(/"/, "", key)
        if (key in lookup) print $0","lookup[key]
    }' "$file2" "$file1"
}

csv_to_json() {
    local file=$1

    awk -F',' '
    NR==1 {
        n = NF
        for (i=1; i<=n; i++) {
            h[i] = $i
            gsub(/"/, "", h[i])
        }
        print "["
        next
    }
    {
        printf "%s{",(FNR>2 ? "," : "")
        for (i=1; i<=NF; i++) {
            v = $i; gsub(/"/, "", v)
            printf "%s\"%s\":\"%s\"", (i>1 ? "," : ""), h[i], v
        }
        print "}"
    }
    END { print "]" }' "$file"
}
```

---

## 71.4 JSON Stream Processing

```bash
#!/bin/bash
# json_proc.sh - JSON data processing pipelines

json_flatten() {
    local input=${1:-/dev/stdin}
    local separator=${2:-.}

    jq -r '[paths(scalars) as $p | {([$p[] | tostring] | join("'\'$separator\''")):getpath($p)}] | add | to_entries[] | .key + "=" + (.value | tostring)' "$input" 2>/dev/null
}

json_merge() {
    local -a files=("$@")
    jq -s 'reduce .[] as $item ({}; . * $item)' "${files[@]}"
}

json_array_merge() {
    local -a files=("$@")
    jq -s '[.[] | .[]]' "${files[@]}"
}

json_extract_field() {
    local input=${1:-/dev/stdin} field=$2
    jq -r ".[\"$field\"] // empty" "$input" 2>/dev/null
}

json_filter_array() {
    local input=${1:-/dev/stdin} field=$2 op=$3 value=$4

    case "$op" in
        =)        jq --arg v "$value" ".[] | select(.${field} == \$v)" "$input" ;;
        !=)       jq --arg v "$value" ".[] | select(.${field} != \$v)" "$input" ;;
        contains) jq --arg v "$value" ".[] | select(.${field} | tostring | contains(\$v))" "$input" ;;
        exists)   jq ".[] | select(.${field} != null)" "$input" ;;
    esac
}

json_schema_infer() {
    local input=${1:-/dev/stdin}

    jq '.
        | if type == "array" then .[0] else . end
        | to_entries
        | map({key: .key, type: (.value | type)})
        | from_entries' "$input" 2>/dev/null
}

json_stats() {
    local input=${1:-/dev/stdin} field=$2

    jq --arg f "$field" '[
        .[]."\($f)" | numbers
    ] | {
        count: length,
        sum: add,
        min: min,
        max: max,
        avg: (if length > 0 then (add / length) else null end)
    }' "$input" 2>/dev/null
}

json_diff() {
    local file1=$1 file2=$2

    diff \
        <(jq -S . "$file1") \
        <(jq -S . "$file2")
}

json_validate_schema() {
    local input=$1 schema=$2

    if command -v jsonschema &>/dev/null; then
        jsonschema -i "$input" "$schema"
    elif command -v ajv &>/dev/null; then
        ajv validate -s "$schema" -d "$input"
    else
        echo "No JSON Schema validator available (install python-jsonschema or ajv)"
        return 1
    fi
}
```

---

## 71.5 Data Validation

```bash
#!/bin/bash
# data_validation.sh - Data quality validation

validate_record() {
    local record=$1
    local -a errors=()

    # Type validations
    _validate_field() {
        local value=$1 type=$2 field=$3
        case "$type" in
            integer)
                [[ "$value" =~ ^-?[0-9]+$ ]] || errors+=("$field: expected integer, got '$value'")
                ;;
            float)
                [[ "$value" =~ ^-?[0-9]*\.?[0-9]+$ ]] || errors+=("$field: expected float, got '$value'")
                ;;
            email)
                [[ "$value" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]] || \
                    errors+=("$field: invalid email '$value'")
                ;;
            nonempty)
                [[ -n "$value" ]] || errors+=("$field: must not be empty")
                ;;
        esac
    }

    # Extract and validate fields from JSON record
    local name; name=$(echo "$record" | jq -r '.name // empty')
    local email; email=$(echo "$record" | jq -r '.email // empty')
    local age; age=$(echo "$record" | jq -r '.age // empty')

    _validate_field "$name" nonempty "name"
    _validate_field "$email" email "email"
    [[ -n "$age" ]] && _validate_field "$age" integer "age"
    [[ -n "$age" ]] && (( age >= 0 && age <= 150 )) 2>/dev/null || \
        [[ -z "$age" ]] || errors+=("age: out of range ($age)")

    if (( ${#errors[@]} == 0 )); then
        echo "VALID"
        return 0
    else
        printf 'INVALID: %s\n' "${errors[@]}"
        return 1
    fi
}

validate_csv_schema() {
    local file=$1

    local -A required_cols=([name]=1 [email]=1 [status]=1)
    local -a valid_statuses=(active inactive pending)

    local header; header=$(head -1 "$file")
    local errors=0

    # Check required columns
    for col in "${!required_cols[@]}"; do
        echo "$header" | grep -q "$col" || {
            echo "SCHEMA ERROR: Missing column: $col"
            (( errors++ ))
        }
    done

    # Check status values
    local status_col; status_col=$(csv_col_index "$file" "status" 2>/dev/null || echo 0)
    if (( status_col > 0 )); then
        awk -F',' -v col="$status_col" 'NR>1 {print $col}' "$file" | \
        while IFS= read -r status; do
            local valid=false
            for s in "${valid_statuses[@]}"; do
                [[ "$status" == "$s" ]] && valid=true && break
            done
            $valid || echo "DATA ERROR: Invalid status: '$status'"
        done
    fi

    echo "Validation complete: $file"
    (( errors == 0 ))
}
```

---

## 71.6 ETL Pipeline

```bash
#!/bin/bash
# etl.sh - Extract, Transform, Load pipeline

ETL_WORK_DIR="${ETL_WORK_DIR:-/tmp/etl}"
ETL_LOG="${ETL_LOG:-/tmp/etl.log}"

mkdir -p "$ETL_WORK_DIR"

etl_log() {
    printf '%s [ETL] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "$ETL_LOG"
}

# ─── Extract ───────────────────────────────────────────────────────────
etl_extract_csv() {
    local source_file=$1 output_name=${2:-extract}
    local dest="${ETL_WORK_DIR}/${output_name}.csv"

    cp "$source_file" "$dest"
    local rows; rows=$(csv_row_count "$dest")
    etl_log "Extracted $rows rows from $source_file"
    echo "$dest"
}

etl_extract_db() {
    local db=$1 query=$2 output_name=${3:-extract}
    local dest="${ETL_WORK_DIR}/${output_name}.csv"

    mysql_query "$db" "$query" csv > "$dest"
    local rows; rows=$(csv_row_count "$dest" 2>/dev/null || echo 0)
    etl_log "Extracted $rows rows from DB: $db"
    echo "$dest"
}

etl_extract_api() {
    local url=$1 output_name=${2:-extract}
    local dest="${ETL_WORK_DIR}/${output_name}.json"

    curl -sfL "$url" > "$dest"
    local count; count=$(jq 'if type=="array" then length else 1 end' "$dest" 2>/dev/null || echo 0)
    etl_log "Extracted $count records from API: $url"
    echo "$dest"
}

# ─── Transform ──────────────────────────────────────────────────────────
etl_transform_csv() {
    local input_file=$1 output_name=${2:-transform}
    local dest="${ETL_WORK_DIR}/${output_name}.csv"

    awk -F',' 'OFS="," {
        # Normalize dates
        for(i=1;i<=NF;i++) {
            if ($i ~ /^[0-9]{4}-[0-9]{2}-[0-9]{2}$/) {
                # already ISO format, keep
            }
            # Trim whitespace
            gsub(/^[ \t]+|[ \t]+$/, "", $i)
        }
        print
    }' "$input_file" > "$dest"

    etl_log "Transform complete: $dest"
    echo "$dest"
}

# ─── Load ─────────────────────────────────────────────────────────────────
etl_load_sqlite() {
    local csv_file=$1 table=$2 db=${3:-$SQLITE_DB}
    local mode=${4:-replace}

    etl_log "Loading to SQLite: $table ($mode)"

    local import_cmd=".mode csv\n.headers on\n"
    case "$mode" in
        replace) import_cmd+="DROP TABLE IF EXISTS $table;\n" ;;
    esac
    import_cmd+=".import $csv_file $table"

    printf "$import_cmd" | sqlite3 "$db"
    local rows; rows=$(sqlite3 "$db" "SELECT COUNT(*) FROM $table" 2>/dev/null)
    etl_log "Loaded $rows rows to $table"
}

etl_run_pipeline() {
    local pipeline_name=$1
    local ts; ts=$(date +%Y%m%d_%H%M%S)

    etl_log "=== ETL Pipeline: $pipeline_name ($ts) ==="

    # Override in pipeline-specific scripts:
    # extract_stage, transform_stage, load_stage

    etl_log "Pipeline complete: $pipeline_name"
}
```

---

## 71.7 Report Generation

```bash
#!/bin/bash
# reports.sh - Data report generation

report_table() {
    local data_file=$1 title=${2:-Report}
    local col_widths=()

    echo "=== $title ==="
    echo ""

    awk -F',' '
    NR==1 {
        n=NF
        for(i=1;i<=n;i++) header[i]=$i
        next
    }
    { for(i=1;i<=NF;i++) if(length($i)>w[i]) w[i]=length($i) }
    END {
        # Print header
        for(i=1;i<=n;i++) printf "%-"w[i]"s  ", header[i]
        printf "\n"
        for(i=1;i<=n;i++) { for(j=0;j<w[i];j++) printf "-"; printf "  " }
        printf "\n"
    }' "$data_file"

    awk -F',' 'NR>1 { print }' "$data_file" | \
    awk -F',' -v OFS='  ' '{ $1=$1; print }'
}

report_summary_stats() {
    local data_file=$1 numeric_col=$2 group_col=${3:-}
    local col_idx; col_idx=$(csv_col_index "$data_file" "$numeric_col")

    awk -F',' -v col="$col_idx" '
    NR==1 { next }
    {
        v = $col + 0
        sum += v; count++
        if (count==1 || v<min) min=v
        if (count==1 || v>max) max=v
    }
    END {
        printf "Count:   %d\n", count
        printf "Sum:     %.2f\n", sum
        printf "Mean:    %.2f\n", (count>0 ? sum/count : 0)
        printf "Min:     %.2f\n", min
        printf "Max:     %.2f\n", max
        printf "Range:   %.2f\n", max-min
    }' "$data_file"
}

generate_html_report() {
    local data_file=$1 output_file=${2:-report.html} title=${3:-Report}

    {
        echo '<!DOCTYPE html><html><head>'
        echo "<title>$title</title>"
        echo '<style>table{border-collapse:collapse;width:100%}th,td{border:1px solid #ddd;padding:8px}th{background:#4CAF50;color:white}</style>'
        echo "</head><body><h1>$title</h1><p>Generated: $(date)</p>"
        echo '<table>'

        local first=true
        while IFS=',' read -ra fields; do
            if $first; then
                echo '<tr>'
                for f in "${fields[@]}"; do echo "<th>$f</th>"; done
                echo '</tr>'
                first=false
            else
                echo '<tr>'
                for f in "${fields[@]}"; do echo "<td>$f</td>"; done
                echo '</tr>'
            fi
        done < "$data_file"

        echo '</table></body></html>'
    } > "$output_file"

    echo "HTML report: $output_file"
}
```

---

## 71.8 Exercises

### Exercise 1: Database Sync
สร้าง sync tool ที่:
- Compare two DB tables
- Generate diff SQL
- Apply inserts/updates/deletes
- Verify sync accuracy

### Exercise 2: Data Pipeline Monitor
สร้าง monitor ที่:
- Track row counts per run
- Detect anomalies (sudden drops/spikes)
- SLA for pipeline completion time
- Lineage tracking

### Exercise 3: Analytics Engine
สร้าง engine ที่:
- Funnel analysis
- Cohort retention
- Moving averages
- Export to visualization format

---

## สรุป Part 71

✅ MySQL/PostgreSQL/SQLite query wrappers with JSON/CSV/table output
✅ SQLite key-value store (kv_set/get/delete)
✅ Schema migration runner: versioned SQL files, applied-version tracking
✅ CSV: row count, column index, select, filter, aggregate, join, to_json
✅ JSON: flatten, merge, filter array, schema infer, stats, diff
✅ Data validation: type, range, enum, email format, schema check
✅ ETL pipeline: extract from CSV/DB/API, transform (normalize/trim), load to SQLite
✅ Report generation: ASCII table, summary stats, HTML table export

---

**→ Part 72: Text Processing and Natural Language Tools**
