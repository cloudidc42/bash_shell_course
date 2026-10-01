# Part 40: Monitoring & Observability
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 40.1 Metrics Collection

```bash
#!/bin/bash
# metrics_collector.sh

get_cpu_metrics() {
    local interval=${1:-1}
    local cpu1 cpu2
    cpu1=$(grep '^cpu ' /proc/stat)
    sleep "$interval"
    cpu2=$(grep '^cpu ' /proc/stat)
    
    awk -v c1="$cpu1" -v c2="$cpu2" '
    BEGIN {
        split(c1, a1)
        split(c2, a2)
        for (i=2; i<=9; i++) {
            prev_total += a1[i]
            curr_total += a2[i]
        }
        prev_idle = a1[5] + a1[6]
        curr_idle = a2[5] + a2[6]
        total_delta = curr_total - prev_total
        idle_delta = curr_idle - prev_idle
        cpu_usage = (total_delta - idle_delta) / total_delta * 100
        printf "cpu_usage %.2f\n", cpu_usage
        printf "cpu_user %.2f\n", (a2[2]-a1[2])/total_delta*100
        printf "cpu_system %.2f\n", (a2[4]-a1[4])/total_delta*100
        printf "cpu_iowait %.2f\n", (a2[6]-a1[6])/total_delta*100
    }' /dev/null
}

get_memory_metrics() {
    awk '
    /MemTotal/     { total = $2 }
    /MemAvailable/ { available = $2 }
    /MemFree/      { free = $2 }
    /Buffers/      { buffers = $2 }
    /^Cached/      { cached = $2 }
    /SwapTotal/    { swap_total = $2 }
    /SwapFree/     { swap_free = $2 }
    END {
        used = total - available
        swap_used = swap_total - swap_free
        printf "mem_total_kb %d\n", total
        printf "mem_used_kb %d\n", used
        printf "mem_available_kb %d\n", available
        printf "mem_usage_pct %.2f\n", used/total*100
        printf "swap_total_kb %d\n", swap_total
        printf "swap_used_kb %d\n", swap_used
        if (swap_total > 0)
            printf "swap_usage_pct %.2f\n", swap_used/swap_total*100
        else
            printf "swap_usage_pct 0\n"
    }' /proc/meminfo
}

get_disk_metrics() {
    df -P 2>/dev/null | awk '
    NR > 1 && $1 !~ /^(tmpfs|devtmpfs|udev)/ {
        gsub(/%/, "", $5)
        printf "disk_use_pct{mount=\"%s\"} %d\n", $6, $5
        printf "disk_avail_kb{mount=\"%s\"} %d\n", $6, $4
    }'
    
    awk '
    $3 ~ /^(sd|nvme|xvd|vd)[a-z]$/ {
        printf "disk_reads{device=\"%s\"} %d\n", $3, $4
        printf "disk_writes{device=\"%s\"} %d\n", $3, $8
        printf "disk_io_ms{device=\"%s\"} %d\n", $3, $13
    }' /proc/diskstats
}

get_network_metrics() {
    awk '
    /^[[:space:]]*(eth|ens|enp|em|bond|wlan)[0-9]/ {
        gsub(/:/, "", $1)
        iface = $1
        printf "net_rx_bytes{iface=\"%s\"} %d\n", iface, $2
        printf "net_tx_bytes{iface=\"%s\"} %d\n", iface, $10
        printf "net_rx_packets{iface=\"%s\"} %d\n", iface, $3
        printf "net_tx_packets{iface=\"%s\"} %d\n", iface, $11
        printf "net_rx_errors{iface=\"%s\"} %d\n", iface, $4
        printf "net_tx_errors{iface=\"%s\"} %d\n", iface, $12
    }' /proc/net/dev
}

export_prometheus_metrics() {
    local output_file=${1:-/var/lib/node_exporter/textfile_collector/custom.prom}
    mkdir -p "$(dirname "$output_file")"
    {
        echo "# HELP system_cpu_usage CPU usage percentage"
        echo "# TYPE system_cpu_usage gauge"
        get_cpu_metrics | awk '{printf "system_%s %s %d\n", $1, $2, systime()*1000}'
        echo ""
        echo "# HELP system_memory Memory metrics"
        echo "# TYPE system_memory gauge"
        get_memory_metrics | awk '{printf "system_%s %s %d\n", $1, $2, systime()*1000}'
        echo ""
        echo "# HELP system_disk Disk metrics"
        echo "# TYPE system_disk gauge"
        get_disk_metrics
        echo ""
        echo "# HELP system_network Network metrics"
        echo "# TYPE system_network gauge"
        get_network_metrics
    } > "${output_file}.tmp" && mv "${output_file}.tmp" "$output_file"
}
```

