# Part 26: SSH Automation & Remote Management
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 26.1 SSH Key Management

```bash
# ─── Generate Keys ────────────────────────────────────────────
# Ed25519 (recommended - fast, secure)
ssh-keygen -t ed25519 -C "deploy@myserver" -f ~/.ssh/deploy_key

# RSA 4096 (for legacy compatibility)
ssh-keygen -t rsa -b 4096 -C "user@host" -f ~/.ssh/id_rsa

# No passphrase (for automation)
ssh-keygen -t ed25519 -N "" -f ~/.ssh/automation_key

# With custom options
ssh-keygen -t ed25519 \
    -C "deploy-$(date +%Y-%m-%d)" \
    -f ~/.ssh/deploy_key \
    -a 100  # key derivation rounds (security)

# ─── Key Distribution ─────────────────────────────────────────
# Manual copy
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# With ssh-copy-id
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 user@host

# Distribute to multiple hosts
distribute_key() {
    local pub_key=$1
    local hosts=("${@:2}")
    
    for host in "${hosts[@]}"; do
        echo "Copying key to $host..."
        ssh-copy-id -i "$pub_key" "$host" 2>/dev/null && \
            echo "✓ $host" || echo "✗ $host (failed)"
    done
}

hosts=(server1 server2 server3)
distribute_key ~/.ssh/deploy_key.pub "${hosts[@]}"

# ─── authorized_keys management ───────────────────────────────
# Format: [options] key_type key_data comment
# Options can restrict what key can do:
cat >> ~/.ssh/authorized_keys << 'EOF'
# Restrict to specific commands
command="/usr/local/bin/backup.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... backup-key

# Restrict to specific IP
from="192.168.1.100",no-pty ssh-ed25519 AAAA... admin-from-office

# Allow only port forwarding (no shell)
no-pty,no-agent-forwarding ssh-ed25519 AAAA... tunnel-only
EOF
```

---

## 26.2 Running Commands on Remote Hosts

```bash
#!/bin/bash
# remote_management.sh

# ─── Single host ──────────────────────────────────────────────
# Run command
ssh user@host "uname -a"
ssh user@host "df -h; free -h; uptime"

# Pipe data
cat local_file.txt | ssh user@host "cat > remote_file.txt"
ssh user@host "cat remote_file.txt" | grep "pattern"

# Run script
ssh user@host "bash -s" < local_script.sh
ssh user@host < local_script.sh  # same thing

# Send and run
scp deploy.sh user@host:/tmp/
ssh user@host "chmod +x /tmp/deploy.sh && /tmp/deploy.sh"

# Heredoc
ssh user@host << 'ENDSSH'
#!/bin/bash
echo "Running on remote: $(hostname)"
cd /var/www/myapp
git pull
npm install
pm2 restart myapp
echo "Deploy complete"
ENDSSH

# ─── Multi-host operations ────────────────────────────────────
HOSTS=(web1 web2 web3 db1 db2)

# Sequential
for host in "${HOSTS[@]}"; do
    echo "=== $host ==="
    ssh "$host" "uptime; df -h /" || echo "Failed: $host"
done

# Parallel
run_parallel() {
    local cmd=$1
    shift
    local hosts=("$@")
    local pids=()
    
    for host in "${hosts[@]}"; do
        ssh -o ConnectTimeout=10 "$host" "$cmd" &
        pids+=("$!:$host")
    done
    
    # Wait and collect results
    for pidhost in "${pids[@]}"; do
        local pid=${pidhost%%:*}
        local host=${pidhost##*:}
        
        if wait "$pid"; then
            echo "✓ $host"
        else
            echo "✗ $host (exit: $?)"
        fi
    done
}

run_parallel "sudo apt-get update -q" "${HOSTS[@]}"

# With timeout and parallel output
run_ssh_parallel() {
    local command=$1
    shift
    local hosts=("$@")
    
    printf '%s\n' "${hosts[@]}" | \
        xargs -P "${#hosts[@]}" -I{} \
        bash -c 'echo "=== {} ==="; ssh -o ConnectTimeout=10 {} "'"$command"'" 2>&1 | sed "s/^/  /"; echo ""'
}

run_ssh_parallel "df -h / && free -h" "${HOSTS[@]}"
```

