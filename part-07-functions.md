# Part 07: Functions
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 7.1 Function Basics

```bash
#!/bin/bash

# Syntax 1: function keyword
function greet() {
    echo "Hello, World!"
}

# Syntax 2: parentheses only (preferred)
greet() {
    echo "Hello, World!"
}

# Call function
greet

# Function with arguments
greet_user() {
    local name=$1    # ใช้ local เสมอ
    echo "Hello, $name!"
}

greet_user "Alice"
greet_user "Bob"

# Multiple arguments
full_greet() {
    local first=$1
    local last=$2
    local title=${3:-"Mr/Ms"}
    echo "Hello, $title $first $last!"
}

full_greet "John" "Doe"
full_greet "Jane" "Smith" "Dr"
```

---

## 7.2 Function Return Values

```bash
# Bash functions return exit codes (0-255)
# ไม่สามารถ return values ได้โดยตรง

# Method 1: echo (command substitution)
get_name() {
    echo "Alice"
}

name=$(get_name)
echo "Name: $name"

# Method 2: global/nameref variable
get_user_info() {
    local -n result=$1
    result="user_info_here"
}

get_user_info my_var
echo "$my_var"

# Method 3: return exit code (0=success, 1+=error)
is_even() {
    local num=$1
    (( num % 2 == 0 )) && return 0 || return 1
}

if is_even 4; then echo "4 is even"; fi
is_even 3; echo "Exit: $?"    # 1

# Method 4: print multiple values
get_dimensions() {
    echo "width=1920"
    echo "height=1080"
}

eval "$(get_dimensions)"
echo "Resolution: ${width}x${height}"

# Method 5: associative array output
get_user() {
    local id=$1
    local -n out=$2
    
    out[name]="Alice"
    out[age]=30
    out[email]="alice@example.com"
}

declare -A user
get_user 1 user
echo "Name: ${user[name]}, Age: ${user[age]}"

# Method 6: stdout + stderr separation
complex_function() {
    local data="result_data"
    
    # Debug/log to stderr
    echo "Processing..." >&2
    
    # Return value to stdout
    echo "$data"
}

result=$(complex_function 2>/dev/null)   # capture stdout only
complex_function > /dev/null            # discard stdout, see stderr
```

---

## 7.3 Local Variables & Scope

```bash
outer_var="I am outer"

test_scope() {
    local local_var="I am local"
    echo "Inside: $outer_var"    # ✓ accessible
    echo "Inside: $local_var"    # ✓ accessible
    
    outer_var="Modified by function"  # modifies global!
    local local_var="shadowed"        # new local, doesn't affect outer scope
}

test_scope
echo "After: $outer_var"             # "Modified by function"
echo "After: ${local_var:-unset}"    # "unset" (local is gone)

# Subshell doesn't affect parent
modify_in_subshell() (
    outer_var="Modified in subshell"  # Note: () not {} = subshell
    echo "In subshell: $outer_var"
)

modify_in_subshell
echo "After subshell: $outer_var"    # Unchanged!

# local -r: readonly local
const_function() {
    local -r PI=3.14159
    echo "PI = $PI"
    # PI=3  # would error: readonly
}

# local with default
with_defaults() {
    local name=${1:-"World"}
    local greeting=${2:-"Hello"}
    echo "$greeting, $name!"
}

with_defaults              # Hello, World!
with_defaults "Alice"      # Hello, Alice!
with_defaults "Bob" "Hi"   # Hi, Bob!
```

---

## 7.4 Advanced Function Techniques

