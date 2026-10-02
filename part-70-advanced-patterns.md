# Part 70: Advanced Shell Scripting Patterns
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 70.1 Object-Oriented Bash

```bash
#!/bin/bash
# oop_bash.sh - Object-oriented patterns in Bash

# ─── Class Simulation via Associative Arrays ─────────────────────────────────

# Constructor
Server::new() {
    local instance=$1 host=$2 port=${3:-80}

    declare -gA "Server_${instance}=([host]=$host [port]=$port [connections]=0 [state]=stopped)"
    declare -gf "${instance}.start" 2>/dev/null || true

    eval "
${instance}.start() { Server::start '${instance}'; }
${instance}.stop()  { Server::stop  '${instance}'; }
${instance}.status(){ Server::status '${instance}'; }
${instance}.get()   { Server::get   '${instance}' \"\$1\"; }
${instance}.set()   { Server::set   '${instance}' \"\$1\" \"\$2\"; }
"
}

Server::start() {
    local instance=$1
    local -n obj="Server_${instance}"

    if [[ "${obj[state]}" == "running" ]]; then
        echo "$instance already running"
        return 1
    fi

    obj[state]="running"
    obj[pid]=$$
    echo "$instance started on ${obj[host]}:${obj[port]}"
}

Server::stop() {
    local instance=$1
    local -n obj="Server_${instance}"

    obj[state]="stopped"
    unset 'obj[pid]'
    echo "$instance stopped"
}

Server::status() {
    local instance=$1
    local -n obj="Server_${instance}"

    printf '%s: state=%s host=%s port=%s connections=%s\n' \
        "$instance" "${obj[state]}" "${obj[host]}" "${obj[port]}" "${obj[connections]}"
}

Server::get() {
    local instance=$1 key=$2
    local -n obj="Server_${instance}"
    echo "${obj[$key]:-}"
}

Server::set() {
    local instance=$1 key=$2 value=$3
    local -n obj="Server_${instance}"
    obj[$key]="$value"
}

Server::destroy() {
    local instance=$1
    unset "Server_${instance}"
    unset -f "${instance}.start" "${instance}.stop" "${instance}.status" \
             "${instance}.get" "${instance}.set" 2>/dev/null || true
}

# Usage:
# Server::new web1 localhost 8080
# web1.start
# web1.status
# web1.set connections 42
# web1.get state
```

---

## 70.2 Event System

```bash
#!/bin/bash
# events.sh - Event pub/sub system

declare -A EVENT_HANDLERS=()

event_on() {
    local event=$1 handler=$2
    local existing="${EVENT_HANDLERS[$event]:-}"

    if [[ -z "$existing" ]]; then
        EVENT_HANDLERS["$event"]="$handler"
    else
        EVENT_HANDLERS["$event"]="$existing $handler"
    fi
}

event_off() {
    local event=$1 handler=$2
    local existing="${EVENT_HANDLERS[$event]:-}"
    EVENT_HANDLERS["$event"]=$(echo "$existing" | tr ' ' '\n' | grep -v "^${handler}$" | tr '\n' ' ')
}

event_emit() {
    local event=$1
    shift
    local args=("$@")

    local handlers="${EVENT_HANDLERS[$event]:-}"
    [[ -z "$handlers" ]] && return 0

    for handler in $handlers; do
        if declare -f "$handler" &>/dev/null; then
            "$handler" "$event" "${args[@]:-}"
        fi
    done
}

event_once() {
    local event=$1 handler=$2
    local wrapper="__once_${event}_${handler//[^a-zA-Z0-9]/_}"

    eval "
${wrapper}() {
    event_off '${event}' '${wrapper}'
    '${handler}' \"\$@\"
}
"
    event_on "$event" "$wrapper"
}

event_list() {
    for event in "${!EVENT_HANDLERS[@]}"; do
        echo "$event: ${EVENT_HANDLERS[$event]}"
    done
}

# ─── Lifecycle Events ──────────────────────────────────────────────────────
setup_lifecycle_events() {
    # Emit exit event on script termination
    trap 'event_emit "exit" "$?"' EXIT
    trap 'event_emit "error" "$LINENO" "$BASH_COMMAND"' ERR
    trap 'event_emit "interrupt"' INT TERM
}
```

---

## 70.3 State Machine

