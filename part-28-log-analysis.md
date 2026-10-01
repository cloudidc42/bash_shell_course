# Part 28: Log Analysis & Monitoring
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 28.1 Log Processing Fundamentals

```bash
# ─── System Logs ──────────────────────────────────────────────
# Linux system logs
tail -f /var/log/syslog         # general system log
tail -f /var/log/auth.log       # authentication
tail -f /var/log/kern.log       # kernel messages
tail -f /var/log/daemon.log     # daemons
journalctl -f                   # systemd journal (follow)
journalctl -u nginx -f          # specific service
journalctl --since "1 hour ago"
journalctl --since "2024-01-15 08:00:00"
journalctl -p err..emerg        # error level and above
journalctl -k                   # kernel messages

# ─── Log Formats ──────────────────────────────────────────────
# Syslog format:
# Jan 15 08:30:01 hostname service[pid]: message

# Apache/Nginx combined log:
# IP - user [date] "method uri version" status size "referer" "UA"

# JSON log (modern):
# {"timestamp":"2024-01-15T08:30:01Z","level":"INFO","msg":"..."}

# ─── Quick log analysis ───────────────────────────────────────
# Count errors
grep -c "ERROR" /var/log/app.log

# Show unique errors
grep "ERROR" /var/log/app.log | sort -u

# Errors in last hour
awk -v d="$(date -d '1 hour ago' '+%Y-%m-%dT%H:%M')" \
    '$0 >= d' /var/log/app.log | grep ERROR | wc -l

# Most common errors
grep "ERROR" /var/log/app.log | \
    sed 's/.*ERROR: //' | \
    sort | uniq -c | sort -rn | head -10

# Response time stats
awk '{print $NF}' /var/log/nginx/access.log | \
    sort -n | \
    awk 'BEGIN{print "Min\tMax\tAvg\tP95\tP99"}
    {a[NR]=$1; sum+=$1}
    END{
        printf "%d\t%d\t%.0f\t%d\t%d\n",
        a[1], a[NR], sum/NR,
        a[int(NR*0.95)], a[int(NR*0.99)]
    }'
```

---

## 28.2 Real-time Log Monitoring

```bash
#!/bin/bash
# log_monitor.sh - Real-time log monitoring with alerts

set -euo pipefail

LOG_FILE="${1:-/var/log/syslog}"
ALERT_PATTERNS=("CRITICAL" "FATAL" "panic:" "Out of memory" "disk space")
ALERT_COMMAND="${ALERT_COMMAND:-}"   # Command to run on alert
ALERT_COOLDOWN=300   # seconds between same alerts
CONTEXT_LINES=5

declare -A ALERT_TIMES=()

# Colors
RED='\033[0;31m' YELLOW='\033[1;33m' GREEN='\033[0;32m'
CYAN='\033[0;36m' NC='\033[0m'

alert() {
    local pattern=$1
    local line=$2
    local now
    now=$(date +%s)
    
    # Check cooldown
    local last=${ALERT_TIMES[$pattern]:-0}
    (( now - last < ALERT_COOLDOWN )) && return
    
    ALERT_TIMES[$pattern]=$now
    
    echo -e "${RED}[ALERT]${NC} Pattern: $pattern"
    echo -e "  Line: $line"
    echo -e "  Time: $(date)"
    
    # Send notification
    if [[ -n "$ALERT_COMMAND" ]]; then
        echo "$line" | eval "$ALERT_COMMAND" 2>/dev/null || true
    fi
}

# Start monitoring
echo -e "${GREEN}Monitoring:${NC} $LOG_FILE"
echo -e "${GREEN}Patterns:${NC} ${ALERT_PATTERNS[*]}"
echo "Press Ctrl+C to stop"
echo ""

# Track statistics
declare -i error_count=0
declare -i warn_count=0
declare -i line_count=0
start_time=$(date +%s)

tail -F "$LOG_FILE" 2>/dev/null | while IFS= read -r line; do
    (( line_count++ ))
    
    # Check patterns
    for pattern in "${ALERT_PATTERNS[@]}"; do
        if echo "$line" | grep -qi "$pattern"; then
            alert "$pattern" "$line"
        fi
    done
    
    # Colorize output
    if echo "$line" | grep -qi "ERROR\|CRITICAL\|FATAL"; then
        echo -e "${RED}$line${NC}"
        (( error_count++ ))
    elif echo "$line" | grep -qi "WARN"; then
        echo -e "${YELLOW}$line${NC}"
        (( warn_count++ ))
    else
        echo "$line"
    fi
    
    # Stats every 100 lines
    if (( line_count % 100 == 0 )); then
        local elapsed=$(( $(date +%s) - start_time ))
        local rate=$(( line_count / (elapsed + 1) ))
        echo -e "${CYAN}[Stats]${NC} Lines: $line_count | Errors: $error_count | Warns: $warn_count | Rate: ${rate}/s"
    fi
done
```

