# Part 78: Automation Framework and Task Orchestration
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 78.1 Task Queue with SQLite

```bash
#!/bin/bash
# task_queue.sh - Persistent task queue backed by SQLite

set -euo pipefail

QUEUE_DB="${QUEUE_DB:-/tmp/task_queue.db}"

queue_init() {
    sqlite3 "$QUEUE_DB" <<'SQL'
CREATE TABLE IF NOT EXISTS tasks (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    queue     TEXT    NOT NULL DEFAULT 'default',
    payload   TEXT    NOT NULL,
    priority  INTEGER NOT NULL DEFAULT 5,
    status    TEXT    NOT NULL DEFAULT 'pending',
    worker_id TEXT,
    attempts  INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    scheduled_at DATETIME NOT NULL DEFAULT (datetime('now')),
    started_at   DATETIME,
    finished_at  DATETIME,
    error_msg TEXT,
    created_at DATETIME NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status, queue, priority DESC, scheduled_at);
SQL
    echo "Queue DB initialized: $QUEUE_DB"
}

queue_enqueue() {
    local payload=$1 queue=${2:-default} priority=${3:-5} delay_seconds=${4:-0}

    local scheduled_at
    if (( delay_seconds > 0 )); then
        scheduled_at=$(date -d "+${delay_seconds} seconds" '+%Y-%m-%d %H:%M:%S' 2>/dev/null || \
                       date -v +"${delay_seconds}"S '+%Y-%m-%d %H:%M:%S')
    else
        scheduled_at=$(date '+%Y-%m-%d %H:%M:%S')
    fi

    sqlite3 "$QUEUE_DB" \
        "INSERT INTO tasks (queue, payload, priority, scheduled_at) \
         VALUES ('$queue', '$(echo "$payload" | sed "s/'/''''/g")', $priority, '$scheduled_at');"
    local task_id
    task_id=$(sqlite3 "$QUEUE_DB" "SELECT last_insert_rowid();")
    echo "$task_id"
}

queue_dequeue() {
    local queue=${1:-default} worker_id=${2:-$$}

    local task_id
    task_id=$(sqlite3 "$QUEUE_DB" <<SQL
BEGIN IMMEDIATE;
SELECT id FROM tasks
WHERE queue = '$queue'
  AND status = 'pending'
  AND attempts < max_attempts
  AND scheduled_at <= datetime('now')
ORDER BY priority DESC, id ASC
LIMIT 1;
SQL
)

    [[ -z "$task_id" ]] && return 1

    sqlite3 "$QUEUE_DB" <<SQL
UPDATE tasks
   SET status = 'running',
       worker_id = '$worker_id',
       started_at = datetime('now'),
       attempts = attempts + 1
 WHERE id = $task_id
   AND status = 'pending';
SQL

    sqlite3 "$QUEUE_DB" "SELECT id, payload FROM tasks WHERE id = $task_id;" | tr '|' '	'
}

queue_complete() {
    local task_id=$1
    sqlite3 "$QUEUE_DB" \
        "UPDATE tasks SET status='done', finished_at=datetime('now') WHERE id=$task_id;"
}

queue_fail() {
    local task_id=$1 error=${2:-unknown}
    local max_attempts; max_attempts=$(sqlite3 "$QUEUE_DB" "SELECT max_attempts FROM tasks WHERE id=$task_id;")
    local attempts; attempts=$(sqlite3 "$QUEUE_DB" "SELECT attempts FROM tasks WHERE id=$task_id;")

    local new_status
    if (( attempts >= max_attempts )); then
        new_status="dead"
    else
        new_status="pending"
    fi

    sqlite3 "$QUEUE_DB" \
        "UPDATE tasks SET status='$new_status', finished_at=datetime('now'), \
         error_msg='$(echo "$error" | sed "s/'/''''/g")' WHERE id=$task_id;"
}

queue_stats() {
    local queue=${1:-default}
    echo "Queue: $queue"
    sqlite3 "$QUEUE_DB" \
        "SELECT status, COUNT(*) as count FROM tasks WHERE queue='$queue' GROUP BY status ORDER BY status;" | \
        awk -F'|' '{printf "  %-12s %d\n", $1, $2}'
}

queue_requeue_dead() {
    local queue=${1:-default}
    sqlite3 "$QUEUE_DB" \
        "UPDATE tasks SET status='pending', attempts=0, error_msg=NULL \
         WHERE status='dead' AND queue='$queue';"
    echo "Re-queued dead tasks in: $queue"
}
```

