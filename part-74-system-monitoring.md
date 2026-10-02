# Part 74: System Monitoring and Observability Dashboards
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 74.1 /proc-Based Metric Collection

```bash
#!/bin/bash
# proc_metrics.sh - Low-level system metrics from /proc

set -euo pipefail

# ─── CPU ───────────────────────────────────────────────────────────────────
read_cpu_stats() {
    awk '/^cpu / {print $2,$3,$4,$5,$6,$7,$8}' /proc/stat
}

cpu_percent() {
    local interval=${1:-1}

    local stats1; stats1=$(read_cpu_stats)
    sleep "$interval"
    local stats2; stats2=$(read_cpu_stats)

    awk -v s1="$stats1" -v s2="$stats2" 'BEGIN {
        n1=split(s1,a," "); n2=split(s2,b," ")
        idle1=a[4]; idle2=b[4]
        total1=0; total2=0
        for(i=1;i<=n1;i++) total1+=a[i]
        for(i=1;i<=n2;i++) total2+=b[i]
        diff_idle=idle2-idle1
        diff_total=total2-total1
        printf "%.1f\n", 100*(1 - diff_idle/diff_total)
    }'
}

cpu_per_core() {
    awk '/^cpu[0-9]/ {idle=$5; total=0; for(i=2;i<=NF;i++) total+=$i; printf "%s: %.1f%%\n", $1, 100*(1-idle/total)}' /proc/stat
}

# ─── Memory ─────────────────────────────────────────────────────────────────
memory_stats() {
    awk '
    /^MemTotal:/    { total=$2 }
    /^MemFree:/     { free=$2 }
    /^MemAvailable/ { avail=$2 }
    /^Buffers:/     { buffers=$2 }
    /^Cached:/      { cached=$2 }
    /^SwapTotal:/   { stotal=$2 }
    /^SwapFree:/    { sfree=$2 }
    END {
        used = total - avail
        swap_used = stotal - sfree
        printf "total_mb=%d used_mb=%d free_mb=%d avail_mb=%d swap_used_mb=%d swap_total_mb=%d\n",
            total/1024, used/1024, free/1024, avail/1024, swap_used/1024, stotal/1024
    }' /proc/meminfo
}

memory_percent() {
    awk '/^MemTotal:/ {t=$2} /^MemAvailable:/ {a=$2} END {printf "%.1f", 100*(t-a)/t}' /proc/meminfo
}

# ─── Disk I/O ───────────────────────────────────────────────────────────────
disk_io_stats() {
    local device=${1:-sda}
    awk -v dev="$device" '$3==dev {print $4,$6,$8,$10}' /proc/diskstats
}

disk_io_rate() {
    local device=${1:-sda} interval=${2:-1}

    local before; before=$(disk_io_stats "$device")
    sleep "$interval"
    local after; after=$(disk_io_stats "$device")

    awk -v b="$before" -v a="$after" -v dt="$interval" 'BEGIN {
        split(b,B," "); split(a,A," ")
        reads  = (A[1]-B[1]) / dt
        writes = (A[3]-B[3]) / dt
        read_kb  = (A[2]-B[2]) * 512 / 1024 / dt
        write_kb = (A[4]-B[4]) * 512 / 1024 / dt
        printf "reads=%.0f/s writes=%.0f/s read_kb=%.0f/s write_kb=%.0f/s\n",
            reads, writes, read_kb, write_kb
    }'
}

# ─── Network ─────────────────────────────────────────────────────────────────
net_stats() {
    local iface=${1:-eth0}
    awk -v iface="${iface}:" '$1==iface {print $2,$3,$10,$11}' /proc/net/dev
}

net_bandwidth() {
    local iface=${1:-eth0} interval=${2:-1}

    local before; before=$(net_stats "$iface")
    sleep "$interval"
    local after; after=$(net_stats "$iface")

    awk -v b="$before" -v a="$after" -v dt="$interval" 'BEGIN {
        split(b,B," "); split(a,A," ")
        rx_kb = (A[1]-B[1]) / 1024 / dt
        rx_pkt = (A[2]-B[2]) / dt
        tx_kb = (A[3]-B[3]) / 1024 / dt
        tx_pkt = (A[4]-B[4]) / dt
        printf "rx=%.0fKB/s rx_pkt=%.0f/s tx=%.0fKB/s tx_pkt=%.0f/s\n",
            rx_kb, rx_pkt, tx_kb, tx_pkt
    }'
}

# ─── Load ───────────────────────────────────────────────────────────────────
system_load() {
    awk '{printf "1m=%.2f 5m=%.2f 15m=%.2f procs=%s\n", $1, $2, $3, $4}' /proc/loadavg
}

cpu_count() {
    grep -c '^processor' /proc/cpuinfo
}

uptime_info() {
    awk '{
        total=$1; days=int(total/86400); hours=int((total%86400)/3600)
        mins=int((total%3600)/60); secs=int(total%60)
        printf "uptime=%dd %dh %dm %ds\n", days, hours, mins, secs
    }' /proc/uptime
}
```