```bash
# Variadic functions ($@)
sum() {
    local total=0
    for num in "$@"; do
        (( total += num ))
    done
    echo $total
}

echo "Sum: $(sum 1 2 3 4 5)"    # 15

# Function accepting array
process_list() {
    local label=$1
    shift
    local items=("$@")
    
    echo "$label:"
    for item in "${items[@]}"; do
        echo "  - $item"
    done
}

fruits=("apple" "banana" "cherry")
process_list "Fruits" "${fruits[@]}"

# Function modifying array (nameref)
transform_array() {
    local -n arr=$1
    local transform=$2
    
    for i in "${!arr[@]}"; do
        case $transform in
            upper)  arr[$i]="${arr[$i]^^}" ;;
            lower)  arr[$i]="${arr[$i],,}" ;;
            rev)    arr[$i]=$(echo "${arr[$i]}" | rev) ;;
        esac
    done
}

words=("Hello" "World" "Bash")
transform_array words upper
echo "${words[@]}"   # HELLO WORLD BASH
transform_array words lower
echo "${words[@]}"   # hello world bash

# Higher-order functions
map_array() {
    local func=$1
    shift
    local result=()
    for item in "$@"; do
        result+=("$($func "$item")")
    done
    echo "${result[@]}"
}

double() { echo $((${1} * 2)); }
square() { echo $((${1} ** 2)); }

numbers=(1 2 3 4 5)
echo "Doubled: $(map_array double "${numbers[@]}")"
echo "Squared: $(map_array square "${numbers[@]}")"

# Curry-like pattern
make_multiplier() {
    local factor=$1
    
    multiply() {
        echo $(( $1 * factor ))
    }
    
    echo "multiply"  # return function name
}
```

---

## 7.5 Recursive Functions

```bash
# Factorial
factorial() {
    local n=$1
    (( n <= 1 )) && echo 1 && return
    echo $(( n * $(factorial $((n-1))) ))
}

echo "5! = $(factorial 5)"    # 120
echo "10! = $(factorial 10)"  # 3628800

# Fibonacci (recursive - slow for large n)
fib() {
    local n=$1
    (( n <= 1 )) && echo $n && return
    echo $(( $(fib $((n-1))) + $(fib $((n-2))) ))
}

# Fibonacci (iterative - fast)
fib_fast() {
    local n=$1
    local a=0 b=1 temp
    for ((i=0; i<n; i++)); do
        temp=$((a + b))
        a=$b
        b=$temp
    done
    echo $a
}

# Tower of Hanoi
hanoi() {
    local n=$1
    local from=$2
    local to=$3
    local aux=$4
    
    if (( n == 1 )); then
        echo "Move disk 1 from $from to $to"
        return
    fi
    
    hanoi $((n-1)) $from $aux $to
    echo "Move disk $n from $from to $to"
    hanoi $((n-1)) $aux $to $from
}

echo "=== Tower of Hanoi (3 disks) ==="
hanoi 3 A C B

# Directory tree traversal
walk_tree() {
    local dir=$1
    local indent=${2:-0}
    local prefix=$(printf '%*s' $indent '' | tr ' ' '  ')
    
    echo "${prefix}📁 $(basename "$dir")/"
    
    # Subdirectories
    for subdir in "$dir"/*/; do
        [[ -d "$subdir" ]] || continue
        walk_tree "$subdir" $((indent + 1))
    done
    
    # Files
    for file in "$dir"/*; do
        [[ -f "$file" ]] || continue
        echo "${prefix}  📄 $(basename "$file")"
    done
}

walk_tree "/etc" 0 2>/dev/null | head -30
```

---

## 7.6 Function Libraries

```bash
#!/bin/bash
# lib/logging.sh - Logging library

# Log levels
declare -r LOG_DEBUG=0
declare -r LOG_INFO=1
declare -r LOG_WARN=2
declare -r LOG_ERROR=3
declare -r LOG_FATAL=4

# Current log level (default: INFO)
LOG_LEVEL=${LOG_LEVEL:-$LOG_INFO}

# Colors
_LOG_COLORS=(
    [0]='\033[0;36m'   # DEBUG - cyan
    [1]='\033[0;32m'   # INFO - green
    [2]='\033[1;33m'   # WARN - yellow
    [3]='\033[0;31m'   # ERROR - red
    [4]='\033[1;31m'   # FATAL - bold red
)
_LOG_NC='\033[0m'

_LOG_LABELS=("DEBUG" "INFO" "WARN" "ERROR" "FATAL")

# Log file (optional)
LOG_FILE="${LOG_FILE:-}"

_log() {
    local level=$1
    shift
    local message="$*"
    
    (( level < LOG_LEVEL )) && return 0
    
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local label="${_LOG_LABELS[$level]}"
    local color="${_LOG_COLORS[$level]}"
    
    # Format message
    local formatted="[$timestamp] [$label] $message"
    
    # Print to stderr with color
    printf "${color}%s${_LOG_NC}\n" "$formatted" >&2
    
    # Write to log file if set
    [[ -n "$LOG_FILE" ]] && echo "$formatted" >> "$LOG_FILE"
    
    # Exit on fatal
    [[ $level -eq $LOG_FATAL ]] && exit 1
}

log_debug() { _log $LOG_DEBUG "$@"; }
log_info()  { _log $LOG_INFO  "$@"; }
log_warn()  { _log $LOG_WARN  "$@"; }
log_error() { _log $LOG_ERROR "$@"; }
log_fatal() { _log $LOG_FATAL "$@"; }

# Alias
log() { log_info "$@"; }
```