---

## 28.3 Log Parser & Analyzer

```bash
#!/bin/bash
# access_log_analyzer.sh - Analyze web server access logs

set -euo pipefail

LOG_FILE="${1:-/var/log/nginx/access.log}"
OUTPUT_DIR="${OUTPUT_DIR:-/tmp/log_analysis}"
TOP_N=20

mkdir -p "$OUTPUT_DIR"

echo "Analyzing: $LOG_FILE"
echo "Output: $OUTPUT_DIR"

# Parse Apache/Nginx combined log format
# 1.2.3.4 - user [01/Jan/2024:00:00:01 +0000] "GET /path HTTP/1.1" 200 1234 "-" "Mozilla/5.0"

parse_log() {
    awk '
    {
        # Extract fields
        ip = $1
        
        # Extract date from brackets
        match($0, /\[([^\]]+)\]/, arr)
        timestamp = arr[1]
        
        # Extract request
        match($0, /"([^"]+)"/, arr)
        request = arr[1]
        split(request, req, " ")
        method = req[1]
        url = req[2]
        
        # Status and size
        status = $(NF-3)  # might vary
        # Better: parse by position after the timestamp bracket
        match($0, /\] "([^"]+)" ([0-9]+) ([0-9-]+)/, arr)
        status = arr[2]
        size = arr[3] == "-" ? 0 : arr[3]
        
        print ip, timestamp, method, url, status, size
    }'
}

# Status code analysis
analyze_status() {
    awk '{status=$5; count[status]++; total++}
    END {
        print "Status Codes:"
        for (s in count) printf "  %s: %d (%.1f%%)\n", s, count[s], count[s]/total*100
    }' | sort -k2 -rn
}

# Top IPs
top_ips() {
    awk '{count[$1]++} END {
        for (ip in count) print count[ip], ip
    }' | sort -rn | head -"$TOP_N"
}

# Top URLs
top_urls() {
    awk '{count[$4]++} END {
        for (url in count) print count[url], url
    }' | sort -rn | head -"$TOP_N"
}

# Requests per hour
requests_per_hour() {
    awk '{
        match($2, /([0-9]+)\/([A-Za-z]+)\/([0-9]+):([0-9]+)/, arr)
        hour = arr[4]
        count[hour]++
    }
    END {
        for (h in count) printf "%02d:00 %d\n", h, count[h]
    }' | sort
}

# Large responses
large_responses() {
    awk '$6 > 100000 {print $6, $4, $5}' | \
        sort -rn | head -10 | \
        awk '{printf "%-12s %-10s %s\n", $1, $2, $3}'
}

# Error analysis (4xx, 5xx)
error_analysis() {
    awk '$5 >= 400 {errors[$5" "$4]++}
    END {
        for (e in errors) print errors[e], e
    }' | sort -rn | head -20
}

# Generate report
generate_report() {
    local report="$OUTPUT_DIR/report.txt"
    local tmp_parsed="$OUTPUT_DIR/parsed.tmp"
    
    echo "Parsing log..."
    parse_log < "$LOG_FILE" > "$tmp_parsed"
    
    {
        echo "═══════════════════════════════════════"
        echo "  Log Analysis Report"
        echo "  File: $LOG_FILE"
        echo "  Generated: $(date)"
        echo "═══════════════════════════════════════"
        echo ""
        
        echo "Total requests: $(wc -l < "$tmp_parsed")"
        echo "Unique IPs: $(awk '{print $1}' "$tmp_parsed" | sort -u | wc -l)"
        echo "Unique URLs: $(awk '{print $4}' "$tmp_parsed" | sort -u | wc -l)"
        echo ""
        
        echo "── Status Codes ──"
        analyze_status < "$tmp_parsed"
        echo ""
        
        echo "── Top $TOP_N IPs ──"
        top_ips < "$tmp_parsed"
        echo ""
        
        echo "── Top $TOP_N URLs ──"
        top_urls < "$tmp_parsed"
        echo ""
        
        echo "── Requests per Hour ──"
        requests_per_hour < "$tmp_parsed"
        echo ""
        
        echo "── Top Errors ──"
        error_analysis < "$tmp_parsed"
        echo ""
        
        echo "── Largest Responses ──"
        large_responses < "$tmp_parsed"
        
    } | tee "$report"
    
    rm -f "$tmp_parsed"
    echo ""
    echo "Report saved to: $report"
}

generate_report
```