---

## 74.2 ASCII Real-Time Dashboard

```bash
#!/bin/bash
# dashboard.sh - ASCII real-time system dashboard

DASH_INTERVAL="${DASH_INTERVAL:-2}"
DASH_TOP_PROCS="${DASH_TOP_PROCS:-8}"

draw_bar() {
    local percent=$1 width=${2:-30} filled_char=${3:-#} empty_char=${4:-.}
    local filled=$(( percent * width / 100 ))
    local empty=$(( width - filled ))
    local bar=""
    local i
    for (( i=0; i<filled; i++ )); do bar+="$filled_char"; done
    for (( i=0; i<empty; i++ )); do bar+="$empty_char"; done
    echo "[$bar] ${percent}%"
}

dash_header() {
    local hostname; hostname=$(hostname -s 2>/dev/null || echo host)
    local ts; ts=$(date '+%Y-%m-%d %H:%M:%S')
    printf '\033[1;36m=== System Dashboard: %-20s %s ===\033[0m\n' "$hostname" "$ts"
}

dash_cpu_section() {
    local cpu_pct; cpu_pct=$(cpu_percent 1)
    local load; load=$(awk '{print $1"/"$2"/"$3}' /proc/loadavg)
    local ncpu; ncpu=$(cpu_count)

    printf '\n\033[1mCPU\033[0m  cores=%-2d load=%s\n' "$ncpu" "$load"
    printf '  Usage: %s\n' "$(draw_bar "${cpu_pct%.*}")"
}

dash_memory_section() {
    local stats; stats=$(memory_stats)
    local total_mb; total_mb=$(echo "$stats" | grep -oP 'total_mb=\K\d+')
    local used_mb;  used_mb=$(echo  "$stats" | grep -oP 'used_mb=\K\d+')
    local pct=$(( used_mb * 100 / (total_mb > 0 ? total_mb : 1) ))

    printf '\n\033[1mMemory\033[0m  %dMB / %dMB\n' "$used_mb" "$total_mb"
    printf '  Usage: %s\n' "$(draw_bar "$pct")"
}

dash_disk_section() {
    printf '\n\033[1mDisk\033[0m\n'
    df -h --output=source,size,used,avail,pcent,target 2>/dev/null | \
        grep -v 'tmpfs\|devtmpfs\|udev\|Filesystem' | head -5 | \
    while read -r src size used avail pct mount; do
        local pct_num="${pct//%/}"
        printf '  %-15s %4s used / %4s total %3s  %s\n' \
            "$mount" "$used" "$size" "$pct" "$(draw_bar "${pct_num:-0}" 20)"
    done
}

dash_top_processes() {
    printf '\n\033[1mTop Processes\033[0m\n'
    printf '  %-8s %-25s %6s %6s\n' 'PID' 'Command' 'CPU%' 'MEM%'
    ps --no-headers -eo pid,comm,pcpu,pmem --sort=-pcpu 2>/dev/null | \
        head -"$DASH_TOP_PROCS" | \
    while read -r pid comm cpu mem; do
        printf '  %-8s %-25s %6s %6s\n' "$pid" "$comm" "$cpu" "$mem"
    done
}

dash_services_section() {
    local services=("${@:-ssh nginx mysql postgresql redis docker}")
    printf '\n\033[1mServices\033[0m\n'
    for svc in ${services[@]}; do
        if systemctl is-active --quiet "$svc" 2>/dev/null; then
            printf '  \033[0;32m[RUNNING]\033[0m %s\n' "$svc"
        else
            printf '  \033[0;31m[STOPPED]\033[0m %s\n' "$svc"
        fi
    done
}

run_dashboard() {
    while true; do
        clear
        dash_header
        dash_cpu_section
        dash_memory_section
        dash_disk_section
        dash_top_processes
        echo ""
        echo "Press Ctrl+C to exit  |  Refresh: ${DASH_INTERVAL}s"
        sleep "$DASH_INTERVAL"
    done
}
```

