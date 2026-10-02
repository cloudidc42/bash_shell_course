# Part 85: Shell Script Design Patterns and Architecture
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 85.1 Plugin System

```bash
#!/bin/bash
# plugin_system.sh - Dynamic plugin loading and dispatch

PLUGIN_DIR="${PLUGIN_DIR:-/etc/myapp/plugins}"
declare -A PLUGIN_COMMANDS=()
declare -A PLUGIN_META=()

plugin_load_all() {
    local dir="${1:-$PLUGIN_DIR}"
    [[ -d "$dir" ]] || return 0

    for file in "$dir"/*.sh "$dir"/*.plugin; do
        [[ -f "$file" ]] || continue
        plugin_load "$file"
    done
}

plugin_load() {
    local file=$1
    # shellcheck disable=SC1090
    source "$file" 2>/dev/null || { echo "Failed to load plugin: $file" >&2; return 1; }

    # Auto-discover plugin_register calls made by the file
    local name; name=$(grep -m1 '^PLUGIN_NAME=' "$file" 2>/dev/null | cut -d= -f2 | tr -d '"' || true)
    [[ -n "$name" ]] && PLUGIN_META["$name"]="$file"
    echo "Loaded plugin: ${name:-$(basename "$file")}"
}

plugin_register() {
    local name=$1 command=$2 handler=$3 description=${4:-}
    PLUGIN_COMMANDS["${name}:${command}"]="$handler"
    echo "Registered command: $name $command -> $handler()"
}

plugin_dispatch() {
    local plugin=$1 command=$2; shift 2
    local key="${plugin}:${command}"

    if [[ -n "${PLUGIN_COMMANDS[$key]:-}" ]]; then
        "${PLUGIN_COMMANDS[$key]}" "$@"
    else
        echo "Unknown plugin command: $plugin $command" >&2
        return 1
    fi
}

plugin_list() {
    echo "Loaded plugins:"
    for key in "${!PLUGIN_COMMANDS[@]}"; do
        printf "  %s\n" "$key"
    done | sort
}
```

---

## 85.2 Configuration Management Pattern

```bash
#!/bin/bash
# config_pattern.sh - Layered configuration with defaults, env, file, args

declare -A CONFIG=()

config_set_defaults() {
    CONFIG[log_level]="info"
    CONFIG[log_file]="/var/log/myapp.log"
    CONFIG[workers]=4
    CONFIG[timeout]=30
    CONFIG[db_host]="localhost"
    CONFIG[db_port]="5432"
    CONFIG[dry_run]="false"
}

config_load_file() {
    local file=$1
    [[ -f "$file" ]] || return 0

    while IFS='=' read -r key value; do
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "$key" ]] && continue
        key="${key%"${key##*[![:space:]]}"}"
        key="${key#"${key%%[![:space:]]*}"}"
        value="${value%"${value##*[![:space:]]}"}"
        value="${value#"${value%%[![:space:]]*}"}"
        value="${value#\"}" value="${value%\"}"
        CONFIG["$key"]="$value"
    done < "$file"
}

config_load_env() {
    local prefix=${1:-MYAPP_}
    while IFS='=' read -r key value; do
        if [[ "$key" =~ ^${prefix} ]]; then
            local cfg_key="${key#${prefix}}"
            cfg_key="${cfg_key,,}"
            CONFIG["$cfg_key"]="$value"
        fi
    done < <(env)
}

config_parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --log-level=*) CONFIG[log_level]="${1#*=}" ;;
            --log-level)   CONFIG[log_level]="$2"; shift ;;
            --workers=*)   CONFIG[workers]="${1#*=}" ;;
            --workers)     CONFIG[workers]="$2"; shift ;;
            --dry-run)     CONFIG[dry_run]="true" ;;
            --db-host=*)   CONFIG[db_host]="${1#*=}" ;;
            --config=*)    config_load_file "${1#*=}" ;;
            *)             ;;
        esac
        shift
    done
}

config_get() {
    local key=$1 default=${2:-}
    echo "${CONFIG[$key]:-$default}"
}

config_require() {
    local -a keys=("$@")
    local missing=()
    for key in "${keys[@]}"; do
        [[ -z "${CONFIG[$key]:-}" ]] && missing+=("$key")
    done
    (( ${#missing[@]} == 0 )) || {
        echo "Missing required config: ${missing[*]}" >&2
        return 1
    }
}

config_dump() {
    echo "=== Configuration ==="
    for key in $(echo "${!CONFIG[@]}" | tr ' ' '\n' | sort); do
        printf "  %-20s = %s\n" "$key" "${CONFIG[$key]}"
    done
}
```

---

