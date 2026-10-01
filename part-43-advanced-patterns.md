# Part 43: Advanced Shell Scripting Patterns
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 43.1 Functional Programming in Bash

```bash
#!/bin/bash
# functional_bash.sh - Functional programming patterns

# ─── Map / Filter / Reduce ─────────────────────────────────────
map() {
    local func=$1
    shift
    for item in "$@"; do
        "$func" "$item"
    done
}

filter() {
    local predicate=$1
    shift
    for item in "$@"; do
        "$predicate" "$item" && echo "$item"
    done
}

reduce() {
    local func=$1
    local acc=$2
    shift 2
    for item in "$@"; do
        acc=$("$func" "$acc" "$item")
    done
    echo "$acc"
}

# Usage examples
double() { echo $(( $1 * 2 )); }
is_even() { (( $1 % 2 == 0 )); }
add() { echo $(( $1 + $2 )); }

# Map: double each element
map double 1 2 3 4 5
# Output: 2 4 6 8 10

# Filter: keep even numbers
filter is_even 1 2 3 4 5 6
# Output: 2 4 6

# Reduce: sum all
reduce add 0 1 2 3 4 5
# Output: 15

# ─── Pipeline Composition ──────────────────────────────────────
compose() {
    local funcs=("$@")
    local input
    while IFS= read -r input; do
        local result="$input"
        for func in "${funcs[@]}"; do
            result=$("$func" "$result")
        done
        echo "$result"
    done
}

trim()       { echo "${1// /}"; }
uppercase()  { echo "${1^^}"; }
add_prefix() { echo "PREFIX_${1}"; }

# Compose: trim → uppercase → add_prefix
echo -e "hello world\n  foo  \nbar" | compose trim uppercase add_prefix

# ─── Currying ──────────────────────────────────────────────────
curry_add() {
    local x=$1
    add_to_x() { echo $(( x + $1 )); }
    echo "add_to_x"
}

add5=$(curry_add 5)
"$add5" 3    # 8
"$add5" 10   # 15

# ─── Memoization ───────────────────────────────────────────────
declare -A _memo_cache=()

memoize() {
    local func=$1
    local key="${func}_${*:2}"

    if [[ -v "_memo_cache[$key]" ]]; then
        echo "${_memo_cache[$key]}"
        return
    fi

    local result
    result=$("$func" "${@:2}")
    _memo_cache["$key"]="$result"
    echo "$result"
}

slow_compute() {
    sleep 1
    echo $(( $1 * $1 ))
}

# First call: slow (1s)
memoize slow_compute 5

# Second call: instant (cached)
memoize slow_compute 5
```

---

## 43.2 Event-Driven Architecture

```bash
#!/bin/bash
# event_system.sh - Event bus for bash scripts

declare -A EVENT_HANDLERS=()
declare -a EVENT_QUEUE=()

on() {
    local event=$1
    local handler=$2

    if [[ -v "EVENT_HANDLERS[$event]" ]]; then
        EVENT_HANDLERS["$event"]+=" $handler"
    else
        EVENT_HANDLERS["$event"]="$handler"
    fi
}

emit() {
    local event=$1
    shift
    local data=("$@")

    local handlers="${EVENT_HANDLERS[$event]:-}"
    if [[ -z "$handlers" ]]; then
        return 0
    fi

    for handler in $handlers; do
        "$handler" "$event" "${data[@]}"
    done
}

emit_async() {
    local event=$1
    shift
    emit "$event" "$@" &
}

# ─── Event Queue ───────────────────────────────────────────────
enqueue_event() {
    local event=$1
    shift
    local args="$*"
    EVENT_QUEUE+=("${event}:${args}")
}

process_event_queue() {
    while (( ${#EVENT_QUEUE[@]} > 0 )); do
        local event_entry="${EVENT_QUEUE[0]}"
        EVENT_QUEUE=("${EVENT_QUEUE[@]:1}")

        local event="${event_entry%%:*}"
        local args="${event_entry#*:}"

        emit "$event" $args
    done
}

# ─── Example usage ─────────────────────────────────────────────
on_user_login() {
    local event=$1
    local username=$2
    echo "[$(date)] User logged in: $username"
}

on_user_login_audit() {
    local event=$1
    local username=$2
    echo "$username" >> /var/log/login_audit.log
}

on_error() {
    local event=$1
    local message=$2
    echo "[ERROR] $message" >&2
}

on "user.login" on_user_login
on "user.login" on_user_login_audit
on "error" on_error

emit "user.login" "alice"
emit "error" "Something went wrong"
```

