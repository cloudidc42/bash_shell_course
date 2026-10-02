# Part 68: Monitoring, Alerting, and Observability
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 68.1 System Metrics Collection

```bash
#!/bin/bash
# metrics_collector.sh - Collect system metrics

set -euo pipefail

METRICS_DIR="${METRICS_DIR:-/tmp/metrics}"
mkdir -p "$METRICS_DIR"

collect_cpu() {
    local load1 load5 load15 ncpu
    read -r load1 load5 load15 _ < /proc/loadavg
    ncpu=$(nproc)

    local cpu_pct; cpu_pct=$(awk \
        '/cpu / {idle=$5; total=$2+$3+$4+$5+$6+$7+$8; printf "%.1f", 100*(total-idle)/total}' \
        /proc/stat 2>/dev/null || echo "0")

    cat <<JSON
{"load1":$load1,"load5":$load5,"load15":$load15,"ncpu":$ncpu,"cpu_pct":$cpu_pct}
JSON
}

collect_memory() {
    local total free available buffers cached
    total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    free=$(awk '/MemFree/ {print $2}' /proc/meminfo)
    available=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
    buffers=$(awk '/Buffers/ {print $2}' /proc/meminfo)
    cached=$(awk '/^Cached/ {print $2}' /proc/meminfo)

    local used=$(( total - available ))
    local pct; pct=$(awk "BEGIN{printf \"%.1f\", 100*${used}/${total}}")

    cat <<JSON
{"total_kb":$total,"free_kb":$free,"available_kb":$available,"used_kb":$used,"pct":$pct}
JSON
}

collect_disk() {
    df -h --output=target,size,used,avail,pcent 2>/dev/null | tail -n +2 | \
    while read -r mount size used avail pct; do
        printf '{"mount":"%s","size":"%s","used":"%s","avail":"%s","pct":"%s"}\n' \
            "$mount" "$size" "$used" "$avail" "${pct//%/}"
    done | jq -s '.'
}

collect_network() {
    local iface=${1:-eth0}
    local rx_file="/sys/class/net/${iface}/statistics/rx_bytes"
    local tx_file="/sys/class/net/${iface}/statistics/tx_bytes"

    local rx=0 tx=0
    [[ -f "$rx_file" ]] && rx=$(cat "$rx_file")
    [[ -f "$tx_file" ]] && tx=$(cat "$tx_file")

    local rx_prev_file="${METRICS_DIR}/net_rx_prev"
    local tx_prev_file="${METRICS_DIR}/net_tx_prev"
    local ts_prev_file="${METRICS_DIR}/net_ts_prev"

    local rx_rate=0 tx_rate=0
    if [[ -f "$rx_prev_file" ]]; then
        local rx_prev tx_prev ts_prev now
        rx_prev=$(cat "$rx_prev_file")
        tx_prev=$(cat "$tx_prev_file")
        ts_prev=$(cat "$ts_prev_file")
        now=$(date +%s)
        local interval=$(( now - ts_prev ))
        (( interval > 0 )) && rx_rate=$(( (rx - rx_prev) / interval / 1024 ))
        (( interval > 0 )) && tx_rate=$(( (tx - tx_prev) / interval / 1024 ))
    fi

    echo "$rx" > "$rx_prev_file"
    echo "$tx" > "$tx_prev_file"
    date +%s > "$ts_prev_file"

    cat <<JSON
{"iface":"$iface","rx_bytes":$rx,"tx_bytes":$tx,"rx_kbps":$rx_rate,"tx_kbps":$tx_rate}
JSON
}

collect_all_metrics() {
    local ts; ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)

    jq -n \
        --arg ts "$ts" \
        --argjson cpu "$(collect_cpu)" \
        --argjson mem "$(collect_memory)" \
        --argjson disk "$(collect_disk)" \
        --argjson net "$(collect_network)" \
        '{timestamp:$ts, cpu:$cpu, memory:$mem, disk:$disk, network:$net}'
}
```

---

## 68.2 Process Monitoring