```bash
#!/bin/bash
# lib/utils.sh - Utility functions

# Check command exists
require_command() {
    local cmd=$1
    local install_hint=${2:-""}
    
    if ! command -v "$cmd" &>/dev/null; then
        echo "Error: '$cmd' is required but not installed." >&2
        [[ -n "$install_hint" ]] && echo "Install with: $install_hint" >&2
        return 1
    fi
}

# Confirm before action
confirm() {
    local message=${1:-"Are you sure?"}
    local default=${2:-"n"}
    
    local prompt
    case $default in
        y|Y) prompt="[Y/n]" ;;
        n|N) prompt="[y/N]" ;;
        *)   prompt="[y/n]" ;;
    esac
    
    read -rp "$message $prompt " answer
    answer=${answer:-$default}
    
    case $answer in
        y|Y|yes|YES) return 0 ;;
        *)            return 1 ;;
    esac
}

# Get file size in human-readable format
human_size() {
    local size=$1
    if (( size < 1024 )); then
        echo "${size}B"
    elif (( size < 1048576 )); then
        echo "$(( size / 1024 ))KB"
    elif (( size < 1073741824 )); then
        echo "$(( size / 1048576 ))MB"
    else
        echo "$(( size / 1073741824 ))GB"
    fi
}

# Timer
start_timer() {
    _TIMER_START=$(date +%s%3N)
}

stop_timer() {
    local end=$(date +%s%3N)
    local elapsed=$(( end - _TIMER_START ))
    printf "%d.%03ds\n" $(( elapsed / 1000 )) $(( elapsed % 1000 ))
}

# Run with timeout
run_with_timeout() {
    local timeout=$1
    shift
    
    timeout "$timeout" "$@"
    local exit_code=$?
    
    if (( exit_code == 124 )); then
        echo "Command timed out after ${timeout}s" >&2
    fi
    
    return $exit_code
}

# Temp directory management
make_temp_dir() {
    local prefix=${1:-"tmp"}
    local tmpdir
    tmpdir=$(mktemp -d "/tmp/${prefix}.XXXXXX")
    
    # Register cleanup
    trap "rm -rf '$tmpdir'" EXIT
    
    echo "$tmpdir"
}

# Lock file
acquire_lock() {
    local lockfile=$1
    local timeout=${2:-30}
    local waited=0
    
    while ! (set -C; echo $$ > "$lockfile") 2>/dev/null; do
        if (( waited >= timeout )); then
            echo "Could not acquire lock after ${timeout}s" >&2
            return 1
        fi
        sleep 1
        ((waited++))
    done
    
    trap "rm -f '$lockfile'" EXIT
    return 0
}
```

---

## 7.7 Error Handling in Functions

```bash
#!/bin/bash

# Pattern 1: Return error codes
divide() {
    local a=$1
    local b=$2
    
    if (( b == 0 )); then
        echo "Error: Division by zero" >&2
        return 1
    fi
    
    echo "scale=4; $a / $b" | bc
}

if result=$(divide 10 3); then
    echo "Result: $result"
else
    echo "Division failed"
fi

# Pattern 2: die function
die() {
    local message=$1
    local exit_code=${2:-1}
    
    echo "FATAL: $message" >&2
    exit $exit_code
}

check_root() {
    [[ $(id -u) -eq 0 ]] || die "This script must be run as root" 2
}

# Pattern 3: try/catch emulation
try() {
    local cmd="$*"
    local output
    local exit_code
    
    output=$(eval "$cmd" 2>&1)
    exit_code=$?
    
    if (( exit_code != 0 )); then
        _TRY_ERROR="$output"
        _TRY_CODE=$exit_code
        return 1
    fi
    
    echo "$output"
    return 0
}

catch() {
    echo "Caught error (code: $_TRY_CODE): $_TRY_ERROR"
}

if ! try "ls /nonexistent/path"; then
    catch
fi

# Pattern 4: Trap in function
safe_create() {
    local dir=$1
    local tmpfile
    tmpfile=$(mktemp)
    
    # Cleanup on function exit
    local _cleanup_tmpfile() {
        rm -f "$tmpfile"
    }
    trap _cleanup_tmpfile RETURN
    
    echo "Working with $tmpfile"
    # ... do work ...
    
    mkdir -p "$dir"
    mv "$tmpfile" "$dir/output"
}
```

