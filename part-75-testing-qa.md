# Part 75: Shell Script Testing and Quality Assurance
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 75.1 Test Framework

```bash
#!/bin/bash
# test_framework.sh - Minimal but complete Bash test framework

set -euo pipefail

# ─── State ───────────────────────────────────────────────────────────────────
declare -i TEST_PASS=0 TEST_FAIL=0 TEST_SKIP=0
declare -a TEST_FAILURES=()
TEST_CURRENT=""
TEST_START_TIME=0

# ─── Assertions ─────────────────────────────────────────────────────────────
assert_equals() {
    local expected=$1 actual=$2 message=${3:-}
    if [[ "$expected" == "$actual" ]]; then
        return 0
    fi
    _test_fail "assert_equals" "expected='$expected' actual='$actual'" "$message"
}

assert_not_equals() {
    local expected=$1 actual=$2 message=${3:-}
    if [[ "$expected" != "$actual" ]]; then
        return 0
    fi
    _test_fail "assert_not_equals" "both='$actual'" "$message"
}

assert_contains() {
    local haystack=$1 needle=$2 message=${3:-}
    if [[ "$haystack" == *"$needle"* ]]; then
        return 0
    fi
    _test_fail "assert_contains" "'$haystack' does not contain '$needle'" "$message"
}

assert_matches() {
    local value=$1 pattern=$2 message=${3:-}
    if [[ "$value" =~ $pattern ]]; then
        return 0
    fi
    _test_fail "assert_matches" "'$value' does not match pattern '$pattern'" "$message"
}

assert_exit_code() {
    local expected_code=$1
    shift
    local cmd=("$@")
    local actual_code=0
    "${cmd[@]}" &>/dev/null || actual_code=$?
    if (( expected_code == actual_code )); then
        return 0
    fi
    _test_fail "assert_exit_code" "expected=$expected_code actual=$actual_code for: ${cmd[*]}"
}

assert_file_exists() {
    local file=$1 message=${2:-}
    [[ -f "$file" ]] && return 0
    _test_fail "assert_file_exists" "file not found: $file" "$message"
}

assert_file_contains() {
    local file=$1 pattern=$2 message=${3:-}
    grep -q "$pattern" "$file" 2>/dev/null && return 0
    _test_fail "assert_file_contains" "pattern '$pattern' not in file '$file'" "$message"
}

assert_empty() {
    local value=$1 message=${2:-}
    [[ -z "$value" ]] && return 0
    _test_fail "assert_empty" "value is not empty: '$value'" "$message"
}

assert_not_empty() {
    local value=$1 message=${2:-}
    [[ -n "$value" ]] && return 0
    _test_fail "assert_not_empty" "value is empty" "$message"
}

assert_true() {
    local condition=$1 message=${2:-}
    eval "$condition" 2>/dev/null && return 0
    _test_fail "assert_true" "condition is false: $condition" "$message"
}

_test_fail() {
    local assertion=$1 detail=$2 message=${3:-}
    local caller="${BASH_SOURCE[2]:-}:${BASH_LINENO[1]:-}"

    (( TEST_FAIL++ ))
    local failure_msg="FAIL: $TEST_CURRENT\n  $assertion: $detail"
    [[ -n "$message" ]] && failure_msg+="\n  msg: $message"
    failure_msg+="\n  at: $caller"

    TEST_FAILURES+=("$failure_msg")
    printf '\033[0;31mF\033[0m'
    return 1
}

# ─── Test runner ─────────────────────────────────────────────────────────────
describe() {
    local suite_name=$1
    echo ""
    echo "$suite_name"
}

it() {
    local test_name=$1
    shift
    local test_fn=${1:-}
    local start_ns; start_ns=$(date +%s%N 2>/dev/null || echo 0)

    TEST_CURRENT="$test_name"

    local prev_fail=$TEST_FAIL
    local exit_code=0

    if [[ -n "$test_fn" ]]; then
        "$test_fn" || exit_code=$?
    else
        eval "${test_name//[^a-zA-Z0-9_]/_}" || exit_code=$?
    fi

    local end_ns; end_ns=$(date +%s%N 2>/dev/null || echo 0)
    local ms=$(( (end_ns - start_ns) / 1000000 ))

    if (( TEST_FAIL > prev_fail )); then
        : # already printed F
    else
        (( TEST_PASS++ ))
        printf '\033[0;32m.\033[0m'
    fi
}

skip() {
    local test_name=$1
    (( TEST_SKIP++ ))
    printf '\033[0;33mS\033[0m'
}

test_summary() {
    echo ""
    echo ""
    echo "=== Test Results ==="
    printf 'Passed: \033[0;32m%d\033[0m  Failed: \033[0;31m%d\033[0m  Skipped: \033[0;33m%d\033[0m\n' \
        "$TEST_PASS" "$TEST_FAIL" "$TEST_SKIP"

    if (( ${#TEST_FAILURES[@]} > 0 )); then
        echo ""
        echo "Failures:"
        for failure in "${TEST_FAILURES[@]}"; do
            printf '%b\n' "$failure"
            echo ""
        done
    fi

    (( TEST_FAIL == 0 ))
}

trap test_summary EXIT
```