```bash
#!/bin/bash
# process_monitor.sh - Process health and resource monitoring

monitor_process() {
    local process_name=$1 cpu_threshold=${2:-80} mem_threshold=${3:-500}

    local pid_list
    pid_list=$(pgrep -f "$process_name" 2>/dev/null | tr '\n' ' ')

    if [[ -z "$pid_list" ]]; then
        echo "{\"process\":\"$process_name\",\"status\":\"not_running\",\"pids\":[]}"
        return 1
    fi

    local results=()
    for pid in $pid_list; do
        [[ -d "/proc/$pid" ]] || continue

        local cpu mem cmd
        cpu=$(ps -p "$pid" -o %cpu= 2>/dev/null | tr -d ' ')
        mem=$(ps -p "$pid" -o rss= 2>/dev/null | tr -d ' ')
        cmd=$(ps -p "$pid" -o comm= 2>/dev/null | tr -d ' ')

        local mem_mb=$(( ${mem:-0} / 1024 ))
        local alert_cpu="false" alert_mem="false"

        (( ${cpu%.*} > cpu_threshold )) && alert_cpu="true"
        (( mem_mb > mem_threshold )) && alert_mem="true"

        results+=("$(jq -n \
            --arg pid "$pid" --arg cpu "$cpu" \
            --argjson mem_mb "$mem_mb" --arg cmd "$cmd" \
            --arg alert_cpu "$alert_cpu" --arg alert_mem "$alert_mem" \
            '{pid:($pid|tonumber),cpu:($cpu|tonumber),mem_mb:$mem_mb,cmd:$cmd,alert_cpu:($alert_cpu=="true"),alert_mem:($alert_mem=="true")}')")
    done

    jq -n \
        --arg name "$process_name" \
        --argjson procs "[$(IFS=,; echo "${results[*]}")]" \
        '{process:$name,status:"running",instances:($procs|length),procs:$procs}'
}

watch_processes() {
    local config_file=${1:-/etc/process_monitor.conf}
    local interval=${2:-60}

    while true; do
        while IFS='|' read -r name cpu_thresh mem_thresh; do
            [[ "$name" =~ ^#|^$ ]] && continue
            local result; result=$(monitor_process "$name" "${cpu_thresh:-80}" "${mem_thresh:-500}")
            local status; status=$(echo "$result" | jq -r '.status')

            if [[ "$status" == "not_running" ]]; then
                send_alert "PROCESS_DOWN" "$name" "Process $name is not running"
            fi

            echo "$result"
        done < "$config_file"
        sleep "$interval"
    done
}
```

---

## 68.3 Log Management

```bash
#!/bin/bash
# log_manager.sh - Log aggregation, rotation, and parsing

LOG_BASE="${LOG_BASE:-/var/log/app}"
LOG_MAX_SIZE_MB=${LOG_MAX_SIZE_MB:-100}
LOG_KEEP_DAYS=${LOG_KEEP_DAYS:-30}

log_rotate() {
    local log_file=$1
    local max_size_mb=${2:-$LOG_MAX_SIZE_MB}

    [[ -f "$log_file" ]] || return 0

    local size_mb; size_mb=$(du -m "$log_file" | cut -f1)

    if (( size_mb >= max_size_mb )); then
        local rotated="${log_file}.$(date +%Y%m%d_%H%M%S)"
        mv "$log_file" "$rotated"
        gzip "$rotated" &
        touch "$log_file"
        echo "Rotated: $log_file -> ${rotated}.gz"
    fi
}

log_cleanup_old() {
    local log_dir=$1 days=${2:-$LOG_KEEP_DAYS}

    find "$log_dir" -name '*.gz' -mtime "+${days}" -delete
    find "$log_dir" -name '*.log.*' -mtime "+${days}" -delete
}

log_tail_multiple() {
    local output_file=${1:-/tmp/aggregated.log}
    shift
    local log_files=("$@")

    local tail_pids=()
    for log in "${log_files[@]}"; do
        [[ -f "$log" ]] || continue
        tail -f "$log" | sed "s/^/[$(basename "$log")] /" >> "$output_file" &
        tail_pids+=($!)
    done

    trap 'kill ${tail_pids[@]} 2>/dev/null' EXIT INT TERM
    wait
}

parse_log_line() {
    local line=$1

    # รองรับ format: 2024-01-15T10:30:00 [ERROR] module: message
    if [[ "$line" =~ ^([0-9T:-]+)' '\[([A-Z]+)\]' '([^:]+):(.*) ]]; then
        jq -n \
            --arg ts "${BASH_REMATCH[1]}" \
            --arg level "${BASH_REMATCH[2]}" \
            --arg module "${BASH_REMATCH[3]}" \
            --arg msg "${BASH_REMATCH[4]}" \
            '{ts:$ts,level:$level,module:$module,msg:$msg}'
    else
        jq -n --arg raw "$line" '{raw:$raw}'
    fi
}

log_error_summary() {
    local log_file=$1 hours=${2:-1}

    local since; since=$(date -d "-${hours} hours" '+%Y-%m-%dT%H:%M' 2>/dev/null || \
                        date -v -"${hours}"H '+%Y-%m-%dT%H:%M' 2>/dev/null)

    echo "=== Error Summary (last ${hours}h) ==="

    grep -E '\[(ERROR|FATAL|CRITICAL)\]' "$log_file" 2>/dev/null | \
        awk '{print $4}' | sort | uniq -c | sort -rn | head -20 | \
        awk '{printf "  %5d  %s\n", $1, $2}'
}
```

