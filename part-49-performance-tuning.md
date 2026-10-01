# Part 49: Performance Tuning and Optimization
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 49.1 Script Performance Profiling

```bash
#!/bin/bash
# bash_profiler.sh - Profile Bash script execution

# ─── Built-in Timing with PS4 ──────────────────────────────────
enable_profiling() {
    local profile_log=${1:-/tmp/bash_profile_$$.log}
    export BASH_PROFILE_LOG="$profile_log"

    PS4='+ $(date "+%s%3N") ${BASH_SOURCE}:${LINENO}: '
    exec 3>&2
    exec 2>"$profile_log"
    set -x
    echo "Profiling enabled → $profile_log"
}

disable_profiling() {
    set +x
    exec 2>&3
    exec 3>&-
    echo "Profiling disabled"
}

analyze_profile() {
    local profile_log=${1:-$BASH_PROFILE_LOG}

    [[ ! -f "$profile_log" ]] && { echo "No profile log found" >&2; return 1; }

    echo "=== Profile Analysis ==="
    echo "Top 20 slowest lines:"
    echo ""

    awk '
    /^\+/ {
        match($0, /^\+ ([0-9]+) (.+):([0-9]+): (.*)/, m)
        if (m[1] != "" && prev_ts != "") {
            duration = m[1] - prev_ts
            printf "%6d ms  %s:%s  %s\n", duration, m[2], m[3], m[4]
        }
        prev_ts = m[1]
        prev_src = m[2]
        prev_line = m[3]
    }' "$profile_log" | sort -rn | head -20
}

# ─── Time Individual Functions ─────────────────────────────────
declare -A FUNC_TIMINGS=()
declare -A FUNC_CALLS=()

time_function() {
    local func_name=$1
    shift
    local args=("$@")

    local start_ns
    start_ns=$(date +%s%N 2>/dev/null || date +%s)000000000

    "$func_name" "${args[@]}"
    local exit_code=$?

    local end_ns
    end_ns=$(date +%s%N 2>/dev/null || date +%s)000000000
    local duration_ms=$(( (end_ns - start_ns) / 1000000 ))

    local current_total="${FUNC_TIMINGS[$func_name]:-0}"
    local current_calls="${FUNC_CALLS[$func_name]:-0}"
    FUNC_TIMINGS["$func_name"]=$(( current_total + duration_ms ))
    FUNC_CALLS["$func_name"]=$(( current_calls + 1 ))

    return $exit_code
}

print_timing_report() {
    echo "=== Function Timing Report ==="
    printf "%-30s %8s %8s %10s\n" "FUNCTION" "CALLS" "TOTAL_MS" "AVG_MS"
    printf "%-30s %8s %8s %10s\n" "--------" "-----" "--------" "------"

    for func_name in "${!FUNC_TIMINGS[@]}"; do
        local total="${FUNC_TIMINGS[$func_name]}"
        local calls="${FUNC_CALLS[$func_name]}"
        local avg=$(( total / calls ))
        printf "%-30s %8d %8d %10d\n" "$func_name" "$calls" "$total" "$avg"
    done | sort -k3 -rn
}

# ─── Benchmark Helper ──────────────────────────────────────────
benchmark() {
    local name=$1
    local iterations=${2:-100}
    shift 2
    local cmd=("$@")

    local total_ms=0
    local min_ms=99999999
    local max_ms=0

    for (( i=0; i<iterations; i++ )); do
        local start_ns
        start_ns=$(date +%s%N 2>/dev/null || echo 0)
        "${cmd[@]}" &>/dev/null
        local end_ns
        end_ns=$(date +%s%N 2>/dev/null || echo 0)
        local duration_ms=$(( (end_ns - start_ns) / 1000000 ))

        (( total_ms += duration_ms ))
        (( duration_ms < min_ms )) && min_ms=$duration_ms
        (( duration_ms > max_ms )) && max_ms=$duration_ms
    done

    local avg_ms=$(( total_ms / iterations ))
    printf "Benchmark: %-30s iterations=%d avg=%dms min=%dms max=%dms\n" \
        "$name" "$iterations" "$avg_ms" "$min_ms" "$max_ms"
}
```

