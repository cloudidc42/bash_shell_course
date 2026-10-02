# Part 61: Performance Optimization and Profiling
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 61.1 Script Profiling

```bash
#!/bin/bash
# profiler.sh - Bash script performance profiling

# ─── Timing Infrastructure ───────────────────────────────────────────
declare -A TIMER_START=()
declare -A TIMER_ACCUM=()
declare -A TIMER_CALLS=()

timer_start() {
    local name=$1
    TIMER_START["$name"]=$(date +%s%N)
    (( TIMER_CALLS["$name"]++ )) || TIMER_CALLS["$name"]=1
}

timer_stop() {
    local name=$1
    local now; now=$(date +%s%N)
    local start="${TIMER_START[$name]:-$now}"
    local elapsed=$(( now - start ))

    TIMER_ACCUM["$name"]=$(( ${TIMER_ACCUM[$name]:-0} + elapsed ))
}

timer_report() {
    echo "=== Performance Report ==="
    printf "%-30s %10s %10s %10s\n" "Function" "Calls" "Total(ms)" "Avg(ms)"
    printf "%-30s %10s %10s %10s\n" "--------" "-----" "---------" "-------"

    for name in "${!TIMER_ACCUM[@]}"; do
        local total_ns="${TIMER_ACCUM[$name]}"
        local calls="${TIMER_CALLS[$name]}"
        local total_ms=$(( total_ns / 1000000 ))
        local avg_ms=$(( total_ms / calls ))
        printf "%-30s %10d %10d %10d\n" "$name" "$calls" "$total_ms" "$avg_ms"
    done | sort -k3 -rn
}

# ─── Function Wrapper ──────────────────────────────────────────────
profile_function() {
    local func_name=$1
    shift

    timer_start "$func_name"
    "$func_name" "$@"
    local exit_code=$?
    timer_stop "$func_name"

    return $exit_code
}

# ─── Built-in Profiler via PS4 ────────────────────────────────────────
enable_trace_profiling() {
    local output_file="${1:-/tmp/bash_trace.log}"

    PS4='+ $(date "+%s%N") ${FUNCNAME[0]}:${LINENO} '
    exec 3>"$output_file"
    BASH_XTRACEFD=3
    set -x
}

disable_trace_profiling() {
    set +x
    exec 3>&-
}

analyze_trace_log() {
    local trace_file=${1:-/tmp/bash_trace.log}

    awk '
    /^\+ [0-9]+ / {
        ts = $2
        func = $3
        sub(/:.*/, "", func)
        if (prev_ts && prev_func) {
            elapsed = ts - prev_ts
            accum[prev_func] += elapsed
            calls[prev_func]++
        }
        prev_ts = ts
        prev_func = func
    }
    END {
        for (f in accum) {
            printf "%s\t%d\t%d\n", f, calls[f], accum[f]/1000000
        }
    }
    ' "$trace_file" | sort -k3 -rn | \
    awk 'BEGIN{printf "%-30s %8s %10s\n","Function","Calls","Total(ms)"}
         {printf "%-30s %8d %10d\n",$1,$2,$3}'
}
```

---

## 61.2 Subprocess Optimization

```bash
#!/bin/bash
# subprocess_opt.sh - Minimize subshell overhead

# ─── Avoid Unnecessary Subshells ──────────────────────────────────────────

# SLOW: each $() is a fork
slow_string_ops() {
    local str=$1
    local upper; upper=$(echo "$str" | tr '[:lower:]' '[:upper:]')
    local len; len=$(echo "$upper" | wc -c)
    echo "$upper ($len chars)"
}

# FAST: use parameter expansion and built-ins
fast_string_ops() {
    local str=$1
    local upper="${str^^}"
    local len="${#upper}"
    echo "$upper ($len chars)"
}

# ─── Read File Without Cat ──────────────────────────────────────────────
# SLOW
slow_read_file() {
    local content; content=$(cat "$1")
    echo "$content"
}

# FAST
fast_read_file() {
    local content
    IFS= read -r -d '' content < "$1" || true
    echo "$content"
}

# ─── Arithmetic Without BC ───────────────────────────────────────────
# SLOW: bc spawns a process
slow_math() {
    local result; result=$(echo "scale=2; $1 + $2" | bc)
    echo "$result"
}

# FAST: bash arithmetic (integer only)
fast_math_int() {
    echo $(( $1 + $2 ))
}

# FAST: awk for floats without bc
fast_math_float() {
    awk "BEGIN{printf \"%.2f\n\", $1 + $2}"
}

# ─── Batch Operations ───────────────────────────────────────────────
# SLOW: one find per extension
slow_find_files() {
    local dir=$1
    find "$dir" -name "*.log"
    find "$dir" -name "*.txt"
    find "$dir" -name "*.csv"
}

# FAST: single find with OR
fast_find_files() {
    local dir=$1
    find "$dir" \( -name "*.log" -o -name "*.txt" -o -name "*.csv" \)
}

# ─── Parallel Job Execution ──────────────────────────────────────────
run_parallel() {
    local -n jobs_ref=$1
    local max_workers=${2:-$(nproc)}
    local -a pids=()
    local -A job_names=()

    for job in "${jobs_ref[@]}"; do
        local name="${job%%:*}"
        local cmd="${job#*:}"

        (eval "$cmd") &
        local pid=$!
        pids+=("$pid")
        job_names["$pid"]="$name"

        while (( ${#pids[@]} >= max_workers )); do
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[$i]}" 2>/dev/null; then
                    wait "${pids[$i]}"
                    unset 'pids[i]'
                    pids=("${pids[@]}")
                    break
                fi
            done
            sleep 0.1
        done
    done

    for pid in "${pids[@]}"; do
        wait "$pid"
    done
}
```