---

## 7.8 Memoization & Caching

```bash
#!/bin/bash
# Memoization using associative arrays

declare -A _memo_cache

# Generic memoize wrapper
memoize() {
    local func=$1
    shift
    local key="${func}:$*"
    
    if [[ ! -v _memo_cache[$key] ]]; then
        _memo_cache[$key]=$("$func" "$@")
    fi
    
    echo "${_memo_cache[$key]}"
}

# Expensive computation
slow_fibonacci() {
    local n=$1
    (( n <= 1 )) && echo $n && return
    
    local a=$(memoize slow_fibonacci $((n-1)))
    local b=$(memoize slow_fibonacci $((n-2)))
    echo $((a + b))
}

# Time without memoization (use fib function from before)
time for i in {0..25}; do fib_fast $i > /dev/null; done

# Cache results to disk
declare -A _disk_cache
CACHE_DIR="${XDG_CACHE_HOME:-$HOME/.cache}/bash_memoize"

disk_cache_get() {
    local key=$(echo "$1" | md5sum | cut -d' ' -f1)
    local cache_file="$CACHE_DIR/$key"
    
    [[ -f "$cache_file" ]] && cat "$cache_file" && return 0
    return 1
}

disk_cache_set() {
    local key=$(echo "$1" | md5sum | cut -d' ' -f1)
    local value=$2
    
    mkdir -p "$CACHE_DIR"
    echo "$value" > "$CACHE_DIR/$key"
}

disk_memoize() {
    local func=$1
    shift
    local cache_key="${func}:$*"
    
    if cached=$(disk_cache_get "$cache_key"); then
        echo "$cached"
        return 0
    fi
    
    local result
    result=$("$func" "$@")
    disk_cache_set "$cache_key" "$result"
    echo "$result"
}
```

---

## 7.9 Function Decorators Pattern

```bash
#!/bin/bash
# Function decorator pattern

# Timing decorator
with_timing() {
    local func=$1
    shift
    
    local start=$(date +%s%3N)
    "$func" "$@"
    local exit_code=$?
    local end=$(date +%s%3N)
    
    printf "[TIMING] %s: %dms\n" "$func" $((end - start)) >&2
    return $exit_code
}

# Logging decorator
with_logging() {
    local func=$1
    shift
    
    echo "[LOG] Calling: $func $*" >&2
    "$func" "$@"
    local exit_code=$?
    echo "[LOG] Done: $func (exit: $exit_code)" >&2
    return $exit_code
}

# Retry decorator
with_retry() {
    local max=$1
    local func=$2
    shift 2
    
    local attempt=1
    while (( attempt <= max )); do
        echo "[RETRY] Attempt $attempt/$max: $func" >&2
        if "$func" "$@"; then
            return 0
        fi
        ((attempt++))
        sleep $((attempt * 2))
    done
    return 1
}

# Caching decorator
with_cache() {
    local ttl=$1
    local func=$2
    shift 2
    
    local cache_key="${func}_$*"
    local cache_file="/tmp/cache_${cache_key//[^a-zA-Z0-9]/_}"
    
    if [[ -f "$cache_file" ]]; then
        local age=$(( $(date +%s) - $(stat -c%Y "$cache_file") ))
        if (( age < ttl )); then
            cat "$cache_file"
            return 0
        fi
    fi
    
    local result
    result=$("$func" "$@")
    echo "$result" > "$cache_file"
    echo "$result"
}

# Usage
do_work() {
    sleep 1
    echo "result"
}

with_timing do_work        # logs timing
with_logging do_work       # logs calls
with_retry 3 do_work       # retries on failure
with_cache 60 do_work      # caches for 60s
```

---

## 7.10 Complete Library Example