---

## 68.4 Prometheus Metrics

```bash
#!/bin/bash
# prometheus_metrics.sh - Expose metrics in Prometheus text format

PROM_PORT="${PROM_PORT:-9100}"
PROM_FILE="${PROM_FILE:-/tmp/metrics.prom}"

format_gauge() {
    local name=$1 value=$2
    shift 2
    local labels=""
    while [[ $# -ge 2 ]]; do
        labels+="${1}=\"${2}\","
        shift 2
    done
    labels="${labels%,}"

    if [[ -n "$labels" ]]; then
        echo "${name}{${labels}} ${value}"
    else
        echo "${name} ${value}"
    fi
}

format_counter() {
    local name=$1 value=$2
    echo "# TYPE ${name} counter"
    echo "${name}_total ${value}"
}

generate_prometheus_metrics() {
    local output=${1:-$PROM_FILE}

    {
        echo "# HELP node_cpu_usage_percent CPU usage percent"
        echo "# TYPE node_cpu_usage_percent gauge"
        local cpu_pct; cpu_pct=$(awk \
            '/cpu / {idle=$5; total=$2+$3+$4+$5+$6+$7+$8; printf "%.1f", 100*(total-idle)/total}' \
            /proc/stat 2>/dev/null || echo "0")
        format_gauge "node_cpu_usage_percent" "$cpu_pct"

        echo "# HELP node_memory_used_bytes Memory used in bytes"
        echo "# TYPE node_memory_used_bytes gauge"
        local mem_avail; mem_avail=$(awk '/MemAvailable/ {print $2*1024}' /proc/meminfo)
        local mem_total; mem_total=$(awk '/MemTotal/ {print $2*1024}' /proc/meminfo)
        local mem_used=$(( mem_total - mem_avail ))
        format_gauge "node_memory_used_bytes" "$mem_used"
        format_gauge "node_memory_total_bytes" "$mem_total"

        echo "# HELP node_disk_usage_percent Disk usage percent per mount"
        echo "# TYPE node_disk_usage_percent gauge"
        df --output=target,pcent 2>/dev/null | tail -n +2 | while read -r mount pct; do
            format_gauge "node_disk_usage_percent" "${pct//%/}" "mount" "$mount"
        done

        echo "# HELP node_load_average_1m Load average 1 minute"
        echo "# TYPE node_load_average_1m gauge"
        local load1; read -r load1 _ < /proc/loadavg
        format_gauge "node_load_average_1m" "$load1"

    } > "$output"

    echo "$output"
}

serve_metrics_once() {
    generate_prometheus_metrics

    local body; body=$(cat "$PROM_FILE")
    local length=${#body}

    printf 'HTTP/1.1 200 OK\r\nContent-Type: text/plain; version=0.0.4\r\nContent-Length: %d\r\n\r\n%s' \
        "$length" "$body"
}

start_metrics_server() {
    local port=${1:-$PROM_PORT}

    echo "Starting metrics server on :$port"

    if command -v nc &>/dev/null; then
        while true; do
            nc -l -p "$port" -q 1 -e bash -c 'source '"${BASH_SOURCE[0]}"'; serve_metrics_once' 2>/dev/null || true
        done
    else
        echo "WARN: nc not available for metrics server"
    fi
}
```

