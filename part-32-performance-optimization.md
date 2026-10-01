# Part 32: Performance Optimization & Benchmarking
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 32.1 Bash Performance Fundamentals

```bash
# ─── Cost of Operations ────────────────────────────────────────
# Cheapest → Most expensive:
# 1. Shell builtins (echo, read, printf, [[ ]])
# 2. Shell arithmetic (( ))
# 3. Variable operations ${var//...}
# 4. External commands (each requires fork+exec)
# 5. Subshells $()
# 6. Pipelines

# ─── Avoid Unnecessary Forks ───────────────────────────────────
# Bad: external command for simple math
result=$(echo "2 + 2" | bc)

# Good: arithmetic builtin
(( result = 2 + 2 ))

# Bad: cat then grep
cat file.txt | grep "pattern"

# Good: direct grep
grep "pattern" file.txt

# Bad: tr to uppercase
name=$(echo "$name" | tr '[:lower:]' '[:upper:]')

# Good: bash built-in (bash 4+)
name="${name^^}"

# Bad: wc -l to count lines
count=$(wc -l < file.txt)

# Good: read in loop and count
count=0
while IFS= read -r _; do (( count++ )); done < file.txt

# Bad: cut in loop
while IFS= read -r line; do
    field=$(echo "$line" | cut -d: -f1)
done < file.txt

# Good: read directly
while IFS=: read -r field _; do
    : # field is available
done < file.txt

# ─── Benchmarking ──────────────────────────────────────────────
time_cmd() {
    local cmd=$1
    local iterations=${2:-100}
    
    local start
    start=$(date +%s%N)
    
    for (( i=0; i<iterations; i++ )); do
        eval "$cmd" > /dev/null 2>&1
    done
    
    local end
    end=$(date +%s%N)
    
    local elapsed=$(( (end - start) / 1000000 ))
    local per_iter=$(( elapsed / iterations ))
    
    printf "%-50s %5dms total, %dms/iter\n" \
        "$cmd" "$elapsed" "$per_iter"
}

# Compare approaches
time_cmd 'echo "test" | tr a-z A-Z' 1000
time_cmd 'v="test"; echo "${v^^}"' 1000
```

---

## 32.2 String Processing Optimization

```bash
# ─── Avoid External Commands in Loops ──────────────────────────
# Bad: sed/grep per line
while IFS= read -r line; do
    processed=$(echo "$line" | sed 's/foo/bar/g')
    echo "$processed"
done < input.txt

# Good: single sed on whole file
sed 's/foo/bar/g' input.txt

# ─── Batch Processing vs Line-by-Line ──────────────────────────
# Bad: process line by line
awk '{print $1}' file.txt | while read -r word; do
    echo "Word: $word" >> output.txt
done

# Good: let awk do the work
awk '{print "Word:", $1}' file.txt > output.txt

# ─── String Operations in Pure Bash ────────────────────────────
# Contains check
string="Hello World"
if [[ "$string" == *"World"* ]]; then
    echo "Contains World"
fi

# Replace without sed
text="foo bar foo baz"
echo "${text//foo/qux}"  # qux bar qux baz

# Split without awk/cut
line="a:b:c:d"
IFS=: read -ra parts <<< "$line"
echo "${parts[0]}"  # a
echo "${parts[2]}"  # c

# Trim whitespace without sed/tr
trim() {
    local var=$1
    var="${var#\"${var%%[! $'\t']*}\"}"
    var="${var%\"${var##*[! $'\t']}\"}"    
    echo "$var"
}

# ─── Avoid Useless Use of Cat ──────────────────────────────────
# Bad
cat file | grep pattern
cat file | wc -l
cat file | head -5

# Good
grep pattern file
wc -l < file
head -5 file
```

---

## 32.3 Parallel Processing

```bash
#!/bin/bash
# parallel_processing.sh - Parallel execution patterns

# ─── Basic Parallelism ─────────────────────────────────────────
# Simple parallel jobs
for item in "${items[@]}"; do
    process_item "$item" &
done
wait  # wait for all

# With job limit
MAX_JOBS=4

for item in "${items[@]}"; do
    while (( $(jobs -r | wc -l) >= MAX_JOBS )); do
        wait -n 2>/dev/null || sleep 0.1
    done
    process_item "$item" &
done
wait

# ─── Parallel with Results ─────────────────────────────────────
run_parallel_collect() {
    local -n _items=$1
    local -n _results=$2
    local cmd=$3
    local max_jobs=${4:-$(nproc)}
    
    local tmpdir
    tmpdir=$(mktemp -d)
    local pids=()
    
    for i in "${!_items[@]}"; do
        while (( $(jobs -r | wc -l) >= max_jobs )); do
            wait -n 2>/dev/null || sleep 0.01
        done
        
        (
            result=$($cmd "${_items[$i]}" 2>&1)
            echo "$result" > "$tmpdir/$i"
        ) &
        pids+=($!)
    done
    
    wait
    
    for i in "${!_items[@]}"; do
        _results[$i]=$(cat "$tmpdir/$i" 2>/dev/null)
    done
    
    rm -rf "$tmpdir"
}

# ─── GNU Parallel ──────────────────────────────────────────────
# Basic
parallel echo {} ::: {1..10}

# Process files
find . -name "*.log" | parallel gzip {}

# Custom jobs
parallel -j4 'curl -s {} | wc -c' ::: url1 url2 url3 url4

# With progress bar
parallel --progress 'process {}' ::: "${items[@]}"

# Retry on failure
parallel --retries 3 'risky_command {}' ::: "${items[@]}"

# ─── xargs Parallelism ─────────────────────────────────────────
find . -name "*.jpg" | xargs -P4 -I{} convert {} -resize 800x600 {}

# ─── Worker Pool Pattern ───────────────────────────────────────
WORKERS=4
QUEUE=/tmp/worker_queue_$$
RESULTS=/tmp/worker_results_$$
mkfifo "$QUEUE" "$RESULTS"

# Start workers
for (( i=0; i<WORKERS; i++ )); do
    (
        while read -r job <&7; do
            [[ "$job" == "STOP" ]] && break
            result=$(process "$job")
            echo "$job:$result" >&8
        done
    ) 7<>"$QUEUE" 8>"$RESULTS" &
done

# Feed jobs
exec 7>"$QUEUE" 8<"$RESULTS"

for job in "${jobs[@]}"; do
    echo "$job" >&7
done

# Send stop signals
for (( i=0; i<WORKERS; i++ )); do
    echo "STOP" >&7
done

# Collect results
count=${#jobs[@]}
while (( count > 0 )); do
    read -r result <&8
    echo "Result: $result"
    (( count-- ))
done

exec 7>&- 8<&-
rm -f "$QUEUE" "$RESULTS"
```