---

## 75.2 TAP Output Format

```bash
#!/bin/bash
# tap_runner.sh - TAP (Test Anything Protocol) compatible runner

declare -i TAP_TEST_NUM=0
declare -i TAP_FAILURES=0
declare -a TAP_PLAN=()

tap_plan() {
    local count=$1
    echo "1..${count}"
}

tap_ok() {
    local description=$1
    (( TAP_TEST_NUM++ ))
    printf 'ok %d - %s\n' "$TAP_TEST_NUM" "$description"
}

tap_not_ok() {
    local description=$1 diagnostic=${2:-}
    (( TAP_TEST_NUM++ ))
    (( TAP_FAILURES++ ))
    printf 'not ok %d - %s\n' "$TAP_TEST_NUM" "$description"
    [[ -n "$diagnostic" ]] && printf '# %s\n' "$diagnostic"
}

tap_skip() {
    local description=$1 reason=${2:-}
    (( TAP_TEST_NUM++ ))
    printf 'ok %d # SKIP %s - %s\n' "$TAP_TEST_NUM" "$reason" "$description"
}

tap_todo() {
    local description=$1 explanation=${2:-}
    (( TAP_TEST_NUM++ ))
    printf 'not ok %d # TODO %s - %s\n' "$TAP_TEST_NUM" "$explanation" "$description"
}

tap_run_test() {
    local description=$1
    shift
    local cmd=("$@")

    local output exit_code=0
    output=$("${cmd[@]}" 2>&1) || exit_code=$?

    if (( exit_code == 0 )); then
        tap_ok "$description"
    else
        tap_not_ok "$description" "exit=$exit_code: $output"
    fi
}

tap_results() {
    (( TAP_FAILURES == 0 ))
}
```

---

## 75.3 Mocking and Stubs

