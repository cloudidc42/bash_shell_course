# Part 59: Log Analysis and Event Processing
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 59.1 Log Parsing Framework

```bash
#!/bin/bash
# log_parser.sh - Structured log parsing

LOG_FORMAT="${LOG_FORMAT:-combined}"

# ─── Multi-file Log Processing ───────────────────────────────────
parse_nginx_line() {
    local line=$1

    if [[ "$line" =~ ^([0-9.]+)[[:space:]]+-[[:space:]]+-[[:space:]]+\[([^]]+)\][[:space:]]+\"([A-Z]+)[[:space:]]+([^"]+)[[:space:]]+HTTP[^"]*\"[[:space:]]+([0-9]+)[[:space:]]+([0-9]+) ]]; then
        local ip="${BASH_REMATCH[1]}"
        local time="${BASH_REMATCH[2]}"
        local method="${BASH_REMATCH[3]}"
        local path="${BASH_REMATCH[4]}"
        local status="${BASH_REMATCH[5]}"
        local bytes="${BASH_REMATCH[6]}"

        printf '{"ip":"%s","time":"%s","method":"%s","path":"%s","status":%s,"bytes":%s}\n' \
            "$ip" "$time" "$method" "$path" "$status" "$bytes"
    fi
}

parse_log_file() {
    local log_file=$1 format=${2:-nginx_access}

    while IFS= read -r line; do
        case "$format" in
            nginx_access) parse_nginx_line "$line" ;;
            json)         echo "$line" ;;
            *)            echo "$line" ;;
        esac
    done < <(${LOG_GUNZIP:-cat} "$log_file" 2>/dev/null)
}

process_log_directory() {
    local log_dir=$1 format=${2:-nginx_access} pattern=${3:-*.log}

    find "$log_dir" -name "$pattern" -type f | sort | while read -r log_file; do
        echo "# Processing: $log_file" >&2
        parse_log_file "$log_file" "$format"
    done
}

process_all_rotated() {
    local log_dir=$1 basename=$2

    for log_file in "$log_dir/${basename}"*; do
        [[ -f "$log_file" ]] || continue
        if [[ "$log_file" == *.gz ]]; then
            LOG_GUNZIP="zcat" parse_log_file "$log_file"
        elif [[ "$log_file" == *.bz2 ]]; then
            LOG_GUNZIP="bzcat" parse_log_file "$log_file"
        else
            parse_log_file "$log_file"
        fi
    done
}
```

---

## 59.2 Log Analytics

```bash
#!/bin/bash
# log_analytics.sh - Log analysis and reporting

analyze_access_log() {
    local log_file=${1:-/var/log/nginx/access.log}
    local output_dir=${2:-/tmp/log_report}

    mkdir -p "$output_dir"
    echo "Analyzing: $log_file"

    # Request counts per status code
    awk '
    {
        match($0, /" ([0-9]{3}) /, arr)
        if (arr[1]) status[arr[1]]++
        total++
    }
    END {
        printf "\n=== HTTP Status Distribution ===\n"
        for (s in status) printf "  %s: %d (%.1f%%)\n", s, status[s], status[s]*100/total
        printf "  Total: %d\n", total
    }
    ' "$log_file" | tee "$output_dir/status.txt"

    # Top 20 IPs
    awk '{print $1}' "$log_file" | sort | uniq -c | sort -rn | head -20 | \
        awk 'BEGIN{print "\n=== Top 20 IPs ==="} {printf "  %-15s %d\n", $2, $1}' | \
        tee "$output_dir/top_ips.txt"

    # Top 20 URLs
    awk '{print $7}' "$log_file" | sort | uniq -c | sort -rn | head -20 | \
        awk 'BEGIN{print "\n=== Top 20 URLs ==="} {printf "  %-50s %d\n", $2, $1}' | \
        tee "$output_dir/top_urls.txt"

    # Requests per hour
    awk '
    {
        match($0, /\[([0-9]{2}\/[A-Za-z]+\/[0-9]{4}):([0-9]{2})/, arr)
        if (arr[2]) hourly[arr[1] " " arr[2]]++
    }
    END {
        printf "\n=== Hourly Distribution ===\n"
        for (h in hourly) printf "  %s: %d\n", h, hourly[h]
    }
    ' "$log_file" | sort | tee "$output_dir/hourly.txt"

    # Bandwidth usage
    awk '
    { bytes += $NF+0; count++ }
    END {
        printf "\n=== Bandwidth Summary ===\n"
        printf "  Total requests: %d\n", count
        printf "  Total bytes: %.2f MB\n", bytes/1024/1024
        printf "  Avg bytes/req: %.0f\n", (count > 0 ? bytes/count : 0)
    }
    ' "$log_file" | tee "$output_dir/bandwidth.txt"

    echo "Report saved to: $output_dir"
}

analyze_response_times() {
    local log_file=$1

    awk '
    {
        rt = $NF + 0
        if (rt > 0) {
            count++; sum += rt
            if (count == 1 || rt < min) min = rt
            if (rt > max) max = rt
            if (rt < 0.1)       bucket["<100ms"]++
            else if (rt < 0.5)  bucket["<500ms"]++
            else if (rt < 1.0)  bucket["<1s"]++
            else if (rt < 5.0)  bucket["<5s"]++
            else                bucket[">5s"]++
            times[count] = rt
        }
    }
    END {
        if (count == 0) exit
        printf "=== Response Time Analysis ===\n"
        printf "  Count: %d  Min: %.3fs  Max: %.3fs  Avg: %.3fs\n", count, min, max, sum/count
        n = asort(times)
        printf "  P50: %.3fs  P95: %.3fs  P99: %.3fs\n",
            times[int(n*0.50)], times[int(n*0.95)], times[int(n*0.99)]
        printf "  Distribution:\n"
        for (b in bucket) printf "    %-10s %d\n", b, bucket[b]
    }
    ' "$log_file"
}

analyze_auth_log() {
    local auth_log=${1:-/var/log/auth.log}
    local report_hours=${2:-24}

    echo "=== Auth Log Analysis (last ${report_hours}h) ==="

    grep "Failed password" "$auth_log" 2>/dev/null | \
        grep -oP 'from \K[0-9.]+' | sort | uniq -c | sort -rn | head -20 | \
        awk 'BEGIN{print "\nTop Failed Login IPs:"} {printf "  %-15s %d attempts\n", $2, $1}'

    grep "Accepted " "$auth_log" 2>/dev/null | \
        awk '{print $9, $11}' | sort | uniq -c | sort -rn | head -10 | \
        awk 'BEGIN{print "\nSuccessful Logins:"} {printf "  %-20s from %-15s (%d)\n", $2, $3, $1}'

    local brute_threshold=10
    grep "Failed password" "$auth_log" 2>/dev/null | \
        grep -oP 'from \K[0-9.]+' | sort | uniq -c | \
        awk -v t="$brute_threshold" '$1>=t{print $2,$1}' | \
        awk 'BEGIN{print "\nPotential Brute Force:"} {printf "  %-15s %d failures\n", $1, $2}'

    grep "sudo:" "$auth_log" 2>/dev/null | grep "COMMAND" | \
        awk 'BEGIN{print "\nSudo Commands:"} {print "  " $0}' | tail -20
}
```

