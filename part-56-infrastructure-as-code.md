# Part 56: Infrastructure as Code with Bash
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 56.1 Configuration Management

```bash
#!/bin/bash
# config_mgmt.sh - Declarative configuration management

CONFIG_DIR="${CONFIG_DIR:-/etc/infra}"
STATE_DIR="${STATE_DIR:-/var/lib/infra/state}"
INVENTORY_FILE="${INVENTORY_FILE:-$CONFIG_DIR/inventory.conf}"

# ─── Resource Declaration ────────────────────────────────────
declare -A RESOURCES=()
declare -A RESOURCE_STATES=()

resource_declare() {
    local type=$1 name=$2
    shift 2
    local properties="$*"

    RESOURCES["${type}:${name}"]="$properties"
}

resource_get_state() {
    local type=$1 name=$2
    local state_file="$STATE_DIR/${type}_${name}.state"

    [[ -f "$state_file" ]] && cat "$state_file" || echo "absent"
}

resource_set_state() {
    local type=$1 name=$2 state=$3
    local state_file="$STATE_DIR/${type}_${name}.state"

    mkdir -p "$STATE_DIR"
    echo "$state" > "$state_file"
}

# ─── File Resource ──────────────────────────────────────────────
resource_file_apply() {
    local name=$1
    shift
    local -A props=()

    while [[ $# -gt 0 ]]; do
        local key="${1%%=*}" value="${1#*=}"
        props["$key"]="$value"
        shift
    done

    local path="${props[path]:-$name}"
    local content="${props[content]:-}"
    local owner="${props[owner]:-root}"
    local group="${props[group]:-root}"
    local mode="${props[mode]:-644}"
    local ensure="${props[ensure]:-present}"

    if [[ "$ensure" == "absent" ]]; then
        if [[ -f "$path" ]]; then
            rm -f "$path"
            echo "REMOVED file: $path"
            resource_set_state "file" "$name" "absent"
        fi
        return 0
    fi

    local changed=false

    # Create parent directories
    mkdir -p "$(dirname "$path")"

    # Write content if different
    if [[ ! -f "$path" ]] || [[ "$(cat "$path")" != "$content" ]]; then
        echo "$content" > "$path"
        changed=true
    fi

    # Set ownership
    local current_owner current_group
    current_owner=$(stat -c %U "$path" 2>/dev/null)
    current_group=$(stat -c %G "$path" 2>/dev/null)

    if [[ "$current_owner" != "$owner" ]] || [[ "$current_group" != "$group" ]]; then
        chown "${owner}:${group}" "$path"
        changed=true
    fi

    # Set permissions
    local current_mode
    current_mode=$(stat -c %a "$path" 2>/dev/null)
    if [[ "$current_mode" != "$mode" ]]; then
        chmod "$mode" "$path"
        changed=true
    fi

    if $changed; then
        echo "UPDATED file: $path"
        resource_set_state "file" "$name" "present"
    else
        echo "OK file: $path (no changes)"
    fi
}

# ─── Package Resource ───────────────────────────────────────────
resource_package_apply() {
    local name=$1
    shift
    local ensure="${1:-present}"

    local pkg_mgr
    if command -v apt-get &>/dev/null; then
        pkg_mgr="apt"
    elif command -v yum &>/dev/null; then
        pkg_mgr="yum"
    elif command -v apk &>/dev/null; then
        pkg_mgr="apk"
    else
        echo "ERROR: No package manager found"
        return 1
    fi

    local is_installed=false
    case "$pkg_mgr" in
        apt) dpkg -l "$name" &>/dev/null 2>&1 && is_installed=true ;;
        yum) rpm -q "$name" &>/dev/null && is_installed=true ;;
        apk) apk info -e "$name" &>/dev/null && is_installed=true ;;
    esac

    if [[ "$ensure" == "present" ]] && ! $is_installed; then
        case "$pkg_mgr" in
            apt) DEBIAN_FRONTEND=noninteractive apt-get install -y "$name" ;;
            yum) yum install -y "$name" ;;
            apk) apk add "$name" ;;
        esac
        echo "INSTALLED package: $name"
        resource_set_state "package" "$name" "present"
    elif [[ "$ensure" == "absent" ]] && $is_installed; then
        case "$pkg_mgr" in
            apt) apt-get remove -y "$name" ;;
            yum) yum remove -y "$name" ;;
            apk) apk del "$name" ;;
        esac
        echo "REMOVED package: $name"
        resource_set_state "package" "$name" "absent"
    else
        echo "OK package: $name ($ensure)"
    fi
}

# ─── Service Resource ───────────────────────────────────────────
resource_service_apply() {
    local name=$1
    shift
    local ensure="${1:-running}"
    local enable="${2:-true}"

    local is_active=false
    local is_enabled=false

    if command -v systemctl &>/dev/null; then
        systemctl is-active "$name" &>/dev/null && is_active=true
        systemctl is-enabled "$name" &>/dev/null && is_enabled=true
    fi

    case "$ensure" in
        running)
            if ! $is_active; then
                systemctl start "$name"
                echo "STARTED service: $name"
            else
                echo "OK service: $name (running)"
            fi
            ;;
        stopped)
            if $is_active; then
                systemctl stop "$name"
                echo "STOPPED service: $name"
            else
                echo "OK service: $name (stopped)"
            fi
            ;;
    esac

    if [[ "$enable" == "true" ]] && ! $is_enabled; then
        systemctl enable "$name"
        echo "ENABLED service: $name"
    elif [[ "$enable" == "false" ]] && $is_enabled; then
        systemctl disable "$name"
        echo "DISABLED service: $name"
    fi

    resource_set_state "service" "$name" "$ensure"
}

# ─── Apply All Resources ───────────────────────────────────────
apply_resources() {
    echo "=== Applying Infrastructure State ==="
    local changed=0 ok=0 failed=0

    for resource_key in "${!RESOURCES[@]}"; do
        local type="${resource_key%%:*}"
        local name="${resource_key#*:}"
        local properties="${RESOURCES[$resource_key]}"

        case "$type" in
            file)    resource_file_apply "$name" $properties && (( ok++ )) || (( failed++ )) ;;
            package) resource_package_apply "$name" $properties && (( ok++ )) || (( failed++ )) ;;
            service) resource_service_apply "$name" $properties && (( ok++ )) || (( failed++ )) ;;
        esac
    done

    echo ""
    echo "=== Summary: ok=$ok failed=$failed ==="
}
```