---

## 78.2 Worker Pool

```bash
#!/bin/bash
# worker_pool.sh - Multi-worker pool that drains a queue

WORKER_POOL_SIZE="${WORKER_POOL_SIZE:-4}"
WORKER_POLL_INTERVAL="${WORKER_POLL_INTERVAL:-1}"
WORKER_LOG="${WORKER_LOG:-/tmp/worker.log}"

declare -A WORKER_PIDS=()

worker_log() {
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] [worker-$$] $*" >> "$WORKER_LOG"
}

worker_run() {
    local queue=${1:-default}
    local worker_id="worker-$$"

    worker_log "Started on queue: $queue"

    while true; do
        local row
        row=$(queue_dequeue "$queue" "$worker_id" 2>/dev/null) || {
            sleep "$WORKER_POLL_INTERVAL"
            continue
        }

        local task_id; task_id=$(echo "$row" | cut -f1)
        local payload;  payload=$(echo "$row" | cut -f2)

        worker_log "Processing task $task_id: $payload"

        if dispatch_task "$task_id" "$payload"; then
            queue_complete "$task_id"
            worker_log "Completed task $task_id"
        else
            queue_fail "$task_id" "dispatch_task returned non-zero"
            worker_log "Failed task $task_id"
        fi
    done
}

dispatch_task() {
    local task_id=$1 payload=$2
    # Override this function in your application
    echo "Executing task $task_id: $payload"
    return 0
}

pool_start() {
    local queue=${1:-default} size=${2:-$WORKER_POOL_SIZE}

    echo "Starting $size workers for queue: $queue"
    for (( i=1; i<=size; i++ )); do
        worker_run "$queue" &
        local pid=$!
        WORKER_PIDS["worker-${i}"]=$pid
        echo "  Started worker-${i} (pid $pid)"
    done
}

pool_stop() {
    echo "Stopping worker pool..."
    for name in "${!WORKER_PIDS[@]}"; do
        local pid="${WORKER_PIDS[$name]}"
        kill "$pid" 2>/dev/null || true
        echo "  Stopped $name (pid $pid)"
    done
    WORKER_PIDS=()
}

pool_status() {
    echo "Worker pool status:"
    for name in "${!WORKER_PIDS[@]}"; do
        local pid="${WORKER_PIDS[$name]}"
        if kill -0 "$pid" 2>/dev/null; then
            echo "  $name (pid $pid): running"
        else
            echo "  $name (pid $pid): dead"
        fi
    done
}

trap pool_stop EXIT INT TERM
```

---

## 78.3 Pipeline DSL

```bash
#!/bin/bash
# pipeline.sh - Step-based pipeline with dependency ordering

declare -A PIPELINE_STEPS=()
declare -A PIPELINE_DEPS=()
declare -A PIPELINE_STATUS=()
PIPELINE_NAME=""

pipeline_init() {
    PIPELINE_NAME=$1
    PIPELINE_STEPS=()
    PIPELINE_DEPS=()
    PIPELINE_STATUS=()
    echo "Pipeline: $PIPELINE_NAME"
}

pipeline_step() {
    local name=$1 fn=$2
    shift 2
    local deps=("$@")

    PIPELINE_STEPS["$name"]="$fn"
    PIPELINE_DEPS["$name"]="${deps[*]:-}"
    PIPELINE_STATUS["$name"]="pending"
}

_pipeline_can_run() {
    local step=$1
    local deps="${PIPELINE_DEPS[$step]:-}"

    [[ -z "$deps" ]] && return 0

    for dep in $deps; do
        [[ "${PIPELINE_STATUS[$dep]:-}" != "done" ]] && return 1
    done
    return 0
}

_pipeline_all_done() {
    for step in "${!PIPELINE_STEPS[@]}"; do
        [[ "${PIPELINE_STATUS[$step]}" != "done" && \
           "${PIPELINE_STATUS[$step]}" != "failed" ]] && return 1
    done
    return 0
}

pipeline_run() {
    local failed=0
    local max_iterations=$(( ${#PIPELINE_STEPS[@]} * 2 + 1 ))
    local iteration=0

    echo "Running pipeline: $PIPELINE_NAME"

    while ! _pipeline_all_done; do
        (( iteration++ ))
        if (( iteration > max_iterations )); then
            echo "ERROR: Pipeline stalled (possible cycle)" >&2
            return 1
        fi

        local progress=0
        for step in "${!PIPELINE_STEPS[@]}"; do
            [[ "${PIPELINE_STATUS[$step]}" != "pending" ]] && continue
            _pipeline_can_run "$step" || continue

            local fn="${PIPELINE_STEPS[$step]}"
            echo "  [$(date '+%H:%M:%S')] Running: $step"
            PIPELINE_STATUS["$step"]="running"

            if "$fn" "$step"; then
                PIPELINE_STATUS["$step"]="done"
                echo "  [$(date '+%H:%M:%S')] Done:    $step"
            else
                PIPELINE_STATUS["$step"]="failed"
                echo "  [$(date '+%H:%M:%S')] FAILED:  $step" >&2
                (( failed++ ))
            fi
            (( progress++ ))
        done

        (( progress == 0 )) && sleep 0.1
    done

    echo ""
    echo "Pipeline summary:"
    for step in "${!PIPELINE_STEPS[@]}"; do
        printf '  %-30s %s\n' "$step" "${PIPELINE_STATUS[$step]}"
    done

    (( failed == 0 ))
}
```

