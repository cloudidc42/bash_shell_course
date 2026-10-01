# Part 55: Monitoring and Alerting Systems
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 55.1 Metrics Collection

```bash
#!/bin/bash
# metrics.sh - System metrics collection

METRICS_DIR="${METRICS_DIR:-/var/lib/metrics}"
METRICS_RETENTION_DAYS="${METRICS_RETENTION_DAYS:-30}"

# ─── System Metrics ────────────────────────────────────────────
collect_cpu_metrics() {
    local timestamp
    timestamp=$(date +%s)

    # CPU usage via /proc/stat
    local cpu_line
    cpu_line=$(head -1 /proc/stat)
    read -r _ user nice system idle iowait irq softirq steal _ <<< "$cpu_line"

    local total=$(( user + nice + system + idle + iowait + irq + softirq + steal ))
    local idle_total=$(( idle + iowait ))

    echo "${timestamp} cpu.total ${total}"
    echo "${timestamp} cpu.idle ${idle_total}"
    echo "${timestamp} cpu.user ${user}"
    echo "${timestamp} cpu.system ${system}"
    echo "${timestamp} cpu.iowait ${iowait}"

    # Load average
    local load1 load5 load15
    read -r load1 load5 load15 _ < /proc/loadavg
    echo "${timestamp} load.1m ${load1}"
    echo "${timestamp} load.5m ${load5}"
    echo "${timestamp} load.15m ${load15}"
}

collect_memory_metrics() {
    local timestamp
    timestamp=$(date +%s)

    local mem_total mem_free mem_available mem_buffers mem_cached
    while IFS=: read -r key value; do
        value="${value// /}"
        value="${value//kB/}"
        case "$key" in
            MemTotal)     mem_total=$value ;;
            MemFree)      mem_free=$value ;;
            MemAvailable) mem_available=$value ;;
            Buffers)      mem_buffers=$value ;;
            Cached)       mem_cached=$value ;;
        esac
    done < /proc/meminfo

    local mem_used=$(( mem_total - mem_available ))
    local mem_pct=$(( mem_used * 100 / mem_total ))

    echo "${timestamp} memory.total ${mem_total}"
    echo "${timestamp} memory.used ${mem_used}"
    echo "${timestamp} memory.free ${mem_free}"
    echo "${timestamp} memory.available ${mem_available}"
    echo "${timestamp} memory.pct ${mem_pct}"
}

collect_disk_metrics() {
    local timestamp
    timestamp=$(date +%s)

    df -P 2>/dev/null | awk 'NR>1 && /^\/dev/' | while read -r device total used avail pct mount; do
        local clean_mount="${mount//\/_}"
        clean_mount="${clean_mount:1}"
        local clean_pct="${pct//%/}"
        echo "${timestamp} disk.${clean_mount:-root}.total ${total}"
        echo "${timestamp} disk.${clean_mount:-root}.used ${used}"
        echo "${timestamp} disk.${clean_mount:-root}.pct ${clean_pct}"
    done
}

collect_network_metrics() {
    local timestamp
    timestamp=$(date +%s)

    while IFS=: read -r interface stats; do
        interface="${interface// /}"
        [[ "$interface" == "lo" ]] && continue
        read -r rx_bytes _ _ _ _ _ _ _ tx_bytes _ <<< "$stats"
        echo "${timestamp} net.${interface}.rx_bytes ${rx_bytes}"
        echo "${timestamp} net.${interface}.tx_bytes ${tx_bytes}"
    done < /proc/net/dev
}

# ─── Metrics Storage ───────────────────────────────────────────
store_metrics() {
    local metric_name=$1

    mkdir -p "$METRICS_DIR"
    local metric_file="$METRICS_DIR/${metric_name}.rrd"

    while IFS= read -r line; do
        echo "$line" >> "$metric_file"
    done

    # Prune old data
    local cutoff
    cutoff=$(date -d "${METRICS_RETENTION_DAYS} days ago" +%s 2>/dev/null || \
             date -v -"${METRICS_RETENTION_DAYS}d" +%s 2>/dev/null || \
             echo 0)

    if [[ $cutoff -gt 0 ]]; then
        local tmpfile
        tmpfile=$(mktemp)
        awk -v cutoff="$cutoff" '$1 > cutoff' "$metric_file" > "$tmpfile"
        mv "$tmpfile" "$metric_file"
    fi
}

collect_all_metrics() {
    collect_cpu_metrics | store_metrics "cpu"
    collect_memory_metrics | store_metrics "memory"
    collect_disk_metrics | store_metrics "disk"
    collect_network_metrics | store_metrics "network"
}

# ─── Prometheus Exposition ─────────────────────────────────────
metrics_to_prometheus() {
    local timestamp
    timestamp=$(date +%s%3N)

    echo "# HELP node_cpu_seconds Total CPU time"
    echo "# TYPE node_cpu_seconds counter"

    read -r _ user nice system idle iowait irq softirq steal _ < /proc/stat
    printf 'node_cpu_seconds_total{mode="user"} %d %d\n' "$user" "$timestamp"
    printf 'node_cpu_seconds_total{mode="system"} %d %d\n' "$system" "$timestamp"
    printf 'node_cpu_seconds_total{mode="idle"} %d %d\n' "$idle" "$timestamp"
    printf 'node_cpu_seconds_total{mode="iowait"} %d %d\n' "$iowait" "$timestamp"

    echo ""
    echo "# HELP node_memory_bytes Memory information"
    echo "# TYPE node_memory_bytes gauge"

    while IFS=: read -r key value; do
        value="${value// /}"; value="${value//kB/}"
        case "$key" in
            MemTotal)     printf 'node_memory_MemTotal_bytes %d %d\n' "$(( value * 1024 ))" "$timestamp" ;;
            MemAvailable) printf 'node_memory_MemAvailable_bytes %d %d\n' "$(( value * 1024 ))" "$timestamp" ;;
        esac
    done < /proc/meminfo
}

serve_prometheus_metrics() {
    local port=${1:-9100}

    while true; do
        local response
        response=$(metrics_to_prometheus)
        local content_length=${#response}

        printf 'HTTP/1.1 200 OK\r\nContent-Type: text/plain; version=0.0.4\r\nContent-Length: %d\r\n\r\n%s' \
            "$content_length" "$response" | nc -l -p "$port" -q 1
    done
}
```