```bash
#!/bin/bash
# mocks.sh - Function mocking and command stubbing

declare -A MOCK_CALLS=()
declare -A MOCK_RETURN=()
declare -A MOCK_OUTPUT=()

mock_function() {
    local name=$1 return_code=${2:-0} output=${3:-}

    MOCK_CALLS["$name"]=0
    MOCK_RETURN["$name"]="$return_code"
    MOCK_OUTPUT["$name"]="$output"

    # Save original if exists
    if declare -f "$name" &>/dev/null; then
        eval "_original_${name}() { $(declare -f $name | tail -n +3)"
    fi

    # Create mock
    eval "${name}() {
        MOCK_CALLS[${name}]=\$(( \${MOCK_CALLS[${name}]:-0} + 1 ))
        local out=\"\${MOCK_OUTPUT[${name}]:-}\"
        [[ -n \"\$out\" ]] && echo \"\$out\"
        return \${MOCK_RETURN[${name}]:-0}
    }"
}

mock_restore() {
    local name=$1
    if declare -f "_original_${name}" &>/dev/null; then
        eval "${name}() { $(declare -f _original_${name} | tail -n +3)"
        unset -f "_original_${name}"
    else
        unset -f "$name" 2>/dev/null || true
    fi
    unset "MOCK_CALLS[$name]" "MOCK_RETURN[$name]" "MOCK_OUTPUT[$name]"
}

mock_assert_called() {
    local name=$1 times=${2:-1}
    local actual="${MOCK_CALLS[$name]:-0}"
    if (( actual == times )); then
        return 0
    fi
    _test_fail "mock_assert_called" "$name called $actual times (expected $times)"
}

mock_assert_called_at_least() {
    local name=$1 min=$2
    local actual="${MOCK_CALLS[$name]:-0}"
    (( actual >= min )) || _test_fail "mock_assert_called_at_least" "$name called $actual times (min $min)"
}

# Command stub via PATH shim
stub_command() {
    local cmd=$1 return_code=${2:-0} output=${3:-}
    local stub_dir="${TEST_STUB_DIR:-/tmp/test_stubs}"
    mkdir -p "$stub_dir"

    cat > "${stub_dir}/${cmd}" << EOF
#!/bin/bash
echo "${output}"
exit ${return_code}
EOF
    chmod +x "${stub_dir}/${cmd}"
    export PATH="${stub_dir}:${PATH}"
}

stub_cleanup() {
    local stub_dir="${TEST_STUB_DIR:-/tmp/test_stubs}"
    rm -rf "$stub_dir"
    export PATH="${PATH#${stub_dir}:}"
}

# File fixture helpers
fixture_create_file() {
    local path=$1 content=$2
    mkdir -p "$(dirname "$path")"
    echo "$content" > "$path"
}

fixture_create_dir() {
    local dir=$1
    mkdir -p "$dir"
}

fixture_teardown() {
    local dir=$1
    [[ -n "$dir" && "$dir" != "/" ]] && rm -rf "$dir"
}
```

---

## 75.4 ShellCheck Integration

```bash
#!/bin/bash
# shellcheck_runner.sh - ShellCheck linting wrapper

SHELLCHECK_SEVERITY="${SHELLCHECK_SEVERITY:-warning}"
SHELLCHECK_EXCLUDE="${SHELLCHECK_EXCLUDE:-}"
SHELLCHECK_SHELL="${SHELLCHECK_SHELL:-bash}"

lint_file() {
    local file=$1 severity=${2:-$SHELLCHECK_SEVERITY}

    if ! command -v shellcheck &>/dev/null; then
        echo "shellcheck not installed" >&2
        return 1
    fi

    local -a sc_args=(
        --severity="$severity"
        --shell="$SHELLCHECK_SHELL"
        --format=gcc
    )

    [[ -n "$SHELLCHECK_EXCLUDE" ]] && sc_args+=(--exclude="$SHELLCHECK_EXCLUDE")

    shellcheck "${sc_args[@]}" "$file"
}

lint_directory() {
    local dir=${1:-.} severity=${2:-$SHELLCHECK_SEVERITY}
    local passed=0 failed=0

    while IFS= read -r -d '' file; do
        if lint_file "$file" "$severity" 2>/dev/null; then
            (( passed++ ))
            printf '\033[0;32m.\033[0m'
        else
            (( failed++ ))
            printf '\033[0;31mF\033[0m'
        fi
    done < <(find "$dir" -name '*.sh' -print0 2>/dev/null)

    echo ""
    echo "ShellCheck: $passed passed, $failed failed"
    (( failed == 0 ))
}

lint_changed_files() {
    local base_branch=${1:-main}

    local changed_files
    changed_files=$(git diff --name-only "${base_branch}...HEAD" 2>/dev/null | grep '\.sh$' || true)

    if [[ -z "$changed_files" ]]; then
        echo "No shell files changed"
        return 0
    fi

    local passed=0 failed=0
    while IFS= read -r file; do
        [[ -f "$file" ]] || continue
        if lint_file "$file"; then
            (( passed++ ))
        else
            (( failed++ ))
        fi
    done <<< "$changed_files"

    echo "Changed files: $passed passed, $failed failed"
    (( failed == 0 ))
}
```

