# Part 18: Debugging & Error Handling
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 18.1 Debug Modes

```bash
# ─── set options for debugging ────────────────────────────────
set -x          # print each command before execution (xtrace)
set -v          # print each line as read (verbose)
set -e          # exit on error (errexit)
set -u          # error on unset variable (nounset)
set -f          # disable glob expansion
set -o pipefail # fail on any pipe error (not just last command)
set -n          # syntax check (don't execute)

# Best practice for scripts
set -euo pipefail

# Combined: enable, then disable
set -x
# ... code to debug ...
set +x

# ─── Debug environment variables ──────────────────────────────
BASH_XTRACEFD=6         # redirect xtrace to fd 6
PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'

# Colorful xtrace
PS4='\e[35m+${BASH_SOURCE}:${LINENO}: \e[0m'

# Timestamped xtrace
PS4='+$(date "+%T.%N") ${BASH_SOURCE}:${LINENO}: '

# ─── Redirect debug output ────────────────────────────────────
exec 6>&2 2>/tmp/debug.log
set -x
# ... code ...
set +x
exec 2>&6 6>&-

# ─── BASH_SOURCE & FUNCNAME ───────────────────────────────────
debug_info() {
    echo "File:     ${BASH_SOURCE[1]}"
    echo "Line:     ${BASH_LINENO[0]}"
    echo "Function: ${FUNCNAME[1]}"
    echo "Call stack:"
    local i
    for (( i=1; i<${#FUNCNAME[@]}; i++ )); do
        echo "  ${FUNCNAME[$i]} (${BASH_SOURCE[$i]}:${BASH_LINENO[$i-1]})"
    done
}

# ─── Bash debugger (bashdb) ───────────────────────────────────
# Install: apt-get install bashdb
bashdb myscript.sh
# Inside bashdb:
# (bashdb) step     - step one line
# (bashdb) next     - step over function
# (bashdb) continue - run to next breakpoint
# (bashdb) print var - print variable
# (bashdb) where    - show call stack
# (bashdb) list     - show source
# (bashdb) break 42 - set breakpoint at line 42

# ─── Shellcheck (static analysis) ────────────────────────────
# Install: apt-get install shellcheck
shellcheck myscript.sh
shellcheck -S error myscript.sh      # only errors
shellcheck -e SC2034 myscript.sh     # exclude specific check
shellcheck --format=gcc myscript.sh  # gcc-style output

# Ignore in script:
# shellcheck disable=SC2034
VAR="unused"    # SC2034: VAR appears unused
```

---

## 18.2 Error Handling Patterns

```bash
#!/bin/bash
# error_handling.sh - Comprehensive error handling

set -euo pipefail

# ─── Basic error handling ─────────────────────────────────────
# Exit with error message
die() {
    echo "Error: $1" >&2
    exit "${2:-1}"
}

# Exit with file/line info
die_with_info() {
    local msg=$1
    local code=${2:-1}
    echo "Error: $msg" >&2
    echo "  at ${BASH_SOURCE[1]}:${BASH_LINENO[0]}" >&2
    exit "$code"
}

# ─── ERR trap ─────────────────────────────────────────────────
on_error() {
    local exit_code=$?
    local line_number=$1
    
    echo "Error on line $line_number: exit code $exit_code" >&2
    echo "Command: ${BASH_COMMAND}" >&2
    
    # Print call stack
    local i
    for (( i=1; i<${#FUNCNAME[@]}; i++ )); do
        echo "  at ${FUNCNAME[$i]} (${BASH_SOURCE[$i]}:${BASH_LINENO[$i-1]})" >&2
    done
}

trap 'on_error $LINENO' ERR

# ─── Cleanup on exit ──────────────────────────────────────────
TEMP_DIR=$(mktemp -d)
LOCK_FILE="/var/run/myscript.lock"

cleanup() {
    rm -rf "$TEMP_DIR"
    rm -f "$LOCK_FILE"
    echo "Cleanup complete"
}

trap cleanup EXIT         # always runs
trap 'cleanup; exit 130' INT   # Ctrl+C
trap 'cleanup; exit 143' TERM  # kill

# ─── Multiple traps ───────────────────────────────────────────
# Stacking traps (bash doesn't support natively, but we can fake it)
add_trap() {
    local new_trap=$1
    local signal=$2
    local current_trap
    current_trap=$(trap -p "$signal" | grep -oP "(?<=trap -- ').*(?=' $signal)")
    
    if [[ -n "$current_trap" ]]; then
        trap "${new_trap}; ${current_trap}" "$signal"
    else
        trap "$new_trap" "$signal"
    fi
}

add_trap "echo 'First cleanup'" EXIT
add_trap "echo 'Second cleanup'" EXIT
# Runs: "Second cleanup" then "First cleanup"

# ─── Try/catch emulation ──────────────────────────────────────
try() {
    [[ $- = *e* ]]; SAVED_OPT_E=$?
    set +e
    "$@"
    EXCEPTION=$?
    (( SAVED_OPT_E )) && set -e
    return "$EXCEPTION"
}

catch() {
    (( EXCEPTION > 0 ))
}

# Usage
try command_that_might_fail
if catch; then
    echo "Command failed with: $EXCEPTION"
fi

# ─── Error handling with retry ────────────────────────────────
retry() {
    local max_attempts=${1:-3}
    local delay=${2:-5}
    local cmd=("${@:3}")
    local attempt=1
    
    while (( attempt <= max_attempts )); do
        echo "Attempt $attempt/$max_attempts..."
        
        if "${cmd[@]}"; then
            return 0
        fi
        
        (( attempt++ ))
        
        if (( attempt <= max_attempts )); then
            echo "Failed, retrying in ${delay}s..."
            sleep "$delay"
            delay=$(( delay * 2 ))  # exponential backoff
        fi
    done
    
    echo "Failed after $max_attempts attempts" >&2
    return 1
}

# Usage
retry 3 5 curl -sf https://api.example.com/health
```

