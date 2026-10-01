# Part 48: Log Management and Analysis
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 48.1 Structured Log Parsing

```bash
#!/bin/bash
# log_parser.sh - Advanced log parsing and analysis

# ─── Common Log Formats ────────────────────────────────────────
parse_nginx_access() {
    local log_file=${1:-/var/log/nginx/access.log}
    local format=${2:-combined}

    awk '
    {
        # Combined log format
        # 127.0.0.1 - user [01/Jan/2024:00:00:00 +0000] "GET /path HTTP/1.1" 200 1234 "referer" "agent"
        ip=$1
        # Extract timestamp
        match($0, /\[([^\]]+)\]/, ts)
        # Extract request
        match($0, /"([A-Z]+) ([^ ]+) ([^"]+)"/, req)
        # Extract status and size
        n=split($0, parts)
        for(i=1;i<=n;i++) {
            if(parts[i-1] ~ /^"[A-Z]/) {
                status=parts[i]
                bytes=parts[i+1]
                break
            }
        }
        printf "{\"ip\":\"%s\",\"timestamp\":\"%s\",\"method\":\"%s\",\"path\":\"%s\",\"status\":%s,\"bytes\":%s}\n",
            ip, ts[1], req[1], req[2], status, (bytes ~ /^[0-9]+$/ ? bytes : "0")
    }' "$log_file"
}

parse_syslog() {
    local log_file=${1:-/var/log/syslog}

    awk '
    {
        # Syslog format: Jan  1 00:00:00 hostname process[pid]: message
        month=$1; day=$2; time=$3; host=$4
        split($5, proc_pid, "[")
        process=proc_pid[1]
        gsub(/\]:?/, "", proc_pid[2])
        pid=proc_pid[2]

        message=""
        for(i=6; i<=NF; i++) message = message " " $i
        sub(/^ /, "", message)

        printf "{\"timestamp\":\"%s %s %s\",\"host\":\"%s\",\"process\":\"%s\",\"pid\":\"%s\",\"message\":\"%s\"}\n",
            month, day, time, host, process, pid, message
    }' "$log_file"
}

parse_apache_access() {
    local log_file=${1:-/var/log/apache2/access.log}

    awk '
    {
        ip=$1; ident=$2; user=$3
        match($0, /\[([^\]]+)\]/, ts)
        match($0, /"([^"]+)"/, req)
        n=split(req[1], r, " ")
        method=r[1]; path=r[2]; proto=r[3]
        # Find status and bytes after request
        match($0, /" ([0-9]+) ([0-9-]+)/, sb)
        printf "{\"ip\":\"%s\",\"user\":\"%s\",\"time\":\"%s\",\"method\":\"%s\",\"path\":\"%s\",\"status\":%s,\"bytes\":%s}\n",
            ip, user, ts[1], method, path, sb[1], (sb[2]=="-"?"0":sb[2])
    }' "$log_file"
}

# ─── JSON Log Analysis ─────────────────────────────────────────
analyze_json_logs() {
    local log_file=$1
    local time_field=${2:-timestamp}
    local level_field=${3:-level}
    local message_field=${4:-message}

    echo "=== Log Summary ==="

    echo "Level Distribution:"
    jq -r ".${level_field}" "$log_file" 2>/dev/null | \
        sort | uniq -c | sort -rn | \
        awk '{printf "  %-10s %d\n", $2, $1}'

    echo ""
    echo "Error Messages (last 10):"
    jq -r "select(.${level_field} == \"ERROR\" or .${level_field} == \"FATAL\") | .${message_field}" \
        "$log_file" 2>/dev/null | tail -10 | \
        awk '{print "  " $0}'

    echo ""
    echo "Top 10 Endpoints (if available):"
    jq -r '.path // empty' "$log_file" 2>/dev/null | \
        sort | uniq -c | sort -rn | head -10 | \
        awk '{printf "  %5d  %s\n", $1, $2}'
}
```

---

## 48.2 Real-Time Log Monitoring

