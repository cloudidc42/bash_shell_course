# Part 50: Shell Script Testing and Quality Assurance
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 50.1 Unit Testing Framework (BATS-style)

```bash
#!/bin/bash
# bash_test_framework.sh - Minimal unit test framework

declare -a TEST_SUITE=()
declare -i TEST_PASSED=0
declare -i TEST_FAILED=0
declare -i TEST_SKIPPED=0
declare -a TEST_FAILURES=()
CURRENT_TEST_NAME=""

# ─── Test Registration ────────────────────────────────────────
test() {
    local name=$1
    local func=$2
    TEST_SUITE+=("$name:$func")
}

skip_test() {
    local name=$1
    TEST_SUITE+=("SKIP:$name:")
}

# ─── Assertions ────────────────────────────────────────────────
assert_equal() {
    local actual=$1
    local expected=$2
    local message=${3:-"Expected '$expected', got '$actual'"}

    if [[ "$actual" == "$expected" ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_not_equal() {
    local actual=$1
    local unexpected=$2
    local message=${3:-"Expected value to not be '$unexpected'"}

    if [[ "$actual" != "$unexpected" ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_empty() {
    local value=$1
    local message=${2:-"Expected empty string, got '$value'"}

    if [[ -z "$value" ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_not_empty() {
    local value=$1
    local message=${2:-"Expected non-empty string"}

    if [[ -n "$value" ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_contains() {
    local haystack=$1
    local needle=$2
    local message=${3:-"Expected '$haystack' to contain '$needle'"}

    if [[ "$haystack" == *"$needle"* ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_matches() {
    local value=$1
    local pattern=$2
    local message=${3:-"Expected '$value' to match pattern '$pattern'"}

    if [[ "$value" =~ $pattern ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_true() {
    local expression=$1
    local message=${2:-"Expression is false: $expression"}

    if eval "$expression" 2>/dev/null; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_false() {
    local expression=$1
    local message=${2:-"Expression is true: $expression"}

    if ! eval "$expression" 2>/dev/null; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_file_exists() {
    local path=$1
    local message=${2:-"File not found: $path"}

    if [[ -f "$path" ]]; then
        return 0
    else
        fail_test "$message"
        return 1
    fi
}

assert_exit_code() {
    local cmd=$1
    local expected_code=${2:-0}

    eval "$cmd"
    local actual_code=$?

    if (( actual_code == expected_code )); then
        return 0
    else
        fail_test "Expected exit $expected_code, got $actual_code for: $cmd"
        return 1
    fi
}

fail_test() {
    local message=$1
    local line=${BASH_LINENO[1]}
    TEST_FAILURES+=("FAIL [$CURRENT_TEST_NAME:$line] $message")
    return 1
}

# ─── Test Runner ───────────────────────────────────────────────
run_tests() {
    local pattern=${1:-""}
    local verbose=${VERBOSE:-false}

    echo "=== Test Suite ==="
    echo ""

    for entry in "${TEST_SUITE[@]}"; do
        local name="${entry%%:*}"
        local func="${entry#*:}"

        if [[ "$name" == "SKIP" ]]; then
            (( TEST_SKIPPED++ ))
            echo "  SKIP  ${func}"
            continue
        fi

        [[ -n "$pattern" ]] && ! [[ "$name" == *"$pattern"* ]] && continue

        CURRENT_TEST_NAME="$name"
        local test_failures_before=${#TEST_FAILURES[@]}

        (
            set -e
            "$func"
        )
        local exit_code=$?

        local new_failures=$(( ${#TEST_FAILURES[@]} - test_failures_before ))

        if (( new_failures == 0 && exit_code == 0 )); then
            (( TEST_PASSED++ ))
            $verbose && echo "  PASS  $name" || printf "."
        else
            (( TEST_FAILED++ ))
            $verbose && echo "  FAIL  $name" || printf "F"
        fi
    done

    echo ""
    echo ""
    echo "Results: ${TEST_PASSED} passed / ${TEST_FAILED} failed / ${TEST_SKIPPED} skipped"

    if (( ${#TEST_FAILURES[@]} > 0 )); then
        echo ""
        echo "Failures:"
        for failure in "${TEST_FAILURES[@]}"; do
            echo "  $failure"
        done
        return 1
    fi

    return 0
}
```