---

## 78.4 Event Bus

```bash
#!/bin/bash
# event_bus.sh - In-process pub/sub event bus

declare -A EVENT_HANDLERS=()
declare -a EVENT_HISTORY=()
EVENT_LOG="${EVENT_LOG:-}"

event_on() {
    local event=$1 handler=$2
    local existing="${EVENT_HANDLERS[$event]:-}"
    EVENT_HANDLERS["$event"]="${existing:+${existing} }${handler}"
}

event_off() {
    local event=$1 handler=$2
    local handlers="${EVENT_HANDLERS[$event]:-}"
    EVENT_HANDLERS["$event"]=$(echo "$handlers" | tr ' ' '\n' | grep -v "^${handler}$" | tr '\n' ' ')
}

event_emit() {
    local event=$1
    shift
    local -a args=("$@")

    EVENT_HISTORY+=("$(date '+%Y-%m-%dT%H:%M:%S') $event ${args[*]:-}")
    [[ -n "$EVENT_LOG" ]] && echo "EVENT: $event ${args[*]:-}" >> "$EVENT_LOG"

    local handlers="${EVENT_HANDLERS[$event]:-}"
    local wildcard="${EVENT_HANDLERS['*']:-}"

    for handler in $handlers $wildcard; do
        [[ -z "$handler" ]] && continue
        "$handler" "$event" "${args[@]:-}" 2>/dev/null || true
    done
}

event_history() {
    local filter=${1:-}
    if [[ -n "$filter" ]]; then
        printf '%s\n' "${EVENT_HISTORY[@]}" | grep "$filter"
    else
        printf '%s\n' "${EVENT_HISTORY[@]}"
    fi
}

# File-based inter-process event bus
FILE_BUS_DIR="${FILE_BUS_DIR:-/tmp/event_bus}"

file_bus_publish() {
    local topic=$1
    shift
    local payload=$(printf '%s\n' "$@" | jq -Rs .)

    mkdir -p "${FILE_BUS_DIR}/${topic}"
    local ts; ts=$(date +%s%N)
    echo "{\"ts\":\"$ts\",\"payload\":$payload}" > \
        "${FILE_BUS_DIR}/${topic}/${ts}.json"
}

file_bus_subscribe() {
    local topic=$1 handler=$2 timeout_sec=${3:-30}
    local last_seen=0
    local deadline=$(( $(date +%s) + timeout_sec ))

    mkdir -p "${FILE_BUS_DIR}/${topic}"

    while (( $(date +%s) < deadline )); do
        for msg_file in "${FILE_BUS_DIR}/${topic}"/*.json 2>/dev/null; do
            [[ -f "$msg_file" ]] || continue
            local ts; ts=$(basename "$msg_file" .json)
            (( ts > last_seen )) || continue

            local payload; payload=$(jq -r '.payload' "$msg_file" 2>/dev/null)
            "$handler" "$topic" "$payload"
            last_seen=$ts
        done
        sleep 0.5
    done
}
```