---

## 74.3 Service Watchdog

```bash
#!/bin/bash
# watchdog.sh - Service monitoring and auto-restart

WATCHDOG_INTERVAL="${WATCHDOG_INTERVAL:-10}"
WATCHDOG_MAX_RESTARTS="${WATCHDOG_MAX_RESTARTS:-5}"
WATCHDOG_RESTART_WINDOW="${WATCHDOG_RESTART_WINDOW:-300}"
WATCHDOG_LOG="${WATCHDOG_LOG:-/tmp/watchdog.log}"

declare -A WATCHDOG_RESTART_COUNT=()
declare -A WATCHDOG_RESTART_TIMES=()

watchdog_log() {
    printf '%s [WATCHDOG] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "$WATCHDOG_LOG"
}

watchdog_check_service() {
    local name=$1 check_cmd=$2 restart_cmd=$3 alert_cmd=${4:-}

    if eval "$check_cmd" > /dev/null 2>&1; then
        return 0
    fi

    watchdog_log "Service DOWN: $name"

    # Count restarts within window
    local now; now=$(date +%s)
    local window_start=$(( now - WATCHDOG_RESTART_WINDOW ))
    local times="${WATCHDOG_RESTART_TIMES[$name]:-}"
    local recent=0

    if [[ -n "$times" ]]; then
        recent=$(echo "$times" | tr ' ' '\n' | awk -v ws="$window_start" '$1>=ws' | wc -l)
    fi

    if (( recent >= WATCHDOG_MAX_RESTARTS )); then
        watchdog_log "CRITICAL: $name exceeded max restarts ($WATCHDOG_MAX_RESTARTS) in window"
        [[ -n "$alert_cmd" ]] && eval "$alert_cmd" &
        return 2
    fi

    watchdog_log "Restarting: $name (restart #$((recent+1)))"
    WATCHDOG_RESTART_TIMES["$name"]+="$now "

    if eval "$restart_cmd" > /dev/null 2>&1; then
        sleep 2
        if eval "$check_cmd" > /dev/null 2>&1; then
            watchdog_log "Service recovered: $name"
            return 0
        fi
    fi

    watchdog_log "Restart FAILED: $name"
    return 1
}

watchdog_run() {
    local config_file=${1:-watchdog.conf}

    watchdog_log "Watchdog started (interval=${WATCHDOG_INTERVAL}s max_restarts=${WATCHDOG_MAX_RESTARTS})"

    while true; do
        while IFS='|' read -r name check_cmd restart_cmd alert_cmd; do
            [[ "$name" =~ ^# || -z "$name" ]] && continue
            watchdog_check_service "$name" "$check_cmd" "$restart_cmd" "$alert_cmd"
        done < "$config_file"
        sleep "$WATCHDOG_INTERVAL"
    done
}

# watchdog.conf format: name|check_command|restart_command|alert_command
# Example:
# nginx|systemctl is-active nginx|systemctl restart nginx|notify_slack "nginx died"
# redis|redis-cli ping|systemctl restart redis|
```

---

## 74.4 Alert Engine