---

## 49.2 System Performance Tuning

```bash
#!/bin/bash
# system_tuning.sh - OS-level performance optimization

# ─── CPU Performance ───────────────────────────────────────────
set_cpu_governor() {
    local governor=${1:-performance}
    local valid_governors="performance powersave ondemand conservative schedutil"

    if ! echo "$valid_governors" | grep -qw "$governor"; then
        echo "Invalid governor. Valid: $valid_governors" >&2
        return 1
    fi

    for cpu in /sys/devices/system/cpu/cpu[0-9]*/cpufreq/scaling_governor; do
        [[ -w "$cpu" ]] && echo "$governor" > "$cpu"
    done
    echo "CPU governor set to: $governor"
}

disable_cpu_frequency_scaling() {
    set_cpu_governor "performance"
    echo "CPU frequency scaling disabled (performance mode)"
}

set_cpu_affinity() {
    local pid=$1
    local cpus=${2:-0-3}
    taskset -cp "$cpus" "$pid"
    echo "CPU affinity set: PID $pid → CPUs $cpus"
}

# ─── Memory Tuning ─────────────────────────────────────────────
tune_memory() {
    local profile=${1:-balanced}

    case "$profile" in
        database)
            sysctl -w vm.swappiness=10
            sysctl -w vm.dirty_ratio=40
            sysctl -w vm.dirty_background_ratio=10
            sysctl -w vm.overcommit_memory=0
            echo "Applied database memory profile"
            ;;
        web)
            sysctl -w vm.swappiness=10
            sysctl -w vm.dirty_ratio=20
            sysctl -w vm.dirty_background_ratio=5
            sysctl -w vm.overcommit_memory=1
            echo "Applied web server memory profile"
            ;;
        balanced)
            sysctl -w vm.swappiness=60
            sysctl -w vm.dirty_ratio=20
            sysctl -w vm.dirty_background_ratio=10
            echo "Applied balanced memory profile"
            ;;
        *)
            echo "Unknown profile: $profile" >&2
            return 1
            ;;
    esac
}

drop_caches() {
    local level=${1:-3}
    echo "Dropping caches (level $level)..."
    sync
    echo "$level" > /proc/sys/vm/drop_caches
    echo "Caches dropped"
}

# ─── Network Tuning ────────────────────────────────────────────
tune_network_stack() {
    local profile=${1:-throughput}

    case "$profile" in
        throughput)
            # TCP buffer sizes
            sysctl -w net.core.rmem_max=134217728
            sysctl -w net.core.wmem_max=134217728
            sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
            sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

            # TCP optimizations
            sysctl -w net.ipv4.tcp_congestion_control=bbr
            sysctl -w net.ipv4.tcp_fastopen=3
            sysctl -w net.core.somaxconn=65535
            sysctl -w net.ipv4.tcp_max_syn_backlog=65535

            echo "Applied throughput network profile"
            ;;
        latency)
            sysctl -w net.ipv4.tcp_nodelay=1
            sysctl -w net.ipv4.tcp_low_latency=1
            sysctl -w net.core.busy_poll=50
            sysctl -w net.core.busy_read=50

            echo "Applied low-latency network profile"
            ;;
    esac
}

tune_tcp_keepalive() {
    local time=${1:-60}
    local intvl=${2:-10}
    local probes=${3:-5}

    sysctl -w net.ipv4.tcp_keepalive_time="$time"
    sysctl -w net.ipv4.tcp_keepalive_intvl="$intvl"
    sysctl -w net.ipv4.tcp_keepalive_probes="$probes"
    echo "TCP keepalive: time=${time}s intvl=${intvl}s probes=${probes}"
}

# ─── Disk I/O Tuning ───────────────────────────────────────────
tune_disk_io() {
    local device=$1
    local scheduler=${2:-mq-deadline}

    if [[ ! -b "/dev/${device}" ]]; then
        echo "Device not found: /dev/$device" >&2
        return 1
    fi

    local scheduler_file="/sys/block/${device}/queue/scheduler"
    [[ -w "$scheduler_file" ]] && echo "$scheduler" > "$scheduler_file"

    # Queue depth
    echo "32" > "/sys/block/${device}/queue/nr_requests" 2>/dev/null

    # Read-ahead (512KB)
    blockdev --setra 1024 "/dev/${device}" 2>/dev/null

    echo "Disk I/O tuned: /dev/$device (scheduler=$scheduler)"
}

set_ulimits() {
    local max_files=${1:-65536}
    local max_processes=${2:-32768}

    ulimit -n "$max_files"
    ulimit -u "$max_processes"

    cat > /etc/security/limits.d/99-performance.conf << EOF
* soft nofile ${max_files}
* hard nofile ${max_files}
* soft nproc ${max_processes}
* hard nproc ${max_processes}
root soft nofile ${max_files}
root hard nofile ${max_files}
EOF
    echo "Set ulimits: files=$max_files processes=$max_processes"
}
```