---

## 50.2 Integration Testing

```bash
#!/bin/bash
# integration_tests.sh - Integration test helpers

# ─── Test Fixtures ────────────────────────────────────────────
FIXTURE_DIR=""

setup_fixtures() {
    FIXTURE_DIR=$(mktemp -d)
    echo "Fixtures: $FIXTURE_DIR"
}

teardown_fixtures() {
    [[ -n "$FIXTURE_DIR" && -d "$FIXTURE_DIR" ]] && rm -rf "$FIXTURE_DIR"
    FIXTURE_DIR=""
}

create_fixture_file() {
    local name=$1
    local content=$2
    local path="${FIXTURE_DIR}/${name}"
    mkdir -p "$(dirname "$path")"
    echo "$content" > "$path"
    echo "$path"
}

# ─── Process Testing ───────────────────────────────────────────
test_command_output() {
    local cmd=$1
    local expected_output=$2
    local expected_exit=${3:-0}

    local actual_output actual_exit
    actual_output=$(eval "$cmd" 2>&1)
    actual_exit=$?

    if (( actual_exit != expected_exit )); then
        echo "FAIL: Exit code $actual_exit != $expected_exit" >&2
        echo "Command: $cmd" >&2
        echo "Output: $actual_output" >&2
        return 1
    fi

    if [[ "$actual_output" != *"$expected_output"* ]]; then
        echo "FAIL: Output missing expected string" >&2
        echo "Expected to contain: $expected_output" >&2
        echo "Actual: $actual_output" >&2
        return 1
    fi

    return 0
}

# ─── HTTP Testing ──────────────────────────────────────────────
assert_http_status() {
    local url=$1
    local expected_status=$2
    local method=${3:-GET}
    local body=${4:-}

    local actual_status
    actual_status=$(curl -sf -o /dev/null -w "%{http_code}" \
        --max-time 10 \
        --request "$method" \
        ${body:+--data "$body"} \
        "$url" 2>/dev/null)

    if [[ "$actual_status" == "$expected_status" ]]; then
        echo "PASS: $method $url → $actual_status"
        return 0
    else
        echo "FAIL: $method $url → $actual_status (expected $expected_status)" >&2
        return 1
    fi
}

assert_json_field() {
    local url=$1
    local json_path=$2
    local expected_value=$3

    local response
    response=$(curl -sf --max-time 10 "$url" 2>/dev/null)

    local actual_value
    actual_value=$(echo "$response" | jq -r "$json_path" 2>/dev/null)

    if [[ "$actual_value" == "$expected_value" ]]; then
        echo "PASS: $json_path = $actual_value"
        return 0
    else
        echo "FAIL: $json_path = $actual_value (expected $expected_value)" >&2
        return 1
    fi
}

# ─── Database Testing ──────────────────────────────────────────
test_db_connection() {
    local db_type=$1
    local dsn=${2:-}

    case "$db_type" in
        mysql)
            mysql -e "SELECT 1;" &>/dev/null && echo "PASS: MySQL connected" || \
                { echo "FAIL: MySQL connection failed" >&2; return 1; }
            ;;
        postgres)
            psql -c "SELECT 1;" &>/dev/null && echo "PASS: PostgreSQL connected" || \
                { echo "FAIL: PostgreSQL connection failed" >&2; return 1; }
            ;;
        redis)
            redis-cli PING 2>/dev/null | grep -q PONG && echo "PASS: Redis connected" || \
                { echo "FAIL: Redis connection failed" >&2; return 1; }
            ;;
    esac
}

# ─── Docker Integration Tests ──────────────────────────────────
test_with_docker() {
    local image=$1
    local run_args=${2:-}
    local test_func=$3

    echo "Starting test container: $image"
    local container_id
    container_id=$(docker run -d $run_args "$image")

    trap "docker rm -f $container_id &>/dev/null" EXIT

    local timeout=30
    local waited=0
    while ! docker exec "$container_id" true &>/dev/null && (( waited < timeout )); do
        sleep 1
        (( waited++ ))
    done

    "$test_func" "$container_id"
    local result=$?

    docker rm -f "$container_id" &>/dev/null
    trap - EXIT

    return $result
}
```