---

## 68.5 Alert Engine

```bash
#!/bin/bash
# alert_engine.sh - Threshold-based alerting with dedup and cooldown

ALERT_STATE_DIR="${ALERT_STATE_DIR:-/tmp/alert_state}"
ALERT_COOLDOWN_SEC=${ALERT_COOLDOWN_SEC:-300}

mkdir -p "$ALERT_STATE_DIR"

# ─── Alert Evaluation ──────────────────────────────────────────────────
evaluate_threshold() {
    local name=$1 value=$2 threshold=$3 operator=${4:->}
    local severity=${5:-WARNING}

    local is_firing=false
    case "$operator" in
        '>') (( $(awk "BEGIN{print ($value > $threshold) ? 1 : 0}") == 1 )) && is_firing=true ;;
        '<') (( $(awk "BEGIN{print ($value < $threshold) ? 1 : 0}") == 1 )) && is_firing=true ;;
        '>=') (( $(awk "BEGIN{print ($value >= $threshold) ? 1 : 0}") == 1 )) && is_firing=true ;;
        '<=') (( $(awk "BEGIN{print ($value <= $threshold) ? 1 : 0}") == 1 )) && is_firing=true ;;
        '==') [[ "$value" == "$threshold" ]] && is_firing=true ;;
    esac

    if $is_firing; then
        send_alert "$name" "$severity" "$name: $value $operator $threshold (value=$value)"
    else
        resolve_alert "$name"
    fi
}

send_alert() {
    local name=$1 severity=$2 message=$3
    local fingerprint; fingerprint=$(printf '%s%s' "$name" "$severity" | md5sum | cut -d' ' -f1)
    local state_file="${ALERT_STATE_DIR}/${fingerprint}"

    # Check cooldown
    if [[ -f "$state_file" ]]; then
        local last_sent; last_sent=$(cat "$state_file")
        local elapsed=$(( $(date +%s) - last_sent ))
        (( elapsed < ALERT_COOLDOWN_SEC )) && return 0
    fi

    date +%s > "$state_file"

    local ts; ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    local payload; payload=$(jq -n \
        --arg name "$name" --arg severity "$severity" \
        --arg message "$message" --arg ts "$ts" \
        '{name:$name,severity:$severity,message:$message,timestamp:$ts,status:"firing"}')

    dispatch_alert "$payload"
}

resolve_alert() {
    local name=$1
    local pattern; pattern=$(printf '%s' "$name" | md5sum | cut -d' ' -f1 | cut -c1-8)
    rm -f "${ALERT_STATE_DIR}"/*"${pattern}"* 2>/dev/null || true
}

dispatch_alert() {
    local payload=$1

    printf '%s\n' "$payload" | tee -a "${ALERT_LOG:-/tmp/alerts.log}"

    [[ -n "${SLACK_WEBHOOK:-}" ]] && \
        curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d "{\"text\":\"$(echo "$payload" | jq -r '.severity + ": " + .message')\"}"

    [[ -n "${PAGERDUTY_KEY:-}" ]] && \
        curl -s -X POST 'https://events.pagerduty.com/v2/enqueue' \
            -H 'Content-Type: application/json' \
            -d "$(echo "$payload" | jq \
                --arg key "$PAGERDUTY_KEY" \
                '{routing_key:$key,event_action:"trigger",payload:{summary:.message,severity:(.severity|ascii_downcase),source:"bash-monitor"}}')"
}

# ─── Alertmanager Webhook Receiver ───────────────────────────────────────────
process_alertmanager_webhook() {
    local payload=$1

    echo "$payload" | jq -r '.alerts[] | [.status, .labels.alertname, .annotations.summary // ""] | join(" | ")' | \
    while IFS=' | ' read -r status alert_name summary; do
        case "$status" in
            firing)
                echo "ALERT FIRING: $alert_name - $summary"
                dispatch_alert "$(jq -n \
                    --arg name "$alert_name" \
                    --arg msg "$summary" \
                    '{name:$name,severity:"WARNING",message:$msg,status:"firing"}')" ;;
            resolved)
                echo "ALERT RESOLVED: $alert_name"
                resolve_alert "$alert_name" ;;
        esac
    done
}
```