## 85.3 Command Dispatcher (CLI Framework)

```bash
#!/bin/bash
# cli_framework.sh - Self-documenting CLI command dispatcher

declare -A CMD_HANDLERS=()
declare -A CMD_DESCRIPTIONS=()
declare -A CMD_USAGE=()

cmd_register() {
    local name=$1 handler=$2 description=$3 usage=${4:-}
    CMD_HANDLERS["$name"]="$handler"
    CMD_DESCRIPTIONS["$name"]="$description"
    CMD_USAGE["$name"]="$usage"
}

cmd_dispatch() {
    local cmd=${1:-help}; shift || true

    if [[ -n "${CMD_HANDLERS[$cmd]:-}" ]]; then
        "${CMD_HANDLERS[$cmd]}" "$@"
    else
        echo "Unknown command: $cmd" >&2
        cmd_dispatch help
        return 1
    fi
}

cmd_help() {
    local cmd=${1:-}
    if [[ -n "$cmd" && -n "${CMD_HANDLERS[$cmd]:-}" ]]; then
        echo "Usage: $0 $cmd ${CMD_USAGE[$cmd]:-}"
        echo ""
        echo "${CMD_DESCRIPTIONS[$cmd]}"
    else
        echo "Usage: $0 <command> [options]"
        echo ""
        echo "Commands:"
        for name in $(echo "${!CMD_HANDLERS[@]}" | tr ' ' '\n' | sort); do
            printf "  %-20s %s\n" "$name" "${CMD_DESCRIPTIONS[$name]:-}"
        done
    fi
}

cmd_register "help" cmd_help "Show this help message" "[command]"

# Example usage
_cmd_start() {
    echo "Starting service..."
}
_cmd_stop() {
    echo "Stopping service..."
}
_cmd_status() {
    echo "Service status..."
}

cmd_register "start"  _cmd_start  "Start the service"  ""
cmd_register "stop"   _cmd_stop   "Stop the service"   ""
cmd_register "status" _cmd_status "Show service status" ""
```

---

## 85.4 Template Engine

```bash
#!/bin/bash
# template.sh - Simple variable substitution template engine

template_render() {
    local template=$1
    shift
    local -A vars=()

    # Load vars from key=value pairs
    while [[ $# -gt 0 ]]; do
        local key="${1%%=*}"
        local val="${1#*=}"
        vars["$key"]="$val"
        shift
    done

    local result="$template"
    for key in "${!vars[@]}"; do
        local val="${vars[$key]}"
        result="${result//\{\{${key}\}\}/$val}"
        result="${result//\$\{${key}\}/$val}"
    done

    echo "$result"
}

template_render_file() {
    local file=$1; shift
    local content; content=$(cat "$file")
    template_render "$content" "$@"
}

template_render_env() {
    local template=$1
    local result="$template"

    # Replace {{VAR}} with env var values
    while [[ "$result" =~ \{\{([A-Z_][A-Z0-9_]*)\}\} ]]; do
        local varname="${BASH_REMATCH[1]}"
        local value="${!varname:-}"
        result="${result//\{\{${varname}\}\}/$value}"
    done

    echo "$result"
}

template_apply_dir() {
    local src_dir=$1 dst_dir=$2
    shift 2

    mkdir -p "$dst_dir"
    find "$src_dir" -name "*.tmpl" | while read -r tmpl; do
        local rel="${tmpl#${src_dir}/}"
        local dst="${dst_dir}/${rel%.tmpl}"
        mkdir -p "$(dirname "$dst")"
        template_render_file "$tmpl" "$@" > "$dst"
        echo "Rendered: $dst"
    done
}
```

---

## 85.5 Observer Pattern / Event Hooks

```bash
#!/bin/bash
# hooks.sh - Lifecycle hooks and observer pattern

declare -A HOOK_HANDLERS=()

hook_register() {
    local event=$1 handler=$2 priority=${3:-50}
    local key="${event}:${priority}:${handler}"
    HOOK_HANDLERS["$key"]="$handler"
}

hook_unregister() {
    local event=$1 handler=$2
    for key in "${!HOOK_HANDLERS[@]}"; do
        [[ "$key" =~ ^${event}:.*:${handler}$ ]] && unset "HOOK_HANDLERS[$key]"
    done
}

hook_fire() {
    local event=$1; shift
    local -a args=("$@")

    # Execute hooks in priority order
    local -a sorted_keys=()
    while IFS= read -r k; do
        sorted_keys+=("$k")
    done < <(
        for key in "${!HOOK_HANDLERS[@]}"; do
            [[ "$key" =~ ^${event}: ]] && echo "$key"
        done | sort -t: -k2 -n
    )

    for key in "${sorted_keys[@]}"; do
        local handler="${HOOK_HANDLERS[$key]}"
        "$handler" "$event" "${args[@]}" 2>/dev/null || true
    done
}

hook_list() {
    local event=${1:-}
    echo "Registered hooks:"
    for key in $(echo "${!HOOK_HANDLERS[@]}" | tr ' ' '\n' | sort); do
        if [[ -z "$event" || "$key" =~ ^${event}: ]]; then
            printf "  %s\n" "$key"
        fi
    done
}

# Middleware chain pattern
declare -a MIDDLEWARE=()

middleware_use() {
    MIDDLEWARE+=("$1")
}

middleware_run() {
    local -a args=("$@")
    local idx=0

    _next() {
        if (( idx < ${#MIDDLEWARE[@]} )); then
            local fn="${MIDDLEWARE[$idx]}"
            (( idx++ ))
            "$fn" _next "${args[@]}"
        fi
    }
    _next
}
```

