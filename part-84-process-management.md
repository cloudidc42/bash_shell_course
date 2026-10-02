# Part 84: Advanced Process Management and Job Control
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 84.1 Process Inspection

```bash
#!/bin/bash
# proc_inspect.sh - Process inspection and management

set -euo pipefail

proc_info() {
    local pid=$1
    [[ -d "/proc/$pid" ]] || { echo "Process $pid not found"; return 1; }

    printf "PID:     %s\n" "$pid"
    printf "Name:    %s\n" "$(cat /proc/$pid/comm 2>/dev/null)"
    printf "State:   %s\n" "$(awk '{print $3}' /proc/$pid/stat 2>/dev/null)"
    printf "PPID:    %s\n" "$(awk '{print $4}' /proc/$pid/stat 2>/dev/null)"
    printf "Threads: %s\n" "$(awk '{print $20}' /proc/$pid/stat 2>/dev/null)"
    printf "RSS:     %s kB\n" "$(awk '/VmRSS/{print $2}' /proc/$pid/status 2>/dev/null)"
    printf "VSZ:     %s kB\n" "$(awk '/VmSize/{print $2}' /proc/$pid/status 2>/dev/null)"
    printf "Cmdline: %s\n" "$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)"
}

proc_children() {
    local ppid=$1
    awk -v ppid="$ppid" '$4==ppid{print $1}' /proc/*/stat 2>/dev/null | \
        grep -oP '\d+' || true
}

proc_tree() {
    local pid=${1:-1} indent=${2:-0}
    local name; name=$(cat /proc/$pid/comm 2>/dev/null || echo "?")
    printf '%*s%s (%s)\n' $((indent*2)) '' "$name" "$pid"
    for child in $(proc_children "$pid"); do
        proc_tree "$child" $(( indent + 1 ))
    done
}

proc_find_by_name() {
    local name=$1
    pgrep -a "$name" 2>/dev/null || \
    ps aux | awk -v n="$name" '$0 ~ n && !/awk/{print $2, $11}'
}

proc_find_by_port() {
    local port=$1
    ss -tlnp 2>/dev/null | awk -v p=":$port" '$0 ~ p {print}' || \
    lsof -i :"$port" -t 2>/dev/null
}

proc_memory_map() {
    local pid=$1
    cat /proc/$pid/maps 2>/dev/null | \
        awk '{print $NF, $1}' | grep -v '^$' | sort -u | head -30
}

proc_open_files() {
    local pid=$1
    ls -la /proc/$pid/fd 2>/dev/null | awk 'NR>1{print $NF, "->", $11}'
}

proc_cpu_time() {
    local pid=$1
    local stat; stat=$(cat /proc/$pid/stat 2>/dev/null)
    local utime stime cutime cstime
    utime=$(echo "$stat" | awk '{print $14}')
    stime=$(echo "$stat" | awk '{print $15}')
    local clk_tck=100
    printf "CPU user: %.2fs  sys: %.2fs\n" \
        "$(echo "$utime $clk_tck" | awk '{printf "%.2f", $1/$2}')" \
        "$(echo "$stime $clk_tck" | awk '{printf "%.2f", $1/$2}')"
}
```

---

## 84.2 Job Control