---

## 26.3 SSH Configuration & Multiplexing

```bash
# ─── Advanced SSH Config ──────────────────────────────────────
cat > ~/.ssh/config << 'EOF'
# Global defaults
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
    
    # Multiplexing (reuse connections)
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h:%p
    ControlPersist 10m
    
    # Compression
    Compression yes
    
    # Connection timeout
    ConnectTimeout 10

# Jump host / bastion
Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/bastion_key
    ForwardAgent yes

# Internal hosts through bastion
Host *.internal
    User deploy
    IdentityFile ~/.ssh/deploy_key
    ProxyJump bastion

# Specific server
Host prod-web1
    HostName 10.0.1.10
    User ubuntu
    IdentityFile ~/.ssh/prod_key

# Development environments
Host dev-*
    User vagrant
    IdentityFile ~/.vagrant.d/insecure_private_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
EOF

# Create socket directory
mkdir -p ~/.ssh/sockets

# ─── Multiplexing ─────────────────────────────────────────────
# First connection creates master socket
ssh -M -S ~/.ssh/sockets/myhost.sock user@host -f -N

# Subsequent connections reuse it (very fast)
ssh -S ~/.ssh/sockets/myhost.sock user@host "ls"

# Close master connection
ssh -S ~/.ssh/sockets/myhost.sock -O exit user@host

# Check connection status
ssh -S ~/.ssh/sockets/myhost.sock -O check user@host

# ─── SSH Agent Forwarding ─────────────────────────────────────
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l  # verify loaded

# Forward agent to remote (allows hopping to other servers)
ssh -A user@bastion
# On bastion, can now ssh to internal hosts using local keys

# ─── Port Forwarding ──────────────────────────────────────────
# Local: forward local port to remote destination
ssh -L 5432:db.internal:5432 user@bastion
# localhost:5432 → bastion → db.internal:5432

# Remote: expose local port on remote server
ssh -R 8080:localhost:3000 user@remote
# remote:8080 → localhost:3000

# Dynamic SOCKS proxy
ssh -D 9050 user@server
# Use in browser: SOCKS5 proxy at 127.0.0.1:9050

# Background (-f -N)
ssh -fNL 8080:db.internal:5432 user@bastion
ssh -fND 9050 user@server

# Kill tunnel
kill $(lsof -t -i:8080)
```

---

## 26.4 Ansible-style Automation Script