---

## 56.2 Inventory Management

```bash
#!/bin/bash
# inventory.sh - Host inventory and grouping

declare -A HOSTS=()
declare -A GROUPS=()
declare -A HOST_VARS=()

# ─── Inventory Parser ──────────────────────────────────────────
parse_inventory() {
    local inventory_file=$1
    local current_group="ungrouped"

    while IFS= read -r line; do
        # Skip comments and empty lines
        [[ "$line" =~ ^[[:space:]]*# ]] && continue
        [[ -z "${line// /}" ]] && continue

        # Group header
        if [[ "$line" =~ ^\[(.+)\]$ ]]; then
            current_group="${BASH_REMATCH[1]}"
            GROUPS["$current_group"]="${GROUPS[$current_group]:-}"
            continue
        fi

        # Host entry with optional vars
        local host="${line%% *}"
        local vars="${line#* }"
        [[ "$vars" == "$host" ]] && vars=""

        HOSTS["$host"]="$current_group"
        GROUPS["$current_group"]+" $host"

        # Parse inline vars
        if [[ -n "$vars" ]]; then
            HOST_VARS["$host"]="$vars"
        fi
    done < "$inventory_file"
}

# ─── Host Query ─────────────────────────────────────────────────get_hosts_in_group() {
    local group=$1
    echo "${GROUPS[$group]:-}" | tr ' ' '\n' | grep -v '^$'
}

get_host_var() {
    local host=$1 var_name=$2

    local vars="${HOST_VARS[$host]:-}"
    echo "$vars" | tr ' ' '\n' | grep "^${var_name}=" | cut -d= -f2-
}

list_groups() {
    for group in "${!GROUPS[@]}"; do
        local count
        count=$(echo "${GROUPS[$group]}" | tr ' ' '\n' | grep -vc '^$')
        printf "%-30s %d hosts\n" "$group" "$count"
    done
}

# ─── Dynamic Inventory ────────────────────────────────────────
inventory_from_aws_tags() {
    local tag_key=$1 tag_value=$2

    if command -v aws &>/dev/null; then
        aws ec2 describe-instances \
            --filters "Name=tag:${tag_key},Values=${tag_value}" \
                      "Name=instance-state-name,Values=running" \
            --query 'Reservations[].Instances[].[PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]' \
            --output text 2>/dev/null | while read -r ip name; do
                echo "${name:-$ip} ansible_host=$ip"
            done
    fi
}

inventory_to_json() {
    echo "{"
    echo "  \"_meta\": {\"hostvars\": {"

    local first_host=true
    for host in "${!HOSTS[@]}"; do
        $first_host || echo ","
        first_host=false
        printf '    "%s": {' "$host"

        local vars="${HOST_VARS[$host]:-}"
        local first_var=true
        while IFS= read -r var; do
            [[ -z "$var" ]] && continue
            local key="${var%%=*}" value="${var#*=}"
            $first_var || echo ","
            first_var=false
            printf '"%s": "%s"' "$key" "$value"
        done <<< "$(echo "$vars" | tr ' ' '\n')"

        printf '}'
    done

    echo ""
    echo "  }},"

    local first_group=true
    for group in "${!GROUPS[@]}"; do
        $first_group || echo ","
        first_group=false
        printf '  "%s": {"hosts": [' "$group"

        local first_h=true
        while IFS= read -r h; do
            [[ -z "$h" ]] && continue
            $first_h || echo ","
            first_h=false
            printf '"%s"' "$h"
        done <<< "$(get_hosts_in_group "$group")"

        printf ']}'
    done

    echo ""
    echo "}"
}
```

