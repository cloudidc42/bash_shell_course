# Part 87: Shell Script Performance and Profiling
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 87.1 Built-in Profiler with BASH_XTRACEFD

```bash
#!/bin/bash
# profiler.sh - Performance profiling for shell scripts

PROFILE_LOG="${PROFILE_LOG:-/tmp/bash_profile.log}"
declare -A FUNC_TIMES=()
declare -A FUNC_CALLS=()

profile_start() {
    exec {BASH_XTRACEFD}>"$PROFILE_LOG"
    PS4='+ $(date "+%s%N") ${BASH_SOURCE}:${LINENO}: '
    set -x
}

profile_stop() {
    set +x
    exec {BASH_XTRACEFD}>&-
    echo "Profile log: $PROFILE_LOG"
}

profile_analyze() {
    local log=${1:-$PROFILE_LOG}

    awk '
    /^\+ [0-9]+ / {
        ts = $2
        func = $3
        sub(/:[0-9]+:$/, "", func)
        if (prev_ts && prev_func) {
            elapsed = ts - prev_ts
            times[prev_func] += elapsed
            calls[prev_func]++
        }
        prev_ts = ts
        prev_func = func
    }
    END {
        printf "%-40s %10s %8s %12s\n", "Function", "Calls", "Total(ms)", "Avg(ms)"
        printf "%-40s %10s %8s %12s\n", "--------", "-----", "---------", "-------"
        for (f in times) {
            total_ms = times[f] / 1000000
            avg_ms = total_ms / calls[f]
            printf "%-40s %10d %8.2f %12.3f\n", f, calls[f], total_ms, avg_ms
        }
    }
    ' "$log" | sort -k3 -rn
}

# ─── Timer utilities ──────────────────────────────────────────
timer_start() {
    local name=${1:-default}
    printf -v "TIMER_${name}_START" '%s' "$(date +%s%N)"
}

timer_stop() {
    local name=${1:-default}
    local varname="TIMER_${name}_START"
    local start="${!varname}"
    local end; end=$(date +%s%N)
    local elapsed_ms=$(( (end - start) / 1000000 ))
    echo "Timer [$name]: ${elapsed_ms}ms"
    echo "$elapsed_ms"
}

time_function() {
    local fn=$1; shift
    timer_start "$fn"
    "$fn" "$@"
    local rc=$?
    timer_stop "$fn" >&2
    return $rc
}
```

---

## 87.2 Benchmark Utilities

```bash
#!/bin/bash
# benchmark.sh - Micro-benchmarking tools

bench_run() {
    local name=$1 iterations=${2:-100}; shift 2
    local -a cmd=("$@")

    local start end elapsed
    start=$(date +%s%N)

    for (( i=0; i<iterations; i++ )); do
        "${cmd[@]}" > /dev/null 2>&1
    done

    end=$(date +%s%N)
    elapsed=$(( (end - start) / 1000000 ))

    printf "Bench %-30s %d iter: %dms total, %.3fms/iter\n" \
        "$name" "$iterations" "$elapsed" \
        "$(awk "BEGIN{printf \"%.3f\", $elapsed/$iterations}")"
}

bench_compare() {
    local iterations=${1:-100}
    shift

    local results=()
    while [[ $# -ge 2 ]]; do
        local name=$1; shift
        local cmd=$1; shift
        local start end elapsed
        start=$(date +%s%N)
        for (( i=0; i<iterations; i++ )); do
            eval "$cmd" > /dev/null 2>&1
        done
        end=$(date +%s%N)
        elapsed=$(( (end - start) / 1000000 ))
        results+=("$elapsed|$name")
    done

    echo "Benchmark comparison ($iterations iterations):"
    printf "%-40s %10s %12s\n" "Name" "Total(ms)" "Per-iter(ms)"
    for r in $(echo "${results[@]}" | tr ' ' '\n' | sort -n); do
        local ms="${r%%|*}"
        local nm="${r#*|}"
        printf "%-40s %10d %12.3f\n" "$nm" "$ms" \
            "$(awk "BEGIN{printf \"%.3f\", $ms/$iterations}")"
    done
}

bench_memory() {
    local cmd=$1
    local pid_file; pid_file=$(mktemp)

    eval "$cmd" &
    local bg_pid=$!
    echo "$bg_pid" > "$pid_file"

    local peak_rss=0
    while kill -0 "$bg_pid" 2>/dev/null; do
        local rss; rss=$(awk '/VmRSS/{print $2}' /proc/$bg_pid/status 2>/dev/null || echo 0)
        (( rss > peak_rss )) && peak_rss=$rss
        sleep 0.01
    done
    wait "$bg_pid" 2>/dev/null

    printf "Peak RSS: %d kB (%.1f MB)\n" "$peak_rss" "$(awk "BEGIN{printf \"%.1f\", $peak_rss/1024}")"
    rm -f "$pid_file"
}
```