---

## 43.3 Plugin System

```bash
#!/bin/bash
# plugin_system.sh - Extensible plugin architecture

PLUGIN_DIR="${PLUGIN_DIR:-${HOME}/.config/myapp/plugins}"
declare -A PLUGINS=()
declare -A PLUGIN_HOOKS=()

plugin_register() {
    local name=$1
    local version=$2
    local description=$3

    PLUGINS["$name"]="${version}:${description}"
    echo "Registered plugin: $name v$version"
}

hook_register() {
    local hook_name=$1
    local plugin_name=$2
    local func=$3

    if [[ -v "PLUGIN_HOOKS[$hook_name]" ]]; then
        PLUGIN_HOOKS["$hook_name"]+=" ${plugin_name}::${func}"
    else
        PLUGIN_HOOKS["$hook_name"]="${plugin_name}::${func}"
    fi
}

hook_run() {
    local hook_name=$1
    shift
    local args=("$@")

    local handlers="${PLUGIN_HOOKS[$hook_name]:-}"
    [[ -z "$handlers" ]] && return 0

    local result=0
    for handler in $handlers; do
        local plugin_name="${handler%%::*}"
        local func="${handler##*::}"

        if ! "$func" "${args[@]}"; then
            echo "Hook failed: $plugin_name::$func" >&2
            (( result++ ))
        fi
    done

    return $result
}

load_plugins() {
    local plugin_dir="${1:-$PLUGIN_DIR}"
    [[ ! -d "$plugin_dir" ]] && return 0

    for plugin_file in "${plugin_dir}"/*.sh; do
        [[ -f "$plugin_file" ]] || continue

        local plugin_name
        plugin_name=$(basename "$plugin_file" .sh)

        # shellcheck disable=SC1090
        source "$plugin_file"

        if declare -f "plugin_init_${plugin_name}" &>/dev/null; then
            "plugin_init_${plugin_name}"
        fi

        echo "Loaded plugin: $plugin_name"
    done
}

list_plugins() {
    echo "=== Installed Plugins ==="
    for name in "${!PLUGINS[@]}"; do
        local info="${PLUGINS[$name]}"
        local version="${info%%:*}"
        local desc="${info#*:}"
        printf "  %-20s %-10s %s\n" "$name" "v$version" "$desc"
    done
}

# ─── Example Plugin: logging ───────────────────────────────────
# plugins/logging.sh

logging_before_command() {
    local cmd=$1
    echo "[LOG] Executing: $cmd"
}

logging_after_command() {
    local cmd=$1
    local exit_code=$2
    echo "[LOG] Completed: $cmd (exit=$exit_code)"
}

plugin_init_logging() {
    plugin_register "logging" "1.0.0" "Command execution logger"
    hook_register "before_command" "logging" "logging_before_command"
    hook_register "after_command" "logging" "logging_after_command"
}
```

---

## 43.4 State Machine

