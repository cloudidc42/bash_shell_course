# Part 52: Network Programming and Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 52.1 TCP/UDP Socket Programming

```bash
#!/bin/bash
# network_sockets.sh - Low-level network operations

# ─── TCP Client (pure bash /dev/tcp) ──────────────────────────
tcp_connect() {
    local host=$1 port=$2 timeout=${3:-5}

    # Open TCP connection using bash built-in
    exec {fd}<>/dev/tcp/"$host"/"$port" 2>/dev/null || return 1
    echo "$fd"
}

tcp_send() {
    local fd=$1 data=$2
    echo -e "$data" >&"$fd"
}

tcp_recv() {
    local fd=$1 timeout=${2:-3}
    local line
    IFS= read -r -t "$timeout" line <&"$fd" && echo "$line"
}

tcp_close() {
    local fd=$1
    exec {fd}>&- 2>/dev/null
    exec {fd}<&- 2>/dev/null
}

# ─── HTTP Client (raw TCP) ─────────────────────────────────────
http_raw_get() {
    local host=$1 path=${2:-/} port=${3:-80}
    local response=""

    exec {fd}<>/dev/tcp/"$host"/"$port"

    # Send HTTP/1.1 request
    printf 'GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\n\r\n' \
        "$path" "$host" >&"$fd"

    # Read response
    while IFS= read -r -t 10 line <&"$fd"; do
        response+="$line"$'\n'
    done

    exec {fd}>&-
    exec {fd}<&-

    echo "$response"
}

# ─── Port Scanner ──────────────────────────────────────────────
scan_port() {
    local host=$1 port=$2 timeout=${3:-1}
    (echo >/dev/tcp/"$host"/"$port") 2>/dev/null
    local status=$?
    [[ $status -eq 0 ]] && echo "open" || echo "closed"
}

scan_ports() {
    local host=$1
    local start_port=${2:-1}
    local end_port=${3:-1024}
    local timeout=${4:-1}

    echo "Scanning $host ports $start_port-$end_port..."
    echo ""

    local open_ports=()
    for ((port=start_port; port<=end_port; port++)); do
        if (echo >/dev/tcp/"$host"/"$port") 2>/dev/null; then
            open_ports+=("$port")
            echo "  OPEN: $host:$port"
        fi
    done &

    wait
    echo ""
    echo "Open ports: ${open_ports[*]}"
}

# Parallel port scanner
fast_scan_ports() {
    local host=$1
    local ports=("${@:2}")
    local -a pids=()
    local tmpdir
    tmpdir=$(mktemp -d)

    for port in "${ports[@]}"; do
        (
            if (echo >/dev/tcp/"$host"/"$port") 2>/dev/null; then
                echo "$port" > "$tmpdir/$port"
            fi
        ) &
        pids+=($!)
    done

    wait "${pids[@]}" 2>/dev/null

    # Collect results
    local open_ports=()
    for f in "$tmpdir"/[0-9]*; do
        [[ -f "$f" ]] && open_ports+=("$(cat "$f")")
    done

    rm -rf "$tmpdir"

    # Sort and output
    printf '%s\n' "${open_ports[@]}" | sort -n
}

# ─── Netcat Server ─────────────────────────────────────────────
nc_server() {
    local port=$1
    local handler=$2  # function name to handle connections

    while true; do
        nc -l -p "$port" | while IFS= read -r request; do
            $handler "$request"
        done
    done
}

# ─── UDP Tools ─────────────────────────────────────────────────
udp_send() {
    local host=$1 port=$2 message=$3
    echo "$message" | nc -u -w1 "$host" "$port"
}

send_syslog_udp() {
    local host=$1 port=${2:-514}
    local facility=${3:-1} severity=${4:-5}
    local message=$5

    local priority=$(( (facility * 8) + severity ))
    local timestamp
    timestamp=$(date '+%b %d %H:%M:%S')
    local hostname
    hostname=$(hostname)

    echo "<${priority}>${timestamp} ${hostname} bash: ${message}" | \
        nc -u -w1 "$host" "$port"
}
```

---

## 52.2 DNS Tools