---

## 49.3 Shell Script Optimization Techniques

```bash
#!/bin/bash
# script_optimization.sh - Bash optimization patterns

# ─── String Operations: Built-ins vs External Tools ────────────

# SLOW: External processes
slow_uppercase() { echo "$1" | tr '[:lower:]' '[:upper:]'; }
slow_trim() { echo "$1" | sed 's/^[[:space:]]*//; s/[[:space:]]*$//'; }
slow_replace() { echo "${1}" | sed "s/${2}/${3}/g"; }

# FAST: Parameter expansion (no subprocess)
fast_uppercase() { echo "${1^^}"; }
fast_lowercase() { echo "${1,,}"; }
fast_trim() {
    local str="$1"
    str="${str#"${str%%[![:space:]]*}"}"
    str="${str%"${str##*[![:space:]]}"}" 
    echo "$str"
}
fast_replace() { echo "${1//$2/$3}"; }

# ─── Arithmetic: Avoid External ────────────────────────────────

# SLOW
slow_add() { echo "$1 + $2" | bc; }

# FAST: Built-in arithmetic
fast_add() { echo $(( $1 + $2 )); }
fast_inc() { (( count++ )); }
fast_compare() { (( $1 > $2 )) && echo "greater"; }

# ─── Array Operations ──────────────────────────────────────────

# Efficient array append
arr=()
arr+=("item1")
arr+=("item2")

# Efficient array search
array_contains() {
    local needle=$1
    shift
    local element
    for element in "$@"; do
        [[ "$element" == "$needle" ]] && return 0
    done
    return 1
}

# Efficient deduplication
array_unique() {
    local -A seen=()
    local result=()
    for item in "$@"; do
        [[ -v "seen[$item]" ]] || { result+=("$item"); seen["$item"]=1; }
    done
    echo "${result[@]}"
}

# ─── File Processing Optimization ─────────────────────────────

process_file_optimized() {
    local input_file=$1
    local output_file=$2

    # Read file into array once (avoid re-reading)
    mapfile -t lines < "$input_file"

    local result=()
    for line in "${lines[@]}"; do
        [[ -z "$line" ]] && continue
        result+=("${line^^}")
    done

    printf '%s\n' "${result[@]}" > "$output_file"
}

# ─── Avoid Subshells in Loops ──────────────────────────────────

# SLOW: Spawns subshell for each word count
slow_count_words() {
    local total=0
    while IFS= read -r line; do
        count=$(echo "$line" | wc -w)
        (( total += count ))
    done < "$1"
    echo "$total"
}

# FAST: Accumulate with awk (single subprocess)
fast_count_words() {
    awk '{total += NF} END {print total}' "$1"
}

# ─── Parallel File Processing ──────────────────────────────────
parallel_process_files() {
    local input_pattern=$1
    local processor=$2
    local max_jobs=${3:-$(nproc)}

    local pids=()
    local job_count=0

    while IFS= read -r file; do
        "$processor" "$file" &
        pids+=($!)
        (( job_count++ ))

        if (( job_count >= max_jobs )); then
            wait "${pids[0]}"
            pids=("${pids[@]:1}")
            (( job_count-- ))
        fi
    done < <(find . -name "$input_pattern" -type f)

    wait "${pids[@]}"
}

# ─── Caching Expensive Operations ─────────────────────────────
CACHE_DIR="${XDG_CACHE_HOME:-$HOME/.cache}/bash_script_cache"
mkdir -p "$CACHE_DIR"

cached_operation() {
    local cache_key=$1
    local ttl_seconds=${2:-3600}
    shift 2
    local compute_func=$1
    shift
    local args=("$@")

    local cache_file="${CACHE_DIR}/${cache_key//\//_}"

    if [[ -f "$cache_file" ]]; then
        local mtime age
        mtime=$(stat -c %Y "$cache_file" 2>/dev/null || stat -f %m "$cache_file")
        age=$(( $(date +%s) - mtime ))
        if (( age < ttl_seconds )); then
            cat "$cache_file"
            return 0
        fi
    fi

    local result
    result=$("$compute_func" "${args[@]}")
    echo "$result" > "$cache_file"
    echo "$result"
}
```