```bash
#!/bin/bash
# state_machine.sh - Finite state machine

declare -A SM_TRANSITIONS=()
declare -A SM_ENTRY_ACTIONS=()
declare -A SM_EXIT_ACTIONS=()
declare -A SM_CURRENT_STATE=()

sm_define_transition() {
    local machine=$1
    local from_state=$2
    local event=$3
    local to_state=$4
    local action=${5:-}

    local key="${machine}:${from_state}:${event}"
    SM_TRANSITIONS["$key"]="${to_state}:${action}"
}

sm_on_entry() {
    local machine=$1
    local state=$2
    local action=$3
    SM_ENTRY_ACTIONS["${machine}:${state}"]="$action"
}

sm_on_exit() {
    local machine=$1
    local state=$2
    local action=$3
    SM_EXIT_ACTIONS["${machine}:${state}"]="$action"
}

sm_init() {
    local machine=$1
    local initial_state=$2
    SM_CURRENT_STATE["$machine"]="$initial_state"

    local entry="${SM_ENTRY_ACTIONS[${machine}:${initial_state}]:-}"
    [[ -n "$entry" ]] && "$entry" "$machine" "$initial_state"
}

sm_send() {
    local machine=$1
    local event=$2
    shift 2
    local event_data=("$@")

    local current="${SM_CURRENT_STATE[$machine]}"
    local key="${machine}:${current}:${event}"
    local transition="${SM_TRANSITIONS[$key]:-}"

    if [[ -z "$transition" ]]; then
        echo "Invalid transition: [$current] + [$event]" >&2
        return 1
    fi

    local new_state="${transition%%:*}"
    local action="${transition#*:}"

    local exit_action="${SM_EXIT_ACTIONS[${machine}:${current}]:-}"
    [[ -n "$exit_action" ]] && "$exit_action" "$machine" "$current"

    [[ -n "$action" ]] && "$action" "$machine" "$current" "$new_state" "${event_data[@]}"

    SM_CURRENT_STATE["$machine"]="$new_state"

    local entry_action="${SM_ENTRY_ACTIONS[${machine}:${new_state}]:-}"
    [[ -n "$entry_action" ]] && "$entry_action" "$machine" "$new_state"

    echo "[$current] --${event}--> [$new_state]"
}

sm_state() {
    local machine=$1
    echo "${SM_CURRENT_STATE[$machine]}"
}

# ─── Example: Order FSM ────────────────────────────────────────
order_on_payment() { echo "  Action: Processing payment for order"; }
order_on_ship()    { echo "  Action: Creating shipment"; }
order_on_cancel()  { echo "  Action: Initiating refund"; }

order_enter_shipped() { echo "  Entry: Sending tracking email"; }
order_enter_delivered() { echo "  Entry: Requesting review"; }

# Define states and transitions
sm_define_transition "order" "pending"    "pay"     "paid"      "order_on_payment"
sm_define_transition "order" "paid"       "ship"    "shipped"   "order_on_ship"
sm_define_transition "order" "shipped"    "deliver" "delivered" ""
sm_define_transition "order" "pending"    "cancel"  "cancelled" "order_on_cancel"
sm_define_transition "order" "paid"       "cancel"  "cancelled" "order_on_cancel"

sm_on_entry "order" "shipped"   "order_enter_shipped"
sm_on_entry "order" "delivered" "order_enter_delivered"

sm_init "order" "pending"

echo "Order FSM Demo:"
sm_send "order" "pay"
sm_send "order" "ship"
sm_send "order" "deliver"
echo "Final state: $(sm_state 'order')"
```

---

## 43.5 Template Engine