---

## 68.6 Distributed Tracing

```bash
#!/bin/bash
# tracing.sh - Distributed tracing helpers

TRACE_LOG="${TRACE_LOG:-/tmp/traces.log}"

trace_generate_id() {
    head -c 8 /dev/urandom | xxd -p 2>/dev/null || \
        printf '%016x' $(( RANDOM * RANDOM * RANDOM ))
}

trace_start() {
    local operation=$1
    local trace_id=${TRACE_ID:-$(trace_generate_id)}
    local span_id; span_id=$(trace_generate_id)

    export TRACE_ID="$trace_id"
    export SPAN_ID="$span_id"
    export TRACE_OPERATION="$operation"
    export SPAN_START_NS; SPAN_START_NS=$(date +%s%N)

    trace_log "START" "$operation" "{}"
    echo "$span_id"
}

trace_end() {
    local span_id=$1 status=${2:-ok}

    local duration_ms=$(( ( $(date +%s%N) - ${SPAN_START_NS:-0} ) / 1000000 ))

    trace_log "END" "${TRACE_OPERATION:-unknown}" \
        "{\"duration_ms\":${duration_ms},\"status\":\"${status}\"}"

    unset SPAN_ID TRACE_OPERATION SPAN_START_NS
}

trace_log() {
    local event=$1 operation=$2 attrs=$3

    printf '%s trace_id=%s span_id=%s parent=%s event=%s operation=%s attrs=%s\n' \
        "$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ 2>/dev/null || date -u +%Y-%m-%dT%H:%M:%SZ)" \
        "${TRACE_ID:-unknown}" \
        "${SPAN_ID:-unknown}" \
        "${PARENT_SPAN_ID:-root}" \
        "$event" \
        "$operation" \
        "$attrs" >> "$TRACE_LOG"
}

trace_wrap() {
    local operation=$1; shift
    local span_id; span_id=$(trace_start "$operation")

    local exit_code=0
    "$@" || exit_code=$?

    local status="ok"
    (( exit_code != 0 )) && status="error"
    trace_end "$span_id" "$status"

    return $exit_code
}

propagate_trace_headers() {
    echo "X-Trace-Id: ${TRACE_ID:-}"
    echo "X-Span-Id: ${SPAN_ID:-}"
    echo "X-Parent-Span-Id: ${PARENT_SPAN_ID:-}"
}
```

---

## 68.7 Health Check Framework

