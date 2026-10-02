# Part 64: Networking and Service Discovery
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 64.1 Network Diagnostics

```bash
#!/bin/bash
# net_diag.sh - Network diagnostics toolkit

# ─── Connectivity Tests ───────────────────────────────────────────────
ping_test() {
    local host=$1 count=${2:-4} timeout=${3:-2}

    local result
    result=$(ping -c "$count" -W "$timeout" "$host" 2>&1)
    local exit_code=$?

    if (( exit_code == 0 )); then
        local rtt; rtt=$(echo "$result" | grep 'rtt\|round-trip' | \
            grep -oP 'min/avg/max[^=]*=\s*\K[0-9.]+/[0-9.]+/[0-9.]+' | head -1)
        echo "REACHABLE: $host  RTT: $rtt ms"
    else
        echo "UNREACHABLE: $host"
    fi

    return $exit_code
}

traceroute_host() {
    local host=$1 max_hops=${2:-30}

    if command -v traceroute &>/dev/null; then
        traceroute -m "$max_hops" "$host"
    elif command -v tracepath &>/dev/null; then
        tracepath -m "$max_hops" "$host"
    else
        echo "Neither traceroute nor tracepath available"
        return 1
    fi
}

check_port() {
    local host=$1 port=$2 timeout=${3:-3}

    if timeout "$timeout" bash -c "echo >/dev/tcp/${host}/${port}" 2>/dev/null; then
        echo "OPEN: ${host}:${port}"
        return 0
    else
        echo "CLOSED/FILTERED: ${host}:${port}"
        return 1
    fi
}

scan_ports() {
    local host=$1
    shift
    local ports=("$@")
    local open=() closed=()

    for port in "${ports[@]}"; do
        if timeout 1 bash -c "echo >/dev/tcp/${host}/${port}" 2>/dev/null; then
            open+=("$port")
        else
            closed+=("$port")
        fi
    done

    echo "Host: $host"
    echo "Open ports: ${open[*]:-none}"
    echo "Closed ports: ${closed[*]:-none}"
}

# ─── DNS Tools ────────────────────────────────────────────────────
dns_lookup() {
    local hostname=$1 record_type=${2:-A}

    if command -v dig &>/dev/null; then
        dig +short "$hostname" "$record_type"
    elif command -v nslookup &>/dev/null; then
        nslookup -type="$record_type" "$hostname" | \
            awk '/^Address/ && !/server/ {print $2}'
    elif command -v host &>/dev/null; then
        host -t "$record_type" "$hostname" | awk '{print $NF}'
    else
        getent hosts "$hostname" | awk '{print $1}'
    fi
}

reverse_dns() {
    local ip=$1

    if command -v dig &>/dev/null; then
        dig +short -x "$ip"
    else
        getent hosts "$ip" | awk '{print $2}'
    fi
}

dns_check_propagation() {
    local hostname=$1 record_type=${2:-A}
    local -a nameservers=(
        "8.8.8.8"
        "1.1.1.1"
        "9.9.9.9"
        "208.67.222.222"
    )

    echo "DNS Propagation Check: $hostname ($record_type)"
    echo "================================================"

    for ns in "${nameservers[@]}"; do
        local result
        result=$(dig +short +time=3 "@${ns}" "$hostname" "$record_type" 2>/dev/null | head -3)
        printf "%-20s -> %s\n" "$ns" "${result:-[no answer]}"
    done
}

# ─── Network Interface Info ───────────────────────────────────────────
get_interface_info() {
    local iface=${1:-}

    if [[ -n "$iface" ]]; then
        ip addr show "$iface" 2>/dev/null
    else
        ip addr show 2>/dev/null
    fi
}

get_default_gateway() {
    ip route show default 2>/dev/null | awk '/default via/ {print $3}' | head -1
}

get_public_ip() {
    local services=(
        "https://api.ipify.org"
        "https://ifconfig.me"
        "https://icanhazip.com"
    )

    for svc in "${services[@]}"; do
        local ip
        ip=$(curl -s --max-time 5 "$svc" 2>/dev/null | tr -d '[:space:]')
        if [[ "$ip" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
            echo "$ip"
            return 0
        fi
    done

    echo "Unable to determine public IP"
    return 1
}

network_summary() {
    echo "=== Network Summary ==="

    echo "Interfaces:"
    ip -brief addr show 2>/dev/null | awk '{printf "  %-15s %-12s %s\n", $1, $2, $3}'

    echo ""
    echo "Default Gateway: $(get_default_gateway)"

    echo ""
    echo "DNS Servers:"
    if command -v resolvectl &>/dev/null; then
        resolvectl status 2>/dev/null | grep 'DNS Servers:' | head -3
    else
        grep '^nameserver' /etc/resolv.conf | awk '{print "  " $2}'
    fi
}
```

---

## 64.2 Service Discovery