---

## 75.5 Code Coverage

```bash
#!/bin/bash
# coverage.sh - Line-hit coverage via PS4 tracing

COVERAGE_DIR="${COVERAGE_DIR:-/tmp/coverage}"
mkdir -p "$COVERAGE_DIR"

coverage_start() {
    local script_under_test=$1
    local trace_file="${COVERAGE_DIR}/trace.log"

    export COVERAGE_SCRIPT="$script_under_test"
    export COVERAGE_TRACE="$trace_file"

    PS4='+COV:${BASH_SOURCE}:${LINENO}: '
    exec 4>"$trace_file"
    BASH_XTRACEFD=4
    set -x
}

coverage_stop() {
    set +x
    exec 4>&-
}

coverage_report() {
    local script=$1
    local trace_file="${COVERAGE_TRACE:-${COVERAGE_DIR}/trace.log}"

    if [[ ! -f "$trace_file" ]]; then
        echo "No coverage data found"
        return 1
    fi

    local total_lines; total_lines=$(wc -l < "$script")
    local hit_lines; hit_lines=$(grep "+COV:${script}:" "$trace_file" 2>/dev/null | \
        grep -oP ':([0-9]+):' | sort -un | wc -l)

    local pct=0
    (( total_lines > 0 )) && pct=$(( hit_lines * 100 / total_lines ))

    echo "=== Coverage Report: $script ==="
    echo "Total lines:   $total_lines"
    echo "Executed:      $hit_lines"
    echo "Coverage:      ${pct}%"

    # Show uncovered lines
    local -A covered=()
    while IFS= read -r line; do
        local lineno; lineno=$(echo "$line" | grep -oP ':([0-9]+):' | tr -d ':')
        [[ -n "$lineno" ]] && covered["$lineno"]=1
    done < <(grep "+COV:${script}:" "$trace_file" 2>/dev/null)

    echo ""
    echo "Uncovered lines:"
    local lnum=1
    while IFS= read -r line; do
        if [[ -z "${covered[$lnum]:-}" ]]; then
            printf '  %4d: %s\n' "$lnum" "$line"
        fi
        (( lnum++ ))
    done < "$script"
}
```

---

## 75.6 Property-Based Testing

```bash
#!/bin/bash
# property_test.sh - Property-based testing (QuickCheck-style)

PROP_RUNS="${PROP_RUNS:-100}"
PROP_SEED="${PROP_SEED:-$RANDOM}"

gen_integer() {
    local min=${1:--1000} max=${2:-1000}
    echo $(( min + RANDOM % (max - min + 1) ))
}

gen_string() {
    local len=${1:-20} charset=${2:-'a-zA-Z0-9'}
    tr -dc "$charset" < /dev/urandom | head -c "$len"
}

gen_nonempty_string() {
    local len=${1:-20}
    local s
    s=$(gen_string "$len")
    [[ -z "$s" ]] && s="x"
    echo "$s"
}

gen_positive_int() {
    echo $(( 1 + RANDOM % 10000 ))
}

gen_float() {
    awk 'BEGIN{srand(); printf "%.4f", rand()*2000-1000}'
}

gen_array() {
    local size=${1:-10} min=${2:-0} max=${3:-100}
    local -a arr=()
    local i
    for (( i=0; i<size; i++ )); do
        arr+=( "$(gen_integer $min $max)" )
    done
    echo "${arr[@]}"
}

property_test() {
    local name=$1 property_fn=$2 gen_fn=$3
    local runs=${4:-$PROP_RUNS}
    local shrink_attempts=50

    local passed=0 failed=0
    local failing_input=""

    for (( i=0; i<runs; i++ )); do
        local input; input=$("$gen_fn")

        if ! "$property_fn" "$input" 2>/dev/null; then
            (( failed++ ))
            failing_input="$input"
            # Simple shrinking: try empty/zero/minimal input
            for shrunk in "" "0" "1" "a"; do
                if ! "$property_fn" "$shrunk" 2>/dev/null; then
                    failing_input="$shrunk"
                    break
                fi
            done
            break
        fi
        (( passed++ ))
    done

    if (( failed > 0 )); then
        printf '\033[0;31mFAIL\033[0m property: %s\n' "$name"
        printf '  Failing input: %s\n' "$failing_input"
        printf '  Passed %d/%d runs before failure\n' "$passed" "$runs"
        return 1
    fi

    printf '\033[0;32mPASS\033[0m property: %s (%d runs)\n' "$name" "$passed"
}

# Example properties:
# reverse_twice_is_identity() {
#     local s=$1
#     local rev; rev=$(echo "$s" | rev)
#     local rev2; rev2=$(echo "$rev" | rev)
#     [[ "$s" == "$rev2" ]]
# }
# property_test "reverse_twice" reverse_twice_is_identity gen_nonempty_string
```