---

## 55.2 Health Checks

```bash
#!/bin/bash
# health_checks.sh - Service and endpoint health monitoring

HEALTH_STATE_DIR="/var/lib/health"
declare -A CHECK_HISTORY=()

# ─── Health Check Framework ────────────────────────────────────
health_check_register() {
    local name=$1 handler=$2 interval=${3:-60} timeout=${4:-10}

    mkdir -p "$HEALTH_STATE_DIR"
    echo "${handler}:${interval}:${timeout}" > "$HEALTH_STATE_DIR/${name}.check"
}

run_health_check() {
    local name=$1
    local check_file="$HEALTH_STATE_DIR/${name}.check"

    [[ ! -f "$check_file" ]] && return 1

    IFS=: read -r handler interval timeout < "$check_file"

    local start_time
    start_time=$(date +%s%N)

    local output
    output=$(timeout "$timeout" bash -c "$handler" 2>&1)
    local exit_code=$?

    local end_time
    end_time=$(date +%s%N)
    local duration_ms=$(( (end_time - start_time) / 1000000 ))

    local state
    state=$([[ $exit_code -eq 0 ]] && echo "healthy" || echo "unhealthy")

    # Save state
    local state_file="$HEALTH_STATE_DIR/${name}.state"
    printf '{"name":"%s","state":"%s","exit_code":%d,"duration_ms":%d,"timestamp":%d,"output":"%s"}\n' \
        "$name" "$state" "$exit_code" "$duration_ms" "$(date +%s)" \
        "$(echo "$output" | head -1 | tr '"' "'")" > "$state_file"

    echo "$state" "$name" "(${duration_ms}ms)"
    return $exit_code
}

# ─── Built-in Checks ───────────────────────────────────────────
check_http() {
    local url=$1 expected_code=${2:-200} timeout=${3:-10}

    local code
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time "$timeout" "$url" 2>/dev/null)

    if [[ "$code" == "$expected_code" ]]; then
        echo "HTTP $code: $url"
        return 0
    else
        echo "HTTP $code (expected $expected_code): $url"
        return 1
    fi
}

check_tcp() {
    local host=$1 port=$2 timeout=${3:-5}

    if (echo >/dev/tcp/"$host"/"$port") 2>/dev/null; then
        echo "TCP OK: $host:$port"
        return 0
    else
        echo "TCP FAIL: $host:$port"
        return 1
    fi
}

check_process() {
    local process_name=$1

    if pgrep -x "$process_name" &>/dev/null; then
        local pid
        pid=$(pgrep -x "$process_name" | head -1)
        echo "Process running: $process_name (PID: $pid)"
        return 0
    else
        echo "Process not found: $process_name"
        return 1
    fi
}

check_disk_space() {
    local path=${1:-/}
    local threshold=${2:-90}

    local used_pct
    used_pct=$(df "$path" | awk 'NR==2{print $5}' | tr -d '%')

    if (( used_pct < threshold )); then
        echo "Disk OK: ${used_pct}% used on $path"
        return 0
    else
        echo "Disk CRITICAL: ${used_pct}% used on $path (threshold: ${threshold}%)"
        return 1
    fi
}

check_memory() {
    local threshold=${1:-90}

    local mem_available mem_total
    while IFS=: read -r key value; do
        value="${value// /}"; value="${value//kB/}"
        case "$key" in
            MemTotal)     mem_total=$value ;;
            MemAvailable) mem_available=$value ;;
        esac
    done < /proc/meminfo

    local mem_used_pct=$(( (mem_total - mem_available) * 100 / mem_total ))

    if (( mem_used_pct < threshold )); then
        echo "Memory OK: ${mem_used_pct}% used"
        return 0
    else
        echo "Memory CRITICAL: ${mem_used_pct}% used (threshold: ${threshold}%)"
        return 1
    fi
}

check_certificate_expiry() {
    local domain=$1 port=${2:-443} min_days=${3:-14}

    local expiry_date
    expiry_date=$(echo | openssl s_client \
        -connect "${domain}:${port}" -servername "$domain" 2>/dev/null | \
        openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)

    [[ -z "$expiry_date" ]] && { echo "Cannot get cert for $domain"; return 1; }

    local expiry_epoch now_epoch days_left
    expiry_epoch=$(date -d "$expiry_date" +%s 2>/dev/null || echo 0)
    now_epoch=$(date +%s)
    days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

    if (( days_left > min_days )); then
        echo "Cert OK: $domain expires in ${days_left} days"
        return 0
    else
        echo "Cert WARNING: $domain expires in ${days_left} days"
        return 1
    fi
}

# ─── Health Status Dashboard ───────────────────────────────────
health_dashboard() {
    local interval=${1:-5}

    while true; do
        clear
        echo "=== Health Check Dashboard: $(date) ==="
        echo ""

        local all_healthy=true

        for state_file in "$HEALTH_STATE_DIR"/*.state; do
            [[ -f "$state_file" ]] || continue

            local data
            data=$(cat "$state_file")
            local name state duration_ms
            name=$(echo "$data" | grep -o '"name":"[^"]*"' | cut -d'"' -f4)
            state=$(echo "$data" | grep -o '"state":"[^"]*"' | cut -d'"' -f4)
            duration_ms=$(echo "$data" | grep -o '"duration_ms":[0-9]*' | cut -d: -f2)

            if [[ "$state" == "healthy" ]]; then
                printf "  [OK]   %-30s %dms\n" "$name" "$duration_ms"
            else
                printf "  [FAIL] %-30s %dms\n" "$name" "$duration_ms"
                all_healthy=false
            fi
        done

        echo ""
        $all_healthy && echo "Overall: HEALTHY" || echo "Overall: DEGRADED"

        sleep "$interval"
    done
}
```

