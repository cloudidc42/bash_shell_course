# Part 82: Log Management and Analysis Pipelines
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 82.1 Log Collection and Tailing

```bash
#!/bin/bash
# log_collect.sh - Multi-file log collection

set -euo pipefail

log_tail_multi() {
    local -a files=("$@")
    local tmpdir; tmpdir=$(mktemp -d)
    local -a pids=()

    trap 'kill "${pids[@]}" 2>/dev/null; rm -rf "$tmpdir"' EXIT INT TERM

    for file in "${files[@]}"; do
        [[ -f "$file" ]] || continue
        tail -F "$file" | \
            awk -v src="$file" '{ print strftime("%Y-%m-%dT%H:%M:%S"), src": "$0 }' &
        pids+=($!)
    done

    wait
}

log_watch_dir() {
    local dir=$1 pattern=${2:-*.log}
    local tmpdir; tmpdir=$(mktemp -d)
    local -a pids=()

    trap 'kill "${pids[@]}" 2>/dev/null; rm -rf "$tmpdir"' EXIT INT TERM

    # Start tailing existing files
    for f in "${dir}"/${pattern}; do
        [[ -f "$f" ]] || continue
        tail -F "$f" | awk -v src="$(basename "$f")" '{print src": "$0}' &
        pids+=($!)
    done

    echo "Watching: $dir/$pattern (${#pids[@]} files)"
    wait
}

log_pipe_to_file() {
    local output_file=$1 max_size_mb=${2:-100}
    local max_bytes=$(( max_size_mb * 1024 * 1024 ))
    local current_bytes=0
    local rotate_count=0

    while IFS= read -r line; do
        echo "$line" >> "$output_file"
        current_bytes=$(( current_bytes + ${#line} + 1 ))

        if (( current_bytes >= max_bytes )); then
            mv "$output_file" "${output_file}.${rotate_count}"
            gzip -f "${output_file}.${rotate_count}" &
            (( rotate_count++ ))
            current_bytes=0
        fi
    done
}
```

---

## 82.2 Log Parsers

```bash
#!/bin/bash
# log_parsers.sh - Structured log field extractors

parse_nginx_access() {
    # Combined Log Format: IP - user [time] "METHOD /path HTTP/x" status bytes "referer" "ua"
    awk '{
        ip     = $1
        user   = $3
        ts     = substr($4, 2)  # strip leading [
        method = substr($6, 2)  # strip leading "
        path   = $7
        status = $9
        bytes  = $10
        printf "{\"ip\":\"%s\",\"user\":\"%s\",\"ts\":\"%s\",\"method\":\"%s\",\"path\":\"%s\",\"status\":%s,\"bytes\":%s}\n",
            ip, user, ts, method, path, status, (bytes=="-" ? 0 : bytes)
    }'
}

parse_apache_access() { parse_nginx_access; }

parse_syslog() {
    # Syslog: Mon DD HH:MM:SS hostname process[pid]: message
    awk '{
        month = $1; day = $2; time = $3; host = $4
        proc_pid = $5
        split(proc_pid, a, "[")
        proc = a[1]; sub("\\]:?", "", a[2]); pid = a[2]
        $1=$2=$3=$4=$5=""; msg = substr($0, 6)
        printf "{\"month\":\"%s\",\"day\":\"%s\",\"time\":\"%s\",\"host\":\"%s\",\"proc\":\"%s\",\"pid\":\"%s\",\"msg\":\"%s\"}\n",
            month, day, time, host, proc, pid, msg
    }'
}

parse_json_log() {
    # Pass-through JSON, adding source file if given
    local source_file=${1:-}
    if [[ -n "$source_file" ]]; then
        jq -c --arg src "$source_file" '. + {_source: $src}' 2>/dev/null
    else
        jq -c '.' 2>/dev/null
    fi
}

parse_key_value_log() {
    # key=value key2="multi word" style logs
    awk '{
        printf "{"
        first = 1
        n = split($0, fields, " ")
        for (i=1; i<=n; i++) {
            if (fields[i] ~ /=/) {
                split(fields[i], kv, "=")
                val = kv[2]; gsub(/^"|"$/, "", val)
                if (!first) printf ","
                printf "\"" kv[1] "\":\"" val "\""
                first = 0
            }
        }
        printf "}\n"
    }'
}

extract_log_level() {
    awk '{
        level = "INFO"
        if ($0 ~ /CRIT|CRITICAL|FATAL/)  level = "CRITICAL"
        else if ($0 ~ /ERROR|ERR/)        level = "ERROR"
        else if ($0 ~ /WARN|WARNING/)     level = "WARN"
        else if ($0 ~ /DEBUG/)            level = "DEBUG"
        print level, $0
    }'
}
```