---

## 61.3 I/O Optimization

```bash
#!/bin/bash
# io_opt.sh - I/O performance optimization

# ─── Buffered Writing ───────────────────────────────────────────────
declare -a WRITE_BUFFER=()
declare -i WRITE_BUFFER_SIZE=0
WRITE_BUFFER_MAX=${WRITE_BUFFER_MAX:-1000}

buffer_write() {
    local file=$1 line=$2

    WRITE_BUFFER+=("$line")
    (( WRITE_BUFFER_SIZE++ ))

    if (( WRITE_BUFFER_SIZE >= WRITE_BUFFER_MAX )); then
        buffer_flush "$file"
    fi
}

buffer_flush() {
    local file=$1

    (( WRITE_BUFFER_SIZE == 0 )) && return

    printf '%s\n' "${WRITE_BUFFER[@]}" >> "$file"
    WRITE_BUFFER=()
    WRITE_BUFFER_SIZE=0
}

# ─── Efficient File Processing ──────────────────────────────────────────
# SLOW: grep + wc on each line
slow_count_errors() {
    local log_file=$1
    local count=0
    while IFS= read -r line; do
        if echo "$line" | grep -q "ERROR"; then
            (( count++ ))
        fi
    done < "$log_file"
    echo "$count"
}

# FAST: single grep pass
fast_count_errors() {
    grep -c "ERROR" "${1:-/dev/stdin}" 2>/dev/null || echo 0
}

# ─── Stream Processing ───────────────────────────────────────────────
process_large_file() {
    local file=$1 chunk_size=${2:-10000}
    local line_count=0 chunk_count=0
    local -a chunk=()

    while IFS= read -r line; do
        chunk+=("$line")
        (( line_count++ ))

        if (( line_count % chunk_size == 0 )); then
            (( chunk_count++ ))
            process_chunk "$chunk_count" "${chunk[@]}"
            chunk=()
        fi
    done < "$file"

    if (( ${#chunk[@]} > 0 )); then
        (( chunk_count++ ))
        process_chunk "$chunk_count" "${chunk[@]}"
    fi

    echo "Processed $line_count lines in $chunk_count chunks"
}

process_chunk() {
    local chunk_num=$1
    shift
    echo "Chunk $chunk_num: $# lines" >&2
}

# ─── Named Pipe for Producer/Consumer ──────────────────────────────────
pipeline_process() {
    local input_file=$1 output_file=$2
    local pipe; pipe=$(mktemp -u)

    mkfifo "$pipe"
    trap "rm -f '$pipe'" EXIT

    grep -v '^#' "$input_file" | \
        awk '{print toupper($0)}' > "$pipe" &

    while IFS= read -r line; do
        echo "$line" | sed 's/  */ /g'
    done < "$pipe" > "$output_file"

    wait
}

# ─── Temp File Management ──────────────────────────────────────────────
declare -a TEMP_FILES=()

make_temp() {
    local suffix=${1:-.tmp}
    local f; f=$(mktemp "/tmp/bash_opt_XXXXXX${suffix}")
    TEMP_FILES+=("$f")
    echo "$f"
}

cleanup_temps() {
    for f in "${TEMP_FILES[@]}"; do
        [[ -f "$f" ]] && rm -f "$f"
    done
    TEMP_FILES=()
}

trap cleanup_temps EXIT INT TERM
```