---

## 40.2 Application Performance Monitoring

```bash
#!/bin/bash
# apm.sh

monitor_endpoints() {
    local config_file=$1
    local interval=${2:-30}
    local metrics_file=${3:-/tmp/endpoint_metrics.tsv}
    
    while true; do
        while IFS=$'\t' read -r name url expected_code; do
            [[ "$name" =~ ^# ]] && continue
            [[ -z "$name" ]] && continue
            
            local start_ns
            start_ns=$(date +%s%N)
            
            local http_code content_length
            read -r http_code content_length < <(
                curl -s -o /dev/null \
                    -w "%{http_code} %{size_download}" \
                    --max-time 10 \
                    "$url" 2>/dev/null
            )
            
            local end_ns
            end_ns=$(date +%s%N)
            local response_ms=$(( (end_ns - start_ns) / 1000000 ))
            
            local status
            if [[ "$http_code" == "${expected_code:-200}" ]]; then
                status="ok"
            else
                status="error"
            fi
            
            local ts
            ts=$(date +%s)
            
            printf "%d\t%s\t%s\t%d\t%d\t%s\n" \
                "$ts" "$name" "$status" "$response_ms" "$content_length" "$http_code" \
                >> "$metrics_file"
            
            if [[ "$status" == "error" ]]; then
                echo "$(date): ALERT $name → HTTP $http_code (expected ${expected_code:-200})"
            fi
        done < "$config_file"
        sleep "$interval"
    done
}

calculate_sla() {
    local metrics_file=$1
    local service_name=$2
    local start_time=$3
    local end_time=$4
    
    awk -v svc="$service_name" -v start="$start_time" -v end="$end_time" '
    $2 == svc && $1 >= start && $1 <= end {
        total++
        if ($3 == "ok") ok++
        total_ms += $4
    }
    END {
        if (total == 0) { print "No data"; exit }
        uptime_pct = ok/total * 100
        avg_ms = total_ms / total
        printf "Service: %s\n", svc
        printf "Total checks: %d\n", total
        printf "Successful: %d\n", ok
        printf "Uptime: %.4f%%\n", uptime_pct
        printf "Avg response: %.0fms\n", avg_ms
        if (uptime_pct >= 99.99) sla = "4 nines"
        else if (uptime_pct >= 99.9) sla = "3 nines"
        else if (uptime_pct >= 99.0) sla = "2 nines"
        else sla = "below 2 nines"
        printf "SLA tier: %s\n", sla
    }' "$metrics_file"
}

monitor_process() {
    local process_name=$1
    local interval=${2:-5}
    local log_file=${3:-/tmp/process_metrics.tsv}
    
    while true; do
        local pid_list
        pid_list=$(pgrep -f "$process_name" 2>/dev/null)
        
        if [[ -z "$pid_list" ]]; then
            printf "%d\t%s\tnot_running\t0\t0\t0\n" \
                "$(date +%s)" "$process_name" >> "$log_file"
        else
            echo "$pid_list" | while read -r pid; do
                local cpu mem vsz rss
                read -r cpu mem vsz rss < <(
                    ps -p "$pid" -o %cpu,%mem,vsz,rss --no-headers 2>/dev/null || echo "0 0 0 0"
                )
                printf "%d\t%s\trunning\t%s\t%s\t%s\n" \
                    "$(date +%s)" "$process_name" "$cpu" "$mem" "$rss" >> "$log_file"
            done
        fi
        sleep "$interval"
    done
}
```

---

## 40.3 Log Aggregation & Analysis