---

## 78.5 State Machine

```bash
#!/bin/bash
# state_machine.sh - Finite state machine framework

declare -A FSM_TRANSITIONS=()
declare -A FSM_ENTRY_HOOKS=()
declare -A FSM_EXIT_HOOKS=()
FSM_STATE=""
FSM_NAME=""

fsm_init() {
    FSM_NAME=$1
    FSM_STATE=$2
    echo "FSM '$FSM_NAME' initialized in state: $FSM_STATE"
}

fsm_add_transition() {
    local from=$1 event=$2 to=$3
    local key="${from}:${event}"
    FSM_TRANSITIONS["$key"]="$to"
}

fsm_on_enter() {
    local state=$1 hook=$2
    FSM_ENTRY_HOOKS["$state"]="$hook"
}

fsm_on_exit() {
    local state=$1 hook=$2
    FSM_EXIT_HOOKS["$state"]="$hook"
}

fsm_send() {
    local event=$1
    shift
    local key="${FSM_STATE}:${event}"

    local next_state="${FSM_TRANSITIONS[$key]:-}"
    if [[ -z "$next_state" ]]; then
        echo "FSM: No transition from '$FSM_STATE' on event '$event'" >&2
        return 1
    fi

    # Run exit hook
    local exit_hook="${FSM_EXIT_HOOKS[$FSM_STATE]:-}"
    [[ -n "$exit_hook" ]] && "$exit_hook" "$FSM_STATE" "$event" "$@" 2>/dev/null || true

    local prev_state=$FSM_STATE
    FSM_STATE=$next_state

    # Run entry hook
    local entry_hook="${FSM_ENTRY_HOOKS[$FSM_STATE]:-}"
    [[ -n "$entry_hook" ]] && "$entry_hook" "$FSM_STATE" "$event" "$@" 2>/dev/null || true

    echo "FSM: $prev_state --[$event]--> $FSM_STATE"
}

fsm_state() { echo "$FSM_STATE"; }

fsm_can_transition() {
    local event=$1
    local key="${FSM_STATE}:${event}"
    [[ -n "${FSM_TRANSITIONS[$key]:-}" ]]
}

# Example: deployment pipeline state machine
fsm_demo_deployment() {
    fsm_init "deployment" "idle"

    fsm_add_transition "idle"        "start"    "building"
    fsm_add_transition "building"    "success"  "testing"
    fsm_add_transition "building"    "failure"  "failed"
    fsm_add_transition "testing"     "success"  "deploying"
    fsm_add_transition "testing"     "failure"  "failed"
    fsm_add_transition "deploying"   "success"  "done"
    fsm_add_transition "deploying"   "failure"  "failed"
    fsm_add_transition "done"        "reset"    "idle"
    fsm_add_transition "failed"      "reset"    "idle"

    fsm_on_enter "building"  _demo_build
    fsm_on_enter "testing"   _demo_test
    fsm_on_enter "deploying" _demo_deploy

    fsm_send "start"
}

_demo_build()   { echo "  -> Running build..."; }
_demo_test()    { echo "  -> Running tests..."; }
_demo_deploy()  { echo "  -> Deploying..."; }
```

---

## 78.6 Circuit Breaker