---

## 18.3 Logging Framework

```bash
#!/bin/bash
# logging.sh - Professional logging for shell scripts

# ─── Log levels ───────────────────────────────────────────────
declare -A LOG_LEVELS=(
    [DEBUG]=0
    [INFO]=1
    [WARN]=2
    [ERROR]=3
    [FATAL]=4
)

LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FILE="${LOG_FILE:-}"
LOG_COLOR="${LOG_COLOR:-true}"

# ─── Colors ───────────────────────────────────────────────────
if [[ "$LOG_COLOR" == "true" && -t 2 ]]; then
    C_DEBUG='\033[36m'    # cyan
    C_INFO='\033[32m'     # green
    C_WARN='\033[33m'     # yellow
    C_ERROR='\033[31m'    # red
    C_FATAL='\033[1;31m'  # bold red
    C_RESET='\033[0m'
else
    C_DEBUG='' C_INFO='' C_WARN='' C_ERROR='' C_FATAL='' C_RESET=''
fi

# ─── Core log function ────────────────────────────────────────
_log() {
    local level=$1
    shift
    local message="$*"
    
    # Check if level is enabled
    [[ "${LOG_LEVELS[$level]}" -lt "${LOG_LEVELS[$LOG_LEVEL]}" ]] && return 0
    
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local caller_info="${BASH_SOURCE[2]##*/}:${BASH_LINENO[1]}"
    
    # Select color
    local color
    case $level in
        DEBUG) color=$C_DEBUG ;;
        INFO)  color=$C_INFO ;;
        WARN)  color=$C_WARN ;;
        ERROR) color=$C_ERROR ;;
        FATAL) color=$C_FATAL ;;
    esac
    
    # Format
    local log_entry="[$timestamp] [$level] [$caller_info] $message"
    
    # Output to stderr (colored)
    printf "%b%s%b\n" "$color" "$log_entry" "$C_RESET" >&2
    
    # Output to file (no color)
    if [[ -n "$LOG_FILE" ]]; then
        echo "$log_entry" >> "$LOG_FILE"
    fi
}

# Public log functions
log_debug() { _log DEBUG "$@"; }
log_info()  { _log INFO "$@"; }
log_warn()  { _log WARN "$@"; }
log_error() { _log ERROR "$@"; }
log_fatal() { _log FATAL "$@"; exit 1; }

# Usage
LOG_LEVEL=DEBUG LOG_FILE=/tmp/app.log ./myapp.sh

# Source this in other scripts:
# source /path/to/logging.sh
# log_info "Starting application"
# log_debug "Debug information"
# log_warn "Something might be wrong"
# log_error "Something went wrong"
# log_fatal "Fatal error, exiting"
```

---

## 18.4 Script Testing