```bash
#!/bin/bash
# state_machine.sh - Finite state machine implementation

# ─── State Machine Definition ───────────────────────────────────────────────
fsm_create() {
    local name=$1 initial_state=$2

    declare -gA "FSM_${name}_transitions=()"
    declare -gA "FSM_${name}_guards=()"
    declare -gA "FSM_${name}_actions=()"
    declare -g "FSM_${name}_state=$initial_state"

    echo "Created FSM: $name (initial: $initial_state)"
}

fsm_add_transition() {
    local name=$1 from=$2 event=$3 to=$4
    local guard=${5:-} action=${6:-}
    local key="${from}:${event}"

    local -n transitions="FSM_${name}_transitions"
    local -n guards="FSM_${name}_guards"
    local -n actions="FSM_${name}_actions"

    transitions["$key"]="$to"
    [[ -n "$guard" ]] && guards["$key"]="$guard"
    [[ -n "$action" ]] && actions["$key"]="$action"
}

fsm_trigger() {
    local name=$1 event=$2
    shift 2
    local args=("$@")

    local state_var="FSM_${name}_state"
    local current_state="${!state_var}"
    local key="${current_state}:${event}"

    local -n transitions="FSM_${name}_transitions"
    local -n guards="FSM_${name}_guards"
    local -n actions="FSM_${name}_actions"

    local next_state="${transitions[$key]:-}"

    if [[ -z "$next_state" ]]; then
        echo "FSM $name: no transition from '$current_state' on '$event'" >&2
        return 1
    fi

    # Check guard
    local guard="${guards[$key]:-}"
    if [[ -n "$guard" ]]; then
        if ! "$guard" "$name" "$current_state" "$event" "${args[@]:-}"; then
            echo "FSM $name: guard '$guard' prevented transition"
            return 1
        fi
    fi

    # Execute action
    local action="${actions[$key]:-}"
    [[ -n "$action" ]] && "$action" "$name" "$current_state" "$next_state" "${args[@]:-}"

    # Transition
    printf -v "FSM_${name}_state" '%s' "$next_state"

    event_emit "fsm:transition" "$name" "$current_state" "$event" "$next_state"
    echo "FSM $name: $current_state --[$event]--> $next_state"
}

fsm_state() {
    local name=$1
    local state_var="FSM_${name}_state"
    echo "${!state_var}"
}

# ─── Example: Deployment State Machine ───────────────────────────────────
setup_deployment_fsm() {
    fsm_create "deploy" "idle"
    fsm_add_transition "deploy" "idle"      "start"    "building"
    fsm_add_transition "deploy" "building"  "success"  "testing"
    fsm_add_transition "deploy" "building"  "failure"  "failed"
    fsm_add_transition "deploy" "testing"   "pass"     "deploying"
    fsm_add_transition "deploy" "testing"   "fail"     "failed"
    fsm_add_transition "deploy" "deploying" "done"     "running"
    fsm_add_transition "deploy" "deploying" "error"    "failed"
    fsm_add_transition "deploy" "running"   "rollback" "idle"
    fsm_add_transition "deploy" "failed"    "reset"    "idle"
}
```

---

## 70.4 Plugin Architecture

```bash
#!/bin/bash
# plugin_system.sh - Dynamic plugin loading

PLUGIN_DIR="${PLUGIN_DIR:-./plugins}"
declare -A PLUGIN_REGISTRY=()
declare -A PLUGIN_HOOKS=()

plugin_register() {
    local name=$1 version=${2:-1.0} capabilities=${3:-}

    PLUGIN_REGISTRY["$name"]="version=${version}|caps=${capabilities}"
    echo "Plugin registered: $name v$version"
}

plugin_hook_add() {
    local hook_name=$1 func=$2
    local existing="${PLUGIN_HOOKS[$hook_name]:-}"

    if [[ -z "$existing" ]]; then
        PLUGIN_HOOKS["$hook_name"]="$func"
    else
        PLUGIN_HOOKS["$hook_name"]="$existing $func"
    fi
}

plugin_hook_run() {
    local hook_name=$1; shift
    local args=("$@")
    local handlers="${PLUGIN_HOOKS[$hook_name]:-}"

    for handler in $handlers; do
        declare -f "$handler" &>/dev/null && "$handler" "${args[@]:-}"
    done
}

plugin_load() {
    local plugin_file=$1

    [[ -f "$plugin_file" ]] || { echo "Plugin not found: $plugin_file" >&2; return 1; }

    # Source plugin in subshell to verify, then in current shell
    if ( source "$plugin_file" 2>/dev/null ); then
        source "$plugin_file"
        echo "Plugin loaded: $plugin_file"
    else
        echo "ERROR: Plugin load failed: $plugin_file" >&2
        return 1
    fi
}

plugin_load_all() {
    local dir=${1:-$PLUGIN_DIR}

    for plugin in "$dir"/*.plugin.sh; do
        [[ -f "$plugin" ]] && plugin_load "$plugin"
    done
}

plugin_list() {
    echo "=== Loaded Plugins ==="
    for name in "${!PLUGIN_REGISTRY[@]}"; do
        local info="${PLUGIN_REGISTRY[$name]}"
        echo "  $name: $info"
    done
}
```