```bash
#!/bin/bash
# circuit_breaker.sh - Circuit breaker for fault-tolerant automation

CB_STATE_DIR="${CB_STATE_DIR:-/tmp/circuit_breakers}"
mkdir -p "$CB_STATE_DIR"

# States: CLOSED (normal), OPEN (tripped), HALF_OPEN (testing)

cb_state_file() { echo "${CB_STATE_DIR}/${1}.json"; }

cb_init() {
    local name=$1
    local fail_threshold=${2:-5}
    local reset_timeout=${3:-60}

    local file; file=$(cb_state_file "$name")
    [[ -f "$file" ]] && return 0

    jq -n \
        --arg name "$name" \
        --argjson threshold "$fail_threshold" \
        --argjson timeout "$reset_timeout" \
        '{name:$name, state:"CLOSED", failures:0, successes:0,
          fail_threshold:$threshold, reset_timeout:$timeout,
          last_state_change:null, last_failure:null}' > "$file"
}

cb_get_state() {
    local name=$1
    local file; file=$(cb_state_file "$name")
    jq -r '.state' "$file" 2>/dev/null || echo "CLOSED"
}

cb_call() {
    local name=$1 fn=$2
    shift 2
    local -a args=("$@")

    local file; file=$(cb_state_file "$name")
    local state; state=$(cb_get_state "$name")
    local now; now=$(date +%s)

    if [[ "$state" == "OPEN" ]]; then
        local last_change; last_change=$(jq -r '.last_state_change' "$file")
        local reset_timeout; reset_timeout=$(jq -r '.reset_timeout' "$file")
        if (( now - last_change >= reset_timeout )); then
            jq --argjson now "$now" '.state="HALF_OPEN" | .last_state_change=$now' "$file" > "${file}.tmp"
            mv "${file}.tmp" "$file"
            state="HALF_OPEN"
        else
            echo "Circuit breaker OPEN for: $name" >&2
            return 1
        fi
    fi

    if "$fn" "${args[@]}"; then
        # Success
        jq --argjson now "$now" '.failures=0 | .successes+=1 | .state="CLOSED" | .last_state_change=$now' \
            "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
        return 0
    else
        # Failure
        local failures; failures=$(jq '.failures + 1' "$file")
        local threshold; threshold=$(jq '.fail_threshold' "$file")

        if (( failures >= threshold )); then
            jq --argjson now "$now" --argjson f "$failures" \
                '.failures=$f | .state="OPEN" | .last_state_change=$now | .last_failure=$now' \
                "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
            echo "Circuit breaker OPENED for: $name (failures=$failures)" >&2
        else
            jq --argjson now "$now" --argjson f "$failures" \
                '.failures=$f | .last_failure=$now' \
                "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
        fi
        return 1
    fi
}

cb_reset() {
    local name=$1
    local file; file=$(cb_state_file "$name")
    local now; now=$(date +%s)
    jq --argjson now "$now" '.state="CLOSED" | .failures=0 | .last_state_change=$now' \
        "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
    echo "Circuit breaker reset: $name"
}

cb_status() {
    local name=$1
    jq -r '"State: " + .state + "  Failures: " + (.failures|tostring) + "/" + (.fail_threshold|tostring)' \
        "$(cb_state_file "$name")" 2>/dev/null
}
```

---

## 78.7 Idempotency and Step Records

```bash
#!/bin/bash
# idempotency.sh - Run steps exactly once (idempotent automation)

STEP_DB="${STEP_DB:-/tmp/steps.db}"

steps_init() {
    sqlite3 "$STEP_DB" <<'SQL'
CREATE TABLE IF NOT EXISTS steps (
    run_id    TEXT NOT NULL,
    step_name TEXT NOT NULL,
    status    TEXT NOT NULL DEFAULT 'done',
    output    TEXT,
    run_at    DATETIME NOT NULL DEFAULT (datetime('now')),
    PRIMARY KEY (run_id, step_name)
);
SQL
}

step_was_run() {
    local run_id=$1 step=$2
    local count; count=$(sqlite3 "$STEP_DB" \
        "SELECT COUNT(*) FROM steps WHERE run_id='$run_id' AND step_name='$step' AND status='done';")
    (( count > 0 ))
}

step_record() {
    local run_id=$1 step=$2 output=${3:-}
    sqlite3 "$STEP_DB" \
        "INSERT OR REPLACE INTO steps (run_id, step_name, status, output) \
         VALUES ('$run_id', '$step', 'done', '$(echo "$output" | sed "s/'/''''/g")');" 
}

run_step() {
    local run_id=$1 step_name=$2 fn=$3
    shift 3
    local -a args=("$@")

    if step_was_run "$run_id" "$step_name"; then
        echo "[SKIP] $step_name (already done in run $run_id)"
        return 0
    fi

    echo "[RUN ] $step_name"
    local output
    output=$("$fn" "${args[@]}" 2>&1)
    local exit_code=$?

    if (( exit_code == 0 )); then
        step_record "$run_id" "$step_name" "$output"
        echo "[DONE] $step_name"
    else
        echo "[FAIL] $step_name: $output" >&2
    fi
    return $exit_code
}

steps_list() {
    local run_id=$1
    sqlite3 -column -header "$STEP_DB" \
        "SELECT step_name, status, run_at FROM steps WHERE run_id='$run_id' ORDER BY run_at;"
}

steps_reset() {
    local run_id=$1 step=${2:-}
    if [[ -n "$step" ]]; then
        sqlite3 "$STEP_DB" "DELETE FROM steps WHERE run_id='$run_id' AND step_name='$step';"
        echo "Reset step '$step' in run '$run_id'"
    else
        sqlite3 "$STEP_DB" "DELETE FROM steps WHERE run_id='$run_id';"
        echo "Reset all steps for run '$run_id'"
    fi
}
```