---

## 82.3 Log Rotation

```bash
#!/bin/bash
# log_rotate.sh - Log rotation with compression and retention

LOG_ROTATE_CONF_DIR="${LOG_ROTATE_CONF_DIR:-/etc/log_rotate.d}"

rotate_log_file() {
    local log_file=$1
    local max_size_mb=${2:-100}
    local keep_days=${3:-30}
    local compress=${4:-true}

    [[ -f "$log_file" ]] || return 0

    local size_mb; size_mb=$(( $(stat -c '%s' "$log_file" 2>/dev/null || echo 0) / 1024 / 1024 ))

    if (( size_mb < max_size_mb )); then
        return 0
    fi

    local ts; ts=$(date '+%Y%m%d_%H%M%S')
    local rotated="${log_file}.${ts}"

    # Rotate
    mv "$log_file" "$rotated"
    touch "$log_file"
    chmod --reference="$rotated" "$log_file" 2>/dev/null || true

    # Signal process to reopen log
    local service; service=$(basename "${log_file%.*}")
    systemctl reload "$service" 2>/dev/null || \
        pkill -HUP -x "$service" 2>/dev/null || true

    # Compress
    if $compress; then
        gzip -f "$rotated" &
        rotated="${rotated}.gz"
    fi

    # Cleanup old rotated logs
    find "$(dirname "$log_file")" \
        -name "$(basename "$log_file").*" \
        -mtime +"$keep_days" \
        -delete 2>/dev/null

    echo "Rotated: $log_file -> $rotated"
}

rotate_dir() {
    local dir=$1 pattern=${2:-*.log}
    local max_size_mb=${3:-100} keep_days=${4:-30}

    for log_file in "${dir}"/${pattern}; do
        [[ -f "$log_file" ]] || continue
        rotate_log_file "$log_file" "$max_size_mb" "$keep_days"
    done
}

rotate_time_based() {
    local log_file=$1 interval=${2:-daily}
    local ts; ts=$(date '+%Y%m%d')
    [[ "$interval" == "hourly" ]] && ts=$(date '+%Y%m%d_%H')

    local rotated="${log_file}.${ts}"
    [[ -f "$rotated" ]] && return 0  # Already rotated this period

    mv "$log_file" "$rotated"
    touch "$log_file"
    gzip -f "$rotated" &
    echo "Time-rotated: $rotated"
}
```

---

## 82.4 Real-time Log Alerting