---

## 70.5 Functional Programming Patterns

```bash
#!/bin/bash
# functional.sh - Functional programming in Bash

# ─── Map / Filter / Reduce ─────────────────────────────────────────────────

array_map() {
    local func=$1; shift
    local -a result=()

    for item in "$@"; do
        result+=("$("$func" "$item")")
    done

    printf '%s\n' "${result[@]}"
}

array_filter() {
    local predicate=$1; shift
    local -a result=()

    for item in "$@"; do
        "$predicate" "$item" && result+=("$item")
    done

    printf '%s\n' "${result[@]}"
}

array_reduce() {
    local func=$1 acc=$2; shift 2

    for item in "$@"; do
        acc=$("$func" "$acc" "$item")
    done

    echo "$acc"
}

# ─── Curry and Compose ────────────────────────────────────────────────────
curry() {
    local func=$1; shift
    local partial_args=("$@")
    local curried="__curried_${func}_${RANDOM}"

    eval "
${curried}() {
    '${func}' $(printf "'%s' " "${partial_args[@]}") "\$@"
}
"
    echo "$curried"
}

compose() {
    local funcs=("$@")
    local composed="__composed_${RANDOM}"

    local body="local __val=\"\$1\""
    for (( i=${#funcs[@]}-1; i>=0; i-- )); do
        body+="; __val=\"\$(${funcs[$i]} \"\$__val\")\""
    done
    body+="; echo \"\$__val\""

    eval "${composed}() { $body; }"
    echo "$composed"
}

# ─── Examples ──────────────────────────────────────────────────────────────────
to_upper() { echo "${1^^}"; }
add_prefix() { echo "prefix_${1}"; }
is_even() { (( $1 % 2 == 0 )); }
add_nums() { echo $(( $1 + $2 )); }

# map: apply to_upper to array
# array_map to_upper a b c  -> A B C

# filter: keep only even numbers
# array_filter is_even 1 2 3 4 5  -> 2 4

# reduce: sum array
# array_reduce add_nums 0 1 2 3 4 5  -> 15

# compose: upper -> prefix
# transform=$(compose to_upper add_prefix)
# $transform hello  -> prefix_HELLO
```

---

## 70.6 Command Pattern (Undo/Redo)

```bash
#!/bin/bash
# command_pattern.sh - Undo/redo with command objects

declare -a UNDO_STACK=()
declare -a REDO_STACK=()

command_execute() {
    local execute_func=$1 undo_func=$2
    shift 2
    local args=("$@")

    # Execute
    "$execute_func" "${args[@]:-}"
    local exit_code=$?

    if (( exit_code == 0 )); then
        # Push to undo stack: "func arg1 arg2..."
        UNDO_STACK+=("$undo_func $(printf '%q ' "${args[@]:-}")")
        # Clear redo stack
        REDO_STACK=()
    fi

    return $exit_code
}

command_undo() {
    local n=${#UNDO_STACK[@]}
    (( n == 0 )) && { echo "Nothing to undo"; return 1; }

    local last_idx=$(( n - 1 ))
    local cmd="${UNDO_STACK[$last_idx]}"
    unset 'UNDO_STACK[last_idx]'
    UNDO_STACK=("${UNDO_STACK[@]}")

    REDO_STACK+=("$cmd")
    eval "$cmd"
}

command_redo() {
    local n=${#REDO_STACK[@]}
    (( n == 0 )) && { echo "Nothing to redo"; return 1; }

    local last_idx=$(( n - 1 ))
    local cmd="${REDO_STACK[$last_idx]}"
    unset 'REDO_STACK[last_idx]'
    REDO_STACK=("${REDO_STACK[@]}")

    UNDO_STACK+=("$cmd")
    eval "$cmd"
}

command_history() {
    echo "Undo stack (${#UNDO_STACK[@]} items):"
    for cmd in "${UNDO_STACK[@]}"; do
        echo "  $cmd"
    done
}
```