```bash
#!/bin/bash
# cluster_manager.sh - Manage multiple servers

set -euo pipefail

# Config file: hosts.conf
# Format: hostname user:port:key
# web1 deploy:22:~/.ssh/deploy_key
# web2 deploy:22:~/.ssh/deploy_key
# db1  admin:2222:~/.ssh/admin_key

HOSTS_FILE="${HOSTS_FILE:-hosts.conf}"
MAX_PARALLEL=${MAX_PARALLEL:-5}
TIMEOUT=${TIMEOUT:-30}

# Parse hosts file
parse_hosts() {
    local group=${1:-all}
    
    while IFS=' ' read -r hostname config; do
        [[ "$hostname" =~ ^# ]] && continue
        [[ -z "$hostname" ]] && continue
        
        local user port key
        IFS=: read -r user port key <<< "$config"
        
        user=${user:-deploy}
        port=${port:-22}
        key=${key:-~/.ssh/id_ed25519}
        
        echo "$hostname $user $port $key"
    done < "$HOSTS_FILE"
}

# Run command on single host
run_on_host() {
    local hostname=$1
    local user=$2
    local port=$3
    local key=$4
    local cmd=$5
    
    ssh \
        -o ConnectTimeout="$TIMEOUT" \
        -o StrictHostKeyChecking=no \
        -o BatchMode=yes \
        -i "$key" \
        -p "$port" \
        "${user}@${hostname}" "$cmd" 2>&1
}

# Run on all hosts (parallel)
run_all() {
    local command=$1
    local pids=()
    local results_dir
    results_dir=$(mktemp -d)
    
    while IFS=' ' read -r hostname user port key; do
        local result_file="$results_dir/${hostname}"
        
        (
            if run_on_host "$hostname" "$user" "$port" "$key" "$command" > "$result_file" 2>&1; then
                echo "SUCCESS"
            else
                echo "FAILED"
            fi
        ) &
        pids+=("$!:$hostname")
        
        # Limit parallelism
        while (( $(jobs -r | wc -l) >= MAX_PARALLEL )); do
            sleep 0.5
        done
    done < <(parse_hosts)
    
    # Wait and report
    local success=0 failed=0
    
    for pidhost in "${pids[@]}"; do
        local pid=${pidhost%%:*}
        local host=${pidhost##*:}
        
        wait "$pid"
        local result
        result=$(cat "$results_dir/$host" 2>/dev/null || echo "No output")
        
        echo "── $host ──"
        echo "$result"
        echo ""
    done
    
    rm -rf "$results_dir"
}

# Deploy file to all hosts
deploy_file() {
    local local_file=$1
    local remote_path=$2
    
    while IFS=' ' read -r hostname user port key; do
        echo "Deploying to $hostname..."
        scp \
            -o StrictHostKeyChecking=no \
            -i "$key" \
            -P "$port" \
            "$local_file" \
            "${user}@${hostname}:${remote_path}" && \
            echo "✓ $hostname" || echo "✗ $hostname"
    done < <(parse_hosts)
}

# Health check all hosts
health_check() {
    echo "=== Cluster Health Check ==="
    local healthy=0 unhealthy=0
    
    while IFS=' ' read -r hostname user port key; do
        local result
        result=$(run_on_host "$hostname" "$user" "$port" "$key" \
            "uptime; df -h /; free -h | head -2" 2>&1)
        
        if [[ $? -eq 0 ]]; then
            echo -e "\033[32m✓ $hostname\033[0m"
            echo "$result" | sed 's/^/  /'
            (( healthy++ ))
        else
            echo -e "\033[31m✗ $hostname (unreachable)\033[0m"
            (( unhealthy++ ))
        fi
        echo ""
    done < <(parse_hosts)
    
    echo "Summary: $healthy healthy, $unhealthy unhealthy"
}

# Main
case "${1:-help}" in
    run)   run_all "$2" ;;
    deploy) deploy_file "$2" "$3" ;;
    health) health_check ;;
    help)
        echo "Usage: $0 [run <cmd>|deploy <file> <path>|health]"
        ;;
esac
```

---

## 26.5 Exercises

### Exercise 1: Server Provisioner
สร้าง script ที่ provision new server:
- Copy SSH keys
- Install required packages
- Configure firewall
- Setup monitoring

### Exercise 2: Deployment Orchestrator
สร้าง rolling deployment:
- Deploy to servers one by one
- Health check after each
- Rollback if failure

### Exercise 3: SSH Tunnel Manager
สร้าง tool จัดการ SSH tunnels:
- Create named tunnels
- List active tunnels
- Auto-reconnect on failure
- Save/restore config

---

## สรุป Part 26

✅ SSH key generation and distribution  
✅ Running commands on remote hosts  
✅ Parallel SSH execution  
✅ SSH config (multiplexing, jump hosts)  
✅ Port forwarding (local, remote, dynamic)  
✅ Cluster management script  
✅ Health checks across multiple servers  

---

**→ Part 27: Git Automation & GitHub/GitLab API**
