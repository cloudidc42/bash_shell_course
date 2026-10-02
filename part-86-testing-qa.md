# Part 86: Shell Script Testing and Quality Assurance
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 86.1 Unit Testing Framework

```bash
#!/bin/bash
# test_framework.sh - Minimal unit testing framework for Bash

declare -i TEST_PASS=0 TEST_FAIL=0 TEST_SKIP=0
declare -a TEST_FAILURES=()
declare -a TEST_SUITE=()
declare -A TEST_BEFORE_EACH=()
declare -A TEST_AFTER_EACH=()

# ─── Assertions ───────────────────────────────────────────────
assert_eq() {
    local expected=$1 actual=$2 message=${3:-"assert_eq"}
    if [[ "$expected" == "$actual" ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | expected='$expected' got='$actual'")
        return 1
    fi
}

assert_ne() {
    local unexpected=$1 actual=$2 message=${3:-"assert_ne"}
    if [[ "$unexpected" != "$actual" ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | '$actual' should not equal '$unexpected'")
        return 1
    fi
}

assert_contains() {
    local haystack=$1 needle=$2 message=${3:-"assert_contains"}
    if [[ "$haystack" == *"$needle"* ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | '$haystack' does not contain '$needle'")
        return 1
    fi
}

assert_empty() {
    local val=$1 message=${2:-"assert_empty"}
    if [[ -z "$val" ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | expected empty, got '$val'")
        return 1
    fi
}

assert_not_empty() {
    local val=$1 message=${2:-"assert_not_empty"}
    if [[ -n "$val" ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | expected non-empty, got empty string")
        return 1
    fi
}

assert_exits_ok() {
    local message=${1:-"assert_exits_ok"}; shift
    if "$@"; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | command returned non-zero: $*")
        return 1
    fi
}

assert_exits_err() {
    local message=${1:-"assert_exits_err"}; shift
    if ! "$@" 2>/dev/null; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | expected non-zero exit from: $*")
        return 1
    fi
}

assert_file_exists() {
    local file=$1 message=${2:-"assert_file_exists"}
    if [[ -f "$file" ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | file not found: $file")
        return 1
    fi
}

assert_match() {
    local value=$1 pattern=$2 message=${3:-"assert_match"}
    if [[ "$value" =~ $pattern ]]; then
        (( TEST_PASS++ ))
    else
        (( TEST_FAIL++ ))
        TEST_FAILURES+=("FAIL: $message | '$value' does not match /$pattern/")
        return 1
    fi
}

# ─── Test runner ──────────────────────────────────────────────
it() {
    local description=$1 fn=$2
    TEST_SUITE+=("$description|$fn")
}

skip() {
    local description=$1
    (( TEST_SKIP++ ))
    echo "  SKIP: $description"
}

run_tests() {
    local suite=${1:-default}
    echo "=== Test Suite: $suite ==="
    echo ""

    for entry in "${TEST_SUITE[@]}"; do
        local desc="${entry%%|*}"
        local fn="${entry#*|}"

        printf "  %-60s" "$desc"
        local fail_before=$TEST_FAIL

        # Run before_each if registered
        [[ -n "${TEST_BEFORE_EACH[$suite]:-}" ]] && "${TEST_BEFORE_EACH[$suite]}" 2>/dev/null || true

        if "$fn" 2>/dev/null; then
            if (( TEST_FAIL == fail_before )); then
                echo " PASS"
            else
                echo " FAIL"
            fi
        else
            (( TEST_FAIL++ ))
            echo " FAIL (error)"
        fi

        # Run after_each
        [[ -n "${TEST_AFTER_EACH[$suite]:-}" ]] && "${TEST_AFTER_EACH[$suite]}" 2>/dev/null || true
    done

    echo ""
    echo "Results: ${TEST_PASS} passed, ${TEST_FAIL} failed, ${TEST_SKIP} skipped"

    if (( ${#TEST_FAILURES[@]} > 0 )); then
        echo ""
        echo "Failures:"
        for f in "${TEST_FAILURES[@]}"; do
            echo "  $f"
        done
    fi

    (( TEST_FAIL == 0 ))
}
```

---

## 86.2 Integration Test Helpers