```bash
#!/usr/bin/env bash
# =====================================================================
# lib/system.sh - System utility functions
# =====================================================================

# Check if running as root
is_root() { [[ $(id -u) -eq 0 ]]; }

# Get OS info
get_os() {
    if [[ -f /etc/os-release ]]; then
        . /etc/os-release
        echo "$ID"
    elif command -v uname &>/dev/null; then
        uname -s | tr '[:upper:]' '[:lower:]'
    else
        echo "unknown"
    fi
}

# Install package (cross-distro)
install_package() {
    local pkg=$1
    
    if command -v "$pkg" &>/dev/null; then
        echo "$pkg already installed"
        return 0
    fi
    
    local os
    os=$(get_os)
    
    case $os in
        ubuntu|debian)
            is_root || { echo "Need root"; return 1; }
            apt-get install -y "$pkg"
            ;;
        centos|rhel|fedora)
            is_root || { echo "Need root"; return 1; }
            yum install -y "$pkg" 2>/dev/null || dnf install -y "$pkg"
            ;;
        arch)
            pacman -S --noconfirm "$pkg"
            ;;
        darwin)
            command -v brew &>/dev/null && brew install "$pkg"
            ;;
        *)
            echo "Unknown OS: $os, manual install required" >&2
            return 1
            ;;
    esac
}

# Service management
service_action() {
    local action=$1
    local service=$2
    
    if command -v systemctl &>/dev/null; then
        systemctl "$action" "$service"
    elif command -v service &>/dev/null; then
        service "$service" "$action"
    else
        echo "No service manager found" >&2
        return 1
    fi
}

# Get memory info
get_memory_info() {
    local -n result=$1
    
    if [[ -f /proc/meminfo ]]; then
        result[total]=$(awk '/MemTotal/{print $2}' /proc/meminfo)
        result[free]=$(awk '/MemFree/{print $2}' /proc/meminfo)
        result[available]=$(awk '/MemAvailable/{print $2}' /proc/meminfo)
        result[used]=$(( result[total] - result[available] ))
        result[percent]=$(( result[used] * 100 / result[total] ))
    fi
}

# Network check
is_online() {
    ping -c1 -W3 8.8.8.8 &>/dev/null || \
    ping -c1 -W3 1.1.1.1 &>/dev/null
}

get_public_ip() {
    curl -s --max-time 5 https://api.ipify.org 2>/dev/null || \
    curl -s --max-time 5 https://ifconfig.me 2>/dev/null || \
    echo "unknown"
}

# Wait for service
wait_for_port() {
    local host=$1
    local port=$2
    local timeout=${3:-30}
    local waited=0
    
    while ! bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null; do
        if (( waited >= timeout )); then
            echo "Timeout waiting for $host:$port" >&2
            return 1
        fi
        sleep 1
        ((waited++))
    done
    return 0
}
```

---

## 7.11 Exercises

### Exercise 1: Math Library
สร้าง math library ที่มีฟังก์ชัน:
- `abs(n)` - absolute value
- `max(a, b, ...)` - maximum of values
- `min(a, b, ...)` - minimum
- `gcd(a, b)` - greatest common divisor
- `lcm(a, b)` - least common multiple
- `is_prime(n)` - primality test

### Exercise 2: String Library
สร้าง string library:
- `trim(str)` - trim whitespace
- `pad_left(str, width, char)` - left padding
- `pad_right(str, width, char)` - right padding
- `center(str, width)` - center text
- `repeat(str, n)` - repeat string n times
- `contains(str, substr)` - check if contains

### Exercise 3: File Library
สร้าง file utility library:
- `file_exists(path)`
- `is_readable(path)`
- `get_size(file)` - human readable
- `get_extension(filename)`
- `safe_delete(file)` - confirm before delete

### Exercise 4: Decorator
สร้าง function decorator ที่:
- Limits execution rate (rate limiting)
- Logs all calls to file
- Times execution

---

## สรุป Part 07

✅ Function syntax (function keyword vs ())  
✅ Arguments ($1, $2, $@, $*)  
✅ Return values (echo, nameref, exit code)  
✅ Local variables & scope  
✅ Variadic functions  
✅ Higher-order functions  
✅ Recursive functions  
✅ Function libraries  
✅ Error handling patterns  
✅ Memoization & caching  
✅ Decorator pattern  

---

**→ Part 08: Input/Output & Redirection**