---

## 59.3 Real-time Log Monitoring

```bash
#!/bin/bash
# log_monitor.sh - Real-time log event processing

declare -A MONITOR_RULES=()
declare -A RULE_COUNTS=()
declare -A RULE_WINDOWS=()

# ─── Rule Engine ──────────────────────────────────────────────────
rule_add() {
    local name=$1 pattern=$2 threshold=${3:-1} window=${4:-60} action=${5:-alert}
    MONITOR_RULES["$name"]="${pattern}|${threshold}|${window}|${action}"
    RULE_COUNTS["$name"]=0
    RULE_WINDOWS["$name"]=$(date +%s)
}

rule_evaluate() {
    local name=$1 line=$2
    local rule="${MONITOR_RULES[$name]}"
    [[ -z "$rule" ]] && return 0

    IFS='|' read -r pattern threshold window action <<< "$rule"

    if echo "$line" | grep -q "$pattern"; then
        local now; now=$(date +%s)
        local window_start="${RULE_WINDOWS[$name]}"

        if (( now - window_start > window )); then
            RULE_COUNTS["$name"]=0
            RULE_WINDOWS["$name"]=$now
        fi

        (( RULE_COUNTS["$name"]++ ))

        if (( RULE_COUNTS["$name"] >= threshold )); then
            rule_trigger "$name" "$line" "${RULE_COUNTS[$name]}"
            RULE_COUNTS["$name"]=0
            RULE_WINDOWS["$name"]=$now
        fi
    fi
}

rule_trigger() {
    local name=$1 line=$2 count=$3
    local rule="${MONITOR_RULES[$name]}"
    IFS='|' read -r _ _ _ action <<< "$rule"

    local message="[ALERT] Rule '${name}' triggered (${count}x): ${line:0:100}"
    echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') $message"

    case "$action" in
        alert) echo "$message" ;;
        slack)
            [[ -n "${SLACK_WEBHOOK:-}" ]] && \
                curl -s -X POST "$SLACK_WEBHOOK" \
                    -H "Content-Type: application/json" \
                    -d "{\"text\":\"$message\"}" &>/dev/null & ;;
        block_ip)
            local ip; ip=$(echo "$line" | grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' | head -1)
            [[ -n "$ip" ]] && iptables -I INPUT -s "$ip" -j DROP 2>/dev/null && \
                echo "Blocked IP: $ip" ;;
    esac
}

monitor_log() {
    local log_file=$1
    [[ ! -f "$log_file" ]] && { echo "Log not found: $log_file"; return 1; }

    echo "Monitoring: $log_file"
    tail -n 0 -F "$log_file" 2>/dev/null | while IFS= read -r line; do
        for rule_name in "${!MONITOR_RULES[@]}"; do
            rule_evaluate "$rule_name" "$line"
        done
    done
}

load_web_security_rules() {
    rule_add "sql_injection"  "union.*select\|drop.*table\|xp_cmdshell" 5 60 "alert"
    rule_add "xss_attempt"    "<script\|javascript:\|onerror=" 10 60 "alert"
    rule_add "path_traversal" "\.\./\|%2E%2E" 5 60 "alert"
    rule_add "brute_force"    '" 401 ' 20 60 "alert"
    rule_add "scanner_detect" "nikto\|sqlmap\|nmap\|masscan" 1 3600 "block_ip"
    echo "Loaded web security rules"
}

load_system_rules() {
    rule_add "oom_killer"    "Out of memory.*killed process" 1 60 "alert"
    rule_add "disk_full"     "No space left on device"       1 300 "alert"
    rule_add "ssh_brute"     "Failed password for"           10 60 "alert"
    rule_add "service_crash" "segfault\|core dumped\|killed" 1 60 "alert"
    echo "Loaded system rules"
}
```