```bash
#!/bin/bash
# integration_test.sh - Integration testing helpers

# ─── Temp environment ─────────────────────────────────────────
TEST_TMPDIR=""

test_setup_tmpdir() {
    TEST_TMPDIR=$(mktemp -d)
    export TEST_TMPDIR
    trap "rm -rf '$TEST_TMPDIR'" RETURN
}

test_fixture_file() {
    local name=$1 content=$2
    echo "$content" > "${TEST_TMPDIR}/${name}"
    echo "${TEST_TMPDIR}/${name}"
}

# ─── Mock commands ────────────────────────────────────────────
declare -A MOCK_COMMANDS=()
declare -A MOCK_CALL_COUNTS=()

mock_command() {
    local name=$1 output=${2:-} exit_code=${3:-0}
    MOCK_COMMANDS["$name"]="$output|$exit_code"
    MOCK_CALL_COUNTS["$name"]=0

    # Create wrapper function
    eval "${name}() {
        MOCK_CALL_COUNTS[\"${name}\"]=$(( MOCK_CALL_COUNTS[\"${name}\"] + 1 ))
        local _mock_data=\"\${MOCK_COMMANDS[\"${name}\"]:-|}\"
        local _mock_output=\"\${_mock_data%%|*}\"
        local _mock_exit=\"\${_mock_data#*|}\"
        [[ -n \"\$_mock_output\" ]] && echo \"\$_mock_output\"
        return \"\$_mock_exit\"
    }"
}

mock_assert_called() {
    local name=$1 times=${2:-}
    local count="${MOCK_CALL_COUNTS[$name]:-0}"
    if [[ -n "$times" ]]; then
        assert_eq "$times" "$count" "mock '$name' called $times times"
    else
        [[ "$count" -gt 0 ]] || {
            (( TEST_FAIL++ ))
            TEST_FAILURES+=("FAIL: mock '$name' was never called")
            return 1
        }
        (( TEST_PASS++ ))
    fi
}

mock_reset() {
    local name=${1:-}
    if [[ -n "$name" ]]; then
        MOCK_CALL_COUNTS["$name"]=0
    else
        for k in "${!MOCK_CALL_COUNTS[@]}"; do
            MOCK_CALL_COUNTS["$k"]=0
        done
    fi
}

# ─── HTTP mock server ─────────────────────────────────────────
mock_http_start() {
    local port=${1:-18080}
    local response=${2:-'{"status":"ok"}'}

    # Use nc to serve single response
    echo -e "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n${response}" | \
        nc -l -p "$port" -q 1 &
    MOCK_HTTP_PID=$!
    sleep 0.2
    echo "$MOCK_HTTP_PID"
}

mock_http_stop() {
    local pid=${1:-${MOCK_HTTP_PID:-}}
    [[ -n "$pid" ]] && kill "$pid" 2>/dev/null || true
}
```

---

## 86.3 Linting and Static Analysis

```bash
#!/bin/bash
# lint_check.sh - Wrapper around shellcheck and custom rules

lint_shellcheck() {
    local -a files=("$@")
    local failed=0

    if ! command -v shellcheck &>/dev/null; then
        echo "shellcheck not installed, skipping" >&2
        return 0
    fi

    for file in "${files[@]}"; do
        [[ -f "$file" ]] || continue
        echo "Checking: $file"
        shellcheck --severity=warning --shell=bash "$file" || (( failed++ ))
    done

    return $failed
}

lint_custom_rules() {
    local file=$1
    local issues=0

    # No 'set -e' or 'set -euo pipefail' at top
    if ! grep -qP '^set -.*e' "$file" 2>/dev/null; then
        echo "  WARN: $file: no 'set -e' or 'set -euo pipefail'" >&2
        (( issues++ ))
    fi

    # Unquoted $variables in conditions
    if grep -nP '\[\[ \$[A-Za-z_]+ ' "$file" 2>/dev/null | grep -qv '"'; then
        echo "  WARN: $file: possibly unquoted variables in [[ ]]" >&2
    fi

    # echo without printf for formatted output
    if grep -cnP '^echo .*%[sd]' "$file" 2>/dev/null | grep -qv '^0$'; then
        echo "  INFO: $file: consider printf for formatted output" >&2
    fi

    # Using backticks instead of $()
    if grep -nqP '`[^`]+`' "$file" 2>/dev/null; then
        echo "  WARN: $file: use \$() instead of backticks" >&2
        (( issues++ ))
    fi

    return $issues
}

lint_dir() {
    local dir=${1:-.}
    local failed=0

    find "$dir" -name "*.sh" -o -name "*.bash" 2>/dev/null | while read -r f; do
        lint_shellcheck "$f" || (( failed++ ))
        lint_custom_rules "$f" || true
    done

    (( failed == 0 ))
}
```

---

## 86.4 Code Coverage for Shell Scripts

```bash
#!/bin/bash
# coverage.sh - Basic line coverage tracking for shell scripts

COVERAGE_DB="${COVERAGE_DB:-/tmp/coverage.db}"
COVERAGE_TARGET=""

coverage_init() {
    local script=$1
    COVERAGE_TARGET="$script"
    rm -f "$COVERAGE_DB"
    sqlite3 "$COVERAGE_DB" "
        CREATE TABLE lines (
            line_no INTEGER PRIMARY KEY,
            content TEXT,
            executed INTEGER DEFAULT 0
        );
    "

    local line_no=1
    while IFS= read -r line; do
        sqlite3 "$COVERAGE_DB" "
            INSERT INTO lines (line_no, content) VALUES ($line_no, $(printf '%s' "$line" | sed "s/'/''/g" | awk '{print "\x27" $0 "\x27"}'));
        "
        (( line_no++ ))
    done < "$script"
}