```bash
#!/bin/bash
# dns_tools.sh - DNS lookup and management

# ─── DNS Resolution ────────────────────────────────────────────
dns_resolve() {
    local host=$1
    local type=${2:-A}

    # Try multiple tools
    if command -v dig &>/dev/null; then
        dig +short "$type" "$host" 2>/dev/null
    elif command -v nslookup &>/dev/null; then
        nslookup -type="$type" "$host" 2>/dev/null | grep -E "Address:|name ="
    elif command -v host &>/dev/null; then
        host -t "$type" "$host" 2>/dev/null
    else
        # Fallback: getent
        getent hosts "$host" 2>/dev/null | awk '{print $1}'
    fi
}

# Resolve all record types
dns_full_lookup() {
    local domain=$1
    local types=(A AAAA MX NS TXT SOA CNAME PTR)

    echo "=== DNS Full Lookup: $domain ==="
    for type in "${types[@]}"; do
        local result
        result=$(dig +short "$type" "$domain" 2>/dev/null)
        if [[ -n "$result" ]]; then
            echo ""
            echo "[$type]"
            echo "$result"
        fi
    done
}

# Reverse DNS lookup
rdns() {
    local ip=$1
    dig +short -x "$ip" 2>/dev/null || \
    nslookup "$ip" 2>/dev/null | awk '/name =/ {print $NF}'
}

# ─── DNS Monitoring ────────────────────────────────────────────
dns_propagation_check() {
    local domain=$1
    local record_type=${2:-A}
    local nameservers=(
        "8.8.8.8"      # Google
        "1.1.1.1"      # Cloudflare
        "9.9.9.9"      # Quad9
        "208.67.222.222" # OpenDNS
    )

    echo "Checking DNS propagation for: $domain ($record_type)"
    echo ""

    declare -A results
    for ns in "${nameservers[@]}"; do
        local result
        result=$(dig "@${ns}" +short "$record_type" "$domain" 2>/dev/null | head -1)
        results["$ns"]="$result"
        printf "  %-20s -> %s\n" "$ns" "${result:-[no answer]}"
    done

    # Check consistency
    local unique_results
    unique_results=$(printf '%s\n' "${results[@]}" | sort -u | wc -l)

    echo ""
    if [[ "$unique_results" -eq 1 ]]; then
        echo "DNS is fully propagated (all servers agree)"
    else
        echo "DNS is still propagating ($unique_results different answers)"
    fi
}

# ─── DNS Zone Transfer Check ───────────────────────────────────
check_zone_transfer() {
    local domain=$1
    local nameservers
    nameservers=$(dig +short NS "$domain" 2>/dev/null)

    echo "Testing zone transfer for: $domain"
    while IFS= read -r ns; do
        [[ -z "$ns" ]] && continue
        echo -n "  Testing $ns... "
        if dig AXFR "$domain" "@$ns" 2>/dev/null | grep -q "SOA"; then
            echo "VULNERABLE (zone transfer allowed)"
        else
            echo "secure"
        fi
    done <<< "$nameservers"
}
```

---

## 52.3 SSH Automation