```bash
#!/bin/bash
# log_aggregator.sh

parse_json_logs() {
    local log_file=$1
    jq -r '
    select(. != null) |
    [
        .timestamp // .time // .ts // "unknown",
        .level // .severity // "info",
        .service // .app // "unknown",
        .message // .msg // "",
        (.duration_ms // .latency_ms // 0 | tostring)
    ] | @tsv' "$log_file" 2>/dev/null
}

error_rate_analysis() {
    local log_file=$1
    local window_minutes=${2:-5}
    
    echo "=== Error Rate Analysis ==="
    echo "Window: ${window_minutes}m buckets"
    echo ""
    
    awk -v window="$window_minutes" '
    {
        if (match($0, /[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}/, m)) {
            ts = m[0]
        } else {
            ts = "unknown"
        }
        split(ts, t, "T")
        split(t[2], hm, ":")
        bucket = t[1] "T" hm[1] ":" int(hm[2]/window)*window
        total[bucket]++
        if ($0 ~ /ERROR|FATAL|CRIT|error|fatal/) errors[bucket]++
    }
    END {
        printf "%-20s %8s %8s %8s\n", "Time Bucket", "Total", "Errors", "Rate"
        printf "%-20s %8s %8s %8s\n", "───────────", "─────", "──────", "────"
        n = asorti(total, buckets)
        for (i=1; i<=n; i++) {
            b = buckets[i]
            err = errors[b]+0
            rate = (total[b] > 0) ? err/total[b]*100 : 0
            flag = (rate > 5) ? " ⚠" : ""
            printf "%-20s %8d %8d %7.1f%%%s\n", b, total[b], err, rate, flag
        }
    }' "$log_file"
}

ship_logs() {
    local log_file=$1
    local destination=$2
    local batch_size=${3:-100}
    local state_file="${log_file}.offset"
    
    local offset=0
    [[ -f "$state_file" ]] && offset=$(cat "$state_file")
    
    local batch=()
    local count=0
    
    tail -n "+$((offset + 1))" "$log_file" | while IFS= read -r line; do
        batch+=("$line")
        (( count++ ))
        
        if (( count % batch_size == 0 )); then
            printf '%s\n' "${batch[@]}" | \
                curl -s -X POST \
                    -H "Content-Type: application/json" \
                    -d "$(printf '%s\n' "${batch[@]}" | jq -R . | jq -s .)" \
                    "$destination"
            batch=()
            echo "$(( offset + count ))" > "$state_file"
        fi
    done
    
    if (( ${#batch[@]} > 0 )); then
        printf '%s\n' "${batch[@]}" | \
            curl -s -X POST \
                -H "Content-Type: application/json" \
                -d "$(printf '%s\n' "${batch[@]}" | jq -R . | jq -s .)" \
                "$destination"
    fi
    
    echo "$(( offset + count ))" > "$state_file"
}
```

---

## 40.4 Alerting & Incident Management