---

## 85.6 Retry and Resilience Patterns

```bash
#!/bin/bash
# resilience.sh - Retry, fallback, and bulkhead patterns

with_retry() {
    local max=${1:-3} delay=${2:-5} backoff=${3:-2}; shift 3
    local -a cmd=("$@")
    local attempt=1 wait_time=$delay

    while (( attempt <= max )); do
        if "${cmd[@]}"; then
            return 0
        fi
        (( attempt < max )) || break
        echo "Attempt $attempt/$max failed. Retrying in ${wait_time}s..." >&2
        sleep "$wait_time"
        wait_time=$(( wait_time * backoff ))
        (( attempt++ ))
    done

    echo "All $max attempts failed: ${cmd[*]}" >&2
    return 1
}

with_fallback() {
    local primary=$1 fallback=$2
    shift 2

    if "$primary" "$@" 2>/dev/null; then
        return 0
    fi
    echo "Primary '$primary' failed, using fallback '$fallback'" >&2
    "$fallback" "$@"
}

with_timeout() {
    local sec=$1; shift
    timeout "$sec" "$@"
}

with_bulkhead() {
    local max_concurrent=$1 lockdir=${2:-/tmp/bulkhead}; shift 2
    local -a cmd=("$@")

    mkdir -p "$lockdir"
    local count; count=$(ls "$lockdir"/*.lock 2>/dev/null | wc -l)
    if (( count >= max_concurrent )); then
        echo "Bulkhead full ($count/$max_concurrent), rejecting" >&2
        return 1
    fi

    local lockfile="${lockdir}/$$.lock"
    touch "$lockfile"
    trap "rm -f '$lockfile'" RETURN

    "${cmd[@]}"
}

cache_result() {
    local key=$1 ttl_sec=${2:-300}; shift 2
    local -a cmd=("$@")
    local cache_dir="/tmp/bash_cache"
    local cache_file="${cache_dir}/$(echo "$key" | md5sum | cut -d' ' -f1)"

    mkdir -p "$cache_dir"

    if [[ -f "$cache_file" ]]; then
        local age=$(( $(date +%s) - $(stat -c '%Y' "$cache_file" 2>/dev/null || echo 0) ))
        if (( age < ttl_sec )); then
            cat "$cache_file"
            return 0
        fi
    fi

    local result; result=$("${cmd[@]}")
    echo "$result" > "$cache_file"
    echo "$result"
}
```

---

## 85.7 Exercises

### Exercise 1: Plugin-based CLI
สร้าง CLI tool ที่:
- Load plugins from a directory
- Each plugin exports commands
- Support --help per plugin
- Version check and conflict detection

### Exercise 2: Config Hierarchy
สร้าง config system ที่:
- /etc/myapp/defaults.conf (lowest priority)
- ~/.myapp/config (user overrides)
- .myapp.local (project overrides)
- Env vars (highest priority)

### Exercise 3: Middleware Pipeline
สร้าง HTTP-like middleware stack ที่:
- Auth middleware
- Rate-limit middleware
- Logging middleware
- Response caching

---

## สรุป Part 85

✅ Plugin system: load_all, register, dispatch, list — file-based dynamic loading
┅ Config: defaults → file → env → args layered priority, config_require validation
┅ CLI dispatcher: register + dispatch + self-documenting help
┅ Template engine: {{VAR}} and ${VAR} substitution, file rendering, dir apply
┅ Observer/hooks: priority-ordered fire, middleware chain pattern
┅ Resilience: with_retry (exponential backoff), with_fallback, with_timeout, with_bulkhead
┅ cache_result: TTL-based file caching for expensive commands

---

**→ Part 86: Shell Script Testing and Quality Assurance**