```bash
#!/bin/bash
# log_monitor.sh - Real-time log monitoring with alerting

declare -A MONITOR_PATTERNS=()
declare -A MONITOR_COUNTS=()
declare -A MONITOR_LAST_ALERT=()
MONITOR_ALERT_COOLDOWN=300

monitor_add_rule() {
    local name=$1
    local pattern=$2
    local threshold=${3:-1}
    local action=${4:-echo}

    MONITOR_PATTERNS["${name}_pattern"]="$pattern"
    MONITOR_PATTERNS["${name}_threshold"]="$threshold"
    MONITOR_PATTERNS["${name}_action"]="$action"
    MONITOR_COUNTS["$name"]=0
    echo "Added monitor rule: $name"
}

tail_and_monitor() {
    local log_file=$1
    local window=${2:-60}

    echo "Monitoring: $log_file"
    echo "Rules: ${!MONITOR_PATTERNS[*]}"

    local line_count=0

    tail -F -n 0 "$log_file" 2>/dev/null | while IFS= read -r line; do
        local now
        now=$(date +%s)

        for rule_key in "${!MONITOR_PATTERNS[@]}"; do
            [[ "$rule_key" != *_pattern ]] && continue
            local name="${rule_key%_pattern}"
            local pattern="${MONITOR_PATTERNS[$rule_key]}"
            local threshold="${MONITOR_PATTERNS[${name}_threshold]}"
            local action="${MONITOR_PATTERNS[${name}_action]}"

            if echo "$line" | grep -qP "$pattern" 2>/dev/null || \
               echo "$line" | grep -q "$pattern"; then
                (( MONITOR_COUNTS[$name]++ ))
                local count="${MONITOR_COUNTS[$name]}"
                local last_alert="${MONITOR_LAST_ALERT[$name]:-0}"
                local elapsed=$(( now - last_alert ))

                if (( count >= threshold && elapsed > MONITOR_ALERT_COOLDOWN )); then
                    MONITOR_LAST_ALERT["$name"]=$now
                    MONITOR_COUNTS["$name"]=0
                    echo "[ALERT:$(date)] Rule '$name': $count matches in window | Line: $line" >&2
                    eval "$action '$name' '$count' '$line'"
                fi
            fi
        done

        (( line_count++ ))
        if (( line_count % 1000 == 0 )); then
            for name in "${!MONITOR_COUNTS[@]}"; do
                MONITOR_COUNTS["$name"]=0
            done
        fi
    done
}

# ─── Log Aggregation ───────────────────────────────────────────
aggregate_logs() {
    local log_dir=$1
    local pattern=${2:-"*.log"}
    local output_file=${3:-/tmp/aggregated.log}
    local since=${4:-"1 hour ago"}

    echo "Aggregating logs from: $log_dir"

    find "$log_dir" -name "$pattern" -newer <(date -d "$since" +%Y%m%d%H%M%S 2>/dev/null || \
        date -v-1H +%Y%m%d%H%M%S) -exec cat {} \; | \
        sort > "$output_file" 2>/dev/null

    echo "Aggregated to: $output_file ($(wc -l < "$output_file") lines)"
}

# ─── Error Rate Analysis ───────────────────────────────────────
error_rate_analysis() {
    local log_file=$1
    local window_minutes=${2:-5}
    local error_pattern=${3:-'ERROR|FATAL|CRITICAL'}

    echo "=== Error Rate Analysis ==="
    echo "Window: ${window_minutes} minutes"
    echo ""

    awk -v window="$window_minutes" -v pattern="$error_pattern" '
    {
        # Extract timestamp (ISO or common formats)
        if (match($0, /[0-9]{4}-[0-9]{2}-[0-9]{2}T?[0-9]{2}:[0-9]{2}/, ts)) {
            timestamp = ts[0]
            # Bucket by window
            gsub(/:/, "", timestamp)
            # Truncate to window
            bucket = substr(timestamp, 1, length(timestamp) - (window > 1 ? 2 : 0))
        } else {
            bucket = "unknown"
        }
        total[bucket]++
        if ($0 ~ pattern) errors[bucket]++
    }
    END {
        printf "%-20s %8s %8s %8s\n", "TIME_BUCKET", "TOTAL", "ERRORS", "ERROR%"
        for (b in total) {
            e = errors[b]+0
            pct = (total[b] > 0) ? int(e*100/total[b]) : 0
            printf "%-20s %8d %8d %7d%%\n", b, total[b], e, pct
        }
    }' "$log_file" | sort
}
```

---

## 48.3 Log Rotation and Archival