```bash
#!/bin/bash
# service_discovery.sh - Service discovery utilities

# ─── mDNS/Bonjour Discovery ─────────────────────────────────────────────
discover_mdns() {
    local service_type=${1:-_http._tcp}
    local timeout=${2:-5}

    if command -v avahi-browse &>/dev/null; then
        timeout "$timeout" avahi-browse -t "$service_type" 2>/dev/null | \
            grep '^=' | awk '{print $4, $5, $7}'
    elif command -v dns-sd &>/dev/null; then
        timeout "$timeout" dns-sd -B "$service_type" local 2>/dev/null
    else
        echo "No mDNS browser available (install avahi-utils)"
        return 1
    fi
}

# ─── Port-based Service Detection ──────────────────────────────────────────
detect_service() {
    local host=$1 port=$2 timeout=${3:-2}

    if ! timeout "$timeout" bash -c "echo >/dev/tcp/${host}/${port}" 2>/dev/null; then
        echo "port $port is closed"
        return 1
    fi

    local banner
    banner=$(timeout "$timeout" bash -c "cat </dev/tcp/${host}/${port}" 2>/dev/null | head -c 256 | strings | head -2)

    local service="unknown"
    case "$port" in
        22)   service="SSH" ;;
        80)   service="HTTP" ;;
        443)  service="HTTPS" ;;
        3306) service="MySQL" ;;
        5432) service="PostgreSQL" ;;
        6379) service="Redis" ;;
        27017) service="MongoDB" ;;
        9200) service="Elasticsearch" ;;
        *)
            if [[ "$banner" =~ SSH ]]; then service="SSH"
            elif [[ "$banner" =~ HTTP ]]; then service="HTTP"
            elif [[ "$banner" =~ MySQL|MariaDB ]]; then service="MySQL"
            fi
            ;;
    esac

    echo "${service} on ${host}:${port}"
    [[ -n "$banner" ]] && echo "  Banner: ${banner:0:100}"
}

scan_common_services() {
    local host=$1
    local common_ports=(21 22 23 25 53 80 110 143 443 3306 5432 6379 8080 8443 27017)

    echo "Service scan: $host"
    echo "=========================="

    for port in "${common_ports[@]}"; do
        if timeout 1 bash -c "echo >/dev/tcp/${host}/${port}" 2>/dev/null; then
            detect_service "$host" "$port" 2 2>/dev/null
        fi
    done
}

# ─── JSON Service Registry ───────────────────────────────────────────────
SERVICE_REGISTRY_DIR="${SERVICE_REGISTRY_DIR:-/tmp/service_registry}"

service_register() {
    local name=$1 host=$2 port=$3
    local tags=${4:-} meta=${5:-}

    mkdir -p "$SERVICE_REGISTRY_DIR"

    local service_id="${name}-${host}-${port}"
    local file="${SERVICE_REGISTRY_DIR}/${service_id}.json"

    jq -n \
        --arg id "$service_id" \
        --arg name "$name" \
        --arg host "$host" \
        --argjson port "$port" \
        --arg tags "$tags" \
        --arg registered "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        '{id:$id,name:$name,host:$host,port:$port,tags:($tags|split(",")),registered:$registered}' \
        > "$file"

    echo "Registered service: $service_id"
}

service_deregister() {
    local service_id=$1
    rm -f "${SERVICE_REGISTRY_DIR}/${service_id}.json"
    echo "Deregistered: $service_id"
}

service_discover() {
    local name=$1

    local results=()
    for file in "${SERVICE_REGISTRY_DIR}"/*.json 2>/dev/null; do
        [[ -f "$file" ]] || continue

        local svc_name; svc_name=$(jq -r '.name' "$file" 2>/dev/null)
        if [[ "$svc_name" == "$name" ]]; then
            results+=("$(jq -c '{host:.host, port:.port}' "$file")")
        fi
    done

    if (( ${#results[@]} == 0 )); then
        echo "No services found for: $name"
        return 1
    fi

    printf '%s\n' "${results[@]}" | jq -s .
}

service_list() {
    echo "=== Registered Services ==="
    for file in "${SERVICE_REGISTRY_DIR}"/*.json 2>/dev/null; do
        [[ -f "$file" ]] || continue
        jq -r '"  \(.name)  \(.host):\(.port)  [\(.registered)]"' "$file"
    done
}
```

---

## 64.3 Load Balancer and Health Checking

```bash
#!/bin/bash
# load_balancer.sh - Simple load balancer with health checks

declare -A LB_BACKENDS=()
declare -A LB_HEALTH=()
declare -i LB_CURRENT=0

lb_add_backend() {
    local name=$1 host=$2 port=$3 weight=${4:-1}
    LB_BACKENDS["$name"]="${host}:${port}:${weight}"
    LB_HEALTH["$name"]="healthy"
}

lb_remove_backend() {
    local name=$1
    unset "LB_BACKENDS[$name]"
    unset "LB_HEALTH[$name]"
}

lb_check_health() {
    local name=$1
    local endpoint="${LB_BACKENDS[$name]}"
    local host="${endpoint%%:*}"
    local rest="${endpoint#*:}"
    local port="${rest%%:*}"

    if timeout 3 bash -c "echo >/dev/tcp/${host}/${port}" 2>/dev/null; then
        LB_HEALTH["$name"]="healthy"
        return 0
    else
        LB_HEALTH["$name"]="unhealthy"
        return 1
    fi
}

lb_run_health_checks() {
    for name in "${!LB_BACKENDS[@]}"; do
        lb_check_health "$name"
        echo "  ${name}: ${LB_HEALTH[$name]}"
    done
}

lb_get_next_backend() {
    local -a healthy_backends=()

    for name in "${!LB_BACKENDS[@]}"; do
        [[ "${LB_HEALTH[$name]}" == "healthy" ]] && healthy_backends+=("$name")
    done

    (( ${#healthy_backends[@]} == 0 )) && { echo "ERROR: No healthy backends" >&2; return 1; }

    local idx=$(( LB_CURRENT % ${#healthy_backends[@]} ))
    (( LB_CURRENT++ ))

    local selected="${healthy_backends[$idx]}"
    local endpoint="${LB_BACKENDS[$selected]}"
    local host="${endpoint%%:*}"
    local rest="${endpoint#*:}"
    local port="${rest%%:*}"

    echo "${host}:${port}"
}
```

