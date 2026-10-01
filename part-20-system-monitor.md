# Part 20: Mini Project - System Monitor
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 20.1 Project Overview

สร้าง System Monitor ที่ complete และ production-ready:
- Real-time monitoring (CPU, Memory, Disk, Network)
- Alerting via email/webhook
- Historical data logging
- Web dashboard (simple HTML)
- Configuration file support
- Daemon mode

---

## 20.2 Main Monitor Script

```bash
#!/usr/bin/env bash
# sysmon.sh - System Monitor
# Version: 2.0.0

set -euo pipefail

# ─── Configuration ────────────────────────────────────────────
readonly VERSION="2.0.0"
readonly SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
readonly CONFIG_FILE="${CONFIG_FILE:-$SCRIPT_DIR/sysmon.conf}"
readonly LOG_DIR="${LOG_DIR:-$SCRIPT_DIR/logs}"
readonly DATA_DIR="${DATA_DIR:-$SCRIPT_DIR/data}"
readonly WEB_DIR="${WEB_DIR:-$SCRIPT_DIR/web}"

# Defaults (overridden by config)
INTERVAL=5
ALERT_CPU=80
ALERT_MEM=85
ALERT_DISK=90
ALERT_EMAIL=""
ALERT_WEBHOOK=""
HISTORY_RETENTION=7    # days
LOG_LEVEL="INFO"
DAEMON_MODE=false

# ─── Colors ───────────────────────────────────────────────────
if [[ -t 1 ]]; then
    RED='\033[0;31m'    BOLD_RED='\033[1;31m'
    GREEN='\033[0;32m'  BOLD_GREEN='\033[1;32m'
    YELLOW='\033[1;33m' BLUE='\033[0;34m'
    CYAN='\033[0;36m'   WHITE='\033[1;37m'
    NC='\033[0m'
else
    RED='' BOLD_RED='' GREEN='' BOLD_GREEN='' YELLOW=''
    BLUE='' CYAN='' WHITE='' NC=''
fi

# ─── Logging ──────────────────────────────────────────────────
log() {
    local level=$1; shift
    local msg="[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*"
    echo "$msg" >> "$LOG_DIR/sysmon.log"
    [[ "$level" != "DEBUG" || "$LOG_LEVEL" == "DEBUG" ]] && echo "$msg" >&2
}
info()  { log "INFO"  "$@"; }
warn()  { log "WARN"  "$@"; }
error() { log "ERROR" "$@"; }
debug() { log "DEBUG" "$@"; }

# ─── Setup ────────────────────────────────────────────────────
setup() {
    mkdir -p "$LOG_DIR" "$DATA_DIR" "$WEB_DIR"
    
    if [[ -f "$CONFIG_FILE" ]]; then
        # shellcheck source=/dev/null
        source "$CONFIG_FILE"
        info "Loaded config: $CONFIG_FILE"
    else
        info "No config file found, using defaults"
    fi
}

# ─── Metrics Collection ───────────────────────────────────────

# CPU usage (percentage)
get_cpu_usage() {
    # Read /proc/stat twice with interval for accuracy
    local cpu1 cpu2
    cpu1=$(grep '^cpu ' /proc/stat)
    sleep 0.5
    cpu2=$(grep '^cpu ' /proc/stat)
    
    # Parse fields: cpu user nice system idle iowait irq softirq
    read -r _ u1 n1 s1 i1 w1 _ _ <<< "$cpu1"
    read -r _ u2 n2 s2 i2 w2 _ _ <<< "$cpu2"
    
    local total1=$(( u1 + n1 + s1 + i1 + w1 ))
    local total2=$(( u2 + n2 + s2 + i2 + w2 ))
    local idle1=$i1
    local idle2=$i2
    
    local delta_total=$(( total2 - total1 ))
    local delta_idle=$(( idle2 - idle1 ))
    
    if (( delta_total > 0 )); then
        echo $(( 100 * (delta_total - delta_idle) / delta_total ))
    else
        echo 0
    fi
}

# Memory usage
get_memory_info() {
    local total used free available percent
    
    # Parse /proc/meminfo
    local mem_total mem_free mem_available mem_buffers mem_cached
    mem_total=$(awk '/^MemTotal:/ {print $2}' /proc/meminfo)
    mem_free=$(awk '/^MemFree:/ {print $2}' /proc/meminfo)
    mem_available=$(awk '/^MemAvailable:/ {print $2}' /proc/meminfo)
    mem_buffers=$(awk '/^Buffers:/ {print $2}' /proc/meminfo)
    mem_cached=$(awk '/^Cached:/ {print $2}' /proc/meminfo)
    
    local mem_used=$(( mem_total - mem_available ))
    local mem_percent=$(( 100 * mem_used / mem_total ))
    
    echo "$mem_total $mem_used $mem_free $mem_available $mem_percent"
}

# Format bytes
format_bytes() {
    local bytes=$1
    # bytes are in kB from /proc/meminfo
    bytes=$(( bytes * 1024 ))
    
    if (( bytes >= 1073741824 )); then
        printf "%.1f GB" "$(echo "scale=1; $bytes/1073741824" | bc)"
    elif (( bytes >= 1048576 )); then
        printf "%.1f MB" "$(echo "scale=1; $bytes/1048576" | bc)"
    else
        printf "%.0f KB" "$(echo "scale=0; $bytes/1024" | bc)"
    fi
}

# Disk usage
get_disk_info() {
    df -h / 2>/dev/null | awk 'NR==2 {
        gsub(/%/, "", $5)
        print $2, $3, $4, $5
    }'
}

# Network I/O
declare -A NET_RX_PREV=()
declare -A NET_TX_PREV=()

get_network_info() {
    local interface
    interface=$(ip route get 8.8.8.8 2>/dev/null | awk '{print $5; exit}' || echo "eth0")
    
    local rx_bytes tx_bytes
    rx_bytes=$(cat "/sys/class/net/$interface/statistics/rx_bytes" 2>/dev/null || echo 0)
    tx_bytes=$(cat "/sys/class/net/$interface/statistics/tx_bytes" 2>/dev/null || echo 0)
    
    local prev_rx=${NET_RX_PREV[$interface]:-$rx_bytes}
    local prev_tx=${NET_TX_PREV[$interface]:-$tx_bytes}
    
    local rx_rate=$(( (rx_bytes - prev_rx) / INTERVAL ))
    local tx_rate=$(( (tx_bytes - prev_tx) / INTERVAL ))
    
    NET_RX_PREV[$interface]=$rx_bytes
    NET_TX_PREV[$interface]=$tx_bytes
    
    echo "$interface $rx_rate $tx_rate $rx_bytes $tx_bytes"
}

# Top processes
get_top_processes() {
    ps aux --no-header --sort=-%cpu | head -5 | \
        awk '{printf "%-20s %5.1f%% %5.1f%%\n", substr($11,1,20), $3, $4}'
}

# System load
get_load_average() {
    cat /proc/loadavg | cut -d' ' -f1-3
}

# Uptime
get_uptime() {
    uptime -p 2>/dev/null || uptime | sed 's/.*up //' | sed 's/,.*//'
}
```

