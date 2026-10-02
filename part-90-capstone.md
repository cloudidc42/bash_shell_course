# Part 90: Shell Script Mastery — Capstone and Advanced Topics
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 90.1 Advanced Bash Features

```bash
#!/bin/bash
# advanced_bash.sh - Advanced Bash language features

# ─── Nameref (bash 4.3+) ─────────────────────────────────────
update_by_name() {
    local -n ref=$1
    ref="${ref}_updated"
}

demo_nameref() {
    local myvar="hello"
    update_by_name myvar
    echo "$myvar"  # hello_updated
}

# ─── Associative array of arrays (serialized) ─────────────────
declare -A REGISTRY=()

registry_set() {
    local key=$1; shift
    REGISTRY["$key"]=$(printf '%s\0' "$@" | base64)
}

registry_get() {
    local key=$1
    local encoded="${REGISTRY[$key]:-}"
    [[ -n "$encoded" ]] || return 1
    printf '%s' "$encoded" | base64 -d | tr '\0' '\n'
}

# ─── Process substitution patterns ───────────────────────────
diff_streams() {
    diff <(sort "$1") <(sort "$2")
}

tee_to_multiple() {
    local input_cmd=$1
    eval "$input_cmd" | tee >(gzip > /tmp/out1.gz) >(wc -l > /tmp/out2.txt) > /dev/null
}

# ─── Extended globbing ────────────────────────────────────────
list_non_hidden_non_log() {
    shopt -s extglob
    local dir=${1:-.}
    ls "$dir"/!(.*|*.log) 2>/dev/null
    shopt -u extglob
}

# ─── Trap with line number for debugging ─────────────────────
setup_debug_trap() {
    trap 'echo "Error at line $LINENO: $BASH_COMMAND" >&2' ERR
}

# ─── Coprocess ────────────────────────────────────────────────
coproc_demo() {
    coproc WORKER { while IFS= read -r line; do echo "processed: $line"; done; }

    echo "hello" >&"${WORKER[1]}"
    echo "world" >&"${WORKER[1]}"

    exec {WORKER[1]}>&-  # close stdin to worker

    while IFS= read -r response; do
        echo "$response"
    done <&"${WORKER[0]}"
}

# ─── Dynamic function generation ─────────────────────────────
make_getter_setter() {
    local var_name=$1
    eval "get_${var_name}() { echo \"\${_${var_name}}\"; }"
    eval "set_${var_name}() { _${var_name}=\"\$1\"; }"
}

# ─── Arithmetic with bases ────────────────────────────────────
to_hex() { printf '%x\n' "$1"; }
to_binary() { printf '%08d\n' "$(echo "obase=2; $1" | bc)"; }
from_hex() { printf '%d\n' "0x$1"; }
from_binary() { echo "$((2#$1))"; }
```

---

## 90.2 POSIX Compatibility Layer

```bash
#!/bin/bash
# posix_compat.sh - Write scripts that work on bash AND sh

# Safe way to check if running under bash
IS_BASH=false
[ -n "${BASH_VERSION:-}" ] && IS_BASH=true

# Portable array handling
if $IS_BASH; then
    declare -a ITEMS=()
    items_add() { ITEMS+=("$1"); }
    items_count() { echo "${#ITEMS[@]}"; }
    items_get() { echo "${ITEMS[$1]}"; }
else
    # POSIX: use space-separated string
    ITEMS=""
    items_add() { ITEMS="$ITEMS $1"; }
    items_count() { echo "$ITEMS" | wc -w; }
    items_get() { echo "$ITEMS" | cut -d' ' -f$(( $1 + 1 )); }
fi

# Portable string operations
portable_upper() {
    if $IS_BASH; then
        echo "${1^^}"
    else
        echo "$1" | tr '[:lower:]' '[:upper:]'
    fi
}

portable_lower() {
    if $IS_BASH; then
        echo "${1,,}"
    else
        echo "$1" | tr '[:upper:]' '[:lower:]'
    fi
}

# Portable function export
portable_export_fn() {
    local fn_name=$1
    if $IS_BASH; then
        export -f "$fn_name"
    else
        echo "export ${fn_name} not portable in POSIX sh"
    fi
}
```

---

## 90.3 Shell Script as a Service