---

## 55.3 Alerting Engine

```bash
#!/bin/bash
# alerting.sh - Alert rules and notification engine

ALERT_STATE_DIR="/var/lib/alerts"
ALERT_LOG="/var/log/alerts.log"

declare -A ALERT_RULES=()
declare -A ALERT_THRESHOLDS=()
declare -A ALERT_CHANNELS=()

# ─── Alert Rules ───────────────────────────────────────────────
alert_rule_add() {
    local name=$1
    local metric=$2
    local operator=$3  # gt, lt, eq, gte, lte
    local threshold=$4
    local duration=${5:-0}  # seconds metric must be in violation
    local severity=${6:-warning}  # info, warning, critical

    ALERT_RULES["$name"]="${metric}:${operator}:${threshold}:${duration}:${severity}"
}

alert_evaluate() {
    local name=$1
    local current_value=$2

    local rule="${ALERT_RULES[$name]}"
    [[ -z "$rule" ]] && return 0

    IFS=: read -r metric operator threshold duration severity <<< "$rule"

    local triggered=false
    case "$operator" in
        gt)  (( $(echo "$current_value > $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        lt)  (( $(echo "$current_value < $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        gte) (( $(echo "$current_value >= $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        lte) (( $(echo "$current_value <= $threshold" | bc -l 2>/dev/null || echo 0) )) && triggered=true ;;
        eq)  [[ "$current_value" == "$threshold" ]] && triggered=true ;;
    esac

    local state_file="$ALERT_STATE_DIR/${name}.state"
    mkdir -p "$ALERT_STATE_DIR"

    if $triggered; then
        local first_triggered
        first_triggered=$(cat "$state_file" 2>/dev/null || date +%s)
        echo "$first_triggered" > "$state_file"

        local elapsed=$(( $(date +%s) - first_triggered ))
        if (( elapsed >= duration )); then
            alert_fire "$name" "$current_value" "$threshold" "$operator" "$severity"
            return 1
        fi
    else
        # Clear state if condition resolved
        if [[ -f "$state_file" ]]; then
            rm -f "$state_file"
            alert_resolve "$name" "$severity"
        fi
    fi

    return 0
}

alert_fire() {
    local name=$1 value=$2 threshold=$3 operator=$4 severity=$5

    local message="ALERT [$severity]: $name - value=$value ${operator} threshold=$threshold"
    echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') FIRING $message" >> "$ALERT_LOG"

    # Prevent alert storm (only notify once per hour)
    local last_notified_file="$ALERT_STATE_DIR/${name}.notified"
    local last_notified=0
    [[ -f "$last_notified_file" ]] && last_notified=$(cat "$last_notified_file")
    local now
    now=$(date +%s)

    if (( now - last_notified > 3600 )); then
        echo "$now" > "$last_notified_file"
        send_alert_notifications "$name" "$message" "$severity"
    fi
}

alert_resolve() {
    local name=$1 severity=$2

    local message="RESOLVED: $name is back to normal"
    echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') RESOLVED $message" >> "$ALERT_LOG"

    rm -f "$ALERT_STATE_DIR/${name}.notified"
    send_alert_notifications "$name" "$message" "info"
}

send_alert_notifications() {
    local name=$1 message=$2 severity=$3

    # Slack
    if [[ -n "${ALERT_SLACK_WEBHOOK:-}" ]]; then
        local color
        case "$severity" in
            critical) color="#ff0000" ;;
            warning)  color="#ffcc00" ;;
            *)        color="#36a64f" ;;
        esac

        local payload
        payload=$(printf '{"text":"*%s*","attachments":[{"color":"%s","text":"%s","ts":%d}]}' \
            "$name" "$color" "$message" "$(date +%s)")

        curl -s -X POST -H "Content-Type: application/json" \
            -d "$payload" "$ALERT_SLACK_WEBHOOK" &>/dev/null &
    fi

    # PagerDuty
    if [[ -n "${PAGERDUTY_ROUTING_KEY:-}" ]]; then
        local event_action
        event_action=$([[ "$severity" == "info" ]] && echo "resolve" || echo "trigger")

        local payload
        payload=$(printf '{"routing_key":"%s","event_action":"%s","payload":{"summary":"%s","severity":"%s","source":"%s"}}' \
            "$PAGERDUTY_ROUTING_KEY" "$event_action" "$message" "$severity" "$(hostname)")

        curl -s -X POST \
            -H "Content-Type: application/json" \
            -H "Accept: application/vnd.pagerduty+json;version=2" \
            -d "$payload" \
            "https://events.pagerduty.com/v2/enqueue" &>/dev/null &
    fi
}

# ─── Alert Monitor Loop ────────────────────────────────────────
run_alert_monitor() {
    local interval=${1:-60}

    echo "Alert monitor started (interval: ${interval}s)"

    while true; do
        # CPU
        local cpu_idle
        read -r _ user _ system idle _ < /proc/stat
        local total=$(( user + system + idle ))
        local cpu_usage=$(( (user + system) * 100 / total ))
        alert_evaluate "cpu_high" "$cpu_usage"

        # Memory
        local mem_total mem_available
        while IFS=: read -r key value; do
            value="${value// /}"; value="${value//kB/}"
            case "$key" in
                MemTotal)     mem_total=$value ;;
                MemAvailable) mem_available=$value ;;
            esac
        done < /proc/meminfo
        local mem_pct=$(( (mem_total - mem_available) * 100 / mem_total ))
        alert_evaluate "memory_high" "$mem_pct"

        # Disk
        local disk_pct
        disk_pct=$(df / | awk 'NR==2{print $5}' | tr -d '%')
        alert_evaluate "disk_high" "$disk_pct"

        sleep "$interval"
    done
}
```