---

## 20.3 Display & Alert Logic

```bash
# ─── Progress Bars ────────────────────────────────────────────
draw_bar() {
    local value=$1      # 0-100
    local width=${2:-20}
    
    local filled=$(( value * width / 100 ))
    local empty=$(( width - filled ))
    
    # Color based on value
    local color
    if (( value >= 90 )); then
        color=$RED
    elif (( value >= 75 )); then
        color=$YELLOW
    else
        color=$GREEN
    fi
    
    printf "${color}["
    printf '%*s' "$filled" | tr ' ' '█'
    printf '%*s' "$empty" | tr ' ' '░'
    printf "]${NC}"
}

# ─── Display Dashboard ────────────────────────────────────────
display_dashboard() {
    local cpu_usage=$1
    local mem_total mem_used mem_free mem_available mem_percent
    read -r mem_total mem_used mem_free mem_available mem_percent <<< "$2"
    local disk_total disk_used disk_avail disk_percent
    read -r disk_total disk_used disk_avail disk_percent <<< "$3"
    local net_iface net_rx_rate net_tx_rate
    read -r net_iface net_rx_rate net_tx_rate _ _ <<< "$4"
    local load_1 load_5 load_15
    read -r load_1 load_5 load_15 <<< "$5"
    
    clear
    echo -e "${WHITE}╔══════════════════════════════════════════════════════════╗${NC}"
    printf "${WHITE}║${NC}  ${CYAN}SYSTEM MONITOR${NC} %-40s${WHITE}║${NC}\n" "v$VERSION"
    printf "${WHITE}║${NC}  Host: %-52s${WHITE}║${NC}\n" "$(hostname)"
    printf "${WHITE}║${NC}  Time: %-52s${WHITE}║${NC}\n" "$(date '+%Y-%m-%d %H:%M:%S')"
    printf "${WHITE}║${NC}  Up:   %-52s${WHITE}║${NC}\n" "$(get_uptime)"
    echo -e "${WHITE}╠══════════════════════════════════════════════════════════╣${NC}"
    
    # CPU
    printf "${WHITE}║${NC}  ${CYAN}CPU${NC}  %3d%%  $(draw_bar "$cpu_usage")  Load: %s %s %s      ${WHITE}║${NC}\n" \
        "$cpu_usage" "$load_1" "$load_5" "$load_15"
    
    # Memory
    printf "${WHITE}║${NC}  ${CYAN}MEM${NC}  %3d%%  $(draw_bar "$mem_percent")  %s / %s   ${WHITE}║${NC}\n" \
        "$mem_percent" "$(format_bytes "$mem_used")" "$(format_bytes "$mem_total")"
    
    # Disk
    printf "${WHITE}║${NC}  ${CYAN}DSK${NC}  %3d%%  $(draw_bar "$disk_percent")  %s / %s   ${WHITE}║${NC}\n" \
        "$disk_percent" "$disk_used" "$disk_total"
    
    # Network
    printf "${WHITE}║${NC}  ${CYAN}NET${NC}  ↓%8s/s  ↑%8s/s  [%s]              ${WHITE}║${NC}\n" \
        "$(numfmt --to=iec "$net_rx_rate" 2>/dev/null || echo "${net_rx_rate}B")" \
        "$(numfmt --to=iec "$net_tx_rate" 2>/dev/null || echo "${net_tx_rate}B")" \
        "$net_iface"
    
    echo -e "${WHITE}╠══════════════════════════════════════════════════════════╣${NC}"
    echo -e "${WHITE}║${NC}  ${CYAN}TOP PROCESSES (by CPU)${NC}                                   ${WHITE}║${NC}"
    
    while IFS= read -r proc; do
        printf "${WHITE}║${NC}  %-56s${WHITE}║${NC}\n" "$proc"
    done < <(get_top_processes)
    
    echo -e "${WHITE}╚══════════════════════════════════════════════════════════╝${NC}"
    echo "  Press Ctrl+C to exit | Interval: ${INTERVAL}s"
}

# ─── Alerting ─────────────────────────────────────────────────
send_alert() {
    local subject=$1
    local message=$2
    local severity=${3:-WARNING}
    
    info "ALERT [$severity]: $subject"
    
    # Email
    if [[ -n "$ALERT_EMAIL" ]]; then
        echo "$message" | mail -s "[SysMon $severity] $subject" "$ALERT_EMAIL" 2>/dev/null || \
            warn "Failed to send email alert"
    fi
    
    # Webhook (Slack/Discord/Teams compatible)
    if [[ -n "$ALERT_WEBHOOK" ]]; then
        local color
        case $severity in
            CRITICAL) color="#FF0000" ;;
            WARNING)  color="#FFA500" ;;
            *)        color="#00FF00" ;;
        esac
        
        local payload
        payload=$(cat << EOF
{
    "text": "**[$severity] $subject**",
    "attachments": [{
        "color": "$color",
        "text": "$message",
        "footer": "$(hostname) | $(date)",
        "ts": $(date +%s)
    }]
}
EOF
)
        curl -s -X POST "$ALERT_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d "$payload" &>/dev/null || warn "Failed to send webhook alert"
    fi
}

# Track alert state (avoid spam)
declare -A ALERT_SENT=()

check_thresholds() {
    local cpu_usage=$1
    local mem_percent=$2
    local disk_percent=$3
    
    # CPU alert
    if (( cpu_usage >= ALERT_CPU )); then
        local key="cpu_${ALERT_CPU}"
        if [[ -z "${ALERT_SENT[$key]:-}" ]]; then
            send_alert "High CPU Usage" "CPU usage is ${cpu_usage}% (threshold: ${ALERT_CPU}%)" "WARNING"
            ALERT_SENT[$key]=$(date +%s)
        fi
    else
        unset 'ALERT_SENT[cpu_*]' 2>/dev/null || true
    fi
    
    # Memory alert
    if (( mem_percent >= ALERT_MEM )); then
        local key="mem_${ALERT_MEM}"
        if [[ -z "${ALERT_SENT[$key]:-}" ]]; then
            send_alert "High Memory Usage" "Memory usage is ${mem_percent}% (threshold: ${ALERT_MEM}%)" "WARNING"
            ALERT_SENT[$key]=$(date +%s)
        fi
    else
        unset 'ALERT_SENT[mem_*]' 2>/dev/null || true
    fi
    
    # Disk alert
    if (( disk_percent >= ALERT_DISK )); then
        local key="disk_${ALERT_DISK}"
        if [[ -z "${ALERT_SENT[$key]:-}" ]]; then
            send_alert "Low Disk Space" "Disk usage is ${disk_percent}% (threshold: ${ALERT_DISK}%)" "CRITICAL"
            ALERT_SENT[$key]=$(date +%s)
        fi
    else
        unset 'ALERT_SENT[disk_*]' 2>/dev/null || true
    fi
}
```