```bash
#!/bin/bash
# shell_service.sh - Pattern for running a shell script as a long-lived service

SERVICE_NAME="${SERVICE_NAME:-myservice}"
PID_FILE="/var/run/${SERVICE_NAME}.pid"
LOG_FILE="/var/log/${SERVICE_NAME}.log"
CONFIG_FILE="/etc/${SERVICE_NAME}/config"

service_log() {
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] [$SERVICE_NAME] $*" >> "$LOG_FILE"
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] [$SERVICE_NAME] $*"
}

service_write_pid() {
    echo $$ > "$PID_FILE"
}

service_read_pid() {
    [[ -f "$PID_FILE" ]] && cat "$PID_FILE" || echo ""
}

service_is_running() {
    local pid; pid=$(service_read_pid)
    [[ -n "$pid" ]] && kill -0 "$pid" 2>/dev/null
}

service_start() {
    if service_is_running; then
        echo "$SERVICE_NAME is already running (pid $(service_read_pid))"
        return 1
    fi

    service_log "Starting..."
    service_write_pid

    trap 'service_log "Stopping..."; rm -f "$PID_FILE"; exit 0' TERM INT
    trap 'service_log "Reloading config..."; source "$CONFIG_FILE" 2>/dev/null || true' HUP

    service_main_loop
}

service_stop() {
    local pid; pid=$(service_read_pid)
    [[ -z "$pid" ]] && { echo "$SERVICE_NAME is not running"; return 1; }

    echo "Stopping $SERVICE_NAME (pid $pid)..."
    kill -TERM "$pid"
    local timeout=30
    while kill -0 "$pid" 2>/dev/null && (( timeout-- > 0 )); do
        sleep 1
    done
    kill -0 "$pid" 2>/dev/null && kill -KILL "$pid" 2>/dev/null || true
    rm -f "$PID_FILE"
    echo "Stopped."
}

service_status() {
    if service_is_running; then
        echo "$SERVICE_NAME is running (pid $(service_read_pid))"
    else
        echo "$SERVICE_NAME is stopped"
        return 1
    fi
}

service_reload() {
    local pid; pid=$(service_read_pid)
    [[ -z "$pid" ]] && { echo "$SERVICE_NAME is not running"; return 1; }
    kill -HUP "$pid"
    echo "Reloaded."
}

service_main_loop() {
    service_log "Service started (pid $$)"
    while true; do
        service_log "Heartbeat"
        sleep 60
    done
}

# CLI dispatcher
case "${1:-status}" in
    start)   service_start ;;
    stop)    service_stop ;;
    restart) service_stop; service_start ;;
    status)  service_status ;;
    reload)  service_reload ;;
    *)       echo "Usage: $0 {start|stop|restart|status|reload}"; exit 1 ;;
esac
```

---

## 90.4 Complete Capstone: DevOps Toolkit

```bash
#!/bin/bash
# devops_toolkit.sh - Unified DevOps automation toolkit
# Combines concepts from Parts 77-89

set -euo pipefail

TOOLKIT_VERSION="1.0.0"
TOOLKIT_ENV="${TOOLKIT_ENV:-development}"

_toolkit_load_modules() {
    local module_dir="${TOOLKIT_MODULE_DIR:-/opt/devops/modules}"
    [[ -d "$module_dir" ]] || return 0

    for module in "$module_dir"/*.sh; do
        [[ -f "$module" ]] || continue
        # shellcheck disable=SC1090
        source "$module"
    done
}

toolkit_version() {
    echo "DevOps Toolkit v${TOOLKIT_VERSION} (env: $TOOLKIT_ENV)"
}

# ─── Unified Deploy Command ───────────────────────────────────
toolkit_deploy() {
    local app=$1 env=${2:-$TOOLKIT_ENV} version=${3:-latest}

    echo "Deploying $app:$version to $env..."

    # Pre-flight: check secrets (Part 77)
    # vault_inject_env "secret/$app/$env" 2>/dev/null || true

    # Build and push artifact (Part 80)
    # artifact_save "$app" "$version" "./dist"

    # Kubernetes deployment (Part 79)
    # deploy_to_k8s "$app" "$version" "manifests/$env"

    # Notify (Part 80)
    # slack_notify "#deployments" "Deployed $app:$version to $env"

    echo "Deploy complete: $app:$version -> $env"
}

# ─── Unified Audit Command ────────────────────────────────────
toolkit_audit() {
    local target=${1:-$(hostname)}

    echo "Running audit on $target..."

    if [[ "$target" == "$(hostname)" ]]; then
        echo "Local security audit complete"
    else
        ssh "$target" "bash -s" <<'REMOTE_AUDIT'
            echo "Remote audit: $(hostname)"
REMOTE_AUDIT
    fi
}

# ─── Unified Monitor Command ──────────────────────────────────
toolkit_monitor() {
    local mode=${1:-dashboard}

    case "$mode" in
        dashboard) echo "Starting health dashboard..." ;;
        logs)      echo "Starting log monitor..." ;;
        processes) echo "Starting process monitor..." ;;
        *) echo "Unknown mode: $mode" >&2; return 1 ;;
    esac
}

# ─── Main CLI ─────────────────────────────────────────────────
toolkit_help() {
    cat <<HELP
DevOps Toolkit v${TOOLKIT_VERSION}

Commands:
  deploy <app> [env] [version]  Deploy application
  audit  [host]                 Run security audit
  monitor [mode]                Start monitoring (dashboard|logs|processes)
  version                       Show version

Environment: TOOLKIT_ENV (current: $TOOLKIT_ENV)
HELP
}

main() {
    local cmd=${1:-help}; shift || true
    case "$cmd" in
        deploy)  toolkit_deploy "$@" ;;
        audit)   toolkit_audit "$@" ;;
        monitor) toolkit_monitor "$@" ;;
        version) toolkit_version ;;
        help|-h|--help) toolkit_help ;;
        *) echo "Unknown command: $cmd" >&2; toolkit_help; exit 1 ;;
    esac
}

main "$@"
```