```bash
#!/bin/bash
# alert_engine.sh - Threshold-based alerting

ALERT_COOLDOWN="${ALERT_COOLDOWN:-300}"
ALERT_STATE_DIR="${ALERT_STATE_DIR:-/tmp/alerts}"
mkdir -p "$ALERT_STATE_DIR"

alert_check_threshold() {
    local name=$1 value=$2 operator=$3 threshold=$4
    local severity=${5:-WARNING} message=${6:-"$name is $value"}

    local triggered=false
    case "$operator" in
        '>')  (( $(echo "$value > $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        '<')  (( $(echo "$value < $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        '>=') (( $(echo "$value >= $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        '<=') (( $(echo "$value <= $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        '!=') [[ "$value" != "$threshold" ]] && triggered=true ;;
    esac

    if $triggered; then
        alert_fire "$name" "$severity" "$message (value=$value, threshold=${operator}${threshold})"
    else
        alert_resolve "$name"
    fi
}

alert_fire() {
    local name=$1 severity=$2 message=$3
    local state_file="${ALERT_STATE_DIR}/${name//[^a-zA-Z0-9_-]/_}"
    local now; now=$(date +%s)

    # Check cooldown
    if [[ -f "$state_file" ]]; then
        local last_fired; last_fired=$(cat "$state_file")
        local elapsed=$(( now - last_fired ))
        if (( elapsed < ALERT_COOLDOWN )); then
            return 0
        fi
    fi

    echo "$now" > "$state_file"

    local ts; ts=$(date '+%Y-%m-%dT%H:%M:%S')
    printf '[ALERT] %s severity=%s name=%s msg=%s\n' "$ts" "$severity" "$name" "$message" >&2

    # Dispatch
    alert_dispatch "$severity" "$name" "$message"
}

alert_resolve() {
    local name=$1
    local state_file="${ALERT_STATE_DIR}/${name//[^a-zA-Z0-9_-]/_}"
    rm -f "$state_file"
}

alert_dispatch() {
    local severity=$1 name=$2 message=$3

    if [[ -n "${SLACK_WEBHOOK_URL:-}" ]]; then
        curl --silent --request POST "$SLACK_WEBHOOK_URL" \
            --header 'Content-Type: application/json' \
            --data "{\"text\":\"[${severity}] ${name}: ${message}\"}" &
    fi

    if [[ -n "${PAGERDUTY_KEY:-}" ]] && [[ "$severity" == "CRITICAL" ]]; then
        curl --silent --request POST 'https://events.pagerduty.com/v2/enqueue' \
            --header 'Content-Type: application/json' \
            --data "{\"routing_key\":\"${PAGERDUTY_KEY}\",\"event_action\":\"trigger\",\"payload\":{\"summary\":\"${message}\",\"severity\":\"critical\"}}" &
    fi
}

run_alert_checks() {
    local cpu_pct; cpu_pct=$(cpu_percent 1 2>/dev/null || echo 0)
    local mem_pct; mem_pct=$(memory_percent 2>/dev/null || echo 0)
    local load_1m; load_1m=$(awk '{print $1}' /proc/loadavg 2>/dev/null || echo 0)
    local ncpu; ncpu=$(cpu_count 2>/dev/null || echo 1)

    alert_check_threshold cpu_percent  "$cpu_pct"     '>'  90 CRITICAL  "CPU usage critical"
    alert_check_threshold cpu_percent  "$cpu_pct"     '>'  80 WARNING   "CPU usage high"
    alert_check_threshold mem_percent  "$mem_pct"     '>'  90 CRITICAL  "Memory usage critical"
    alert_check_threshold mem_percent  "$mem_pct"     '>'  80 WARNING   "Memory usage high"
    alert_check_threshold load_average "$load_1m"     '>=' "$((ncpu * 2))" WARNING "Load average high"

    # Disk usage
    df -h --output=pcent,target 2>/dev/null | tail -n +2 | \
    while read -r pct mount; do
        local pct_n="${pct//%/}"
        alert_check_threshold "disk${mount//\//_}" "$pct_n" '>' 90 CRITICAL "Disk $mount full"
        alert_check_threshold "disk${mount//\//_}" "$pct_n" '>' 80 WARNING  "Disk $mount high"
    done
}
```

---

## 74.5 Cron Job Monitor