---

## 20.4 Data Logging & History

```bash
# ─── Data Logging ─────────────────────────────────────────────
log_metrics() {
    local timestamp
    timestamp=$(date +%Y-%m-%dT%H:%M:%S)
    local date_str
    date_str=$(date +%Y-%m-%d)
    
    local cpu_usage=$1
    local mem_info=$2
    local disk_info=$3
    
    local mem_percent
    mem_percent=$(echo "$mem_info" | awk '{print $5}')
    local disk_percent
    disk_percent=$(echo "$disk_info" | awk '{print $4}')
    
    # JSON format for easy parsing
    local data_file="$DATA_DIR/metrics_${date_str}.jsonl"
    
    cat >> "$data_file" << EOF
{"timestamp":"$timestamp","cpu":$cpu_usage,"memory":$mem_percent,"disk":$disk_percent}
EOF
    
    # Rotate old data
    find "$DATA_DIR" -name "metrics_*.jsonl" -mtime "+$HISTORY_RETENTION" -delete 2>/dev/null || true
}

# Generate simple HTML report
generate_html_report() {
    local date_str
    date_str=$(date +%Y-%m-%d)
    local data_file="$DATA_DIR/metrics_${date_str}.jsonl"
    local report_file="$WEB_DIR/index.html"
    
    [[ -f "$data_file" ]] || return
    
    # Parse data
    local avg_cpu avg_mem avg_disk
    avg_cpu=$(awk -F'"cpu":' '{print $2}' "$data_file" | tr -d ',' | awk '{sum+=$1; count++} END{printf "%.0f", sum/count}')
    avg_mem=$(awk -F'"memory":' '{print $2}' "$data_file" | tr -d ',' | awk '{sum+=$1; count++} END{printf "%.0f", sum/count}')
    avg_disk=$(awk -F'"disk":' '{print $2}' "$data_file" | tr -d ',' | awk '{sum+=$1; count++} END{printf "%.0f", sum/count}')
    
    cat > "$report_file" << HTML
<!DOCTYPE html>
<html>
<head>
    <title>System Monitor - $(hostname)</title>
    <meta http-equiv="refresh" content="30">
    <style>
        body { font-family: monospace; background: #1a1a2e; color: #eee; margin: 20px; }
        h1 { color: #00d4ff; }
        .metric { background: #16213e; padding: 15px; margin: 10px 0; border-radius: 8px; }
        .bar { height: 20px; background: #0f3460; border-radius: 4px; margin-top: 8px; }
        .fill-green  { height: 100%; border-radius: 4px; background: #00b894; }
        .fill-yellow { height: 100%; border-radius: 4px; background: #fdcb6e; }
        .fill-red    { height: 100%; border-radius: 4px; background: #e17055; }
        .value { font-size: 2em; font-weight: bold; }
    </style>
</head>
<body>
    <h1>System Monitor: $(hostname)</h1>
    <p>Last updated: $(date) | Refresh: every 30s</p>
    
    <div class="metric">
        <h3>CPU Usage (avg today)</h3>
        <div class="value">${avg_cpu}%</div>
        <div class="bar">
            <div class="$([ "$avg_cpu" -ge 80 ] && echo fill-red || ([ "$avg_cpu" -ge 60 ] && echo fill-yellow || echo fill-green))"
                 style="width:${avg_cpu}%"></div>
        </div>
    </div>
    
    <div class="metric">
        <h3>Memory Usage (avg today)</h3>
        <div class="value">${avg_mem}%</div>
        <div class="bar">
            <div class="$([ "$avg_mem" -ge 85 ] && echo fill-red || ([ "$avg_mem" -ge 70 ] && echo fill-yellow || echo fill-green))"
                 style="width:${avg_mem}%"></div>
        </div>
    </div>
    
    <div class="metric">
        <h3>Disk Usage</h3>
        <div class="value">${avg_disk}%</div>
        <div class="bar">
            <div class="$([ "$avg_disk" -ge 90 ] && echo fill-red || ([ "$avg_disk" -ge 75 ] && echo fill-yellow || echo fill-green))"
                 style="width:${avg_disk}%"></div>
        </div>
    </div>
</body>
</html>
HTML
    
    debug "HTML report generated: $report_file"
}
```