```bash
#!/bin/bash
# ssh_automation.sh - SSH key management and automation

SSH_CONFIG_DIR="$HOME/.ssh"

# ─── Key Management ────────────────────────────────────────────
generate_ssh_key() {
    local name=$1
    local type=${2:-ed25519}
    local comment=${3:-"$(whoami)@$(hostname)"}
    local key_file="$SSH_CONFIG_DIR/${name}_${type}"

    mkdir -p "$SSH_CONFIG_DIR"
    chmod 700 "$SSH_CONFIG_DIR"

    ssh-keygen -t "$type" -C "$comment" -f "$key_file" -N "" -q
    echo "Generated: $key_file"
    cat "${key_file}.pub"
}

rotate_ssh_key() {
    local host=$1 user=$2 old_key_file=$3 new_key_file=$4

    local new_pub
    new_pub=$(cat "${new_key_file}.pub")

    # Add new key
    ssh -i "$old_key_file" "${user}@${host}" \
        "echo '$new_pub' >> ~/.ssh/authorized_keys"

    # Verify new key works
    if ssh -i "$new_key_file" -o BatchMode=yes "${user}@${host}" 'exit 0' 2>/dev/null; then
        echo "Key rotated successfully"
    else
        echo "ERROR: New key failed, keeping old key"
        return 1
    fi
}

# ─── Remote Execution ──────────────────────────────────────────
ssh_exec() {
    local host=$1
    local command=$2
    local timeout=${3:-30}

    ssh -o ConnectTimeout="$timeout" \
        -o BatchMode=yes \
        -o StrictHostKeyChecking=accept-new \
        "$host" "$command"
}

# Parallel SSH execution
ssh_parallel() {
    local hosts_file=$1 command=$2
    local max_jobs=${3:-10}
    local -a pids=()

    while IFS= read -r host; do
        [[ -z "$host" || "$host" =~ ^# ]] && continue

        (
            output=$(ssh_exec "$host" "$command" 2>&1)
            status=$?
            printf '%s\t%d\t%s\n' "$host" "$status" "$output"
        ) &
        pids+=($!)

        if (( ${#pids[@]} >= max_jobs )); then
            wait "${pids[0]}"
            pids=("${pids[@]:1}")
        fi
    done < "$hosts_file"

    wait
}

# ─── SSH Tunnel Management ─────────────────────────────────────
ssh_tunnel_local() {
    local local_port=$1 remote_host=$2 remote_port=$3 jump_host=$4

    ssh -N -f \
        -L "${local_port}:${remote_host}:${remote_port}" \
        -o ServerAliveInterval=30 \
        -o ExitOnForwardFailure=yes \
        "$jump_host"

    echo "Tunnel: localhost:$local_port -> $remote_host:$remote_port via $jump_host"
}

ssh_tunnel_reverse() {
    local remote_port=$1 local_host=$2 local_port=$3 jump_host=$4

    ssh -N -f \
        -R "${remote_port}:${local_host}:${local_port}" \
        -o ServerAliveInterval=30 \
        "$jump_host"

    echo "Reverse tunnel: $jump_host:$remote_port -> $local_host:$local_port"
}

kill_ssh_tunnels() {
    pkill -f "ssh.*-[LR].*-N" || echo "No tunnels found"
}

# ─── SFTP Operations ───────────────────────────────────────────
sftp_sync() {
    local local_dir=$1 remote_dir=$2 host=$3

    rsync -avz --progress \
        -e "ssh -o StrictHostKeyChecking=accept-new" \
        "$local_dir/" \
        "${host}:${remote_dir}/"
}
```

---

## 52.4 Network Monitoring

```bash
#!/bin/bash
# network_monitoring.sh - Continuous network health monitoring

# ─── ICMP Monitoring ───────────────────────────────────────────
ping_monitor() {
    local host=$1
    local interval=${2:-5}
    local threshold=${3:-200}  # ms

    echo "Monitoring: $host (interval: ${interval}s, threshold: ${threshold}ms)"

    local success=0 failure=0

    while true; do
        local output
        output=$(ping -c 1 -W 3 "$host" 2>&1)
        local status=$?

        if [[ $status -eq 0 ]]; then
            local rtt
            rtt=$(echo "$output" | grep -oP 'time=\K[\d.]+')
            ((success++))
        else
            ((failure++))
            echo "$(date '+%H:%M:%S') DOWN  $host (failures: $failure)"
        fi

        local total=$(( success + failure ))
        if (( total % 10 == 0 && total > 0 )); then
            local loss=$(( failure * 100 / total ))
            echo "$(date '+%H:%M:%S') STATS $host - success: $success, loss: ${loss}%"
        fi

        sleep "$interval"
    done
}

# ─── Bandwidth Monitor ─────────────────────────────────────────
bandwidth_monitor() {
    local interface=${1:-eth0}
    local interval=${2:-1}

    local prev_rx=0 prev_tx=0

    printf "%-12s %12s %12s\n" "Time" "RX (KB/s)" "TX (KB/s)"

    while true; do
        local curr_rx curr_tx
        curr_rx=$(cat "/sys/class/net/${interface}/statistics/rx_bytes" 2>/dev/null || echo 0)
        curr_tx=$(cat "/sys/class/net/${interface}/statistics/tx_bytes" 2>/dev/null || echo 0)

        if [[ $prev_rx -gt 0 ]]; then
            local rx_rate=$(( (curr_rx - prev_rx) / interval / 1024 ))
            local tx_rate=$(( (curr_tx - prev_tx) / interval / 1024 ))
            printf "%-12s %12s %12s\n" "$(date '+%H:%M:%S')" "${rx_rate}" "${tx_rate}"
        fi

        prev_rx=$curr_rx
        prev_tx=$curr_tx
        sleep "$interval"
    done
}

# ─── SSL Certificate Monitor ───────────────────────────────────
check_ssl_cert() {
    local domain=$1
    local port=${2:-443}
    local warn_days=${3:-30}

    local cert_info
    cert_info=$(echo | openssl s_client \
        -connect "${domain}:${port}" \
        -servername "$domain" \
        2>/dev/null | openssl x509 -noout -dates -subject 2>/dev/null)

    if [[ -z "$cert_info" ]]; then
        echo "ERROR: Could not retrieve certificate for $domain"
        return 1
    fi

    local expiry_date
    expiry_date=$(echo "$cert_info" | grep "notAfter=" | cut -d= -f2)
    local expiry_epoch
    expiry_epoch=$(date -d "$expiry_date" +%s 2>/dev/null)

    local now_epoch
    now_epoch=$(date +%s)
    local days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

    echo "Domain:     $domain"
    echo "Expires:    $expiry_date"
    echo "Days left:  $days_left"

    if (( days_left < 0 )); then
        echo "Status:     EXPIRED"; return 2
    elif (( days_left < warn_days )); then
        echo "Status:     WARNING"; return 1
    else
        echo "Status:     OK"
    fi
}
```