coverage_mark_executed() {
    local line_no=$1
    sqlite3 "$COVERAGE_DB" "UPDATE lines SET executed=1 WHERE line_no=$line_no;"
}

coverage_report() {
    local total executed pct
    total=$(sqlite3 "$COVERAGE_DB" "SELECT COUNT(*) FROM lines;")
    executed=$(sqlite3 "$COVERAGE_DB" "SELECT COUNT(*) FROM lines WHERE executed=1;")
    pct=$(awk "BEGIN{printf \"%.1f\", ($executed/$total)*100}")

    echo "=== Coverage Report: $COVERAGE_TARGET ==="
    printf "Lines: %d/%d (%.1f%%)\n" "$executed" "$total" "$pct"
    echo ""
    echo "Uncovered lines:"
    sqlite3 "$COVERAGE_DB" "SELECT line_no || ': ' || content FROM lines WHERE executed=0 LIMIT 20;"
}

coverage_threshold() {
    local threshold=$1
    local total executed pct
    total=$(sqlite3 "$COVERAGE_DB" "SELECT COUNT(*) FROM lines;")
    executed=$(sqlite3 "$COVERAGE_DB" "SELECT COUNT(*) FROM lines WHERE executed=1;")
    pct=$(awk "BEGIN{printf \"%.0f\", ($executed/$total)*100}")

    if (( pct < threshold )); then
        echo "FAIL: Coverage $pct% < threshold $threshold%" >&2
        return 1
    fi
    echo "PASS: Coverage $pct% >= threshold $threshold%"
}
```

---

## 86.5 Test Runner CLI

```bash
#!/bin/bash
# run_tests.sh - Test suite runner with reporting

declare -a TEST_FILES=()
declare -i TOTAL_PASS=0 TOTAL_FAIL=0
REPORT_FILE="${REPORT_FILE:-/tmp/test-report.txt}"

discover_tests() {
    local dir=${1:-.}
    while IFS= read -r f; do
        TEST_FILES+=("$f")
    done < <(find "$dir" -name "test_*.sh" -o -name "*_test.sh" 2>/dev/null | sort)
}

run_test_file() {
    local file=$1
    echo "--- $file ---"

    local output exit_code
    output=$(bash "$file" 2>&1)
    exit_code=$?

    echo "$output"

    local pass fail
    pass=$(echo "$output" | grep -c ' PASS$' || true)
    fail=$(echo "$output" | grep -c ' FAIL' || true)

    TOTAL_PASS=$(( TOTAL_PASS + pass ))
    TOTAL_FAIL=$(( TOTAL_FAIL + fail ))

    echo "" >> "$REPORT_FILE"
    echo "=== $file ===" >> "$REPORT_FILE"
    echo "$output" >> "$REPORT_FILE"

    return $exit_code
}

run_all_tests() {
    local dir=${1:-.}
    discover_tests "$dir"

    echo "Running ${#TEST_FILES[@]} test files..."
    echo "" > "$REPORT_FILE"

    local failed_files=()
    for file in "${TEST_FILES[@]}"; do
        run_test_file "$file" || failed_files+=("$file")
    done

    echo ""
    echo "=============================="
    echo "Total: ${TOTAL_PASS} passed, ${TOTAL_FAIL} failed"
    echo "Report: $REPORT_FILE"

    if (( ${#failed_files[@]} > 0 )); then
        echo ""
        echo "Failed files:"
        printf "  %s\n" "${failed_files[@]}"
        return 1
    fi
}
```

---

## 86.6 Exercises

### Exercise 1: Unit Test for Part 78 Queue
เขียน test suite สำหรับ `queue_enqueue()` และ `queue_dequeue()`:
- Test enqueue single item
- Test priority ordering
- Test dequeue when empty
- Test complete + fail transitions

### Exercise 2: Mock HTTP Test
ทดสอบ scraper functions จาก Part 83:
- Mock HTTP server with nc
- Assert correct URL encoding
- Assert rate limiting behavior

### Exercise 3: CI Integration
สร้าง `ci_test.sh` ที่:
- Runs shellcheck on all .sh files
- Runs unit tests
- Checks coverage threshold
- Outputs JUnit XML format

---

## สรุป Part 86

✅ assert_eq/ne/contains/empty/not_empty/exits_ok/exits_err/file_exists/match
┅ it() + run_tests(): test registry and runner with before/after_each hooks
┅ mock_command: eval-based wrapper, call count tracking, mock_assert_called
┅ mock_http_start/stop: nc-based single-response HTTP mock
┅ lint_shellcheck: wrapper + severity filter; lint_custom_rules: set -e, backticks, etc.
┅ coverage_init/mark/report/threshold: SQLite-backed line coverage
┅ run_test_file + run_all_tests: discover *_test.sh files, aggregate pass/fail

---

**→ Part 87: Shell Script Performance and Profiling**