---

## 49.4 I/O Optimization

```bash
#!/bin/bash
# io_optimization.sh - Optimize I/O operations

# ─── Buffered Writing ──────────────────────────────────────────
buffered_write() {
    local output_file=$1
    local buffer_size=${2:-1000}
    local buffer=()

    flush_buffer() {
        if (( ${#buffer[@]} > 0 )); then
            printf '%s\n' "${buffer[@]}" >> "$output_file"
            buffer=()
        fi
    }

    while IFS= read -r line; do
        buffer+=("$line")
        if (( ${#buffer[@]} >= buffer_size )); then
            flush_buffer
        fi
    done

    flush_buffer
}

# ─── Efficient Large File Processing ──────────────────────────
process_large_file() {
    local input=$1
    local chunk_size=${2:-10000}
    local processor=${3:-cat}
    local output=${4:-/dev/stdout}

    local total_lines
    total_lines=$(wc -l < "$input")
    local chunks=$(( (total_lines + chunk_size - 1) / chunk_size ))

    echo "Processing $total_lines lines in $chunks chunks..."

    for (( i=0; i<chunks; i++ )); do
        local start=$(( i * chunk_size + 1 ))
        local end=$(( (i + 1) * chunk_size ))

        sed -n "${start},${end}p" "$input" | "$processor"
    done > "$output"
}

# ─── Atomic File Operations ────────────────────────────────────
atomic_write() {
    local target_file=$1
    local content=$2

    local tmp_file
    tmp_file=$(mktemp "${target_file}.XXXXXX")

    echo "$content" > "$tmp_file"
    mv -f "$tmp_file" "$target_file"
}

atomic_append() {
    local target_file=$1
    local content=$2

    local lock_file="${target_file}.lock"
    local max_wait=10
    local waited=0

    while ! mkdir "$lock_file" 2>/dev/null; do
        (( waited++ ))
        (( waited >= max_wait )) && { echo "Lock timeout" >&2; return 1; }
        sleep 0.1
    done

    trap "rmdir '$lock_file'" EXIT
    echo "$content" >> "$target_file"
    rmdir "$lock_file"
}

# ─── Named Pipe Throughput ─────────────────────────────────────
fifo_pipeline() {
    local stages=("$@")
    local num_stages=${#stages[@]}

    if (( num_stages < 2 )); then
        echo "Need at least 2 stages" >&2
        return 1
    fi

    local fifos=()
    for (( i=0; i<num_stages-1; i++ )); do
        local fifo
        fifo=$(mktemp -u)
        mkfifo "$fifo"
        fifos+=("$fifo")
    done

    trap "rm -f ${fifos[*]}" EXIT

    cat | "${stages[0]}" > "${fifos[0]}" &

    for (( i=1; i<num_stages-1; i++ )); do
        "${stages[$i]}" < "${fifos[$((i-1))]}" > "${fifos[$i]}" &
    done

    "${stages[$((num_stages-1))]}" < "${fifos[$((num_stages-2))]}"
    wait
}
```