```bash
#!/bin/bash
# cron_monitor.sh - Track cron job execution

CRON_DB="${CRON_DB:-/tmp/cron_monitor.db}"

cron_db_init() {
    sqlite3 "$CRON_DB" "
        CREATE TABLE IF NOT EXISTS job_runs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            job_name TEXT NOT NULL,
            started_at INTEGER,
            finished_at INTEGER,
            duration_ms INTEGER,
            exit_code INTEGER,
            output TEXT
        );
        CREATE INDEX IF NOT EXISTS idx_job_name ON job_runs(job_name);
        CREATE INDEX IF NOT EXISTS idx_started ON job_runs(started_at);
    " 2>/dev/null
}

cron_job_start() {
    local job_name=$1
    cron_db_init

    local start_ms; start_ms=$(date +%s%3N)
    local run_id
    run_id=$(sqlite3 "$CRON_DB" "
        INSERT INTO job_runs (job_name, started_at) VALUES ('$job_name', $start_ms);
        SELECT last_insert_rowid();
    " 2>/dev/null)

    export CRON_RUN_ID="$run_id"
    export CRON_START_MS="$start_ms"
    echo "$run_id"
}

cron_job_end() {
    local job_name=$1 exit_code=${2:-0} output=${3:-}
    local run_id="${CRON_RUN_ID:-0}"
    local start_ms="${CRON_START_MS:-0}"

    local finish_ms; finish_ms=$(date +%s%3N)
    local duration_ms=$(( finish_ms - start_ms ))
    local output_escaped="${output//\'/\'\'}" 

    sqlite3 "$CRON_DB" "
        UPDATE job_runs
        SET finished_at=$finish_ms, duration_ms=$duration_ms,
            exit_code=$exit_code, output='$output_escaped'
        WHERE id=$run_id;
    " 2>/dev/null
}

cron_wrap() {
    local job_name=$1
    shift
    local cmd=("$@")

    cron_db_init
    cron_job_start "$job_name" > /dev/null

    local output exit_code
    output=$("${cmd[@]}" 2>&1)
    exit_code=$?

    cron_job_end "$job_name" "$exit_code" "${output:0:1000}"

    return "$exit_code"
}

cron_status_report() {
    cron_db_init

    echo "=== Cron Job Status Report ==="
    echo ""

    sqlite3 -column -header "$CRON_DB" "
        SELECT
            job_name,
            COUNT(*) as runs,
            SUM(CASE WHEN exit_code=0 THEN 1 ELSE 0 END) as success,
            SUM(CASE WHEN exit_code!=0 THEN 1 ELSE 0 END) as failures,
            AVG(duration_ms) as avg_ms,
            MAX(started_at) as last_run_epoch
        FROM job_runs
        WHERE started_at > strftime('%s','now','-7 days') * 1000
        GROUP BY job_name
        ORDER BY job_name;
    " 2>/dev/null
}
```

---

## 74.6 System Health Scorecard