```bash
#!/bin/bash
# test_framework.sh - Simple unit testing for bash scripts

# Test counters
TESTS_RUN=0
TESTS_PASSED=0
TESTS_FAILED=0
FAILED_TESTS=()

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# ─── Assertions ───────────────────────────────────────────────
assert_equals() {
    local description=$1
    local expected=$2
    local actual=$3
    
    (( TESTS_RUN++ ))
    
    if [[ "$expected" == "$actual" ]]; then
        echo -e "${GREEN}✓${NC} $description"
        (( TESTS_PASSED++ ))
    else
        echo -e "${RED}✗${NC} $description"
        echo "  Expected: $(printf '%q' "$expected")"
        echo "  Actual:   $(printf '%q' "$actual")"
        (( TESTS_FAILED++ ))
        FAILED_TESTS+=("$description")
    fi
}

assert_not_equals() {
    local description=$1
    local unexpected=$2
    local actual=$3
    
    (( TESTS_RUN++ ))
    
    if [[ "$unexpected" != "$actual" ]]; then
        echo -e "${GREEN}✓${NC} $description"
        (( TESTS_PASSED++ ))
    else
        echo -e "${RED}✗${NC} $description"
        echo "  Should not be: $(printf '%q' "$unexpected")"
        (( TESTS_FAILED++ ))
        FAILED_TESTS+=("$description")
    fi
}

assert_success() {
    local description=$1
    shift
    
    (( TESTS_RUN++ ))
    
    if "$@"; then
        echo -e "${GREEN}✓${NC} $description"
        (( TESTS_PASSED++ ))
    else
        echo -e "${RED}✗${NC} $description"
        echo "  Command failed: $*"
        (( TESTS_FAILED++ ))
        FAILED_TESTS+=("$description")
    fi
}

assert_fails() {
    local description=$1
    shift
    
    (( TESTS_RUN++ ))
    
    if ! "$@" 2>/dev/null; then
        echo -e "${GREEN}✓${NC} $description"
        (( TESTS_PASSED++ ))
    else
        echo -e "${RED}✗${NC} $description"
        echo "  Expected failure but succeeded: $*"
        (( TESTS_FAILED++ ))
        FAILED_TESTS+=("$description")
    fi
}

assert_file_exists() {
    local description=$1
    local file=$2
    
    (( TESTS_RUN++ ))
    
    if [[ -f "$file" ]]; then
        echo -e "${GREEN}✓${NC} $description"
        (( TESTS_PASSED++ ))
    else
        echo -e "${RED}✗${NC} $description"
        echo "  File not found: $file"
        (( TESTS_FAILED++ ))
        FAILED_TESTS+=("$description")
    fi
}

assert_contains() {
    local description=$1
    local expected=$2
    local haystack=$3
    
    (( TESTS_RUN++ ))
    
    if [[ "$haystack" == *"$expected"* ]]; then
        echo -e "${GREEN}✓${NC} $description"
        (( TESTS_PASSED++ ))
    else
        echo -e "${RED}✗${NC} $description"
        echo "  Expected to contain: $expected"
        echo "  In: $haystack"
        (( TESTS_FAILED++ ))
        FAILED_TESTS+=("$description")
    fi
}

# Print test results
print_results() {
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Test Results:"
    echo "  Total:  $TESTS_RUN"
    echo -e "  Passed: ${GREEN}$TESTS_PASSED${NC}"
    
    if (( TESTS_FAILED > 0 )); then
        echo -e "  Failed: ${RED}$TESTS_FAILED${NC}"
        echo ""
        echo "Failed tests:"
        for test in "${FAILED_TESTS[@]}"; do
            echo -e "  ${RED}✗${NC} $test"
        done
        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        return 1
    else
        echo -e "  ${GREEN}All tests passed!${NC}"
        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        return 0
    fi
}

# ─── Example Tests ────────────────────────────────────────────
# Source the file to test
source ./mylib.sh

echo "Running tests for mylib.sh..."
echo ""

# Test: add function
assert_equals "add 2+2" "4" "$(add 2 2)"
assert_equals "add negative" "-1" "$(add 2 -3)"

# Test: is_number function
assert_success "is_number with integer" is_number 42
assert_success "is_number with float" is_number 3.14
assert_fails "is_number with string" is_number "abc"

# Test: file creation
tmpfile=$(mktemp)
create_config "$tmpfile"
assert_file_exists "config file created" "$tmpfile"
assert_contains "config has required key" "database_host" "$(cat "$tmpfile")"
rm -f "$tmpfile"

print_results
```

---

## 18.5 Exercises

### Exercise 1: Error Handling Library
สร้าง library ที่ reusable:
- Configurable error levels
- Stack trace on error
- Notification on critical errors
- Rollback support

### Exercise 2: Test Runner
สร้าง test runner ที่:
- Discover test files automatically
- Run tests in parallel
- Generate HTML/JUnit XML report
- Show coverage metrics

### Exercise 3: Debug Dashboard
สร้าง tool ที่:
- Monitor script execution
- Capture performance metrics
- Display real-time logs
- Alert on errors

---

## สรุป Part 18

✅ Debug modes (set -x, set -e, set -u, pipefail)  
✅ PS4 customization for better traces  
✅ ERR trap and call stacks  
✅ Cleanup on exit/signals  
✅ Try/catch emulation  
✅ Retry with exponential backoff  
✅ Professional logging framework  
✅ Unit testing framework  
✅ Shellcheck static analysis  

---

**→ Part 19: Script Best Practices & Production-Ready Scripts**