---

## 49.5 Load Testing Scripts

```bash
#!/bin/bash
# load_tester.sh - HTTP load testing

http_load_test() {
    local url=$1
    local concurrency=${2:-10}
    local requests=${3:-100}
    local method=${4:-GET}
    local body=${5:-}

    echo "Load Test: $url"
    echo "Concurrency: $concurrency | Requests: $requests"
    echo ""

    local tmpdir
    tmpdir=$(mktemp -d)
    local results_dir="${tmpdir}/results"
    mkdir -p "$results_dir"

    local completed=0
    local pids=()

    make_request() {
        local req_id=$1
        local result_file="${results_dir}/${req_id}"

        local start_ms
        start_ms=$(date +%s%3N)

        local http_code
        http_code=$(curl -sf -o /dev/null -w "%{http_code}" \
            --max-time 30 \
            --request "$method" \
            ${body:+--data "$body"} \
            "$url" 2>/dev/null)

        local end_ms
        end_ms=$(date +%s%3N)
        local duration_ms=$(( end_ms - start_ms ))

        echo "${duration_ms} ${http_code}" > "$result_file"
    }

    for (( i=0; i<requests; i++ )); do
        if (( ${#pids[@]} >= concurrency )); then
            wait "${pids[0]}"
            pids=("${pids[@]:1}")
        fi
        make_request "$i" &
        pids+=($!)
    done

    wait "${pids[@]}"

    local total_ms=0
    local success=0
    local failed=0
    local min_ms=99999999
    local max_ms=0

    while IFS=' ' read -r duration_ms status; do
        (( total_ms += duration_ms ))
        (( duration_ms < min_ms )) && min_ms=$duration_ms
        (( duration_ms > max_ms )) && max_ms=$duration_ms
        if (( status >= 200 && status < 400 )); then
            (( success++ ))
        else
            (( failed++ ))
        fi
    done < <(cat "${results_dir}"/* 2>/dev/null)

    local total=$(( success + failed ))
    local avg_ms=$(( total > 0 ? total_ms / total : 0 ))

    echo "=== Results ==="
    printf "Total requests:  %d\n" "$total"
    printf "Successful:      %d\n" "$success"
    printf "Failed:          %d\n" "$failed"
    printf "Avg response:    %d ms\n" "$avg_ms"
    printf "Min response:    %d ms\n" "$min_ms"
    printf "Max response:    %d ms\n" "$max_ms"

    rm -rf "$tmpdir"
}
```

---

## 49.6 Exercises

### Exercise 1: Script Optimizer
สร้าง tool ที่:
- Scan Bash scripts for anti-patterns
- Suggest optimizations
- Measure before/after performance
- Generate improvement report

### Exercise 2: System Tuning Wizard
สร้าง interactive wizard ที่:
- Detect system workload type
- Apply appropriate kernel parameters
- Benchmark before/after
- Generate tuning report

### Exercise 3: Continuous Load Tester
สร้าง continuous tester ที่:
- Run load tests on schedule
- Track performance trends
- Alert on degradation
- Generate performance SLA report

---

## สรุป Part 49

✅ Bash script profiling with PS4 tracing
✅ Function timing and call count tracking
✅ Benchmark framework
✅ CPU governor and affinity tuning
✅ Memory, network, disk I/O tuning profiles
✅ Shell script optimization (built-ins vs subshells)
✅ Array operations and deduplication
✅ Cached computation framework
✅ Buffered/atomic I/O operations
✅ HTTP load testing framework

---

**→ Part 50: Shell Script Testing and Quality Assurance**