```bash
#!/bin/bash
# template_engine.sh - Advanced Bash template engine

render_template() {
    local template_file=$1
    shift
    local vars=("$@")

    local content
    content=$(cat "$template_file")

    for var in "${vars[@]}"; do
        local key="${var%%=*}"
        local value="${var#*=}"
        content="${content//\{\{${key}\}\}/$value}"
    done

    local include_pattern='\{\{include ([^}]+)\}\}'
    while [[ "$content" =~ $include_pattern ]]; do
        local include_file="${BASH_REMATCH[1]}"
        local include_content
        include_content=$(cat "$include_file" 2>/dev/null || echo "<!-- include not found: $include_file -->")
        content="${content/\{\{include ${include_file}\}\}/$include_content}"
    done

    local if_pattern='\{\{if ([^}]+)\}\}(.*?)\{\{endif\}\}'
    while [[ "$content" =~ \{\{if\ ([^}]+)\}\} ]]; do
        local condition="${BASH_REMATCH[1]}"
        if eval "$condition" 2>/dev/null; then
            content=$(echo "$content" | sed "s/{{if ${condition}}}//; s/{{endif}}//")
        else
            content=$(echo "$content" | perl -0777 -pe "s/\{\{if ${condition}\}\}.*?\{\{endif\}\}//s" 2>/dev/null || echo "$content")
        fi
    done

    local for_pattern='\{\{for ([^ ]+) in ([^}]+)\}\}'
    if [[ "$content" =~ $for_pattern ]]; then
        local var_name="${BASH_REMATCH[1]}"
        local items_var="${BASH_REMATCH[2]}"
        local items_str="${!items_var}"
        local expanded=""
        local body=""
        body=$(echo "$content" | sed -n "/{{for ${var_name} in ${items_var}}}/,/{{endfor}}/p" | \
               sed "1d;\$d")

        for item in $items_str; do
            local item_content="${body//\{\{${var_name}\}\}/$item}"
            expanded+="$item_content"$'\n'
        done

        content=$(echo "$content" | \
            perl -0777 -pe "s/\{\{for ${var_name} in ${items_var}\}\}.*?\{\{endfor\}\}/${expanded}/s" 2>/dev/null || echo "$content")
    fi

    echo "$content"
}

generate_from_template() {
    local template=$1
    local output=$2
    shift 2

    render_template "$template" "$@" > "$output"
    echo "Generated: $output"
}

# ─── Config File Generator ─────────────────────────────────────
generate_nginx_config() {
    local domain=$1
    local backend_host=$2
    local backend_port=${3:-8080}
    local ssl=${4:-false}

    local template_content
    template_content=$(cat << 'TMPL'
server {
    listen {{ssl_port}};
    server_name {{domain}};
    {{ssl_config}}

    location / {
        proxy_pass http://{{backend_host}}:{{backend_port}};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 30;
        proxy_send_timeout 60;
        proxy_read_timeout 60;
    }

    location /health {
        access_log off;
        return 200 "OK";
    }
}
TMPL
)

    local ssl_port="80"
    local ssl_config=""

    if $ssl; then
        ssl_port="443 ssl"
        ssl_config="ssl_certificate /etc/ssl/${domain}.crt;
    ssl_certificate_key /etc/ssl/${domain}.key;
    ssl_protocols TLSv1.2 TLSv1.3;"
    fi

    echo "$template_content" | \
        sed \
            -e "s/{{domain}}/$domain/g" \
            -e "s/{{backend_host}}/$backend_host/g" \
            -e "s/{{backend_port}}/$backend_port/g" \
            -e "s/{{ssl_port}}/$ssl_port/g" \
            -e "s|{{ssl_config}}|$ssl_config|g"
}

generate_systemd_unit() {
    local service_name=$1
    local exec_start=$2
    local user=${3:-www-data}
    local description=${4:-$service_name}

    cat << EOF
[Unit]
Description=${description}
After=network.target

[Service]
Type=simple
User=${user}
ExecStart=${exec_start}
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=${service_name}

[Install]
WantedBy=multi-user.target
EOF
}
```

---

## 43.6 Concurrent Data Processing

```bash
#!/bin/bash
# concurrent_processing.sh - Advanced parallel patterns

# ─── Worker Pool with Bounded Concurrency ──────────────────────
parallel_exec() {
    local max_jobs=${1:-4}
    local input_file=${2:-/dev/stdin}
    local processor=${3:-cat}

    local job_count=0
    local pids=()

    while IFS= read -r item; do
        if (( job_count >= max_jobs )); then
            wait -n 2>/dev/null || wait "${pids[0]}"
            pids=("${pids[@]:1}")
            (( job_count-- ))
        fi

        "$processor" "$item" &
        pids+=($!)
        (( job_count++ ))
    done < "$input_file"

    wait "${pids[@]}"
}

# ─── Fan-out with Result Collection ────────────────────────────
fanout() {
    local func=$1
    shift
    local items=("$@")

    local tmpdir
    tmpdir=$(mktemp -d)
    local pids=()

    for i in "${!items[@]}"; do
        local item="${items[$i]}"
        local result_file="${tmpdir}/${i}.result"
        "$func" "$item" > "$result_file" &
        pids+=($!)
    done

    local failed=0
    for i in "${!pids[@]}"; do
        if wait "${pids[$i]}"; then
            cat "${tmpdir}/${i}.result"
        else
            echo "FAILED: ${items[$i]}" >&2
            (( failed++ ))
        fi
    done

    rm -rf "$tmpdir"
    (( failed > 0 )) && return 1 || return 0
}

# ─── Pipeline Stage Parallelism ────────────────────────────────
parallel_pipeline() {
    local workers=${1:-4}
    local process_func=${2:-cat}
    local input=${3:-/dev/stdin}
    local output=${4:-/dev/stdout}

    local work_fifo output_fifo
    work_fifo=$(mktemp -u)
    output_fifo=$(mktemp -u)

    mkfifo "$work_fifo" "$output_fifo"
    trap "rm -f '$work_fifo' '$output_fifo'" EXIT

    for (( i=0; i<workers; i++ )); do
        (
            while IFS= read -r line; do
                result=$("$process_func" "$line")
                echo "$result" > "$output_fifo"
            done < "$work_fifo"
        ) &
    done

    cat "$input" > "$work_fifo" &

    local line_count
    line_count=$(wc -l < "$input")

    for (( i=0; i<line_count; i++ )); do
        IFS= read -r result < "$output_fifo"
        echo "$result" >> "$output"
    done

    wait
}

# ─── Scatter-Gather Pattern ────────────────────────────────────
scatter_gather() {
    local aggregator=$1
    shift
    local workers=("$@")

    local tmpdir
    tmpdir=$(mktemp -d)
    local pids=()

    for i in "${!workers[@]}"; do
        local worker="${workers[$i]}"
        local result="${tmpdir}/${i}"
        "$worker" > "$result" &
        pids+=($!)
    done

    for pid in "${pids[@]}"; do
        wait "$pid"
    done

    for result_file in "${tmpdir}"/*; do
        cat "$result_file"
    done | "$aggregator"

    rm -rf "$tmpdir"
}

# Example: check multiple services
check_service() { curl -sf "$1/health" > /dev/null && echo "$1: UP" || echo "$1: DOWN"; }

# Parallel health checks
fanout check_service \
    "http://service1:8080" \
    "http://service2:8080" \
    "http://service3:8080"
```