```bash
#!/bin/bash
# log_alert.sh - Pattern matching alerts on live log stream

declare -A LOG_ALERT_LAST=()
LOG_ALERT_COOLDOWN="${LOG_ALERT_COOLDOWN:-300}"

log_alert_cooldown_ok() {
    local key=$1
    local now; now=$(date +%s)
    local last="${LOG_ALERT_LAST[$key]:-0}"
    if (( now - last >= LOG_ALERT_COOLDOWN )); then
        LOG_ALERT_LAST["$key"]=$now
        return 0
    fi
    return 1
}

log_alert_fire() {
    local key=$1 message=$2 level=${3:-warning}
    local ts; ts=$(date '+%Y-%m-%dT%H:%M:%S')

    echo "[ALERT:${level^^}] $ts | $key | $message" >&2

    # Slack notification
    if [[ -n "${SLACK_WEBHOOK:-}" ]]; then
        curl -s -X POST "$SLACK_WEBHOOK" \
            --header 'Content-Type: application/json' \
            --data "{\"text\":\"[$level] $key: $message\"}" > /dev/null &
    fi
}

monitor_log_patterns() {
    local log_file=$1

    # Define patterns: key:pattern:level
    local -a rules=(
        "error:ERROR:error"
        "oom:Out of memory:critical"
        "disk_full:No space left on device:critical"
        "auth_fail:authentication failure:warning"
        "timeout:Connection timed out:warning"
    )

    tail -F "$log_file" 2>/dev/null | while IFS= read -r line; do
        for rule in "${rules[@]}"; do
            IFS=':' read -r key pattern level <<< "$rule"
            if echo "$line" | grep -qi "$pattern"; then
                if log_alert_cooldown_ok "$key"; then
                    log_alert_fire "$key" "$line" "$level"
                fi
            fi
        done
    done
}

monitor_error_rate() {
    local log_file=$1 threshold=${2:-10} window_sec=${3:-60}
    local -i count=0 window_start=0

    tail -F "$log_file" 2>/dev/null | while IFS= read -r line; do
        echo "$line" | grep -qi 'error' || continue

        local now; now=$(date +%s)
        if (( now - window_start >= window_sec )); then
            window_start=$now
            count=0
        fi

        (( count++ ))
        if (( count >= threshold )); then
            log_alert_fire "error_rate" "${count} errors in ${window_sec}s" "critical"
            count=0
        fi
    done
}
```

---

## 82.5 Log Statistical Analysis

```bash
#!/bin/bash
# log_stats.sh - Statistical analysis on log files

log_request_rate() {
    local log_file=$1 time_field=${2:-1}
    awk -v tf="$time_field" '{
        split($tf, t, ":")
        hour_min = t[2] ":" substr(t[3], 1, 2)
        counts[hour_min]++
    }
    END {
        for (hm in counts) print counts[hm], hm
    }' "$log_file" | sort -k2 | \
    awk '{printf "%s  %d req\n", $2, $1}'
}

log_status_distribution() {
    local log_file=$1 status_field=${2:-9}
    awk -v sf="$status_field" '{
        counts[$sf]++; total++
    }
    END {
        for (s in counts)
            printf "%-6s %6d  %5.1f%%\n", s, counts[s], counts[s]/total*100
    }' "$log_file" | sort -k1
}

log_top_ips() {
    local log_file=$1 top_n=${2:-20} ip_field=${3:-1}
    awk -v f="$ip_field" '{print $f}' "$log_file" | \
        sort | uniq -c | sort -rn | head -"$top_n" | \
        awk '{printf "%-20s %d\n", $2, $1}'
}

log_top_paths() {
    local log_file=$1 top_n=${2:-20} path_field=${3:-7}
    awk -v f="$path_field" '{print $f}' "$log_file" | \
        sed 's/?.*//' | \
        sort | uniq -c | sort -rn | head -"$top_n" | \
        awk '{printf "%6d  %s\n", $1, $2}'
}

log_percentile() {
    local log_file=$1 field=${2:-10} percentile=${3:-95}
    awk -v f="$field" '$f ~ /^[0-9]+$/ {print $f}' "$log_file" | \
    sort -n | \
    awk -v p="$percentile" '
    { lines[NR] = $1 }
    END {
        idx = int(NR * p / 100)
        if (idx < 1) idx = 1
        if (idx > NR) idx = NR
        printf "P%d response: %d bytes\n", p, lines[idx]
    }'
}

log_error_summary() {
    local log_file=$1 last_n=${2:-1000}
    echo "=== Error Summary (last $last_n lines) ==="
    tail -n "$last_n" "$log_file" | \
        grep -i 'error\|warn\|crit' | \
        sed 's/[0-9]\+/N/g' | \
        sort | uniq -c | sort -rn | head -20 | \
        awk '{count=$1; $1=""; printf "  %5d  %s\n", count, $0}'
}

log_daily_summary() {
    local log_dir=$1 pattern=${2:-access.log}
    echo "=== Daily Log Summary ==="
    for f in "${log_dir}"/${pattern}*; do
        [[ -f "$f" ]] || continue
        local lines; lines=$(wc -l < "$f")
        local size; size=$(du -sh "$f" | cut -f1)
        printf '  %-40s %8d lines  %s\n' "$(basename "$f")" "$lines" "$size"
    done
}
```

