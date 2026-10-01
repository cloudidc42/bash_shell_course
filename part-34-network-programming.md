# Part 34: Network Programming & Protocol Handling
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 34.1 TCP/UDP with /dev/tcp

```bash
# ─── /dev/tcp Built-in ─────────────────────────────────────────
# Bash can connect to TCP/UDP without external tools

# Simple HTTP request
exec 3<>/dev/tcp/example.com/80
printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3
cat <&3
exec 3>&-

# Check if port is open (quick)
check_port() {
    local host=$1
    local port=$2
    local timeout=${3:-3}
    
    if (exec 3<>/dev/tcp/"$host"/"$port") 2>/dev/null; then
        exec 3>&-
        return 0  # open
    fi
    return 1  # closed/filtered
}

# ─── Port Scanner ──────────────────────────────────────────────
scan_ports() {
    local host=$1
    local start_port=${2:-1}
    local end_port=${3:-1024}
    local timeout=${4:-1}
    
    echo "Scanning $host ports $start_port-$end_port"
    
    for (( port=start_port; port<=end_port; port++ )); do
        (
            if timeout "$timeout" bash -c "exec 3<>/dev/tcp/$host/$port" 2>/dev/null; then
                exec 3>&-
                echo "  $port/tcp OPEN"
            fi
        ) &
        
        if (( port % 50 == 0 )); then
            wait
        fi
    done
    wait
    echo "Scan complete"
}

# ─── HTTP Client ───────────────────────────────────────────────
http_get() {
    local url=$1
    local output=${2:-/dev/stdout}
    
    local host path port
    if [[ "$url" =~ ^https?://([^/:]+)(:([0-9]+))?(/.*)?$ ]]; then
        host="${BASH_REMATCH[1]}"
        port="${BASH_REMATCH[3]:-80}"
        path="${BASH_REMATCH[4]:-/}"
    else
        echo "Invalid URL: $url" >&2
        return 1
    fi
    
    exec 3<>/dev/tcp/"$host"/"$port"
    
    printf "GET %s HTTP/1.1\r\nHost: %s\r\nUser-Agent: bash-http\r\nConnection: close\r\n\r\n" \
        "$path" "$host" >&3
    
    local in_body=false
    while IFS= read -r line <&3; do
        line="${line%$'\r'}"
        
        if [[ -z "$line" ]] && ! $in_body; then
            in_body=true
            continue
        fi
        
        $in_body && echo "$line"
    done > "$output"
    
    exec 3>&-
}
```

---

## 34.2 curl Advanced Usage

```bash
#!/bin/bash
# advanced_curl.sh

API_BASE="https://api.example.com"
API_TOKEN="${API_TOKEN:?Token required}"

api_request() {
    local method=$1
    local endpoint=$2
    local data=${3:-}
    local extra_headers=${4:-}
    
    local args=(
        --silent
        --fail-with-body
        --max-time 30
        --retry 3
        --retry-delay 2
        --retry-on-http-error 429,500,502,503
        -X "$method"
        -H "Authorization: Bearer $API_TOKEN"
        -H "Content-Type: application/json"
        -H "Accept: application/json"
        -w "\n%{http_code}"
    )
    
    [[ -n "$data" ]] && args+=(-d "$data")
    
    while IFS= read -r header; do
        [[ -n "$header" ]] && args+=(-H "$header")
    done <<< "$extra_headers"
    
    local response http_code
    response=$(curl "${args[@]}" "${API_BASE}${endpoint}" 2>&1)
    http_code="${response##*$'\n'}"
    response="${response%$'\n'*}"
    
    if [[ "$http_code" -ge 400 ]]; then
        echo "API Error $http_code: $response" >&2
        return 1
    fi
    
    echo "$response"
}

# ─── Concurrent Downloads ──────────────────────────────────────
download_parallel() {
    local urls_file=$1
    local output_dir=$2
    local max_parallel=${3:-4}
    
    mkdir -p "$output_dir"
    
    while IFS= read -r url; do
        local filename
        filename=$(basename "$url")
        
        while (( $(jobs -r | wc -l) >= max_parallel )); do
            wait -n 2>/dev/null
        done
        
        (
            curl -sL --output "$output_dir/$filename" \
                --progress-bar \
                --retry 3 \
                "$url" && \
                echo "Downloaded: $filename"
        ) &
    done < "$urls_file"
    
    wait
    echo "All downloads complete"
}

# ─── Webhook sender ────────────────────────────────────────────
send_webhook() {
    local webhook_url=$1
    local message=$2
    local title=${3:-"Notification"}
    
    local payload
    payload=$(jq -n \
        --arg title "$title" \
        --arg message "$message" \
        --arg timestamp "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        '{title: $title, message: $message, timestamp: $timestamp}')
    
    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "$webhook_url"
}

# Slack specific
send_slack() {
    local webhook=$1
    local text=$2
    local channel=${3:-"#general"}
    
    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "$(jq -n --arg text "$text" --arg channel "$channel" \
            '{text: $text, channel: $channel}')" \
        "$webhook"
}
```

---

## 34.3 SSH Tunneling & Port Forwarding