```bash
#!/bin/bash
# health_scorecard.sh - Weighted system health scoring

declare -A HEALTH_SCORES=()
declare -A HEALTH_WEIGHTS=()
declare -A HEALTH_STATUS=()

health_check_register() {
    local name=$1 weight=${2:-1}
    HEALTH_WEIGHTS["$name"]="$weight"
}

health_check_record() {
    local name=$1 status=$2 score=$3
    HEALTH_STATUS["$name"]="$status"
    HEALTH_SCORES["$name"]="$score"
}

run_health_checks() {
    # CPU
    local cpu_pct; cpu_pct=$(cpu_percent 1 2>/dev/null | awk '{print int($1)}')
    if   (( cpu_pct < 70 )); then health_check_record cpu PASS 100
    elif (( cpu_pct < 90 )); then health_check_record cpu WARN  50
    else                          health_check_record cpu FAIL   0
    fi
    health_check_register cpu 3

    # Memory
    local mem_pct; mem_pct=$(memory_percent 2>/dev/null | awk '{print int($1)}')
    if   (( mem_pct < 80 )); then health_check_record memory PASS 100
    elif (( mem_pct < 95 )); then health_check_record memory WARN  50
    else                          health_check_record memory FAIL   0
    fi
    health_check_register memory 3

    # Disk (root)
    local disk_pct; disk_pct=$(df / 2>/dev/null | awk 'NR==2{print $5}' | tr -d '%')
    if   (( disk_pct < 80 )); then health_check_record disk PASS 100
    elif (( disk_pct < 95 )); then health_check_record disk WARN  50
    else                          health_check_record disk FAIL   0
    fi
    health_check_register disk 2

    # Load
    local ncpu; ncpu=$(cpu_count 2>/dev/null || echo 1)
    local load; load=$(awk '{print $1}' /proc/loadavg 2>/dev/null || echo 0)
    local load_int; load_int=$(echo "$load" | awk '{print int($1)}')
    if   (( load_int <= ncpu ));           then health_check_record load PASS 100
    elif (( load_int <= ncpu * 2 ));       then health_check_record load WARN  50
    else                                        health_check_record load FAIL   0
    fi
    health_check_register load 2
}

health_score_total() {
    local total_score=0 total_weight=0

    for name in "${!HEALTH_SCORES[@]}"; do
        local score="${HEALTH_SCORES[$name]}"
        local weight="${HEALTH_WEIGHTS[$name]:-1}"
        total_score=$(( total_score + score * weight ))
        total_weight=$(( total_weight + weight ))
    done

    (( total_weight > 0 )) || { echo 0; return; }
    echo $(( total_score / total_weight ))
}

health_report() {
    run_health_checks

    local score; score=$(health_score_total)

    echo "=== System Health Scorecard ==="
    echo ""
    printf '%-20s %-8s %s\n' 'Check' 'Status' 'Score'
    printf '%-20s %-8s %s\n' '-----' '------' '-----'

    for name in $(echo "${!HEALTH_STATUS[@]}" | tr ' ' '\n' | sort); do
        local status="${HEALTH_STATUS[$name]}"
        local s="${HEALTH_SCORES[$name]}"
        local color
        case "$status" in
            PASS) color='\033[0;32m' ;;
            WARN) color='\033[0;33m' ;;
            FAIL) color='\033[0;31m' ;;
            *)    color='\033[0m' ;;
        esac
        printf "%-20s ${color}%-8s\033[0m %s\n" "$name" "$status" "$s"
    done

    echo ""
    local grade
    if   (( score >= 90 )); then grade='\033[0;32mA (Excellent)\033[0m'
    elif (( score >= 75 )); then grade='\033[0;33mB (Good)\033[0m'
    elif (( score >= 50 )); then grade='\033[0;33mC (Fair)\033[0m'
    else                         grade='\033[0;31mF (Critical)\033[0m'
    fi
    printf "Overall Score: %d/100  Grade: ${grade}\n" "$score"
}
```

---

## 74.7 Exercises

### Exercise 1: Metrics Pipeline
สร้าง pipeline ที่:
- Collect metrics every 30s
- Store in SQLite time-series table
- Detect trends (rising/falling)
- Generate daily summary report

### Exercise 2: Multi-Host Monitor
สร้าง monitor ที่:
- SSH to multiple hosts in parallel
- Aggregate metrics
- Highlight outliers
- Cluster-wide health score

### Exercise 3: Capacity Planning Report
สร้าง report ที่:
- Disk growth rate estimation
- Memory leak detection (trend)
- CPU saturation forecast
- Recommendation engine

---

## สรุป Part 74

✅ /proc CPU: per-core utilization, idle-delta calculation
✅ /proc Memory: total/used/avail/swap stats and percentage
┅ /proc Disk I/O: reads/writes per second, KB/s via diskstats
┅ /proc Network: RX/TX KB/s and packet rate per interface
┅ ASCII dashboard: CPU bar, memory bar, disk, top-N processes, service status
┅ Service watchdog: check-restart loop, restart-window throttle, escalation alert
┅ Alert engine: threshold operators, cooldown state file, Slack+PD dispatch
┅ Cron job monitor: SQLite run tracking, duration, exit code, 7-day report
┅ Health scorecard: weighted PASS/WARN/FAIL, letter-grade summary

---

**→ Part 75: Shell Script Testing and Quality Assurance**