---

## 28.4 Centralized Logging with Syslog

```bash
# ─── rsyslog configuration ────────────────────────────────────
# /etc/rsyslog.conf additions:

cat > /etc/rsyslog.d/remote.conf << 'EOF'
# Send to remote syslog server
*.* @192.168.1.100:514    # UDP
*.* @@192.168.1.100:514   # TCP

# Or filter specific facility
local0.* @logserver:514
local1.* @logserver:514

# Template for structured logging
$template myFormat,"%TIMESTAMP:::date-rfc3339% %HOSTNAME% %syslogtag%%msg%\n"
*.* @logserver:514;myFormat
EOF

# ─── Send logs from script ────────────────────────────────────
log_to_syslog() {
    local priority=${1:-info}
    local message=$2
    local facility=${3:-local0}
    
    logger -p "${facility}.${priority}" -t "$(basename "$0")[$$]" "$message"
}

# Redirect stdout/stderr to syslog
exec 1> >(logger -t "myscript" -p local0.info)
exec 2> >(logger -t "myscript" -p local0.error)

# ─── Journald structured logging ──────────────────────────────
# Write structured fields to journald
systemd-cat -t "myapp" -p info echo "Application started"

# In scripts
log_systemd() {
    local message=$1
    local level=${2:-info}
    
    echo "$message" | systemd-cat -t "$(basename "$0")" -p "$level"
}

# Query with filters
journalctl _SYSTEMD_UNIT=myapp.service --since "1 hour ago"
journalctl SYSLOG_IDENTIFIER=myapp
journalctl -p err..emerg -o json-pretty

# ─── Log rotation ─────────────────────────────────────────────
# /etc/logrotate.d/myapp
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 644 myapp myapp
    sharedscripts
    postrotate
        systemctl kill -s HUP myapp.service 2>/dev/null || true
    endscript
}
EOF

# Manual rotation
logrotate -f /etc/logrotate.d/myapp
```

---

## 28.5 Exercises

### Exercise 1: Log Aggregator
สร้าง script ที่:
- Collect logs จากหลาย servers
- Parse different formats
- Store ใน central database
- Query interface

### Exercise 2: Anomaly Detector
สร้าง log anomaly detection:
- Baseline normal behavior
- Alert on deviations
- Rate limiting on alerts
- Dashboard report

### Exercise 3: Audit Trail
สร้าง audit trail system:
- Log all admin commands
- Track file access
- Record user sessions
- Generate compliance report

---

## สรุป Part 28

✅ System log locations and tools  
✅ Real-time log monitoring  
✅ Pattern matching and alerting  
✅ Access log analysis (status, IPs, URLs)  
✅ Requests per hour analysis  
✅ Centralized logging (rsyslog, journald)  
✅ Log rotation  

---

**→ Part 29: Backup & Disaster Recovery**
