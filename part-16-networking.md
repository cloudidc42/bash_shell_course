# Part 16: Networking Basics
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 16.1 Network Configuration

```bash
# ─── Network Interfaces ───────────────────────────────────────
ip addr show                    # show all interfaces
ip addr show eth0               # specific interface
ip link show                    # link layer info
ifconfig                        # old command (net-tools)
ifconfig -a                     # all interfaces

# Interface management
ip link set eth0 up
ip link set eth0 down
ip addr add 192.168.1.100/24 dev eth0
ip addr del 192.168.1.100/24 dev eth0

# ─── Routing ──────────────────────────────────────────────────
ip route show                   # show routing table
ip route add 10.0.0.0/8 via 192.168.1.1
ip route del 10.0.0.0/8
ip route add default via 192.168.1.1   # default gateway
route -n                        # old command

# ─── DNS ──────────────────────────────────────────────────────
cat /etc/resolv.conf            # DNS servers
cat /etc/hosts                  # local hostname resolution

resolvectl status               # systemd-resolved status
systemd-resolve --status

# DNS lookup tools
host example.com
host -t MX example.com
nslookup example.com
dig example.com
dig example.com MX
dig example.com @8.8.8.8       # query specific DNS server
dig +short example.com          # just IP
dig +trace example.com          # trace resolution path
dig -x 1.2.3.4                 # reverse lookup

# ─── Network Statistics ───────────────────────────────────────
ss -tulpn                       # socket statistics
ss -s                           # summary
netstat -tulpn                  # old command
netstat -an | grep ESTABLISHED

# ─── nmcli (NetworkManager) ───────────────────────────────────
nmcli device status
nmcli connection show
nmcli connection up "connection-name"
nmcli device wifi list
nmcli device wifi connect "SSID" password "password"
```

---

## 16.2 Testing & Diagnostics

```bash
# ─── Connectivity ─────────────────────────────────────────────
ping -c 4 8.8.8.8              # ping 4 times
ping -i 0.5 host               # interval 0.5s
ping -f host                   # flood ping (root)
ping6 ::1                      # IPv6 ping

# Traceroute
traceroute google.com
tracepath google.com            # no root required
mtr google.com                  # real-time traceroute

# ─── Port Testing ─────────────────────────────────────────────
# nc (netcat) - Swiss Army knife of networking
nc -zv host 80                  # test port 80 TCP
nc -zuv host 53                 # test port 53 UDP
nc -zv host 80-90               # port range

# Bash built-in TCP/UDP
# Check if port open (no nc needed)
timeout 3 bash -c 'cat < /dev/null > /dev/tcp/host/80' 2>/dev/null \
    && echo "Port 80 open" || echo "Port 80 closed"

check_port() {
    local host=$1
    local port=$2
    local timeout=${3:-3}
    
    timeout "$timeout" bash -c "cat < /dev/null > /dev/tcp/$host/$port" 2>/dev/null
}

# nmap (network mapper)
nmap -p 80,443 example.com     # scan specific ports
nmap -p 1-1000 host            # scan port range
nmap -p- host                  # all ports
nmap -sV host                  # service version detection
nmap -O host                   # OS detection (root)
nmap -sn 192.168.1.0/24       # ping scan (host discovery)

# curl for HTTP testing
curl -I https://example.com    # headers only
curl -v https://example.com    # verbose
curl -s -o /dev/null -w "%{http_code}\n" https://example.com  # status code
curl --connect-timeout 5 https://example.com

# wget
wget -q --spider https://example.com  # check URL
```

---

## 16.3 curl & wget Advanced

```bash
# ─── curl ─────────────────────────────────────────────────────

# HTTP Methods
curl https://api.example.com                           # GET
curl -X POST https://api.example.com/data              # POST
curl -X PUT https://api.example.com/data/1             # PUT
curl -X DELETE https://api.example.com/data/1          # DELETE
curl -X PATCH https://api.example.com/data/1           # PATCH

# Headers
curl -H "Content-Type: application/json" URL
curl -H "Authorization: Bearer $TOKEN" URL
curl -H "X-Custom: value" URL

# Request body
curl -d "param1=value1&param2=value2" URL             # form data
curl -d @data.txt URL                                  # from file
curl -d '{"key":"value"}' -H "Content-Type: application/json" URL  # JSON

# Authentication
curl -u user:password URL                              # basic auth
curl -H "Authorization: Bearer $TOKEN" URL            # bearer token

# File operations
curl -O https://example.com/file.tar.gz               # download
curl -o myfile.tar.gz https://example.com/file.tar.gz # save as
curl -L URL                                            # follow redirects
curl -C - -O URL                                       # resume download

# SSL/TLS
curl -k URL                                            # ignore cert errors
curl --cacert /path/to/ca.pem URL                     # custom CA
curl --cert client.pem --key key.pem URL              # client cert

# Proxy
curl -x http://proxy:3128 URL
curl --socks5 127.0.0.1:9050 URL                      # SOCKS5 (Tor)

# Timing & info
curl -w "@curl-format.txt" -s -o /dev/null URL
# curl-format.txt:
cat > /tmp/curl-format.txt << 'EOF'
    time_namelookup:  %{time_namelookup}s\n
    time_connect:     %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
    time_starttransfer: %{time_starttransfer}s\n
    time_total:       %{time_total}s\n
    speed_download:   %{speed_download} bytes/s\n
    http_code:        %{http_code}\n
EOF

# Retry
curl --retry 3 --retry-delay 2 --retry-max-time 60 URL

# ─── wget ─────────────────────────────────────────────────────
wget URL                                               # download
wget -O filename URL                                   # save as
wget -q URL                                            # quiet
wget --no-check-certificate URL                        # skip SSL
wget -c URL                                            # continue/resume
wget -b URL                                            # background
wget -r -np -k https://example.com/                   # recursive mirror
wget -m https://example.com/                          # mirror website

# Rate limit
wget --limit-rate=1m URL                               # limit to 1MB/s
```