```bash
#!/bin/bash
# job_control.sh - Advanced job and background process control

declare -A JOB_PIDS=()
declare -A JOB_STATUS=()
declare -A JOB_OUTPUTS=()

job_start() {
    local name=$1; shift
    local -a cmd=("$@")
    local output_file; output_file=$(mktemp)

    "${cmd[@]}" > "$output_file" 2>&1 &
    local pid=$!

    JOB_PIDS["$name"]=$pid
    JOB_STATUS["$name"]="running"
    JOB_OUTPUTS["$name"]="$output_file"

    echo "Started job '$name' (pid $pid)"
}

job_wait() {
    local name=$1 timeout_sec=${2:-0}
    local pid="${JOB_PIDS[$name]:-}"

    [[ -z "$pid" ]] && { echo "Job '$name' not found" >&2; return 1; }

    if (( timeout_sec > 0 )); then
        local deadline=$(( $(date +%s) + timeout_sec ))
        while kill -0 "$pid" 2>/dev/null && (( $(date +%s) < deadline )); do
            sleep 0.5
        done
        if kill -0 "$pid" 2>/dev/null; then
            echo "Job '$name' timed out" >&2
            job_kill "$name"
            return 1
        fi
    else
        wait "$pid" 2>/dev/null
    fi

    local exit_code=$?
    JOB_STATUS["$name"]=$(( exit_code == 0 )) && echo "done" || echo "failed"
    return $exit_code
}

job_kill() {
    local name=$1 signal=${2:-TERM}
    local pid="${JOB_PIDS[$name]:-}"

    [[ -z "$pid" ]] && return 0
    kill -"$signal" "$pid" 2>/dev/null || true
    sleep 1
    kill -0 "$pid" 2>/dev/null && kill -KILL "$pid" 2>/dev/null || true
    JOB_STATUS["$name"]="killed"
    echo "Killed job '$name' (pid $pid)"
}

job_output() {
    local name=$1
    local f="${JOB_OUTPUTS[$name]:-}"
    [[ -n "$f" && -f "$f" ]] && cat "$f"
}

job_status() {
    local name=${1:-}
    if [[ -n "$name" ]]; then
        local pid="${JOB_PIDS[$name]:-}"
        local status="${JOB_STATUS[$name]:-unknown}"
        if [[ -n "$pid" ]] && kill -0 "$pid" 2>/dev/null; then
            status="running"
        fi
        printf "%-20s pid=%-8s status=%s\n" "$name" "${pid:-N/A}" "$status"
    else
        for jname in "${!JOB_PIDS[@]}"; do
            job_status "$jname"
        done
    fi
}

job_wait_all() {
    local timeout_sec=${1:-0}
    local failed=0
    for name in "${!JOB_PIDS[@]}"; do
        job_wait "$name" "$timeout_sec" || (( failed++ ))
    done
    (( failed == 0 ))
}

job_cleanup() {
    for name in "${!JOB_PIDS[@]}"; do
        local pid="${JOB_PIDS[$name]}"
        kill -0 "$pid" 2>/dev/null && kill -TERM "$pid" 2>/dev/null || true
        local f="${JOB_OUTPUTS[$name]:-}"
        [[ -n "$f" ]] && rm -f "$f"
    done
    JOB_PIDS=()
    JOB_STATUS=()
    JOB_OUTPUTS=()
}

trap job_cleanup EXIT INT TERM
```

---

## 84.3 Process Supervisor