```bash
#!/bin/bash
# log_archiver.sh - Advanced log rotation and archival

LOG_ARCHIVE_DIR="${LOG_ARCHIVE_DIR:-/var/log/archive}"
LOG_COMPRESS_AFTER_DAYS="${LOG_COMPRESS_AFTER_DAYS:-7}"
LOG_DELETE_AFTER_DAYS="${LOG_DELETE_AFTER_DAYS:-90}"
LOG_MAX_SIZE="${LOG_MAX_SIZE:-100}"

rotate_log() {
    local log_file=$1
    local max_size_mb=${2:-$LOG_MAX_SIZE}
    local keep_count=${3:-5}
    local compress=${4:-true}

    [[ ! -f "$log_file" ]] && return 0

    local size_mb
    size_mb=$(du -m "$log_file" | cut -f1)

    (( size_mb < max_size_mb )) && return 0

    echo "Rotating: $log_file (${size_mb}MB)"

    for (( i=keep_count; i>=1; i-- )); do
        local current="${log_file}.${i}"
        local next="${log_file}.$((i+1))"

        [[ -f "$current.gz" ]] && mv "$current.gz" "${next}.gz"
        [[ -f "$current" ]] && {
            $compress && gzip -c "$current" > "${next}.gz" && rm "$current" || \
                mv "$current" "$next"
        }
    done

    $compress && gzip -c "$log_file" > "${log_file}.1.gz" || cp "$log_file" "${log_file}.1"

    > "$log_file"

    echo "Rotated: $log_file"
}

archive_old_logs() {
    local log_dir=${1:-/var/log}
    local pattern=${2:-"*.log"}

    mkdir -p "$LOG_ARCHIVE_DIR"

    echo "Archiving logs older than $LOG_COMPRESS_AFTER_DAYS days..."

    find "$log_dir" -maxdepth 1 -name "$pattern" \
        -mtime +"$LOG_COMPRESS_AFTER_DAYS" \
        ! -name "*.gz" \
        -type f | while IFS= read -r log_file; do

        local basename
        basename=$(basename "$log_file")
        local date_suffix
        date_suffix=$(date -r "$log_file" +%Y%m%d 2>/dev/null || \
                      stat -f %Sm -t %Y%m%d "$log_file" 2>/dev/null || \
                      date +%Y%m%d)
        local archive_file="${LOG_ARCHIVE_DIR}/${basename%.*}_${date_suffix}.log.gz"

        gzip -c "$log_file" > "$archive_file" && \
            rm "$log_file" && \
            echo "Archived: $basename → $archive_file"
    done

    echo "Deleting archives older than $LOG_DELETE_AFTER_DAYS days..."
    find "$LOG_ARCHIVE_DIR" -name "*.gz" -mtime +"$LOG_DELETE_AFTER_DAYS" -delete -print | \
        awk '{print "Deleted: " $0}'
}

# ─── Log Search ────────────────────────────────────────────────
log_search() {
    local pattern=$1
    local log_dir=${2:-/var/log}
    local since=${3:-"24 hours ago"}
    local case_insensitive=${4:-false}

    local grep_args=(-rn --include="*.log")
    $case_insensitive && grep_args+=(-i)

    echo "Searching for: $pattern (since: $since)"

    local newer_than
    newer_than=$(date -d "$since" +%Y%m%d%H%M%S 2>/dev/null || \
                 date -v-1d +%Y%m%d%H%M%S 2>/dev/null)

    if [[ -n "$newer_than" ]]; then
        find "$log_dir" -name "*.log" -newer <(touch -t "$newer_than" /tmp/_log_search_ref && echo /tmp/_log_search_ref) \
            2>/dev/null | xargs grep "${grep_args[@]:1}" "$pattern" 2>/dev/null
    else
        grep "${grep_args[@]}" "$pattern" "$log_dir/" 2>/dev/null
    fi
}
```

---

## 48.4 Log Shipping to ELK / OpenSearch

```bash
#!/bin/bash
# log_shipper.sh - Ship logs to Elasticsearch/OpenSearch

ES_HOST="${ES_HOST:-localhost}"
ES_PORT="${ES_PORT:-9200}"
ES_INDEX="${ES_INDEX:-app-logs}"
ES_AUTH="${ES_AUTH:-}"

es_send() {
    local document=$1
    local index=${2:-$ES_INDEX}

    local url="http://${ES_HOST}:${ES_PORT}/${index}/_doc"
    local auth_header=()
    [[ -n "$ES_AUTH" ]] && auth_header=(--user "$ES_AUTH")

    curl -sf \
        --request POST \
        --header "Content-Type: application/json" \
        "${auth_header[@]}" \
        "$url" \
        --data "$document" >/dev/null
}

es_bulk_send() {
    local documents_file=$1
    local index=${2:-$ES_INDEX}

    local url="http://${ES_HOST}:${ES_PORT}/_bulk"
    local auth_header=()
    [[ -n "$ES_AUTH" ]] && auth_header=(--user "$ES_AUTH")

    local bulk_data=""
    while IFS= read -r doc; do
        bulk_data+="{\"index\":{\"_index\":\"${index}\"}}"$'\n'
        bulk_data+="$doc"$'\n'
    done < "$documents_file"

    echo "$bulk_data" | curl -sf \
        --request POST \
        --header "Content-Type: application/x-ndjson" \
        "${auth_header[@]}" \
        "$url" \
        --data-binary @- >/dev/null
}

ship_log_file() {
    local log_file=$1
    local index=${2:-$ES_INDEX}
    local batch_size=${3:-100}
    local offset_file="${log_file}.offset"

    local offset=0
    [[ -f "$offset_file" ]] && offset=$(cat "$offset_file")

    local current_line=0
    local batch=()

    while IFS= read -r line; do
        (( current_line++ ))
        (( current_line <= offset )) && continue

        local doc
        doc=$(echo "$line" | python3 -c "
import sys, json
line = sys.stdin.read().strip()
try:
    doc = json.loads(line)
    if '@timestamp' not in doc:
        from datetime import datetime
        doc['@timestamp'] = datetime.utcnow().isoformat() + 'Z'
    print(json.dumps(doc))
except:
    print(json.dumps({'message': line, '@timestamp': '$(date -u +%Y-%m-%dT%H:%M:%SZ)'}))
" 2>/dev/null)

        batch+=("$doc")

        if (( ${#batch[@]} >= batch_size )); then
            local tmpfile
            tmpfile=$(mktemp)
            printf '%s\n' "${batch[@]}" > "$tmpfile"
            es_bulk_send "$tmpfile" "$index"
            rm -f "$tmpfile"
            batch=()
            echo "$current_line" > "$offset_file"
        fi
    done < "$log_file"

    if (( ${#batch[@]} > 0 )); then
        local tmpfile
        tmpfile=$(mktemp)
        printf '%s\n' "${batch[@]}" > "$tmpfile"
        es_bulk_send "$tmpfile" "$index"
        rm -f "$tmpfile"
        echo "$current_line" > "$offset_file"
    fi

    echo "Shipped $(( current_line - offset )) new lines from $log_file"
}
```