---

## 50.3 Static Analysis and Linting

```bash
#!/bin/bash
# static_analysis.sh - Script quality checks

analyze_script() {
    local script_file=$1
    local report_file=${2:-/tmp/analysis_report.txt}
    local issues=0

    echo "Analyzing: $script_file" | tee "$report_file"
    echo "Date: $(date)" | tee -a "$report_file"
    echo "" | tee -a "$report_file"

    # ─── ShellCheck ──────────────────────────────────────────
    if command -v shellcheck &>/dev/null; then
        echo "=== ShellCheck ===" | tee -a "$report_file"
        if shellcheck --format=tty "$script_file" 2>&1 | tee -a "$report_file"; then
            echo "ShellCheck: PASSED" | tee -a "$report_file"
        else
            echo "ShellCheck: ISSUES FOUND" | tee -a "$report_file"
            (( issues++ ))
        fi
        echo "" | tee -a "$report_file"
    fi

    # ─── Custom Rules ─────────────────────────────────────────
    echo "=== Custom Analysis ===" | tee -a "$report_file"

    # Check for missing shebang
    if ! head -1 "$script_file" | grep -q "^#!"; then
        echo "WARN: Missing shebang line" | tee -a "$report_file"
        (( issues++ ))
    fi

    # Check for set -e or set -euo pipefail
    if ! grep -q "set -[eE]" "$script_file" && ! grep -q "set -.*e" "$script_file"; then
        echo "WARN: Consider using 'set -euo pipefail'" | tee -a "$report_file"
    fi

    # Check for unquoted variables
    local unquoted
    unquoted=$(grep -n '\$[A-Za-z_][A-Za-z0-9_]*[^"]' "$script_file" 2>/dev/null | \
        grep -v '#' | grep -v 'echo\|printf' | wc -l)
    if (( unquoted > 5 )); then
        echo "WARN: $unquoted potentially unquoted variables found" | tee -a "$report_file"
    fi

    # Check for eval usage
    local eval_count
    eval_count=$(grep -c "^\s*eval " "$script_file" 2>/dev/null || echo 0)
    if (( eval_count > 0 )); then
        echo "WARN: Found $eval_count eval() calls - review for security" | tee -a "$report_file"
    fi

    # Check for hardcoded passwords
    if grep -qiE "password\s*=\s*['\"][^'\"]+['\"]" "$script_file" 2>/dev/null; then
        echo "ERROR: Potential hardcoded password found" | tee -a "$report_file"
        (( issues++ ))
    fi

    # Check for TODO/FIXME
    local todo_count
    todo_count=$(grep -cE "TODO|FIXME|HACK|XXX" "$script_file" 2>/dev/null || echo 0)
    (( todo_count > 0 )) && echo "INFO: $todo_count TODO/FIXME comments" | tee -a "$report_file"

    # Complexity check (function length)
    awk '
    /\(\)/ { in_func=1; func_name=$1; line_count=0 }
    in_func { line_count++ }
    /^}/ {
        if (in_func && line_count > 50)
            printf "WARN: Function %s is %d lines (consider refactoring)\n", func_name, line_count
        in_func=0
    }' "$script_file" | tee -a "$report_file"

    echo "" | tee -a "$report_file"
    echo "Total issues: $issues" | tee -a "$report_file"
    echo "Report: $report_file"

    return $issues
}

check_all_scripts() {
    local dir=${1:-.}
    local total_issues=0

    while IFS= read -r script; do
        echo "────────────────────────────────"
        analyze_script "$script" 2>/dev/null
        (( total_issues += $? ))
    done < <(find "$dir" -name "*.sh" -type f | sort)

    echo ""
    echo "Total scripts analyzed: $(find "$dir" -name "*.sh" -type f | wc -l)"
    echo "Total issues: $total_issues"
    return $total_issues
}
```