---

## 82.6 Log Shipping

```bash
#!/bin/bash
# log_ship.sh - Forward logs to external systems

ship_to_elasticsearch() {
    local index=$1 es_url=${2:-http://localhost:9200}

    while IFS= read -r line; do
        local ts; ts=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
        local doc; doc=$(echo "$line" | jq -Rc --arg ts "$ts" \
            '{"@timestamp":$ts, "message":.}' 2>/dev/null || \
            echo "{\"@timestamp\":\"$ts\",\"message\":\"$line\"}")

        curl -s -X POST \
            --header 'Content-Type: application/json' \
            --data "$doc" \
            "${es_url}/${index}/_doc" > /dev/null
    done
}

ship_to_loki() {
    local stream_labels=$1 loki_url=${2:-http://localhost:3100}
    local -a batch_lines=()
    local batch_size=100

    push_batch() {
        local -a values=()
        for line in "${batch_lines[@]}"; do
            local ts; ts=$(date +%s%N)
            values+=("[\"$ts\", $(echo "$line" | jq -Rs .)]")
        done

        local payload; payload=$(printf '{"streams":[{"stream":%s,"values":[%s]}]}' \
            "$stream_labels" \
            "$(IFS=','; echo "${values[*]}")")

        curl -s -X POST \
            --header 'Content-Type: application/json' \
            --data "$payload" \
            "${loki_url}/loki/api/v1/push" > /dev/null
        batch_lines=()
    }

    while IFS= read -r line; do
        batch_lines+=("$line")
        (( ${#batch_lines[@]} >= batch_size )) && push_batch
    done
    (( ${#batch_lines[@]} > 0 )) && push_batch
}

ship_to_syslog() {
    local host=${1:-localhost} port=${2:-514} facility=${3:-local0}
    while IFS= read -r line; do
        logger --server "$host" --port "$port" \
            --priority "${facility}.info" \
            -- "$line" 2>/dev/null
    done
}

archive_logs_to_s3() {
    local log_dir=$1 bucket=$2 prefix=${3:-logs}
    local date_path; date_path=$(date '+%Y/%m/%d')

    find "$log_dir" -name '*.gz' -mtime +1 | while read -r f; do
        local key="${prefix}/${date_path}/$(basename "$f")"
        aws s3 cp "$f" "s3://${bucket}/${key}" --only-show-errors
        echo "Archived: $key"
        rm -f "$f"
    done
}
```

---

## 82.7 Exercises

### Exercise 1: Log Pipeline
สร้าง pipeline:
- tail -F nginx/access.log
- parse → JSON
- filter status >= 500
- batch 100 records
- ship to Elasticsearch

### Exercise 2: Log Dashboard
สร้าง terminal dashboard ที่:
- Request rate per minute (ASCII bar)
- HTTP status distribution
- Top 10 IPs
- P50/P95/P99 response time
- Live refresh ทุก 5 วินาที

### Exercise 3: Anomaly Detector
สร้าง anomaly detection ที่:
- คำนวณ baseline error rate
- Alert ถ้า current rate > 3x baseline
- Track per-path anomalies
- Store history ใน SQLite

---

## สรุป Part 82

✅ log_tail_multi: tail -F หลายไฟล์ + prefix source + timestamp
┅ log_pipe_to_file: auto-rotate by size with gzip background
┅ parse_nginx/apache/syslog/json/kv_log: JSON output per record
┅ extract_log_level: CRITICAL/ERROR/WARN/DEBUG/INFO classification
┅ log_rotate: size-based rotation, SIGHUP, gzip, retention cleanup
┅ rotate_time_based: daily/hourly rotation by naming
┅ monitor_log_patterns: configurable rule set with cooldown
┅ monitor_error_rate: sliding window error rate alert
┅ log stats: request rate, status dist, top IPs/paths, percentile, error summary
┅ ship_to_elasticsearch/loki/syslog, archive_logs_to_s3

---

**→ Part 83: Shell Scripting for Web Scraping and Crawling**