```bash
#!/bin/bash
# health_checks.sh - Service health verification

HEALTH_RESULTS_DIR="${HEALTH_RESULTS_DIR:-/tmp/health}"
mkdir -p "$HEALTH_RESULTS_DIR"

check_http_endpoint() {
    local name=$1 url=$2
    local expected_status=${3:-200} timeout=${4:-10}
    local expected_body=${5:-}

    local start_ns; start_ns=$(date +%s%N)

    local response_file; response_file=$(mktemp)
    local http_code
    http_code=$(curl -s -o "$response_file" -w '%{http_code}' \
        --max-time "$timeout" --connect-timeout 5 \
        "$url" 2>/dev/null)

    local duration_ms=$(( ( $(date +%s%N) - start_ns ) / 1000000 ))
    local status="ok" reason=""

    if [[ "$http_code" != "$expected_status" ]]; then
        status="fail"
        reason="HTTP $http_code (expected $expected_status)"
    elif [[ -n "$expected_body" ]] && ! grep -q "$expected_body" "$response_file"; then
        status="fail"
        reason="Body pattern not found: $expected_body"
    fi

    rm -f "$response_file"

    local result; result=$(jq -n \
        --arg name "$name" --arg url "$url" \
        --arg status "$status" --arg reason "$reason" \
        --argjson dur "$duration_ms" --arg http "$http_code" \
        '{name:$name,url:$url,status:$status,reason:$reason,response_ms:$dur,http_code:$http}')

    echo "$result" > "${HEALTH_RESULTS_DIR}/${name}.json"
    echo "$result"
}

run_health_checks() {
    local config_file=${1:-/etc/health_checks.conf}

    while IFS='|' read -r name url expected_status timeout expected_body; do
        [[ "$name" =~ ^#|^$ ]] && continue
        check_http_endpoint "$name" "$url" \
            "${expected_status:-200}" \
            "${timeout:-10}" \
            "${expected_body:-}"
    done < "$config_file"
}

health_summary() {
    local total=0 healthy=0 failed=0

    echo "=== Health Check Summary ==="
    for result_file in "${HEALTH_RESULTS_DIR}"/*.json; do
        [[ -f "$result_file" ]] || continue
        (( total++ ))

        local name status dur
        name=$(jq -r '.name' "$result_file")
        status=$(jq -r '.status' "$result_file")
        dur=$(jq -r '.response_ms' "$result_file")
        local reason; reason=$(jq -r '.reason // ""' "$result_file")

        if [[ "$status" == "ok" ]]; then
            printf "  [OK]   %-30s %dms\n" "$name" "$dur"
            (( healthy++ ))
        else
            printf "  [FAIL] %-30s %s\n" "$name" "$reason"
            (( failed++ ))
        fi
    done
    echo ""
    echo "Total: $total  Healthy: $healthy  Failed: $failed"
    (( failed == 0 ))
}
```

---

## 68.8 SLO/SLA Tracking

```bash
#!/bin/bash
# slo_tracker.sh - Service Level Objective tracking

SLO_DATA_DIR="${SLO_DATA_DIR:-/var/lib/slo}"
mkdir -p "$SLO_DATA_DIR"

slo_record() {
    local service=$1 success=${2:-true}
    local data_file="${SLO_DATA_DIR}/${service}.log"

    printf '%s %s\n' "$(date +%s)" "$success" >> "$data_file"
}

slo_calculate() {
    local service=$1 window_hours=${2:-720}

    local data_file="${SLO_DATA_DIR}/${service}.log"
    [[ -f "$data_file" ]] || { echo "{\"service\":\"$service\",\"uptime_pct\":null}"; return; }

    local cutoff; cutoff=$(( $(date +%s) - window_hours * 3600 ))
    local total=0 successes=0

    while IFS=' ' read -r ts status; do
        (( ts < cutoff )) && continue
        (( total++ ))
        [[ "$status" == "true" ]] && (( successes++ ))
    done < "$data_file"

    local uptime_pct=100
    (( total > 0 )) && uptime_pct=$(awk "BEGIN{printf \"%.4f\", 100*${successes}/${total}}")

    local slo_target=${SLO_TARGET:-99.9}
    local error_budget_pct; error_budget_pct=$(awk "BEGIN{printf \"%.4f\", 100 - $slo_target}")
    local error_rate_pct; error_rate_pct=$(awk "BEGIN{printf \"%.4f\", 100 - $uptime_pct}")
    local budget_remaining; budget_remaining=$(awk "BEGIN{printf \"%.2f\", 100*($error_budget_pct - $error_rate_pct)/$error_budget_pct}")

    jq -n \
        --arg service "$service" \
        --argjson uptime "$uptime_pct" \
        --argjson total "$total" \
        --argjson success "$successes" \
        --argjson budget_remaining "$budget_remaining" \
        --argjson window "$window_hours" \
        '{service:$service,uptime_pct:$uptime,total_checks:$total,successful:$success,error_budget_remaining_pct:$budget_remaining,window_hours:$window}'
}

slo_report() {
    echo "=== SLO Report ==="
    echo ""
    for data_file in "${SLO_DATA_DIR}"/*.log; do
        [[ -f "$data_file" ]] || continue
        local service; service=$(basename "$data_file" .log)
        slo_calculate "$service" | jq -r \
            '"Service: " + .service + " | Uptime: " + (.uptime_pct|tostring) + "% | Budget remaining: " + (.error_budget_remaining_pct|tostring) + "%"'
    done
}
```