---

## 48.5 Log Dashboard

```bash
#!/bin/bash
# log_dashboard.sh - Terminal log dashboard

log_dashboard() {
    local refresh_secs=${1:-5}
    local log_files=("${@:2}")

    [[ ${#log_files[@]} -eq 0 ]] && log_files=(/var/log/syslog /var/log/nginx/access.log)

    while true; do
        clear
        echo "╔════════════════════════════════════════════════════════════╗"
        echo "║           LOG DASHBOARD - $(date '+%Y-%m-%d %H:%M:%S')           ║"
        echo "╚════════════════════════════════════════════════════════════╝"
        echo ""

        for log_file in "${log_files[@]}"; do
            [[ ! -f "$log_file" ]] && continue

            local size
            size=$(du -sh "$log_file" 2>/dev/null | cut -f1)
            local lines
            lines=$(wc -l < "$log_file" 2>/dev/null)
            local errors_1m
            errors_1m=$(tail -n 1000 "$log_file" | grep -cE "ERROR|FATAL|CRITICAL" 2>/dev/null)

            printf "┌─ %-50s ─┐\n" "$log_file"
            printf "│ Size: %-8s Lines: %-10s Errors(recent): %-6s │\n" \
                "$size" "$lines" "$errors_1m"
            echo "├─────────────────────────────────────────────────────────────┤"

            tail -n 5 "$log_file" | while IFS= read -r line; do
                local color=""
                if echo "$line" | grep -qE "ERROR|FATAL"; then
                    color="\033[0;31m"
                elif echo "$line" | grep -qE "WARN"; then
                    color="\033[0;33m"
                fi
                printf "│ ${color}%-62s\033[0m│\n" "${line:0:62}"
            done

            echo "└─────────────────────────────────────────────────────────────┘"
            echo ""
        done

        echo ""
        echo "Refresh every ${refresh_secs}s | Press Ctrl+C to exit"
        sleep "$refresh_secs"
    done
}
```

---

## 48.6 Exercises

### Exercise 1: Multi-Source Log Aggregator
สร้าง aggregator ที่:
- รวม logs จากหลาย servers ผ่าน SSH
- Normalize formats
- Store ใน central location
- Generate daily digest

### Exercise 2: Anomaly Detector
สร้าง detector ที่:
- เก็บ baseline metrics
- ตรวจจับ statistical anomalies
- Alert เมื่อ deviation > threshold
- Auto-create incident ticket

### Exercise 3: Log Cost Optimizer
สร้าง optimizer ที่:
- Analyze log verbosity
- Identify high-volume noisy logs
- Suggest sampling rates
- Calculate storage cost savings

---

## สรุป Part 48

✅ Nginx/Apache/Syslog parsing to JSON
✅ Real-time log monitoring with alerting rules
✅ Error rate analysis with time bucketing
✅ Log rotation with size triggers
✅ Log archival with compression and TTL
✅ Log search across multiple files
✅ Log shipping to Elasticsearch (bulk API)
✅ Terminal log dashboard with auto-refresh

---

**→ Part 49: Performance Tuning and Optimization**