```bash
#!/bin/bash
# supervisor.sh - Simple process supervisor (like supervisord)

SUPERVISOR_DIR="${SUPERVISOR_DIR:-/tmp/supervisor}"
SUPERVISOR_LOG="${SUPERVISOR_DIR}/supervisor.log"

mkdir -p "$SUPERVISOR_DIR"

declare -A SV_PROGRAMS=()
declare -A SV_PIDS=()
declare -A SV_RESTARTS=()
SV_MAX_RESTARTS="${SV_MAX_RESTARTS:-5}"

sv_log() {
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] $*" | tee -a "$SUPERVISOR_LOG"
}

sv_register() {
    local name=$1; shift
    SV_PROGRAMS["$name"]="$*"
    SV_RESTARTS["$name"]=0
    sv_log "Registered: $name -> $*"
}

sv_start_one() {
    local name=$1
    local cmd="${SV_PROGRAMS[$name]:-}"
    [[ -z "$cmd" ]] && { sv_log "Unknown program: $name"; return 1; }

    local logfile="${SUPERVISOR_DIR}/${name}.log"
    eval "$cmd" >> "$logfile" 2>&1 &
    SV_PIDS["$name"]=$!
    sv_log "Started $name (pid ${SV_PIDS[$name]})"
}

sv_stop_one() {
    local name=$1
    local pid="${SV_PIDS[$name]:-}"
    [[ -z "$pid" ]] && return 0
    kill -TERM "$pid" 2>/dev/null || true
    sleep 2
    kill -0 "$pid" 2>/dev/null && kill -KILL "$pid" 2>/dev/null || true
    unset "SV_PIDS[$name]"
    sv_log "Stopped $name"
}

sv_restart_one() {
    local name=$1
    sv_stop_one "$name"
    sleep 1
    sv_start_one "$name"
}

sv_monitor_loop() {
    sv_log "Supervisor started"

    for name in "${!SV_PROGRAMS[@]}"; do
        sv_start_one "$name"
    done

    while true; do
        sleep 5
        for name in "${!SV_PROGRAMS[@]}"; do
            local pid="${SV_PIDS[$name]:-}"
            if [[ -z "$pid" ]] || ! kill -0 "$pid" 2>/dev/null; then
                local restarts="${SV_RESTARTS[$name]:-0}"
                if (( restarts < SV_MAX_RESTARTS )); then
                    sv_log "Restarting $name (attempt $((restarts+1))/$SV_MAX_RESTARTS)"
                    SV_RESTARTS["$name"]=$(( restarts + 1 ))
                    sv_start_one "$name"
                else
                    sv_log "FATAL: $name exceeded max restarts ($SV_MAX_RESTARTS)"
                fi
            fi
        done
    done
}

sv_status() {
    echo "=== Supervisor Status ==="
    for name in "${!SV_PROGRAMS[@]}"; do
        local pid="${SV_PIDS[$name]:-N/A}"
        local alive="down"
        [[ "$pid" != "N/A" ]] && kill -0 "$pid" 2>/dev/null && alive="up"
        printf "  %-20s pid=%-8s %s  restarts=%s\n" \
            "$name" "$pid" "$alive" "${SV_RESTARTS[$name]:-0}"
    done
}
```

---

## 84.4 Signal Handling

```bash
#!/bin/bash
# signals.sh - Comprehensive signal handling patterns

declare -a CLEANUP_FUNS=()
declare -i SHUTDOWN_REQUESTED=0

register_cleanup() {
    CLEANUP_FUNS+=("$1")
}

run_cleanups() {
    for fn in "${CLEANUP_FUNS[@]}"; do
        "$fn" 2>/dev/null || true
    done
}

handle_sigterm() {
    echo "Received SIGTERM, shutting down gracefully..." >&2
    SHUTDOWN_REQUESTED=1
    run_cleanups
    exit 0
}

handle_sigint() {
    echo "Received SIGINT" >&2
    SHUTDOWN_REQUESTED=1
    run_cleanups
    exit 130
}

handle_sighup() {
    echo "Received SIGHUP - reloading config..." >&2
    reload_config 2>/dev/null || true
}

setup_signal_handlers() {
    trap handle_sigterm TERM
    trap handle_sigint  INT
    trap handle_sighup  HUP
    trap run_cleanups   EXIT
}

reload_config() {
    echo "Config reloaded"
}

wait_for_shutdown() {
    while (( SHUTDOWN_REQUESTED == 0 )); do
        sleep 1
    done
}

send_signal_to_group() {
    local signal=$1 pgid=$2
    kill -"$signal" -- -"$pgid" 2>/dev/null || true
}

graceful_shutdown() {
    local pid=$1 timeout_sec=${2:-30}

    echo "Sending SIGTERM to $pid..."
    kill -TERM "$pid" 2>/dev/null || return 0

    local deadline=$(( $(date +%s) + timeout_sec ))
    while kill -0 "$pid" 2>/dev/null && (( $(date +%s) < deadline )); do
        sleep 1
    done

    if kill -0 "$pid" 2>/dev/null; then
        echo "Process $pid did not stop after ${timeout_sec}s, sending SIGKILL"
        kill -KILL "$pid" 2>/dev/null || true
    else
        echo "Process $pid stopped gracefully"
    fi
}
```

---

## 84.5 Process Limits and cgroups