---

## 32.4 Memory & I/O Optimization

```bash
# ─── Avoid Loading Entire Files ────────────────────────────────
# Bad: load whole file into variable
content=$(cat bigfile.txt)
while IFS= read -r line <<< "$content"; do
    process "$line"
done

# Good: stream line by line
while IFS= read -r line; do
    process "$line"
done < bigfile.txt

# ─── Efficient Array Operations ────────────────────────────────
declare -a arr=()
for i in {1..1000}; do
    arr+=("item$i")       # efficient
    # arr=("${arr[@]}" "item$i")  # slow: copies array each time
done

# ─── Reduce I/O with Buffering ─────────────────────────────────
# Bad: write one line at a time
for item in "${items[@]}"; do
    echo "$item" >> output.txt
done

# Good: buffer in variable, write once
{
    for item in "${items[@]}"; do
        echo "$item"
    done
} > output.txt

# Or use printf with format
printf '%s\n' "${items[@]}" > output.txt

# ─── Tmpfs for Temp Files ──────────────────────────────────────
TMPDIR=/dev/shm  # Linux: memory-based FS
tmpfile=$(mktemp "$TMPDIR/tmp.XXXXXX")

# ─── Minimize Disk Reads ───────────────────────────────────────
declare -A file_cache

read_cached() {
    local file=$1
    
    if [[ -z "${file_cache[$file]:-}" ]]; then
        file_cache[$file]=$(cat "$file" 2>/dev/null)
    fi
    
    echo "${file_cache[$file]}"
}

# ─── Profile Script Performance ────────────────────────────────
PS4='+ $(date "+%s.%N") ${FUNCNAME[0]:+${FUNCNAME[0]}():}line ${LINENO}: '
set -x
# ... your script ...
set +x

# Measure individual commands
SECONDS=0
heavy_computation
echo "Took: ${SECONDS}s"
```

---

## 32.5 Benchmark Script

```bash
#!/bin/bash
# benchmark.sh - Performance comparison tool

declare -A BENCHMARKS=()

bench() {
    local name=$1
    local cmd=$2
    local iterations=${3:-1000}
    
    # Warmup
    for (( i=0; i<5; i++ )); do
        eval "$cmd" > /dev/null 2>&1
    done
    
    local start
    start=$(date +%s%N)
    
    for (( i=0; i<iterations; i++ )); do
        eval "$cmd" > /dev/null 2>&1
    done
    
    local end
    end=$(date +%s%N)
    
    local total_ms=$(( (end - start) / 1000000 ))
    local per_op_us=$(( (end - start) / 1000 / iterations ))
    
    BENCHMARKS[$name]="$total_ms ms total / ${per_op_us} µs per op"
}

echo "=== Bash Performance Benchmarks ==="
echo ""

bench "tr uppercase"     'echo "hello" | tr a-z A-Z'
bench "bash uppercase"   'v="hello"; echo "${v^^}"'
bench "bc arithmetic"    'echo "2+2" | bc'
bench "bash arithmetic"  '(( x = 2+2 ))'
bench "array append +="  'a=(); for i in {1..10}; do a+=($i); done'
bench "array rebuild"    'a=(); for i in {1..10}; do a=("${a[@]}" $i); done'
bench "subshell pwd"     'dir=$(pwd)'
bench "builtin PWD"      'dir=$PWD'

echo "Results:"
echo "─────────────────────────────────────"
for name in "${!BENCHMARKS[@]}"; do
    printf "%-30s %s\n" "$name" "${BENCHMARKS[$name]}"
done | sort
```

---

## 32.6 Exercises

### Exercise 1: Script Profiler
สร้าง profiler:
- Time each function call
- Count fork() calls
- Report slowest operations
- Compare before/after optimization

### Exercise 2: Parallel File Processor
สร้าง parallel processor:
- Process N files at once
- Progress bar
- Error handling per file
- Summary report

### Exercise 3: Stream Processor
สร้าง efficient stream processor:
- Process gigabyte files
- No temp files
- Single pass
- Memory bounded

---

## สรุป Part 32

✅ Fork cost awareness  
✅ String operations in pure bash  
✅ Parallel processing (manual, GNU parallel, xargs)  
✅ Worker pool pattern  
✅ Memory and I/O optimization  
✅ Profiling and benchmarking  
✅ Practical benchmark script  

---

**→ Part 33: Text Processing Mastery (awk/sed/perl)**