---

## 70.7 Builder Pattern

```bash
#!/bin/bash
# builder_pattern.sh - Fluent builder interface

Request::new() {
    local instance=$1
    declare -gA "Request_${instance}=(
        [method]=GET
        [url]=''
        [body]=''
        [timeout]=30
    )"
    declare -ga "Request_${instance}_headers=()"
}

Request::method() {
    local instance=$1 method=$2
    local -n obj="Request_${instance}"
    obj[method]="$method"
    echo "$instance"
}

Request::url() {
    local instance=$1 url=$2
    local -n obj="Request_${instance}"
    obj[url]="$url"
    echo "$instance"
}

Request::header() {
    local instance=$1 key=$2 value=$3
    local -n headers="Request_${instance}_headers"
    headers+=("${key}: ${value}")
    echo "$instance"
}

Request::body() {
    local instance=$1 body=$2
    local -n obj="Request_${instance}"
    obj[body]="$body"
    echo "$instance"
}

Request::timeout() {
    local instance=$1 sec=$2
    local -n obj="Request_${instance}"
    obj[timeout]="$sec"
    echo "$instance"
}

Request::execute() {
    local instance=$1
    local -n obj="Request_${instance}"
    local -n headers="Request_${instance}_headers"

    local curl_args=(
        -s
        -X "${obj[method]}"
        --max-time "${obj[timeout]}"
    )

    for header in "${headers[@]:-}"; do
        [[ -n "$header" ]] && curl_args+=(-H "$header")
    done

    [[ -n "${obj[body]}" ]] && curl_args+=(-d "${obj[body]}")

    curl "${curl_args[@]}" "${obj[url]}"
}

# Usage:
# Request::new req1
# req1.method() already via builder
# or:
# Request::method req1 POST
# Request::url req1 https://api.example.com/data
# Request::header req1 Content-Type application/json
# Request::body req1 '{"key":"value"}'
# Request::execute req1
```

---

## 70.8 Reactive Variables

```bash
#!/bin/bash
# reactive.sh - Observable variables and computed properties

declare -A WATCHERS=()
declare -A COMPUTED=()

reactive_set() {
    local var=$1 value=$2
    local old_value="${!var:-}"

    printf -v "$var" '%s' "$value"

    # Notify watchers
    local handlers="${WATCHERS[$var]:-}"
    for handler in $handlers; do
        declare -f "$handler" &>/dev/null && "$handler" "$var" "$old_value" "$value"
    done

    # Update computed properties that depend on this var
    local dependents="${COMPUTED[deps:$var]:-}"
    for computed_var in $dependents; do
        local compute_fn="${COMPUTED[$computed_var]:-}"
        [[ -n "$compute_fn" ]] && printf -v "$computed_var" '%s' "$(eval "$compute_fn")"
    done
}

reactive_watch() {
    local var=$1 handler=$2
    local existing="${WATCHERS[$var]:-}"

    if [[ -z "$existing" ]]; then
        WATCHERS["$var"]="$handler"
    else
        WATCHERS["$var"]="$existing $handler"
    fi
}

reactive_computed() {
    local var=$1 expression=$2
    shift 2
    local deps=("$@")

    COMPUTED["$var"]="$expression"

    for dep in "${deps[@]}"; do
        local existing="${COMPUTED[deps:$dep]:-}"
        COMPUTED["deps:${dep}"]="$existing $var"
    done

    # Initial compute
    printf -v "$var" '%s' "$(eval "$expression")"
}

# ─── Example Usage ─────────────────────────────────────────────────────────
# on_price_change() { echo "Price changed: $2 -> $3"; }
# reactive_watch PRICE on_price_change
# reactive_computed TOTAL '(( PRICE * QUANTITY ))' PRICE QUANTITY
# reactive_set PRICE 100
# reactive_set QUANTITY 5
# echo $TOTAL  # -> 500
```

---

## 70.9 Mini DSL Parser