---

## 61.4 Memory Optimization

```bash
#!/bin/bash
# memory_opt.sh - Memory-efficient bash techniques

# ─── Avoid Large Arrays in Memory ─────────────────────────────────────────
process_file_streaming() {
    local file=$1

    while IFS= read -r line; do
        process_line "$line"
    done < "$file"
}

process_line() {
    echo "${1^^}"
}

# ─── String Deduplication ──────────────────────────────────────────────
deduplicate_stream() {
    awk '!seen[$0]++'
}

# ─── Efficient Associative Array ─────────────────────────────────────────
cache_get() {
    local key=$1
    local var="CACHE_${key//[^a-zA-Z0-9_]/_}"
    echo "${!var:-}"
}

cache_set() {
    local key=$1 value=$2
    local var="CACHE_${key//[^a-zA-Z0-9_]/_}"
    printf -v "$var" '%s' "$value"
}

cache_exists() {
    local key=$1
    local var="CACHE_${key//[^a-zA-Z0-9_]/_}"
    [[ -n "${!var+set}" ]]
}

# ─── Memoization ────────────────────────────────────────────────────
memoize() {
    local func=$1
    shift

    local key="${func}_$(printf '%s_' "$@" | md5sum | cut -d' ' -f1)"

    if cache_exists "$key"; then
        cache_get "$key"
        return 0
    fi

    local result
    result=$("$func" "$@")
    cache_set "$key" "$result"
    echo "$result"
}

compute_hash() {
    local data=$1
    echo "$data" | sha256sum | cut -d' ' -f1
}

# ─── Reduce Global Variable Footprint ─────────────────────────────────────
compute_stats() {
    local -a data=("$@")
    local sum=0 count=${#data[@]} min max

    min="${data[0]}"
    max="${data[0]}"

    for val in "${data[@]}"; do
        (( sum += val ))
        (( val < min )) && min=$val
        (( val > max )) && max=$val
    done

    local avg=$(( count > 0 ? sum / count : 0 ))

    printf '{"sum":%d,"count":%d,"min":%d,"max":%d,"avg":%d}\n' \
        "$sum" "$count" "$min" "$max" "$avg"
}
```

---

## 61.5 Benchmark Tool

```bash
#!/bin/bash
# benchmark.sh - Function benchmarking utility

benchmark() {
    local name=$1 iterations=${2:-100}
    shift 2
    local cmd=("$@")

    local start_ns end_ns elapsed_ns avg_ns
    start_ns=$(date +%s%N)

    local i
    for (( i=0; i<iterations; i++ )); do
        "${cmd[@]}" > /dev/null 2>&1
    done

    end_ns=$(date +%s%N)
    elapsed_ns=$(( end_ns - start_ns ))
    avg_ns=$(( elapsed_ns / iterations ))

    printf "%-40s %8d iter  %8d ms total  %6d us/iter\n" \
        "$name" \
        "$iterations" \
        "$(( elapsed_ns / 1000000 ))" \
        "$(( avg_ns / 1000 ))"
}

benchmark_compare() {
    local desc=$1 iterations=${2:-100}
    shift 2

    echo "=== Benchmark: $desc ==="
    echo "Iterations: $iterations"
    echo ""

    while [[ $# -ge 2 ]]; do
        local label=$1 cmd=$2
        shift 2
        benchmark "$label" "$iterations" bash -c "$cmd"
    done
}
```

---

## 61.6 Exercises

### Exercise 1: Script Profiler
สร้าง profiler ที่:
- Wrap ทุก function อัตโนมัติ
- Report hotspots
- Export flame graph data

### Exercise 2: Parallel Processing Framework
สร้าง framework ที่:
- Worker pool with queue
- Progress reporting
- Error collection
- Result aggregation

### Exercise 3: Caching Layer
สร้าง cache ที่:
- TTL-based expiry
- LRU eviction
- Disk persistence
- Cache statistics

---

## สรุป Part 61

✅ Timer infrastructure with accumulator and call counter
✅ PS4 trace profiling with log analyzer
✅ Subshell optimization: parameter expansion vs $(), built-ins vs external
✅ Parallel job runner with worker-slot throttling
✅ Buffered file writing to reduce syscalls
✅ Streaming large file processing in chunks
✅ Named pipe producer/consumer pipeline
✅ Memoization with variable-name-based cache
✅ Benchmarking utility with ns-precision timing

---

**→ Part 62: Security Hardening and Secure Scripting**