---

## 50.4 CI Integration for Shell Scripts

```bash
#!/bin/bash
# ci_quality_gate.sh - CI quality gate for shell scripts

quality_gate() {
    local dir=${1:-.}
    local exit_code=0

    echo "=== Shell Script Quality Gate ==="
    echo "Directory: $dir"
    echo ""

    echo "1. ShellCheck..."
    if command -v shellcheck &>/dev/null; then
        local shellcheck_issues=0
        while IFS= read -r script; do
            shellcheck --severity=warning "$script" 2>/dev/null || (( shellcheck_issues++ ))
        done < <(find "$dir" -name "*.sh" -type f)

        if (( shellcheck_issues > 0 )); then
            echo "   FAIL: $shellcheck_issues scripts with issues"
            exit_code=1
        else
            echo "   PASS"
        fi
    else
        echo "   SKIP: shellcheck not installed"
    fi

    echo ""
    echo "2. Executable permissions..."
    local non_executable=0
    while IFS= read -r script; do
        [[ -x "$script" ]] || (( non_executable++ ))
    done < <(find "$dir" -name "*.sh" -type f)
    if (( non_executable > 0 )); then
        echo "   WARN: $non_executable scripts missing executable bit"
    else
        echo "   PASS"
    fi

    echo ""
    echo "3. Syntax check..."
    local syntax_errors=0
    while IFS= read -r script; do
        bash -n "$script" 2>/dev/null || (( syntax_errors++ ))
    done < <(find "$dir" -name "*.sh" -type f)
    if (( syntax_errors > 0 )); then
        echo "   FAIL: $syntax_errors syntax errors"
        exit_code=1
    else
        echo "   PASS"
    fi

    echo ""
    echo "4. Tests..."
    if [[ -f "${dir}/tests/run_tests.sh" ]]; then
        bash "${dir}/tests/run_tests.sh" && echo "   PASS" || {
            echo "   FAIL"
            exit_code=1
        }
    else
        echo "   SKIP: no test suite found"
    fi

    echo ""
    echo "5. Secret scanning..."
    local secret_patterns='(password|passwd|secret|api_key|private_key|token)\s*=\s*['"'"'"'"'][^'"'"'"'"']{8,}'
    local secrets_found
    secrets_found=$(grep -rniE "$secret_patterns" "$dir" --include="*.sh" 2>/dev/null | \
        grep -v "^#" | wc -l)
    if (( secrets_found > 0 )); then
        echo "   FAIL: Potential secrets found ($secrets_found lines)"
        exit_code=1
    else
        echo "   PASS"
    fi

    echo ""
    echo "Quality gate: $([ $exit_code -eq 0 ] && echo 'PASSED' || echo 'FAILED')"
    return $exit_code
}
```

---

## 50.5 Test Coverage Tracking