---

## 78.8 Retry Policy Library

```bash
#!/bin/bash
# retry.sh - Configurable retry policies

retry_linear() {
    local fn=$1 attempts=${2:-3} delay=${3:-5}
    shift 3
    local -a args=("$@")
    local i
    for (( i=1; i<=attempts; i++ )); do
        "$fn" "${args[@]}" && return 0
        (( i < attempts )) && sleep "$delay"
        echo "Attempt $i/$attempts failed" >&2
    done
    return 1
}

retry_exponential() {
    local fn=$1 attempts=${2:-5} base=${3:-2} max_delay=${4:-60}
    shift 4
    local -a args=("$@")
    local i delay
    for (( i=1; i<=attempts; i++ )); do
        "$fn" "${args[@]}" && return 0
        delay=$(( base ** (i-1) ))
        (( delay > max_delay )) && delay=$max_delay
        (( i < attempts )) && sleep "$delay"
        echo "Attempt $i/$attempts failed, next in ${delay}s" >&2
    done
    return 1
}

retry_with_jitter() {
    local fn=$1 attempts=${2:-5} base=${3:-2}
    shift 3
    local -a args=("$@")
    local i delay jitter
    for (( i=1; i<=attempts; i++ )); do
        "$fn" "${args[@]}" && return 0
        delay=$(( base ** (i-1) ))
        jitter=$(( RANDOM % (delay + 1) ))
        local total=$(( delay + jitter ))
        (( i < attempts )) && sleep "$total"
        echo "Attempt $i/$attempts failed, next in ${total}s" >&2
    done
    return 1
}

retry_until_success() {
    local fn=$1 timeout_sec=${2:-300} delay=${3:-5}
    shift 3
    local -a args=("$@")
    local deadline=$(( $(date +%s) + timeout_sec ))

    while (( $(date +%s) < deadline )); do
        "$fn" "${args[@]}" && return 0
        sleep "$delay"
    done

    echo "Timed out after ${timeout_sec}s" >&2
    return 1
}
```

---

## 78.9 Exercises

### Exercise 1: Complete Automation Pipeline
สร้าง pipeline สำหรับ deploy web app:
- `step: build` → compile assets
- `step: test` → เซ็น unit test
- `step: push_image` → push Docker image
- `step: migrate_db` → รัน migration
- `step: deploy` → swap containers
- idempotent: รันซ้ำ resume ได้

### Exercise 2: Queue + Worker System
สร้าง task processing system:
- Producer enqueues tasks from CSV
- 3 parallel workers drain queue
- Failed tasks retry 3 times
- Dead tasks alert via Slack

### Exercise 3: State Machine CI/CD
สร้าง CI/CD state machine:
- States: idle → pending → running → done|failed
- Circuit breaker ป้องกัน flapping deploy
- Event bus ส่ง Slack/email notifications

---

## สรุป Part 78

✅ Task queue: SQLite-backed, priority, delay, max_attempts, status tracking
┅ Worker pool: spawn N workers, poll queue, dispatch_task hook, start/stop/status
┅ Pipeline DSL: step registration, dependency graph, topological execution
┅ Event bus: in-process pub/sub with wildcard handler + file-based inter-process bus
┅ State machine: define states/transitions/hooks, CLOSED/OPEN/HALF_OPEN logic
┅ Circuit breaker: fail_threshold, reset_timeout, OPEN/HALF_OPEN/CLOSED lifecycle
┅ Idempotency: SQLite-backed step records, run_step skips already-done steps
┅ Retry policies: linear, exponential, jitter, until-success with timeout

---

**→ Part 79: Container and Kubernetes Automation**