---

## 52.5 Firewall Management

```bash
#!/bin/bash
# firewall_tools.sh - iptables management

iptables_allow_port() {
    local port=$1 proto=${2:-tcp}
    iptables -A INPUT -p "$proto" --dport "$port" -j ACCEPT
    echo "Allowed $proto:$port"
}

iptables_block_ip() {
    local ip=$1
    iptables -A INPUT -s "$ip" -j DROP
    iptables -A OUTPUT -d "$ip" -j DROP
    echo "Blocked: $ip"
}

iptables_rate_limit() {
    local port=$1 rate=${2:-25} burst=${3:-100}

    iptables -A INPUT -p tcp --dport "$port" \
        -m state --state NEW \
        -m limit --limit "${rate}/second" --limit-burst "$burst" \
        -j ACCEPT

    iptables -A INPUT -p tcp --dport "$port" \
        -m state --state NEW \
        -j DROP

    echo "Rate limit on port $port: ${rate}/s burst $burst"
}

save_iptables() {
    local file=${1:-/etc/iptables/rules.v4}
    mkdir -p "$(dirname "$file")"
    iptables-save > "$file"
    echo "Rules saved to $file"
}

restore_iptables() {
    local file=${1:-/etc/iptables/rules.v4}
    iptables-restore < "$file"
    echo "Rules restored from $file"
}

fail2ban_ban() {
    local ip=$1 jail=${2:-sshd}
    fail2ban-client set "$jail" banip "$ip" 2>/dev/null || \
        iptables -A INPUT -s "$ip" -j DROP
    echo "Banned: $ip (jail: $jail)"
}

fail2ban_unban() {
    local ip=$1 jail=${2:-sshd}
    fail2ban-client set "$jail" unbanip "$ip" 2>/dev/null || \
        iptables -D INPUT -s "$ip" -j DROP 2>/dev/null
    echo "Unbanned: $ip"
}
```

---

## 52.6 Exercises

### Exercise 1: Network Health Dashboard
- Status ของ hosts หลายเครื่องพร้อมกัน
- Bandwidth usage real-time
- SSL certificate expiry
- Connection count per port

### Exercise 2: SSH Bastion Automation
- Jump host management
- Key rotation scheduler
- Audit log สำหรับ SSH sessions

### Exercise 3: Security Scanner
- Port scan และ service detection
- SSL/TLS configuration check
- Generate HTML report

---

## สรุป Part 52

✅ TCP/UDP socket programming with /dev/tcp
✅ DNS resolution, propagation check, zone transfer detection
✅ SSH key management, rotation, parallel execution
✅ SSH tunneling (local/reverse) and SFTP sync
✅ Network bandwidth and connection monitoring
✅ SSL certificate monitoring with expiry alerts
✅ iptables firewall management and rate limiting
✅ fail2ban integration

---

**→ Part 53: Security Automation and Hardening**