---

## 20.5 Main Loop & Entry Point

```bash
# ─── Daemon Mode ──────────────────────────────────────────────
start_daemon() {
    local pid_file="/var/run/sysmon.pid"
    
    if [[ -f "$pid_file" ]]; then
        local pid
        pid=$(cat "$pid_file")
        if kill -0 "$pid" 2>/dev/null; then
            echo "sysmon already running (PID: $pid)"
            exit 1
        fi
        rm -f "$pid_file"
    fi
    
    # Fork to background
    (
        echo $BASHPID > "$pid_file"
        trap "rm -f $pid_file; exit 0" EXIT TERM INT
        
        while true; do
            local cpu mem disk net load
            cpu=$(get_cpu_usage)
            mem=$(get_memory_info)
            disk=$(get_disk_info)
            net=$(get_network_info)
            load=$(get_load_average)
            
            log_metrics "$cpu" "$mem" "$disk"
            check_thresholds "$cpu" \
                "$(echo "$mem" | awk '{print $5}')" \
                "$(echo "$disk" | awk '{print $4}')"
            generate_html_report
            
            sleep "$INTERVAL"
        done
    ) &
    
    echo "sysmon started (PID: $!)"
}

stop_daemon() {
    local pid_file="/var/run/sysmon.pid"
    if [[ -f "$pid_file" ]]; then
        kill "$(cat "$pid_file")" && echo "sysmon stopped"
        rm -f "$pid_file"
    else
        echo "sysmon is not running"
    fi
}

# ─── Interactive Mode ─────────────────────────────────────────
run_interactive() {
    trap 'echo ""; echo "Stopped"; exit 0' INT TERM
    
    info "Starting system monitor (Interval: ${INTERVAL}s)"
    
    while true; do
        local cpu mem disk net load
        cpu=$(get_cpu_usage)
        mem=$(get_memory_info)
        disk=$(get_disk_info)
        net=$(get_network_info)
        load=$(get_load_average)
        
        display_dashboard "$cpu" "$mem" "$disk" "$net" "$load"
        log_metrics "$cpu" "$mem" "$disk"
        check_thresholds "$cpu" \
            "$(echo "$mem" | awk '{print $5}')" \
            "$(echo "$disk" | awk '{print $4}')"
        
        sleep "$INTERVAL"
    done
}

# ─── Usage ────────────────────────────────────────────────────
usage() {
    cat << EOF
Usage: $0 [command]

Commands:
  start      Start interactive monitoring
  daemon     Run as background daemon
  stop       Stop daemon
  status     Show current status (one-time)
  report     Generate HTML report
  help       Show this help

Environment:
  CONFIG_FILE  Path to config file (default: ./sysmon.conf)
  INTERVAL     Polling interval in seconds (default: 5)
  LOG_LEVEL    DEBUG|INFO|WARN|ERROR (default: INFO)
EOF
}

# ─── Main ─────────────────────────────────────────────────────
setup

case "${1:-start}" in
    start)
        run_interactive
        ;;
    daemon)
        DAEMON_MODE=true
        start_daemon
        ;;
    stop)
        stop_daemon
        ;;
    status)
        cpu=$(get_cpu_usage)
        mem=$(get_memory_info)
        disk=$(get_disk_info)
        net=$(get_network_info)
        load=$(get_load_average)
        display_dashboard "$cpu" "$mem" "$disk" "$net" "$load"
        ;;
    report)
        generate_html_report
        echo "Report generated: $WEB_DIR/index.html"
        ;;
    help|-h|--help)
        usage
        ;;
    *)
        echo "Unknown command: $1"
        usage
        exit 1
        ;;
esac
```