---

## 16.4 SSH

```bash
# ─── Basic SSH ────────────────────────────────────────────────
ssh user@host
ssh -p 2222 user@host           # non-standard port
ssh -i ~/.ssh/key user@host     # specific key
ssh -v user@host                # verbose (debug)

# Run remote command
ssh user@host "ls -la /tmp"
ssh user@host "bash -s" < local_script.sh
ssh user@host << 'EOF'
echo "Running on remote"
pwd
hostname
EOF

# ─── Key Management ───────────────────────────────────────────
# Generate key
ssh-keygen -t ed25519 -C "email@example.com"         # ed25519 (recommended)
ssh-keygen -t rsa -b 4096 -C "email@example.com"      # RSA 4096

# Copy key to server
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/mykey.pub user@host

# Manual method
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Key agent
eval "$(ssh-agent -s)"         # start agent
ssh-add ~/.ssh/id_ed25519      # add key
ssh-add -l                     # list loaded keys
ssh-add -d ~/.ssh/id_ed25519   # remove key

# ─── SSH Config (~/.ssh/config) ───────────────────────────────
cat > ~/.ssh/config << 'EOF'
# Default settings
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes

# Jump host
Host bastion
    HostName 1.2.3.4
    User admin
    IdentityFile ~/.ssh/bastion_key
    Port 22

# Private server via jump host
Host private-server
    HostName 10.0.0.10
    User deploy
    IdentityFile ~/.ssh/deploy_key
    ProxyJump bastion

# Development server
Host dev
    HostName dev.example.com
    User developer
    Port 2222
    IdentityFile ~/.ssh/dev_key
    LocalForward 8080 localhost:8080
EOF

# Now can do:
ssh dev              # connects with all settings

# ─── SSH Tunneling ────────────────────────────────────────────
# Local port forwarding
ssh -L 8080:target_host:80 user@jumphost
# Access localhost:8080 → jumphost → target_host:80

# Remote port forwarding (expose local to remote)
ssh -R 9090:localhost:3000 user@remote
# remote:9090 → localhost:3000

# Dynamic SOCKS proxy
ssh -D 9050 user@server
# Use with curl --socks5 127.0.0.1:9050

# Keep tunnel alive
ssh -fNL 8080:db.internal:5432 user@bastion
# -f = background, -N = no command, -L = forward

# ─── SCP / SFTP / rsync ───────────────────────────────────────
# scp
scp file.txt user@host:/path/
scp user@host:/path/file.txt .
scp -r dir/ user@host:/path/
scp -P 2222 file user@host:/path/

# rsync (preferred - faster, resumable)
rsync -av source/ user@host:/dest/
rsync -av --delete source/ user@host:/dest/      # mirror (delete extra)
rsync -av -e "ssh -p 2222" source/ user@host:/dest/
rsync -av --progress source/ dest/               # local with progress
rsync -av --exclude="*.log" --exclude=".git" source/ dest/

# sftp
sftp user@host
# Interactive commands: ls, cd, get, put, mkdir, rm, quit
sftp> get remote_file
sftp> put local_file
sftp> ls -la
```

---

## 16.5 Networking Script Examples