---

## 56.3 Template Engine

```bash
#!/bin/bash
# templates.sh - Configuration template rendering

TEMPLATE_DIR="${TEMPLATE_DIR:-/etc/infra/templates}"

# ─── Variable Substitution ───────────────────────────────────
render_template() {
    local template=$1
    shift
    local -A vars=()

    while [[ $# -gt 0 ]]; do
        vars["${1%%=*}"]="${1#*=}"
        shift
    done

    local output="$template"

    for key in "${!vars[@]}"; do
        local value="${vars[$key]}"
        output="${output//\{\{ $key \}\}/$value}"
        output="${output//\{\{$key\}\}/$value}"
    done

    echo "$output"
}

render_template_file() {
    local template_file=$1
    shift
    local -A vars=()

    while [[ $# -gt 0 ]]; do
        vars["${1%%=*}"]="${1#*=}"
        shift
    done

    local content
    content=$(cat "$template_file")

    for key in "${!vars[@]}"; do
        local value="${vars[$key]}"
        content="${content//\{\{ $key \}\}/$value}"
        content="${content//\{\{$key\}\}/$value}"
    done

    echo "$content"
}

# ─── Nginx Config Template ───────────────────────────────────
generate_nginx_config() {
    local server_name=$1
    local upstream_hosts=$2
    local ssl_cert=${3:-}
    local ssl_key=${4:-}

    local upstream_block=""
    IFS=',' read -ra hosts <<< "$upstream_hosts"
    for host in "${hosts[@]}"; do
        upstream_block+="    server ${host};\n"
    done

    printf 'upstream backend_%s {\n    least_conn;\n%s}\n\nserver {\n    listen 80;\n    server_name %s;\n\n    location / {\n        proxy_pass http://backend_%s;\n        proxy_set_header Host $host;\n        proxy_set_header X-Real-IP $remote_addr;\n        proxy_connect_timeout 5s;\n        proxy_read_timeout 30s;\n    }\n\n    location /health {\n        access_log off;\n        return 200 ok;\n    }\n}\n' \
        "${server_name//./_}" "$(printf '%b' "$upstream_block")" \
        "$server_name" "${server_name//./_}"
}
```

---

## 56.4 Drift Detection