```bash
#!/bin/bash
# coverage_tracker.sh - Track which functions are tested

declare -A COVERED_FUNCTIONS=()
declare -A ALL_FUNCTIONS=()

extract_functions() {
    local script_file=$1

    grep -n "^[a-zA-Z_][a-zA-Z0-9_]*\(\)" "$script_file" | \
        while IFS=: read -r line_num func_def; do
            local func_name="${func_def%%(*}"
            echo "${func_name}:${line_num}"
        done
}

track_coverage() {
    local test_file=$1
    local source_file=$2

    while IFS=: read -r func_name line_num; do
        ALL_FUNCTIONS["$func_name"]="$line_num"
    done < <(extract_functions "$source_file")

    # source the file being tested
    # shellcheck disable=SC1090
    source "$source_file" 2>/dev/null

    for func_name in "${!ALL_FUNCTIONS[@]}"; do
        if grep -q "\"${func_name}\"\|${func_name} \|${func_name}(" "$test_file" 2>/dev/null; then
            COVERED_FUNCTIONS["$func_name"]=1
        fi
    done

    local total=${#ALL_FUNCTIONS[@]}
    local covered=${#COVERED_FUNCTIONS[@]}
    local pct=$(( total > 0 ? covered * 100 / total : 0 ))

    echo "=== Coverage Report ==="
    printf "Covered:  %d / %d functions (%d%%)\n" "$covered" "$total" "$pct"
    echo ""

    echo "Uncovered functions:"
    for func in "${!ALL_FUNCTIONS[@]}"; do
        [[ -v "COVERED_FUNCTIONS[$func]" ]] || echo "  - $func (line ${ALL_FUNCTIONS[$func]})"
    done
}
```

---

## 50.6 Example Test Suite

```bash
#!/bin/bash
# example_tests.sh - Example test suite using framework

source ./bash_test_framework.sh

# ─── Unit Tests ────────────────────────────────────────────────
test_string_uppercase() {
    local result
    result=$(echo "hello" | tr '[:lower:]' '[:upper:]')
    assert_equal "$result" "HELLO"
}

test_math_operations() {
    assert_equal $(( 2 + 2 )) "4"
    assert_equal $(( 10 / 3 )) "3"
    assert_true "(( 5 > 3 ))"
    assert_false "(( 1 > 10 ))"
}

test_array_contains() {
    local arr=("apple" "banana" "cherry")
    assert_true "array_contains 'banana' \"${arr[@]}\""
    assert_false "array_contains 'grape' \"${arr[@]}\""
}

test_file_operations() {
    local tmpfile
    tmpfile=$(mktemp)
    echo "test content" > "$tmpfile"

    assert_file_exists "$tmpfile"
    assert_equal "$(cat "$tmpfile")" "test content"

    rm -f "$tmpfile"
}

test_exit_codes() {
    assert_exit_code "true" 0
    assert_exit_code "false" 1
    assert_exit_code "ls /nonexistent" 2
}

# Register tests
test "String uppercase" test_string_uppercase
test "Math operations" test_math_operations
test "Array contains" test_array_contains
test "File operations" test_file_operations
test "Exit codes" test_exit_codes

# Run
run_tests
```

---

## 50.7 Exercises

### Exercise 1: Full Test Suite for Script Library
เขียน test suite ครบถ้วนสำหรับ script library ที่สร้างไว้:
- 100% function coverage
- Edge case testing
- Performance benchmarks
- CI integration

### Exercise 2: Mutation Testing
สร้าง mutation tester ที่:
- แก้ code แบบสุ่ม (change operators)
- รัน test suite
- Report tests ที่ตรวจจับ mutations ไม่ได้

### Exercise 3: Contract Testing
สร้าง contract testing framework ที่:
- Define function contracts (inputs/outputs)
- Auto-generate test cases
- Verify contracts across versions

---

## สรุป Part 50

✅ Unit testing framework (assert_equal/matches/contains/etc.)
✅ Test registration and runner
✅ Integration test helpers
✅ HTTP API testing assertions
✅ Docker integration test setup
✅ Static analysis with ShellCheck integration
✅ Security pattern detection
✅ CI quality gate (shellcheck + syntax + secrets + tests)
✅ Test coverage tracking by function
✅ Full example test suite

---

**→ Part 51: Advanced Text Processing (awk, sed, perl)**