---

## 43.7 Advanced Error Handling

```bash
#!/bin/bash
# advanced_error_handling.sh

# ─── Error Context Stack ───────────────────────────────────────
declare -a ERROR_CONTEXT=()

push_context() { ERROR_CONTEXT+=("$1"); }
pop_context()  { unset 'ERROR_CONTEXT[-1]'; }

error_context() {
    if (( ${#ERROR_CONTEXT[@]} > 0 )); then
        echo "Context: ${ERROR_CONTEXT[*]}"
    fi
}

# ─── Structured Error Handler ──────────────────────────────────
declare -A ERROR_HANDLERS=()

on_error() {
    local pattern=$1
    local handler=$2
    ERROR_HANDLERS["$pattern"]="$handler"
}

handle_error() {
    local code=$1
    local message=$2
    local line=${3:-${BASH_LINENO[0]}}
    local func=${4:-${FUNCNAME[1]}}

    echo "ERROR [$code] in ${func}:${line}: $message" >&2
    error_context >&2

    for pattern in "${!ERROR_HANDLERS[@]}"; do
        if [[ "$message" =~ $pattern ]]; then
            "${ERROR_HANDLERS[$pattern]}" "$code" "$message" "$line" "$func"
            return
        fi
    done
}

# ─── Retry with Backoff ────────────────────────────────────────
retry() {
    local max_attempts=${1:-3}
    local base_delay=${2:-1}
    local max_delay=${3:-30}
    shift 3
    local cmd=("$@")

    local attempt=0

    while (( attempt < max_attempts )); do
        (( attempt++ ))

        push_context "attempt ${attempt}/${max_attempts}"

        if "${cmd[@]}"; then
            pop_context
            return 0
        fi

        local exit_code=$?
        pop_context

        if (( attempt < max_attempts )); then
            local delay=$(( base_delay * (2 ** (attempt - 1)) ))
            (( delay > max_delay )) && delay=$max_delay
            echo "Attempt $attempt failed (exit=$exit_code), retrying in ${delay}s..." >&2
            sleep "$delay"
        fi
    done

    handle_error 1 "Command failed after $max_attempts attempts: ${cmd[*]}"
    return 1
}

# ─── Circuit Breaker ───────────────────────────────────────────
declare -A CIRCUIT_BREAKERS=()

circuit_breaker() {
    local name=$1
    local threshold=${2:-3}
    local reset_time=${3:-60}
    shift 3
    local cmd=("$@")

    local state_key="${name}_state"
    local count_key="${name}_fails"
    local time_key="${name}_opened"

    local state="${CIRCUIT_BREAKERS[$state_key]:-closed}"
    local fail_count="${CIRCUIT_BREAKERS[$count_key]:-0}"
    local opened_at="${CIRCUIT_BREAKERS[$time_key]:-0}"

    if [[ "$state" == "open" ]]; then
        local elapsed=$(( SECONDS - opened_at ))
        if (( elapsed > reset_time )); then
            CIRCUIT_BREAKERS[$state_key]="half-open"
            state="half-open"
            echo "Circuit $name: half-open (testing)" >&2
        else
            echo "Circuit $name: OPEN (${elapsed}s / ${reset_time}s to reset)" >&2
            return 1
        fi
    fi

    if "${cmd[@]}"; then
        CIRCUIT_BREAKERS[$count_key]=0
        CIRCUIT_BREAKERS[$state_key]="closed"
        return 0
    fi

    (( fail_count++ ))
    CIRCUIT_BREAKERS[$count_key]=$fail_count

    if (( fail_count >= threshold )); then
        CIRCUIT_BREAKERS[$state_key]="open"
        CIRCUIT_BREAKERS[$time_key]=$SECONDS
        echo "Circuit $name: OPENED after $fail_count failures" >&2
    fi

    return 1
}

# ─── Timeout Wrapper ───────────────────────────────────────────
with_timeout() {
    local timeout_secs=$1
    shift
    local cmd=("$@")

    (
        "${cmd[@]}" &
        local pid=$!

        (
            sleep "$timeout_secs"
            kill "$pid" 2>/dev/null
        ) &
        local timer_pid=$!

        wait "$pid"
        local exit_code=$?
        kill "$timer_pid" 2>/dev/null
        exit $exit_code
    )
}
```