---

## 75.7 Integration Test Harness

```bash
#!/bin/bash
# integration_harness.sh - Integration test setup/teardown

INT_TEST_DIR="$(mktemp -d /tmp/int_test_XXXXXX)"
INT_TEST_LOG="${INT_TEST_DIR}/test.log"
declare -a INT_CLEANUP_CMDS=()

int_log() {
    echo "$(date '+%H:%M:%S') $*" | tee -a "$INT_TEST_LOG"
}

int_cleanup_register() {
    INT_CLEANUP_CMDS+=("$1")
}

int_cleanup() {
    int_log "Running cleanup..."
    for cmd in "${INT_CLEANUP_CMDS[@]}"; do
        eval "$cmd" 2>/dev/null || true
    done
    rm -rf "$INT_TEST_DIR" 2>/dev/null || true
}

trap int_cleanup EXIT INT TERM

# Start a test service
int_start_server() {
    local name=$1 start_cmd=$2 port=$3 timeout=${4:-10}

    int_log "Starting $name on port $port"
    eval "$start_cmd" &
    local pid=$!
    int_cleanup_register "kill $pid 2>/dev/null || true"

    # Wait for port to open
    local waited=0
    while ! timeout 1 bash -c "echo >/dev/tcp/localhost/${port}" 2>/dev/null; do
        sleep 0.5
        (( waited++ ))
        if (( waited > timeout * 2 )); then
            int_log "TIMEOUT: $name did not start on port $port"
            return 1
        fi
    done

    int_log "$name started (pid=$pid)"
    echo "$pid"
}

int_stop_server() {
    local name=$1 pid=$2
    int_log "Stopping $name (pid=$pid)"
    kill "$pid" 2>/dev/null || true
    wait "$pid" 2>/dev/null || true
}

int_create_test_db() {
    local db="${INT_TEST_DIR}/test.db"
    sqlite3 "$db" "
        CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT);
        INSERT INTO users VALUES (1,'Alice','alice@test.com');
        INSERT INTO users VALUES (2,'Bob','bob@test.com');
    " 2>/dev/null
    int_cleanup_register "rm -f '$db'"
    echo "$db"
}

int_test_http_endpoint() {
    local name=$1 url=$2 expected_status=${3:-200} expected_body=${4:-}

    local response http_code body
    response=$(curl --silent --write-out '\n%{http_code}' --max-time 5 "$url" 2>/dev/null)
    http_code=$(echo "$response" | tail -1)
    body=$(echo "$response" | head -n -1)

    if [[ "$http_code" != "$expected_status" ]]; then
        int_log "FAIL $name: HTTP $http_code (expected $expected_status)"
        return 1
    fi

    if [[ -n "$expected_body" ]] && ! echo "$body" | grep -q "$expected_body"; then
        int_log "FAIL $name: body pattern not found: $expected_body"
        return 1
    fi

    int_log "PASS $name: HTTP $http_code"
}
```