---

## 87.3 Performance Optimization Techniques

```bash
#!/bin/bash
# perf_patterns.sh - High-performance shell scripting patterns

# ─── Avoid subshells ──────────────────────────────────────────
# SLOW: var=$(cat file)
# FAST: read -r var < file  OR  mapfile -t lines < file

fast_read_file() {
    local file=$1
    local -a lines=()
    mapfile -t lines < "$file"
    echo "Read ${#lines[@]} lines"
}

fast_read_line() {
    local file=$1
    local line
    read -r line < "$file"
    echo "$line"
}

# ─── Avoid forks in loops ─────────────────────────────────────
# SLOW: for i in $(seq 1 1000); do ...
# FAST: for (( i=1; i<=1000; i++ )); do ...

fast_loop() {
    local count=$1
    local sum=0
    for (( i=1; i<=count; i++ )); do
        (( sum += i ))
    done
    echo "$sum"
}

# ─── Batch operations ─────────────────────────────────────────
batch_process() {
    local input_file=$1 batch_size=${2:-100}
    local -a batch=()
    local count=0

    while IFS= read -r line; do
        batch+=("$line")
        if (( ${#batch[@]} >= batch_size )); then
            process_batch "${batch[@]}"
            batch=()
        fi
    done < "$input_file"

    (( ${#batch[@]} > 0 )) && process_batch "${batch[@]}" || true
}

process_batch() {
    echo "Processing ${#@} items..."
}

# ─── String operations without forks ─────────────────────────
str_upper() {
    echo "${1^^}"
}

str_lower() {
    echo "${1,,}"
}

str_trim() {
    local s="$1"
    s="${s#"${s%%[![:space:]]*}"}"
    s="${s%"${s##*[![:space:]]}"}"
    echo "$s"
}

str_replace_all() {
    local str=$1 old=$2 new=$3
    echo "${str//$old/$new}"
}

str_split() {
    local str=$1 delim=$2
    local -a parts=()
    IFS="$delim" read -ra parts <<< "$str"
    printf '%s\n' "${parts[@]}"
}

str_count() {
    local str=$1 sub=$2
    local remaining="${str//$sub/}"
    echo $(( (${#str} - ${#remaining}) / ${#sub} ))
}

# ─── Avoid repeated external calls ───────────────────────────
DATE_CACHE=""
DATE_CACHE_TS=0

get_date_cached() {
    local now; now=$(date +%s)
    if (( now - DATE_CACHE_TS >= 1 )); then
        DATE_CACHE=$(date '+%Y-%m-%d %H:%M:%S')
        DATE_CACHE_TS=$now
    fi
    echo "$DATE_CACHE"
}
```

---

## 87.4 Parallel Performance