```bash
#!/bin/bash
# tunnel_manager.sh

create_tunnel() {
    local name=$1
    local local_port=$2
    local remote_host=$3
    local remote_port=$4
    local jump_host=$5
    
    local pid_file="/tmp/tunnel_${name}.pid"
    local log_file="/tmp/tunnel_${name}.log"
    
    if [[ -f "$pid_file" ]]; then
        kill "$(cat "$pid_file")" 2>/dev/null
        rm -f "$pid_file"
    fi
    
    if command -v autossh &>/dev/null; then
        AUTOSSH_PIDFILE="$pid_file" \
        AUTOSSH_LOGFILE="$log_file" \
        autossh -M 0 \
            -o "ServerAliveInterval=30" \
            -o "ServerAliveCountMax=3" \
            -o "ExitOnForwardFailure=yes" \
            -o "StrictHostKeyChecking=no" \
            -fN \
            -L "${local_port}:${remote_host}:${remote_port}" \
            "${jump_host}"
    else
        while true; do
            ssh -fN \
                -o "ServerAliveInterval=30" \
                -o "ServerAliveCountMax=3" \
                -o "ExitOnForwardFailure=yes" \
                -L "${local_port}:${remote_host}:${remote_port}" \
                "${jump_host}"
            
            echo "Tunnel $name died, restarting in 5s..."
            sleep 5
        done &
        echo $! > "$pid_file"
    fi
    
    echo "Tunnel $name: localhost:$local_port → $remote_host:$remote_port (via $jump_host)"
}

create_socks_proxy() {
    local proxy_port=$1
    local ssh_host=$2
    
    ssh -fN -D "$proxy_port" \
        -o "ServerAliveInterval=60" \
        -o "ExitOnForwardFailure=yes" \
        "$ssh_host"
    
    echo "SOCKS5 proxy on port $proxy_port via $ssh_host"
    echo "Configure: export https_proxy=socks5://127.0.0.1:$proxy_port"
}

tcp_relay() {
    local local_port=$1
    local target_host=$2
    local target_port=$3
    
    if command -v socat &>/dev/null; then
        socat "TCP-LISTEN:${local_port},reuseaddr,fork" \
            "TCP:${target_host}:${target_port}" &
        echo "Relay: localhost:$local_port → $target_host:$target_port"
        return
    fi
    
    if command -v ncat &>/dev/null; then
        ncat -l "$local_port" --sh-exec "ncat $target_host $target_port" &
        return
    fi
    
    echo "Neither socat nor ncat found" >&2
    return 1
}
```

---

## 34.4 Network Monitoring

```bash
#!/bin/bash
# network_monitor.sh

ping_monitor() {
    local host=$1
    local interval=${2:-5}
    local alert_threshold=${3:-3}
    
    local failures=0
    local successes=0
    
    while true; do
        if ping -c1 -W2 "$host" &>/dev/null; then
            (( successes++ ))
            (( failures > 0 )) && echo "$(date): $host recovered after $failures failures"
            failures=0
        else
            (( failures++ ))
            if (( failures == alert_threshold )); then
                echo "ALERT: $(date): $host unreachable for $failures attempts"
            fi
        fi
        
        sleep "$interval"
    done
}

bandwidth_monitor() {
    local interface=${1:-eth0}
    local interval=${2:-1}
    
    local prev_rx=0 prev_tx=0
    
    while true; do
        local rx tx
        rx=$(grep "$interface:" /proc/net/dev | awk '{print $2}')
        tx=$(grep "$interface:" /proc/net/dev | awk '{print $10}')
        
        if (( prev_rx > 0 )); then
            local rx_rate=$(( (rx - prev_rx) / interval ))
            local tx_rate=$(( (tx - prev_tx) / interval ))
            
            printf "\r%s: RX: %6s/s  TX: %6s/s" \
                "$interface" \
                "$(numfmt --to=iec "$rx_rate")" \
                "$(numfmt --to=iec "$tx_rate")"
        fi
        
        prev_rx=$rx
        prev_tx=$tx
        
        sleep "$interval"
    done
}

health_check_all() {
    local config_file=$1
    
    echo "=== Service Health Check $(date) ==="
    echo ""
    
    local total=0 healthy=0 unhealthy=0
    
    while IFS=' ' read -r name host port path; do
        [[ "$name" =~ ^# ]] && continue
        [[ -z "$name" ]] && continue
        (( total++ ))
        
        local url="http://${host}:${port}${path:-/}"
        local http_code
        http_code=$(curl -s -o /dev/null -w "%{http_code}" \
            --max-time 5 "$url" 2>/dev/null)
        
        if [[ "$http_code" =~ ^[23] ]]; then
            echo "  ✓ $name ($url) → HTTP $http_code"
            (( healthy++ ))
        else
            echo "  ✗ $name ($url) → HTTP ${http_code:-timeout}"
            (( unhealthy++ ))
        fi
    done < "$config_file"
    
    echo ""
    echo "Summary: $healthy/$total healthy"
    
    (( unhealthy > 0 )) && return 1
    return 0
}
```

---

## 34.5 Exercises

### Exercise 1: HTTP Server Monitor
สร้าง tool ตรวจสอบ web services:
- Check response time
- Validate content
- Alert on degradation
- History/trending

### Exercise 2: Network Scanner
สร้าง network discovery:
- Ping sweep
- Port scan
- Service detection
- Output CSV/JSON

### Exercise 3: Traffic Shaper
สร้าง tool จำลอง network conditions:
- Add latency
- Limit bandwidth
- Simulate packet loss
- Compare performance

---

## สรุป Part 34

✅ TCP/UDP with /dev/tcp  
✅ Port scanning  
✅ Advanced curl (REST API, parallel downloads)  
✅ SSH tunneling & port forwarding  
✅ SOCKS proxy via SSH  
✅ Bandwidth monitoring  
✅ Service health checks  

---

**→ Part 35: System Automation & Configuration Management**