```bash
#!/bin/bash
# mini_dsl.sh - Simple DSL interpreter

# เพิ่มเติม: mini pipeline DSL
# pipeline_def file:
#   input csv /path/to/data.csv
#   filter column status = active
#   transform uppercase name
#   output json /path/to/out.json

DSL_INPUT_FILE=""
DSL_FILTER_COL=""
DSL_FILTER_OP=""
DSL_FILTER_VAL=""
DSL_TRANSFORM=()
DSL_OUTPUT_FORMAT=""
DSL_OUTPUT_FILE=""

dsl_parse() {
    local dsl_file=$1

    while IFS=' ' read -ra tokens; do
        [[ ${#tokens[@]} -eq 0 || "${tokens[0]}" =~ ^# ]] && continue

        local cmd="${tokens[0]}"
        case "$cmd" in
            input)
                DSL_INPUT_FORMAT="${tokens[1]}"
                DSL_INPUT_FILE="${tokens[2]}"
                ;;
            filter)
                DSL_FILTER_COL="${tokens[2]}"
                DSL_FILTER_OP="${tokens[3]}"
                DSL_FILTER_VAL="${tokens[4]}"
                ;;
            transform)
                DSL_TRANSFORM+=("${tokens[1]}:${tokens[2]}")
                ;;
            output)
                DSL_OUTPUT_FORMAT="${tokens[1]}"
                DSL_OUTPUT_FILE="${tokens[2]}"
                ;;
            *)
                echo "DSL WARN: unknown command: $cmd" >&2
                ;;
        esac
    done < "$dsl_file"
}

dsl_execute() {
    echo "Executing DSL pipeline:"
    echo "  Input: $DSL_INPUT_FORMAT $DSL_INPUT_FILE"
    echo "  Filter: $DSL_FILTER_COL $DSL_FILTER_OP $DSL_FILTER_VAL"
    echo "  Transforms: ${DSL_TRANSFORM[*]:-none}"
    echo "  Output: $DSL_OUTPUT_FORMAT $DSL_OUTPUT_FILE"

    local data
    case "$DSL_INPUT_FORMAT" in
        csv) data=$(cut -d, -f1- "$DSL_INPUT_FILE") ;;
        json) data=$(cat "$DSL_INPUT_FILE") ;;
        *) echo "Unknown format: $DSL_INPUT_FORMAT" >&2; return 1 ;;
    esac

    # Apply transforms
    for transform in "${DSL_TRANSFORM[@]}"; do
        local op="${transform%%:*}" target="${transform#*:}"
        case "$op" in
            uppercase) data=$(echo "$data" | awk -v col="$target" 'BEGIN{FS=OFS=","} NR>1{$col=toupper($col)}1') ;;
            lowercase) data=$(echo "$data" | awk -v col="$target" 'BEGIN{FS=OFS=","} NR>1{$col=tolower($col)}1') ;;
        esac
    done

    # Output
    case "$DSL_OUTPUT_FORMAT" in
        json) echo "$data" | python3 -c 'import sys,csv,json; r=csv.DictReader(sys.stdin); json.dump(list(r),sys.stdout,indent=2)' > "$DSL_OUTPUT_FILE" ;;
        csv)  echo "$data" > "$DSL_OUTPUT_FILE" ;;
    esac

    echo "Pipeline complete: $DSL_OUTPUT_FILE"
}
```

---

## 70.10 Exercises

### Exercise 1: OOP Framework
สร้าง OOP framework ที่:
- Inheritance (extends)
- Method overriding
- Private/public visibility
- toString() convention

### Exercise 2: Actor Model
สร้าง actor model ที่:
- Actors as background processes
- Message passing via FIFOs
- Supervision tree
- Dead letter queue

### Exercise 3: Template Engine
สร้าง template engine ที่:
- `{{variable}}` substitution
- `{{#if condition}}` blocks
- `{{#each array}}` loops
- Include partials

---

## สรุป Part 70

✅ Object-oriented Bash: class via associative arrays, namespaced methods, constructor/destructor
✅ Event system: on/off/emit/once with multiple handlers per event
✅ State machine: transition table, guard conditions, action hooks
✅ Plugin architecture: register, hook, dynamic load, capability registry
✅ Functional patterns: map/filter/reduce over arrays; curry/compose function factories
✅ Command pattern: execute with undo/redo stack
✅ Builder pattern: fluent HTTP request builder
✅ Reactive variables: watchers and computed properties
✅ Mini DSL parser/executor with tokenizer and pipeline stages

---

**→ Part 71: Database Operations and Data Processing**