```bash
#!/bin/bash
# network_health.sh - Network health check

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

pass() { echo -e "${GREEN}✓${NC} $1"; }
fail() { echo -e "${RED}✗${NC} $1"; }
warn() { echo -e "${YELLOW}!${NC} $1"; }

# Check internet connectivity
check_internet() {
    echo "=== Internet Connectivity ==="
    
    local hosts=("8.8.8.8" "1.1.1.1" "208.67.222.222")
    local ok=0
    
    for host in "${hosts[@]}"; do
        if ping -c 1 -W 2 "$host" &>/dev/null; then
            pass "Can reach $host"
            (( ok++ ))
        else
            fail "Cannot reach $host"
        fi
    done
    
    (( ok > 0 )) && pass "Internet: OK" || fail "Internet: FAILED"
}

# Check DNS
check_dns() {
    echo ""
    echo "=== DNS Resolution ==="
    
    local domains=("google.com" "github.com" "cloudflare.com")
    
    for domain in "${domains[@]}"; do
        if dig +short "$domain" &>/dev/null; then
            local ip
            ip=$(dig +short "$domain" | head -1)
            pass "DNS $domain → $ip"
        else
            fail "DNS resolution failed: $domain"
        fi
    done
}

# Check local network
check_local() {
    echo ""
    echo "=== Local Network ==="
    
    local interface
    interface=$(ip route get 8.8.8.8 2>/dev/null | awk '{print $5; exit}')
    local gateway
    gateway=$(ip route show default | awk '/default/ {print $3}')
    local ip
    ip=$(ip addr show "$interface" 2>/dev/null | awk '/inet / {print $2; exit}')
    
    pass "Interface: $interface"
    pass "IP: $ip"
    
    if [[ -n "$gateway" ]] && ping -c 1 -W 2 "$gateway" &>/dev/null; then
        pass "Gateway $gateway: reachable"
    else
        fail "Gateway $gateway: unreachable"
    fi
}

# Check ports
check_ports() {
    echo ""
    echo "=== Port Connectivity ==="
    
    local -A services=(
        ["google.com:443"]="Google HTTPS"
        ["github.com:22"]="GitHub SSH"
        ["8.8.8.8:53"]="Google DNS"
    )
    
    for endpoint in "${!services[@]}"; do
        local host="${endpoint%%:*}"
        local port="${endpoint##*:}"
        local name="${services[$endpoint]}"
        
        if timeout 3 bash -c "cat < /dev/null > /dev/tcp/$host/$port" 2>/dev/null; then
            pass "$name ($endpoint)"
        else
            fail "$name ($endpoint)"
        fi
    done
}

# Show network stats
show_stats() {
    echo ""
    echo "=== Network Statistics ==="
    
    echo "Active connections: $(ss -t state established | wc -l)"
    echo "Listening ports:"
    ss -tulpn | awk 'NR>1 {print "  " $1 " " $5}' | sort -u
}

check_internet
check_dns
check_local
check_ports
show_stats

echo ""
echo "Network health check complete: $(date)"
```

---

## 16.6 Port Scanner Script

```bash
#!/bin/bash
# port_scan.sh - Simple port scanner

usage() {
    echo "Usage: $0 <host> [start_port] [end_port]"
    echo "Example: $0 192.168.1.1 1 1000"
    exit 1
}

[[ $# -lt 1 ]] && usage

HOST=$1
START=${2:-1}
END=${3:-1024}
TIMEOUT=1
OPEN_PORTS=()

echo "Scanning $HOST ports $START-$END..."
echo "Started: $(date)"
echo ""

scan_port() {
    local port=$1
    if timeout "$TIMEOUT" bash -c "cat < /dev/null > /dev/tcp/$HOST/$port" 2>/dev/null; then
        # Try to get service name
        local service
        service=$(getent services "$port/tcp" 2>/dev/null | awk '{print $1}')
        OPEN_PORTS+=("$port")
        printf "%-6s OPEN   %s\n" "$port" "${service:-unknown}"
    fi
}

# Parallel scanning
MAX_JOBS=50
for ((port=START; port<=END; port++)); do
    scan_port "$port" &
    
    # Limit concurrent jobs
    while (( $(jobs -r | wc -l) >= MAX_JOBS )); do
        wait -n 2>/dev/null || true
    done
done

wait

echo ""
echo "Scan complete: $(date)"
echo "Open ports found: ${#OPEN_PORTS[@]}"
```

---

## 16.7 Exercises

### Exercise 1: Network Monitor
สร้าง script ที่ monitor network:
- Check connectivity ทุก 5 นาที
- Log outages
- Alert เมื่อ latency สูง

### Exercise 2: SSH Config Manager
สร้าง tool สำหรับ manage ~/.ssh/config:
- Add/remove hosts
- List connections
- Test connectivity

### Exercise 3: Bandwidth Monitor
วัด network bandwidth:
- Upload/download speed
- Per-interface statistics
- Historical graph

---

## สรุป Part 16

✅ Network configuration (ip, ifconfig)  
✅ DNS tools (dig, host, nslookup)  
✅ Connectivity testing (ping, traceroute, nc)  
✅ curl & wget advanced usage  
✅ SSH (keys, tunneling, config, rsync)  
✅ Network health check script  
✅ Port scanner script  

---

**→ Part 17: Cron Jobs & Task Scheduling**