---

## 75.8 CI Test Runner

```bash
#!/bin/bash
# ci_runner.sh - CI-compatible test runner

CI_TEST_DIRS=("tests/unit" "tests/integration")
CI_REPORT_DIR="${CI_REPORT_DIR:-test-results}"
CI_EXIT_CODE=0

mkdir -p "$CI_REPORT_DIR"

run_test_file() {
    local file=$1 suite_name
    suite_name=$(basename "$file" .sh)

    local report_file="${CI_REPORT_DIR}/${suite_name}.xml"
    local start_ns; start_ns=$(date +%s%N 2>/dev/null || echo 0)
    local output exit_code=0

    output=$(bash "$file" 2>&1) || exit_code=$?

    local end_ns; end_ns=$(date +%s%N 2>/dev/null || echo 0)
    local ms=$(( (end_ns - start_ns) / 1000000 ))

    # JUnit XML
    cat > "$report_file" << XML
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="${suite_name}" time="$(( ms / 1000 ))" errors="0" failures="$((exit_code > 0 ? 1 : 0))">
  <testcase name="${suite_name}" time="$(( ms / 1000 ))">
$(if (( exit_code != 0 )); then echo "    <failure message=\"test failed\">$(echo "$output" | sed 's/</\&lt;/g; s/>/\&gt;/g' | head -20)</failure>"; fi)
  </testcase>
</testsuite>
XML

    if (( exit_code == 0 )); then
        printf '\033[0;32mPASS\033[0m %s (%dms)\n' "$suite_name" "$ms"
    else
        printf '\033[0;31mFAIL\033[0m %s (%dms)\n' "$suite_name" "$ms"
        echo "$output" | sed 's/^/  /'
        CI_EXIT_CODE=1
    fi
}

run_all_tests() {
    local test_pattern=${1:-'test_*.sh'}
    echo "=== CI Test Run ==="
    echo ""

    for dir in "${CI_TEST_DIRS[@]}"; do
        [[ -d "$dir" ]] || continue
        for file in "${dir}"/${test_pattern}; do
            [[ -f "$file" ]] && run_test_file "$file"
        done
    done

    echo ""
    if (( CI_EXIT_CODE == 0 )); then
        printf '\033[0;32mAll tests passed\033[0m\n'
    else
        printf '\033[0;31mSome tests failed\033[0m\n'
    fi

    return $CI_EXIT_CODE
}
```

---

## 75.9 Exercises

### Exercise 1: Test a Utility Library
เขียน test suite สำหรับ library จาก Part 72 ที่:
- `word_frequency()` จากไฟล์อื่น
- `csv_filter()` ตรวจ edge cases
- `markdown_to_html()` ตรวจ headings/lists
- ครอบคลุม 100% branches

### Exercise 2: Mock-Based Test
เขียน test suite สำหรับ HTTP client โดย:
- Mock curl ให้ return fixed responses
- Test retry logic
- Test auth header injection
- Test error handling

### Exercise 3: Property Test a Sort
เขียน property tests สำหรับ sort function:
- Output is sorted (each element ≤ next)
- Length preserved
- All input elements present in output
- Idempotent (sort(sort(x)) = sort(x))

---

## สรุป Part 75

✅ Test framework: assert_equals/not_equals/contains/matches/exit_code/file_exists/empty
┅ TAP output: plan, ok, not ok, skip, todo
┅ Mocking: function override with call counter, command stub via PATH shim
┅ Fixture: file/dir creation and teardown helpers
┅ ShellCheck: file/directory lint, changed-files-only mode
┅ Coverage: PS4 trace + line-hit counting, uncovered-line report
┅ Property testing: gen_integer/string/array, property_test with simple shrinking
┅ Integration harness: start_server with port-wait, test DB, HTTP assertion
┅ CI runner: JUnit XML output, per-file timing, aggregate PASS/FAIL exit code

---

**→ Part 76: Package Management and Dependency Automation**