---

## 59.4 Log Aggregation

```bash
#!/bin/bash
# log_aggregation.sh - Centralized log collection

LOG_AGGREGATE_DIR="${LOG_AGGREGATE_DIR:-/var/log/aggregated}"

collect_service_logs() {
    local service=$1 output_dir=${2:-$LOG_AGGREGATE_DIR}

    mkdir -p "$output_dir/$service"

    if command -v journalctl &>/dev/null; then
        journalctl --unit="$service" --since="1 hour ago" \
            --output=json --no-pager 2>/dev/null > \
            "$output_dir/$service/$(date +%Y%m%d_%H%M%S).jsonl"
    fi

    local log_paths=(
        "/var/log/${service}.log"
        "/var/log/${service}/${service}.log"
        "/var/log/${service}/access.log"
        "/var/log/${service}/error.log"
    )

    for log_path in "${log_paths[@]}"; do
        [[ -f "$log_path" ]] || continue
        rsync -a --no-owner "$log_path" "$output_dir/$service/" 2>/dev/null
    done
}

rotate_log() {
    local log_file=$1 max_size_mb=${2:-100} keep_files=${3:-5}

    [[ ! -f "$log_file" ]] && return 0

    local size_mb; size_mb=$(du -m "$log_file" | cut -f1)

    if (( size_mb >= max_size_mb )); then
        for (( i=keep_files-1; i>=1; i-- )); do
            [[ -f "${log_file}.${i}" ]] && mv "${log_file}.${i}" "${log_file}.$((i+1))"
        done

        for (( i=2; i<=keep_files; i++ )); do
            local f="${log_file}.${i}"
            [[ -f "$f" ]] && [[ "$f" != *.gz ]] && gzip -f "$f"
        done

        mv "$log_file" "${log_file}.1"
        touch "$log_file"

        if [[ -f "${log_file}.pid" ]]; then
            kill -USR1 "$(cat "${log_file}.pid")" 2>/dev/null
        fi

        echo "Rotated: $log_file (was ${size_mb}MB)"
        return 1
    fi

    return 0
}

log_statistics() {
    local log_dir=${1:-/var/log}

    echo "=== Log Statistics ==="
    find "$log_dir" -name "*.log" -maxdepth 3 2>/dev/null | while read -r f; do
        local size; size=$(du -sh "$f" | cut -f1)
        local lines; lines=$(wc -l < "$f" 2>/dev/null || echo 0)
        printf "  %-40s %6s  %8d lines\n" "${f#$log_dir/}" "$size" "$lines"
    done | sort -k2 -rh
}
```

---

## 59.5 Exercises

### Exercise 1: Log Dashboard
สร้าง real-time dashboard ที่:
- Show request rate per second
- Display error rate (text-based graph)
- Top IPs and URLs live-updating
- Alert on anomalies

### Exercise 2: Log Correlation
สร้าง correlation engine ที่:
- Correlate events across multiple log files
- Detect attack patterns spanning services
- Timeline reconstruction

### Exercise 3: Anomaly Detection
สร้าง anomaly detector ที่:
- Learn baseline request patterns
- Detect statistical outliers
- Generate anomaly scores
- Auto-block suspicious IPs

---

## สรุป Part 59

✅ Log parsing framework with nginx/apache to JSON
✅ Multi-file and rotated log processing (gz, bz2)
✅ Access log analytics: status codes, top IPs/URLs, hourly distribution
✅ Response time percentiles (P50/P95/P99) via awk asort
✅ Auth log analysis: failed logins, brute force detection, sudo audit
✅ Real-time rule engine with threshold/window/action support
✅ Predefined web security and system rules
✅ Log rotation with compression and signal-based reopen

---

**→ Part 60: API Development and Web Scraping**