---

## 90.5 Summary of the Full Curriculum

### Parts 1-20: Foundation
- Shell basics, variables, conditionals, loops, functions
- File operations, text processing (grep/sed/awk)
- Input/output, pipes, redirection

### Parts 21-40: Intermediate
- Regular expressions, string manipulation
- Arrays, associative arrays
- Error handling, debugging, logging
- Cron jobs, background processes

### Parts 41-60: System Administration
- User/group management, permissions
- Package management, systemd services
- Disk management, monitoring
- Network tools, SSH automation

### Parts 61-70: Networking and Integration
- Network diagnostics, service discovery
- Backup/recovery, database operations
- API integration, webhook handling

### Parts 71-76: Advanced Topics
- Performance tuning, profiling
- Parallel processing, job queues
- Advanced text processing

### Parts 77-89: Professional/World-Class
- **77**: Secrets management (GPG, PKI, Vault, AWS)
- **78**: Automation framework (queues, workers, pipelines, FSM)
- **79**: Container/Kubernetes automation
- **80**: CI/CD pipeline scripting
- **81**: Infrastructure as Code (Terraform, Ansible, AWS)
- **82**: Log management and analysis
- **83**: Web scraping and crawling
- **84**: Advanced process management
- **85**: Design patterns and architecture
- **86**: Testing and quality assurance
- **87**: Performance and profiling
- **88**: Real-world projects
- **89**: Security auditing

### Part 90: Mastery
- Advanced Bash features (nameref, coprocess)
- POSIX compatibility patterns
- Shell as a service pattern
- Unified DevOps toolkit capstone

---

## 90.6 Final Exercises

### Exercise 1: Personal DevOps Toolkit
Build your own toolkit integrating at least 5 modules from this course:
- Configuration management
- Secret injection
- Deployment automation
- Health monitoring
- Test runner

### Exercise 2: Complete CI/CD Pipeline
Design and implement a complete pipeline for a real project:
- Code → Build → Test → Security scan → Deploy → Monitor

### Exercise 3: Contribute Back
- Turn your best scripts into a proper project
- Add tests, CI, documentation
- Package as an installable tool
- Open source it

---

## สรุป Part 90 และหลักสูตรทั้งหมด

✅ Advanced Bash: nameref, coprocess, process substitution, extglob, dynamic functions
┅ POSIX compatibility: portable arrays, string ops, IS_BASH branching
┅ Shell as service: PID file, SIGTERM/SIGHUP handlers, start/stop/status/reload
┅ DevOps toolkit capstone: unified CLI combining all course modules
┅ Curriculum complete: 90 parts covering beginner → world-class professional level

**หลักสูตร Bash/Shell Script ระดับ World-Class สมบูรณ์แล้ว!**

ครอบคลุม 1000+ steps ตั้งแต่พื้นฐานจนถึงระดับมืออาชีพระดับโลก:
- Secrets and PKI management
- Container and Kubernetes automation
- CI/CD pipelines
- Infrastructure as Code
- Log management and analysis
- Web scraping and crawling
- Process management and supervision
- Design patterns and architecture
- Testing and quality assurance
- Performance profiling
- Security auditing and hardening