```bash
#!/bin/bash
# drift.sh - Infrastructure drift detection

BASELINE_DIR="${BASELINE_DIR:-/var/lib/infra/baseline}"
REPORT_DIR="${REPORT_DIR:-/var/lib/infra/reports}"

# ─── Baseline Capture ──────────────────────────────────────────
capture_baseline() {
    local name=${1:-$(hostname)}

    mkdir -p "$BASELINE_DIR"
    local baseline_file="$BASELINE_DIR/${name}.baseline"
    local timestamp
    timestamp=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

    {
        echo "# Baseline captured: $timestamp"
        echo "# Host: $name"
        echo ""

        echo "[packages]"
        if command -v dpkg &>/dev/null; then
            dpkg -l | awk '/^ii/{print $2"="$3}' | sort
        elif command -v rpm &>/dev/null; then
            rpm -qa --qf '%{NAME}=%{VERSION}-%{RELEASE}\n' | sort
        fi
        echo ""

        echo "[services]"
        if command -v systemctl &>/dev/null; then
            systemctl list-units --type=service --state=active --no-legend | \
                awk '{print $1}' | sort
        fi
        echo ""

        echo "[files]"
        local config_paths=("/etc/ssh/sshd_config" "/etc/hosts" "/etc/fstab" \
                            "/etc/passwd" "/etc/sudoers")
        for f in "${config_paths[@]}"; do
            [[ -f "$f" ]] && echo "${f}:$(sha256sum "$f" | cut -d' ' -f1)"
        done
        echo ""

        echo "[users]"
        awk -F: '$3 >= 1000 && $3 < 65534 {print $1":"$3":"$6}' /etc/passwd | sort
        echo ""

        echo "[listening_ports]"
        if command -v ss &>/dev/null; then
            ss -tlnp 2>/dev/null | awk 'NR>1{print $4}' | sort -u
        fi

    } > "$baseline_file"

    echo "Baseline captured: $baseline_file"
}

detect_drift() {
    local name=${1:-$(hostname)}
    local baseline_file="$BASELINE_DIR/${name}.baseline"

    [[ ! -f "$baseline_file" ]] && {
        echo "No baseline found for $name"
        return 1
    }

    mkdir -p "$REPORT_DIR"
    local report_file="$REPORT_DIR/${name}_$(date +%Y%m%d_%H%M%S).drift"
    local drift_found=false

    {
        echo "=== Drift Report: $name ==="
        echo "Baseline: $baseline_file"
        echo "Timestamp: $(date -u '+%Y-%m-%dT%H:%M:%SZ')"
        echo ""

        local in_files=false
        while IFS= read -r line; do
            [[ "$line" == "[files]" ]] && in_files=true && continue
            [[ "$line" =~ ^\[ ]] && in_files=false
            $in_files || continue
            [[ -z "$line" ]] && continue

            local path="${line%%:*}"
            local expected_hash="${line#*:}"

            if [[ ! -f "$path" ]]; then
                echo "DRIFT [file_missing]: $path"
                drift_found=true
            else
                local current_hash
                current_hash=$(sha256sum "$path" | cut -d' ' -f1)
                if [[ "$current_hash" != "$expected_hash" ]]; then
                    echo "DRIFT [file_changed]: $path"
                    drift_found=true
                fi
            fi
        done < "$baseline_file"

        $drift_found || echo "No drift detected."
    } | tee "$report_file"

    $drift_found && return 1 || return 0
}
```

---

## 56.5 Exercises

### Exercise 1: Full IaC Framework
สร้าง IaC framework ที่:
- Declare resources in config files
- Apply idempotently
- Track state changes
- Generate compliance reports

### Exercise 2: Multi-Environment Promotion
สร้าง promotion pipeline ที่:
- dev → staging → production
- Environment-specific variable substitution
- Approval gates between stages

### Exercise 3: Drift Remediation
สร้าง auto-remediation system ที่:
- Detect drift hourly
- Auto-remediate known drifts
- Alert on unknown drifts
- Maintain audit trail

---

## สรุป Part 56

✅ Resource management: file, package, service resources
✅ Idempotent apply with state tracking
✅ Inventory parser with group support and inline vars
✅ Dynamic inventory from AWS tags
✅ Template engine with variable substitution
✅ Nginx config template generator
✅ Infrastructure baseline capture and drift detection
✅ File integrity checking via SHA256

---

**→ Part 57: Container Orchestration with Bash**