---

## 55.4 Alertmanager-style Routing

```bash
#!/bin/bash
# alert_routing.sh - Alert routing and grouping

declare -A ROUTES=()
declare -A INHIBITIONS=()
declare -A SILENCES=()

# ─── Routing Rules ─────────────────────────────────────────────
route_add() {
    local name=$1
    local match_labels=$2  # "severity=critical,team=infra"
    local receiver=$3
    local group_wait=${4:-30}
    local group_interval=${5:-300}
    local repeat_interval=${6:-3600}

    ROUTES["$name"]="${match_labels}|${receiver}|${group_wait}|${group_interval}|${repeat_interval}"
}

route_match() {
    local alert_labels=$1

    for route_name in "${!ROUTES[@]}"; do
        local route_data="${ROUTES[$route_name]}"
        IFS='|' read -r match_labels receiver group_wait group_interval repeat_interval <<< "$route_data"

        local matched=true
        IFS=',' read -ra label_pairs <<< "$match_labels"
        for pair in "${label_pairs[@]}"; do
            local key="${pair%%=*}"
            local value="${pair#*=}"
            if ! echo "$alert_labels" | grep -q "${key}=${value}"; then
                matched=false
                break
            fi
        done

        if $matched; then
            echo "$receiver"
            return 0
        fi
    done

    echo "default"
}

# ─── Silence Management ────────────────────────────────────────
silence_add() {
    local name=$1
    local matchers=$2  # "alertname=DiskHigh"
    local duration_hours=$3
    local created_by=${4:-admin}
    local comment=${5:-"Silenced"}

    local expires
    expires=$(date -d "+${duration_hours} hours" +%s 2>/dev/null || \
              date -v "+${duration_hours}H" +%s 2>/dev/null)

    SILENCES["$name"]="${matchers}|${expires}|${created_by}|${comment}"
    echo "Silence added: $name (expires: $(date -d "@$expires" 2>/dev/null || echo "$expires"))"
}

is_silenced() {
    local alert_name=$1 alert_labels=$2
    local now
    now=$(date +%s)

    for silence_name in "${!SILENCES[@]}"; do
        local silence_data="${SILENCES[$silence_name]}"
        IFS='|' read -r matchers expires _ _ <<< "$silence_data"

        # Check expiry
        (( now > expires )) && continue

        # Check matchers
        local matched=true
        IFS=',' read -ra matcher_pairs <<< "$matchers"
        for pair in "${matcher_pairs[@]}"; do
            local key="${pair%%=*}"
            local value="${pair#*=}"
            if [[ "$key" == "alertname" ]]; then
                [[ "$alert_name" != "$value" ]] && matched=false && break
            elif ! echo "$alert_labels" | grep -q "${key}=${value}"; then
                matched=false
                break
            fi
        done

        $matched && return 0
    done

    return 1
}

silence_list() {
    local now
    now=$(date +%s)

    echo "=== Active Silences ==="
    for name in "${!SILENCES[@]}"; do
        IFS='|' read -r matchers expires created_by comment <<< "${SILENCES[$name]}"
        (( now > expires )) && continue
        local remaining=$(( (expires - now) / 60 ))
        printf "  %-20s matchers=%-30s remaining=%dm by=%s\n" \
            "$name" "$matchers" "$remaining" "$created_by"
    done
}
```

---

## 55.5 Exercises

### Exercise 1: Full Monitoring Stack
สร้าง monitoring stack ที่:
- Collect metrics ทุก 30 วินาที
- Store ใน time-series format
- Expose Prometheus endpoint
- Alert เมื่อ threshold ถูกละเมิด

### Exercise 2: SLA Monitor
สร้าง SLA monitor ที่:
- Track uptime percentage
- Calculate error rate
- Alert เมื่อ SLA ใกล้จะ breach
- Monthly SLA report

### Exercise 3: Capacity Planning
สร้าง capacity planner ที่:
- Trend analysis จาก historical metrics
- Predict เมื่อ resources จะหมด
- Auto-scale triggers
- Cost optimization recommendations

---

## สรุป Part 55

✅ Metrics collection: CPU, memory, disk, network via /proc
✅ Prometheus text format exposition
✅ Health check framework with built-in checks
✅ Alert rules with operator evaluation and duration
✅ Alert suppression (storm prevention, silences)
✅ Multi-channel notifications: Slack, PagerDuty
✅ Alert routing with label matching
✅ Real-time health dashboard

---

**→ Part 56: Infrastructure as Code with Bash**