---

## 68.9 ASCII Dashboard

```bash
#!/bin/bash
# ascii_dashboard.sh - Terminal monitoring dashboard

draw_bar() {
    local value=$1 max=${2:-100} width=${3:-40}
    local filled=$(( value * width / max ))
    local empty=$(( width - filled ))

    printf '['
    printf '#%.0s' $(seq 1 "$filled" 2>/dev/null || echo)
    printf ' %.0s' $(seq 1 "$empty" 2>/dev/null || echo)
    printf '] %3d%%' "$value"
}

display_dashboard() {
    clear
    echo "================================================"
    printf '  System Dashboard  %s\n' "$(date)"
    echo "================================================"
    echo ""

    # CPU
    local cpu_pct; cpu_pct=$(awk \
        '/cpu / {idle=$5; total=$2+$3+$4+$5+$6+$7+$8; printf "%d", 100*(total-idle)/total}' \
        /proc/stat 2>/dev/null || echo 0)
    printf '  CPU:    '
    draw_bar "$cpu_pct"
    echo ""

    # Memory
    local mem_total; mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    local mem_avail; mem_avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
    local mem_pct=$(( (mem_total - mem_avail) * 100 / mem_total ))
    printf '  Memory: '
    draw_bar "$mem_pct"
    echo ""

    # Disk
    echo ""
    echo "  Disk Usage:"
    df -h --output=target,pcent 2>/dev/null | tail -n +2 | head -5 | \
    while read -r mount pct; do
        local pct_val=${pct//%/}
        printf '  %-15s ' "$mount"
        draw_bar "$pct_val"
        echo ""
    done

    # Load
    local load1 load5 load15
    read -r load1 load5 load15 _ < /proc/loadavg
    echo ""
    printf '  Load: 1m=%-6s 5m=%-6s 15m=%s\n' "$load1" "$load5" "$load15"

    echo "================================================"
}

run_dashboard() {
    local interval=${1:-5}
    while true; do
        display_dashboard
        sleep "$interval"
    done
}
```

---

## 68.10 Exercises

### Exercise 1: Custom Exporter
สร้าง Prometheus exporter ที่:
- Business metrics (orders/sec, revenue)
- Application metrics (cache hit rate, queue depth)
- Custom labels per environment
- Pushgateway integration

### Exercise 2: Alert Correlation
สร้าง system ที่:
- Correlate related alerts
- Root cause analysis
- Suppress child alerts
- Escalation policy

### Exercise 3: SLO Burn Rate
สร้าง SLO burn rate monitor ที่:
- Fast burn (1h window)
- Slow burn (6h window)
- Dual-window alerting
- Error budget depletion forecast

---

## สรุป Part 68

✅ System metrics: CPU/load, memory, disk, network with per-interval rates
✅ Process monitoring: per-PID CPU/memory with threshold alerting
✅ Log rotation (size-based + gzip), multi-file tail aggregation, error summary
✅ Prometheus text-format metrics generation (gauge/counter/labels)
✅ Alert engine: threshold eval, dedup by fingerprint, cooldown, Slack/PagerDuty dispatch
✅ Alertmanager webhook receiver with firing/resolved routing
✅ Distributed tracing: trace/span IDs, context propagation, duration measurement
✅ Health endpoint checker: HTTP status + body pattern + response time
✅ SLO tracking: uptime %, error budget calculation and remaining %
✅ ASCII bar chart dashboard for terminal monitoring

---

**→ Part 69: Infrastructure as Code with Bash**