---

## 64.4 Network Monitoring

```bash
#!/bin/bash
# net_monitor.sh - Network bandwidth and connection monitoring

# ─── Bandwidth Monitor ───────────────────────────────────────────────
monitor_bandwidth() {
    local iface=${1:-eth0} interval=${2:-1} samples=${3:-10}

    local prev_rx=0 prev_tx=0
    local rx_file="/sys/class/net/${iface}/statistics/rx_bytes"
    local tx_file="/sys/class/net/${iface}/statistics/tx_bytes"

    [[ -f "$rx_file" ]] || { echo "Interface $iface not found"; return 1; }

    printf "%-10s %15s %15s\n" "Time" "RX (KB/s)" "TX (KB/s)"
    printf "%-10s %15s %15s\n" "----" "---------" "---------"

    local i
    for (( i=0; i<samples; i++ )); do
        local rx tx
        rx=$(cat "$rx_file")
        tx=$(cat "$tx_file")

        if (( i > 0 )); then
            local rx_rate=$(( (rx - prev_rx) / 1024 / interval ))
            local tx_rate=$(( (tx - prev_tx) / 1024 / interval ))
            printf "%-10s %15d %15d\n" "$(date +%H:%M:%S)" "$rx_rate" "$tx_rate"
        fi

        prev_rx=$rx
        prev_tx=$tx
        sleep "$interval"
    done
}

# ─── Connection Monitor ───────────────────────────────────────────────
monitor_connections() {
    local interval=${1:-5}

    while true; do
        clear
        echo "=== Active Connections ($(date)) ==="
        echo ""

        echo "By State:"
        ss -tan 2>/dev/null | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn | \
            awk '{printf "  %-15s %d\n", $2, $1}'

        echo ""
        echo "Top Remote Hosts:"
        ss -tan 2>/dev/null | awk 'NR>1 && $1=="ESTAB" {print $5}' | \
            grep -oP '^[^:]+' | sort | uniq -c | sort -rn | head -10 | \
            awk '{printf "  %-25s %d\n", $2, $1}'

        sleep "$interval"
    done
}

# ─── Latency Test ────────────────────────────────────────────────────
latency_test() {
    local hosts=("$@")
    local count=10

    echo "Latency Test (${count} pings)"
    echo "================================"

    for host in "${hosts[@]}"; do
        local result
        result=$(ping -c "$count" -q "$host" 2>/dev/null)

        if (( $? == 0 )); then
            local stats; stats=$(echo "$result" | grep 'rtt\|round-trip' | \
                grep -oP '[0-9.]+/[0-9.]+/[0-9.]+/[0-9.]+' | head -1)
            local loss; loss=$(echo "$result" | grep -oP '[0-9]+(?=% packet loss)')
            printf "%-30s min/avg/max: %-25s loss: %s%%\n" "$host" "$stats" "${loss:-0}"
        else
            printf "%-30s UNREACHABLE\n" "$host"
        fi
    done
}
```

---

## 64.5 Exercises

### Exercise 1: Network Discovery Tool
สร้าง tool ที่:
- ARP scan local subnet
- Detect OS via TTL
- Service fingerprinting
- Export to JSON/CSV

### Exercise 2: Service Mesh
สร้าง simple mesh ที่:
- Register/deregister services
- Health-based routing
- Load balancing (round-robin/least-conn)
- Circuit breaker

### Exercise 3: Network Monitor Dashboard
สร้าง dashboard ที่:
- Real-time bandwidth graphs (ASCII)
- Connection state breakdown
- Alert on unusual traffic
- Historical data

---

## สรุป Part 64

✅ Ping, traceroute, port check, subnet scan
✅ DNS lookup (dig/nslookup/host fallback), reverse DNS, propagation check
✅ Network interface summary, default gateway, public IP detection
✅ mDNS/Bonjour service discovery via avahi-browse
✅ Port-based service detection with banner grab
✅ JSON-based service registry (register/deregister/discover/list)
✅ Round-robin load balancer with health checks
✅ /sys-based bandwidth monitor with KB/s rates
✅ Connection state and remote host monitoring via ss
✅ Multi-host latency test with min/avg/max/loss

---

**→ Part 65: Backup and Disaster Recovery**