---

## 20.6 Configuration File

```bash
# sysmon.conf - System Monitor Configuration

# Monitoring interval (seconds)
INTERVAL=5

# Alert thresholds (percentage)
ALERT_CPU=80
ALERT_MEM=85
ALERT_DISK=90

# Alerts
ALERT_EMAIL="admin@example.com"
ALERT_WEBHOOK="https://hooks.slack.com/services/xxx/yyy/zzz"

# Data retention (days)
HISTORY_RETENTION=7

# Logging
LOG_LEVEL="INFO"
```

---

## 20.7 Exercises

### Exercise 1: Add Swap Monitoring
เพิ่ม swap usage monitoring:
- Read from /proc/meminfo
- Display ใน dashboard
- Alert เมื่อ swap usage สูง

### Exercise 2: Process Monitor
เพิ่ม functionality:
- Monitor specific process (by name)
- Alert ถ้า process ตาย
- Auto-restart option

### Exercise 3: Custom Metrics
เพิ่ม plugin system:
- Load custom metric scripts จาก plugins/
- Display ใน dashboard
- Include ใน reports

---

## สรุป Part 20 (และ Beginner Level)

✅ Complete system monitor script  
✅ CPU, Memory, Disk, Network metrics  
✅ Real-time terminal dashboard  
✅ Email and webhook alerting  
✅ Historical data logging (JSON)  
✅ HTML report generation  
✅ Daemon mode support  
✅ Configuration file support  

---

## สรุป Level 1: Beginner (Parts 01-20)

| Part | Topic |
|------|-------|
| 01 | Introduction & Shell Basics |
| 02 | Variables & Data Types |
| 03 | String Manipulation |
| 04 | Arrays |
| 05 | Conditionals |
| 06 | Loops |
| 07 | Functions |
| 08 | I/O Redirection |
| 09 | File Operations |
| 10 | Text Processing |
| 11 | Regular Expressions |
| 12 | Process Management |
| 13 | Environment Variables |
| 14 | Permissions & Ownership |
| 15 | Package Management |
| 16 | Networking Basics |
| 17 | Cron Jobs & Scheduling |
| 18 | Debugging & Error Handling |
| 19 | Script Best Practices |
| 20 | Mini Project: System Monitor |

**→ Part 21: Advanced Text Processing (awk ระดับเชี่ยวชาญ)**