```bash
#!/bin/bash
# proc_limits.sh - Process resource limits and cgroup helpers

set_process_limits() {
    local max_files=${1:-1024}
    local max_procs=${2:-256}
    local max_mem_kb=${3:-}

    ulimit -n "$max_files"
    ulimit -u "$max_procs"
    [[ -n "$max_mem_kb" ]] && ulimit -v "$max_mem_kb"

    echo "Limits: files=$max_files procs=$max_procs mem=${max_mem_kb:-unlimited}"
}

run_in_cgroup() {
    local cgroup_name=$1 memory_limit=${2:-512M} cpu_shares=${3:-512}
    shift 3
    local -a cmd=("$@")

    local cgpath="/sys/fs/cgroup/memory/${cgroup_name}"
    if [[ -d /sys/fs/cgroup/memory ]]; then
        mkdir -p "$cgpath" 2>/dev/null || true
        echo "$memory_limit" > "${cgpath}/memory.limit_in_bytes" 2>/dev/null || true

        echo $$ > "${cgpath}/tasks" 2>/dev/null || true
        "${cmd[@]}"
    else
        echo "cgroups v1 memory controller not available, running without limits" >&2
        "${cmd[@]}"
    fi
}

run_with_timeout() {
    local timeout_sec=$1; shift
    timeout "$timeout_sec" "$@"
}

run_as_user() {
    local user=$1; shift
    if [[ "$(id -u)" == "0" ]]; then
        su -s /bin/bash -c "$*" "$user"
    else
        "$@"
    fi
}

watch_process_resources() {
    local pid=$1 interval=${2:-5} iterations=${3:-12}

    printf "%-12s %-8s %-8s %-8s\n" "Time" "CPU%" "RSS(MB)" "Threads"
    for (( i=0; i<iterations; i++ )); do
        [[ -d "/proc/$pid" ]] || { echo "Process $pid ended"; break; }

        local cpu; cpu=$(ps -p "$pid" -o %cpu= 2>/dev/null | tr -d ' ')
        local rss; rss=$(awk '/VmRSS/{print int($2/1024)}' /proc/$pid/status 2>/dev/null || echo 0)
        local threads; threads=$(awk '/Threads/{print $2}' /proc/$pid/status 2>/dev/null || echo 0)

        printf "%-12s %-8s %-8s %-8s\n" "$(date '+%H:%M:%S')" "${cpu:-0}" "$rss" "$threads"
        sleep "$interval"
    done
}
```

---

## 84.6 Exercises

### Exercise 1: Process Monitor Daemon
สร้าง daemon ที่:
- Watch a list of critical processes
- Restart if they die
- Rate-limit restarts (backoff)
- Alert on repeated failures

### Exercise 2: Job Queue Runner
สร้าง runner ที่:
- Read jobs from SQLite queue (Part 78)
- Run with resource limits (ulimit)
- Timeout per job
- Collect exit code + stdout/stderr

### Exercise 3: Supervisor Config
สร้าง supervisor config format:
```ini
[program:web]
command=/opt/app/server
autostart=true
autorestart=true
max_restarts=3
```
- Parse ini format
- Start/stop/restart per program

---

## สรุป Part 84

✅ proc_info: /proc/$pid/stat + status fields (name, state, RSS, VSZ, threads)
┅ proc_children, proc_tree: walk /proc for process hierarchy
┅ proc_find_by_name/port, proc_memory_map, proc_open_files, proc_cpu_time
┅ Job control: start/wait/kill/output/status with per-job output files
┅ job_wait: timeout support + kill on deadline
┅ Process supervisor: register + monitor loop + auto-restart with max_restarts
┅ sv_status: running/down + restart count per program
┅ Signal handlers: SIGTERM, SIGINT, SIGHUP, EXIT with cleanup registry
┅ graceful_shutdown: SIGTERM → poll → SIGKILL after timeout
┅ Process limits: ulimit, cgroups v1 memory, timeout wrapper, watch_process_resources

---

**→ Part 85: Shell Script Design Patterns and Architecture**