```bash
#!/bin/bash
# alerting_system.sh

check_thresholds() {
    local config_file=$1
    
    while IFS=$'\t' read -r metric_name threshold operator alert_name; do
        [[ "$metric_name" =~ ^# ]] && continue
        [[ -z "$metric_name" ]] && continue
        
        local current_value
        current_value=$(get_metric_value "$metric_name")
        
        local triggered=false
        case "$operator" in
            ">")  (( $(echo "$current_value > $threshold" | bc -l) )) && triggered=true ;;
            "<")  (( $(echo "$current_value < $threshold" | bc -l) )) && triggered=true ;;
            ">=") (( $(echo "$current_value >= $threshold" | bc -l) )) && triggered=true ;;
            "<=") (( $(echo "$current_value <= $threshold" | bc -l) )) && triggered=true ;;
            "==") [[ "$current_value" == "$threshold" ]] && triggered=true ;;
        esac
        
        if $triggered; then
            fire_alert "$alert_name" "$metric_name" "$current_value" "$threshold" "$operator"
        fi
    done < "$config_file"
}

get_metric_value() {
    local metric=$1
    case "$metric" in
        cpu_usage)
            get_cpu_metrics | grep "^cpu_usage" | awk '{print $2}'
            ;;
        mem_usage_pct)
            get_memory_metrics | grep "^mem_usage_pct" | awk '{print $2}'
            ;;
        disk_use_pct:*)
            local mount="${metric#disk_use_pct:}"
            df -P "$mount" 2>/dev/null | awk 'NR==2 {gsub(/%/,"",$5); print $5}'
            ;;
        load_avg_1m)
            awk '{print $1}' /proc/loadavg
            ;;
        *) echo "0" ;;
    esac
}

fire_alert() {
    local alert_name=$1
    local metric=$2
    local value=$3
    local threshold=$4
    local operator=$5
    
    local message="Alert: $alert_name | $metric $operator $threshold (current: $value)"
    echo "$(date): $message"
    
    local lock_file="/tmp/alert_lock_${alert_name// /_}"
    if [[ -f "$lock_file" ]]; then
        local lock_age=$(( SECONDS - $(cat "$lock_file") ))
        (( lock_age < 300 )) && return 0
    fi
    echo $SECONDS > "$lock_file"
    
    send_alert "warning" "$alert_name" "$message"
}

alert_status_dashboard() {
    while true; do
        clear
        echo "=== Alert Status Dashboard: $(date) ==="
        echo ""
        
        printf "%-30s %-10s %-15s %s\n" "METRIC" "VALUE" "STATUS" "THRESHOLD"
        printf "%-30s %-10s %-15s %s\n" "──────" "─────" "──────" "─────────"
        
        local cpu
        cpu=$(awk '{print $1}' /proc/loadavg)
        local cpu_status="OK"
        (( $(echo "$cpu > 2" | bc -l) )) && cpu_status="WARNING"
        (( $(echo "$cpu > 4" | bc -l) )) && cpu_status="CRITICAL"
        printf "%-30s %-10s %-15s %s\n" "load_avg_1m" "$cpu" "$cpu_status" "> 2.0 warn"
        
        local mem_pct
        mem_pct=$(get_memory_metrics | grep "^mem_usage_pct" | awk '{printf "%.0f", $2}')
        local mem_status="OK"
        (( mem_pct > 80 )) && mem_status="WARNING"
        (( mem_pct > 90 )) && mem_status="CRITICAL"
        printf "%-30s %-10s %-15s %s\n" "mem_usage_pct" "${mem_pct}%" "$mem_status" "> 80% warn"
        
        df -P 2>/dev/null | awk 'NR>1 && $1 !~ /^(tmpfs|devtmpfs)/' | \
            while read -r dev size used avail use_pct mount; do
                use_num="${use_pct/\%/}"
                local status="OK"
                (( use_num > 80 )) && status="WARNING"
                (( use_num > 90 )) && status="CRITICAL"
                printf "%-30s %-10s %-15s %s\n" \
                    "disk:$mount" "$use_pct" "$status" "> 80% warn"
            done
        
        echo ""
        echo "Refreshes every 10s. Press Ctrl+C to exit."
        sleep 10
    done
}
```

---

## 40.5 Distributed Tracing (Simple)

```bash
#!/bin/bash
# simple_tracing.sh

TRACE_ID=""
SPAN_ID=""

trace_init() {
    TRACE_ID=$(cat /proc/sys/kernel/random/uuid 2>/dev/null || \
               openssl rand -hex 16)
    SPAN_ID=$(openssl rand -hex 8)
    export TRACE_ID SPAN_ID
}

trace_span() {
    local operation=$1
    local parent_span=${SPAN_ID:-}
    local new_span_id
    new_span_id=$(openssl rand -hex 8)
    
    local start_ns
    start_ns=$(date +%s%N)
    
    SPAN_ID="$new_span_id"
    
    "$operation" "${@:2}"
    local exit_code=$?
    
    local end_ns
    end_ns=$(date +%s%N)
    local duration_ms=$(( (end_ns - start_ns) / 1000000 ))
    
    printf '{"trace_id":"%s","span_id":"%s","parent_span":"%s","operation":"%s","duration_ms":%d,"status":"%s"}\n' \
        "$TRACE_ID" "$new_span_id" "$parent_span" "$operation" "$duration_ms" \
        "$([ $exit_code -eq 0 ] && echo ok || echo error)" >> /var/log/traces.jsonl
    
    SPAN_ID="$parent_span"
    return $exit_code
}
```

---

## 40.6 Exercises

### Exercise 1: Custom Prometheus Exporter
สร้าง exporter ที่ expose:
- Application-specific metrics
- Custom business metrics
- Prometheus text format
- systemd service ที่ auto-restart

### Exercise 2: Log-based Alerting
สร้าง alert system:
- Parse application logs
- Detect patterns
- Configurable thresholds
- Alert deduplication

### Exercise 3: SLA Report Generator
สร้าง monthly SLA report:
- Parse monitoring data
- Calculate uptime %
- Response time p50/p95/p99
- Export PDF/HTML

---

## สรุป Part 40

✅ System metrics collection  
✅ Prometheus text format export  
✅ HTTP endpoint monitoring  
✅ SLA calculation  
✅ Log aggregation & shipping  
✅ Error rate analysis  
✅ Threshold-based alerting  
✅ Real-time alert dashboard  

---

**→ Part 41: CI/CD Pipeline Automation**