---

## 43.8 DSL (Domain-Specific Language) Builder

```bash
#!/bin/bash
# dsl_builder.sh - Build a mini-DSL for infrastructure

# ─── Server DSL ────────────────────────────────────────────────
declare -A CURRENT_SERVER=()
declare -a SERVERS=()

server() {
    CURRENT_SERVER=()
    CURRENT_SERVER[name]="$1"
}

has_role() {
    CURRENT_SERVER[role]="$1"
}

with_ip() {
    CURRENT_SERVER[ip]="$1"
}

with_user() {
    CURRENT_SERVER[user]="${1:-ubuntu}"
}

with_port() {
    CURRENT_SERVER[port]="${1:-22}"
}

end_server() {
    local entry
    entry=$(declare -p CURRENT_SERVER)
    SERVERS+=("$entry")
}

# ─── Deployment DSL ────────────────────────────────────────────
declare -A CURRENT_DEPLOY=()

deploy() {
    CURRENT_DEPLOY=()
    CURRENT_DEPLOY[app]="$1"
}

from_image() {
    CURRENT_DEPLOY[image]="$1"
}

to_servers() {
    CURRENT_DEPLOY[servers]="$*"
}

with_env() {
    local key=$1
    local value=$2
    CURRENT_DEPLOY["env_${key}"]="$value"
}

run_deploy() {
    echo "=== Deploying ${CURRENT_DEPLOY[app]} ==="
    echo "Image: ${CURRENT_DEPLOY[image]}"
    echo "Servers: ${CURRENT_DEPLOY[servers]}"

    for key in "${!CURRENT_DEPLOY[@]}"; do
        [[ "$key" == env_* ]] && echo "  ${key#env_}=${CURRENT_DEPLOY[$key]}"
    done
}

# ─── Example usage ─────────────────────────────────────────────
server "web-01"
  with_ip "10.0.1.10"
  has_role "web"
  with_user "ubuntu"
end_server

server "web-02"
  with_ip "10.0.1.11"
  has_role "web"
end_server

deploy "myapp"
  from_image "registry.example.com/myapp:v1.2.3"
  to_servers "web-01" "web-02"
  with_env "NODE_ENV" "production"
  with_env "PORT" "3000"
run_deploy
```

---

## 43.9 Exercises

### Exercise 1: Reactive Script Framework
สร้าง framework ที่:
- Monitor file changes (inotify)
- Trigger reactions automatically
- Debounce rapid changes
- Support multiple watchers

### Exercise 2: Configuration DSL
สร้าง DSL สำหรับ server configuration:
- Declarative syntax
- Idempotent operations
- Diff and apply
- Rollback support

### Exercise 3: Distributed Task Queue
สร้าง task queue system:
- Multiple workers
- Task priorities
- Dead letter queue
- Monitoring dashboard

---

## สรุป Part 43

✅ Functional programming (map/filter/reduce/compose)
✅ Memoization pattern
✅ Event-driven architecture
✅ Plugin system with hooks
✅ Finite state machine
✅ Template engine
✅ Advanced parallel patterns (fanout, scatter-gather)
✅ Circuit breaker pattern
✅ Retry with exponential backoff
✅ Mini-DSL builder

---

**→ Part 44: Web Application Development with Bash**