```bash
#!/bin/bash
# parallel_perf.sh - Parallel processing patterns for performance

parallel_map() {
    local worker_fn=$1 workers=${2:-$(nproc)}; shift 2
    local -a items=("$@")
    local -a pids=()
    local tmpdir; tmpdir=$(mktemp -d)

    local i=0
    for item in "${items[@]}"; do
        (
            local result; result=$("$worker_fn" "$item")
            echo "$result" > "${tmpdir}/${i}.out"
        ) &
        pids+=($!)
        (( i++ ))

        while (( ${#pids[@]} >= workers )); do
            local new_pids=()
            for pid in "${pids[@]}"; do
                kill -0 "$pid" 2>/dev/null && new_pids+=("$pid") || wait "$pid" 2>/dev/null
            done
            pids=("${new_pids[@]}")
            (( ${#pids[@]} >= workers )) && sleep 0.05
        done
    done

    for pid in "${pids[@]}"; do wait "$pid" 2>/dev/null; done

    for (( j=0; j<i; j++ )); do
        cat "${tmpdir}/${j}.out" 2>/dev/null
    done

    rm -rf "$tmpdir"
}

parallel_reduce() {
    local reducer_fn=$1; shift
    local -a values=("$@")
    local accumulator="${values[0]}"

    for (( i=1; i<${#values[@]}; i++ )); do
        accumulator=$("$reducer_fn" "$accumulator" "${values[$i]}")
    done

    echo "$accumulator"
}

xargs_parallel() {
    local fn=$1 workers=${2:-4}; shift 2

    export -f "$fn"
    xargs -P "$workers" -I{} bash -c "${fn} {}" -- "$@"
}
```

---

## 87.5 Memory-Efficient Patterns

```bash
#!/bin/bash
# mem_efficient.sh - Low-memory shell scripting patterns

# ─── Stream processing instead of loading into memory ─────────
stream_transform() {
    local input=$1 output=$2

    while IFS= read -r line; do
        # Transform line by line — never hold entire file
        echo "${line^^}" >> "$output"
    done < "$input"
}

stream_filter() {
    local input=$1 pattern=$2

    while IFS= read -r line; do
        [[ "$line" =~ $pattern ]] && echo "$line"
    done < "$input"
}

stream_aggregate() {
    local input=$1
    local sum=0 count=0

    while IFS= read -r line; do
        [[ "$line" =~ ^[0-9]+$ ]] || continue
        (( sum += line ))
        (( count++ ))
    done < "$input"

    echo "Sum=$sum Count=$count Avg=$(awk "BEGIN{printf \"%.2f\", $sum/$count}")"
}

# ─── Sliding window without arrays ────────────────────────────
sliding_window_avg() {
    local input=$1 window=${2:-5}

    awk -v w="$window" '
    {
        vals[NR % w] = $1
        sum += $1
        if (NR > w) sum -= vals[(NR - w) % w]
        n = (NR < w) ? NR : w
        printf "%.2f\n", sum / n
    }
    ' "$input"
}
```

---

## 87.6 Exercises

### Exercise 1: Profile a Real Script
เลือก script จาก Part 78-83 แล้ว:
- Enable BASH_XTRACEFD profiling
- Identify top 5 slowest functions
- Optimize the slowest one
- Measure improvement with bench_run

### Exercise 2: Parallel Processing Benchmark
เปรียบเทียบ:
- Sequential processing of 1000 URLs
- Parallel with 4 workers
- Parallel with 8 workers
- Graph results

### Exercise 3: Memory Profile
สร้าง script ที่:
- Processes a 100MB file
- Tracks peak RSS every 100ms
- Compares mapfile vs line-by-line streaming
- Reports memory savings

---

## สรุป Part 87

✅ BASH_XTRACEFD profiling: PS4 with timestamp, profile_analyze with awk
┅ timer_start/stop/time_function: nanosecond timers via date +%s%N
┅ bench_run: N-iteration benchmark with ms/iter output
┅ bench_compare: side-by-side comparison, sorted by speed
┅ bench_memory: peak RSS polling while background process runs
┅ Performance patterns: fast_read_file (mapfile), fast_loop (C-style), batch_process
┅ String ops without forks: ${var^^}, ${var,,}, trim/split/replace using pure bash
┅ parallel_map: worker pool with tmpdir result collection
┅ stream_transform/filter/aggregate: O(1) memory line-by-line processing
┅ sliding_window_avg: awk-based windowing without arrays

---

**→ Part 88: Real-World Shell Script Projects**
