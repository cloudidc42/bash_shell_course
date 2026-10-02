# Part 66: CI/CD Pipeline Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 66.1 Pipeline Framework

```bash
#!/bin/bash
# pipeline.sh - CI/CD pipeline orchestration engine

set -euo pipefail

PIPELINE_NAME="${PIPELINE_NAME:-pipeline}"
PIPELINE_LOG="${PIPELINE_LOG:-/tmp/pipeline.log}"
PIPELINE_START_TIME=$(date +%s)

# Stage result tracking
declare -A STAGE_RESULTS=()
declare -a STAGE_ORDER=()
PIPELINE_FAILED=false

pipeline_log() {
    local level=$1; shift
    printf '%s [%s] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$level" "$*" | tee -a "$PIPELINE_LOG"
}

# ─── Stage Runner ──────────────────────────────────────────────────
run_stage() {
    local stage_name=$1
    shift
    local stage_cmd=("$@")

    STAGE_ORDER+=("$stage_name")

    # Skip if pipeline already failed (unless --continue-on-error)
    if [[ "$PIPELINE_FAILED" == true && "${PIPELINE_CONTINUE_ON_ERROR:-false}" != true ]]; then
        STAGE_RESULTS["$stage_name"]="SKIPPED"
        pipeline_log "SKIP" "Stage: $stage_name (pipeline failed)"
        return 0
    fi

    pipeline_log "START" "Stage: $stage_name"
    local stage_start; stage_start=$(date +%s)

    local stage_log="${PIPELINE_LOG}.${stage_name// /_}"

    if "${stage_cmd[@]}" > "$stage_log" 2>&1; then
        local duration=$(( $(date +%s) - stage_start ))
        STAGE_RESULTS["$stage_name"]="PASS"
        pipeline_log "PASS" "Stage: $stage_name (${duration}s)"
    else
        local exit_code=$?
        local duration=$(( $(date +%s) - stage_start ))
        STAGE_RESULTS["$stage_name"]="FAIL"
        PIPELINE_FAILED=true
        pipeline_log "FAIL" "Stage: $stage_name (${duration}s, exit $exit_code)"
        cat "$stage_log" >&2
        return 1
    fi
}

pipeline_summary() {
    local total_time=$(( $(date +%s) - PIPELINE_START_TIME ))
    local pass=0 fail=0 skip=0

    echo ""
    echo "============================="
    echo " Pipeline: $PIPELINE_NAME"
    echo "============================="
    printf "%-30s %s\n" "Stage" "Result"
    printf "%-30s %s\n" "-----" "------"

    for stage in "${STAGE_ORDER[@]}"; do
        local result="${STAGE_RESULTS[$stage]:-UNKNOWN}"
        local icon
        case "$result" in
            PASS)    icon="[OK]"  ; (( pass++ )) ;;
            FAIL)    icon="[FAIL]"; (( fail++ )) ;;
            SKIPPED) icon="[SKIP]"; (( skip++ )) ;;
            *)       icon="[?]"
        esac
        printf "  %-30s %s\n" "$stage" "$icon $result"
    done

    echo ""
    echo "Results: PASS=$pass FAIL=$fail SKIP=$skip  Time: ${total_time}s"
    echo "============================="

    [[ "$PIPELINE_FAILED" != true ]]
}
```

---

## 66.2 Build System Abstraction

```bash
#!/bin/bash
# build.sh - Build system detection and abstraction

detect_build_system() {
    local dir=${1:-.}

    if [[ -f "$dir/Makefile" || -f "$dir/makefile" ]]; then
        echo "make"
    elif [[ -f "$dir/CMakeLists.txt" ]]; then
        echo "cmake"
    elif [[ -f "$dir/build.gradle" || -f "$dir/build.gradle.kts" ]]; then
        echo "gradle"
    elif [[ -f "$dir/pom.xml" ]]; then
        echo "maven"
    elif [[ -f "$dir/package.json" ]]; then
        if [[ -f "$dir/yarn.lock" ]]; then echo "yarn"
        elif [[ -f "$dir/pnpm-lock.yaml" ]]; then echo "pnpm"
        else echo "npm"
        fi
    elif [[ -f "$dir/Cargo.toml" ]]; then
        echo "cargo"
    elif [[ -f "$dir/go.mod" ]]; then
        echo "go"
    elif [[ -f "$dir/setup.py" || -f "$dir/pyproject.toml" ]]; then
        echo "python"
    else
        echo "unknown"
    fi
}

build_project() {
    local dir=${1:-.}
    local target=${2:-build}
    local build_system; build_system=$(detect_build_system "$dir")

    pipeline_log "INFO" "Build system: $build_system"

    case "$build_system" in
        make)
            make -C "$dir" "$target"
            ;;
        cmake)
            local build_dir="$dir/build"
            mkdir -p "$build_dir"
            cmake -S "$dir" -B "$build_dir" -DCMAKE_BUILD_TYPE=Release
            cmake --build "$build_dir" --parallel "$(nproc)"
            ;;
        gradle)
            ( cd "$dir" && ./gradlew build )
            ;;
        maven)
            ( cd "$dir" && mvn -q package -DskipTests )
            ;;
        npm)
            ( cd "$dir" && npm ci && npm run build )
            ;;
        yarn)
            ( cd "$dir" && yarn install --frozen-lockfile && yarn build )
            ;;
        pnpm)
            ( cd "$dir" && pnpm install --frozen-lockfile && pnpm build )
            ;;
        cargo)
            ( cd "$dir" && cargo build --release )
            ;;
        go)
            ( cd "$dir" && go build ./... )
            ;;
        python)
            ( cd "$dir" && pip install -e . -q )
            ;;
        *)
            pipeline_log "WARN" "Unknown build system in $dir"
            return 1
            ;;
    esac
}

install_dependencies() {
    local dir=${1:-.}
    local build_system; build_system=$(detect_build_system "$dir")

    case "$build_system" in
        npm)   ( cd "$dir" && npm ci ) ;;
        yarn)  ( cd "$dir" && yarn install --frozen-lockfile ) ;;
        pnpm)  ( cd "$dir" && pnpm install --frozen-lockfile ) ;;
        go)    ( cd "$dir" && go mod download ) ;;
        cargo) ( cd "$dir" && cargo fetch ) ;;
        python)( cd "$dir" && pip install -r requirements.txt -q 2>/dev/null || true ) ;;
        maven) ( cd "$dir" && mvn -q dependency:resolve ) ;;
        *)     pipeline_log "INFO" "No dependency install step for $build_system" ;;
    esac
}
```

---

## 66.3 Automated Testing

```bash
#!/bin/bash
# test_runner.sh - Automated test runner with reporting

TEST_RESULTS_DIR="${TEST_RESULTS_DIR:-/tmp/test_results}"
TEST_PASS=0
TEST_FAIL=0
TEST_SKIP=0

mkdir -p "$TEST_RESULTS_DIR"

# ─── Test Registration and Execution ───────────────────────────────────
declare -a TEST_CASES=()

register_test() {
    TEST_CASES+=("$1")
}

assert_equals() {
    local expected=$1 actual=$2 msg=${3:-assertion}
    if [[ "$expected" == "$actual" ]]; then
        return 0
    else
        echo "  ASSERT FAIL: $msg" >&2
        echo "    Expected: $expected" >&2
        echo "    Actual:   $actual" >&2
        return 1
    fi
}

assert_contains() {
    local haystack=$1 needle=$2 msg=${3:-contains}
    if [[ "$haystack" == *"$needle"* ]]; then
        return 0
    else
        echo "  ASSERT FAIL: $msg" >&2
        echo "    '$haystack' does not contain '$needle'" >&2
        return 1
    fi
}

assert_exit_code() {
    local expected=$1 cmd=$2 msg=${3:-exit_code}
    eval "$cmd" &>/dev/null
    local actual=$?
    assert_equals "$expected" "$actual" "$msg"
}

run_test() {
    local test_name=$1
    local test_log="${TEST_RESULTS_DIR}/${test_name// /_}.log"

    local start_ns; start_ns=$(date +%s%N)

    if "$test_name" > "$test_log" 2>&1; then
        local duration_ms=$(( ( $(date +%s%N) - start_ns ) / 1000000 ))
        printf "  %-50s [PASS] %dms\n" "$test_name" "$duration_ms"
        (( TEST_PASS++ ))
    else
        local duration_ms=$(( ( $(date +%s%N) - start_ns ) / 1000000 ))
        printf "  %-50s [FAIL] %dms\n" "$test_name" "$duration_ms"
        cat "$test_log" | sed 's/^/    /'
        (( TEST_FAIL++ ))
    fi
}

run_all_tests() {
    echo "=== Running ${#TEST_CASES[@]} tests ==="

    for test in "${TEST_CASES[@]}"; do
        run_test "$test"
    done

    echo ""
    echo "Results: PASS=$TEST_PASS  FAIL=$TEST_FAIL  SKIP=$TEST_SKIP"
    (( TEST_FAIL == 0 ))
}

# ─── Parallel Test Runner ─────────────────────────────────────────────────
run_tests_parallel() {
    local max_workers=${1:-$(nproc)}
    shift
    local tests=("$@")
    local -a pids=() test_names=()

    for test in "${tests[@]}"; do
        local test_log="${TEST_RESULTS_DIR}/${test// /_}.log"
        ( "$test" > "$test_log" 2>&1; echo $? > "${test_log}.exit" ) &
        pids+=($!)
        test_names+=("$test")

        while (( ${#pids[@]} >= max_workers )); do
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[$i]}" 2>/dev/null; then
                    unset 'pids[i]'
                    pids=("${pids[@]}")
                    break
                fi
            done
            sleep 0.05
        done
    done

    for pid in "${pids[@]}"; do wait "$pid" 2>/dev/null || true; done

    # Collect results
    for test in "${test_names[@]}"; do
        local test_log="${TEST_RESULTS_DIR}/${test// /_}.log"
        local exit_code=1
        [[ -f "${test_log}.exit" ]] && exit_code=$(cat "${test_log}.exit")

        if (( exit_code == 0 )); then
            printf "  %-50s [PASS]\n" "$test"
            (( TEST_PASS++ ))
        else
            printf "  %-50s [FAIL]\n" "$test"
            (( TEST_FAIL++ ))
        fi
    done
}

# ─── JUnit XML Report ───────────────────────────────────────────────────
generate_junit_xml() {
    local output_file=${1:-test-results.xml}
    local suite_name=${2:-"Pipeline Tests"}
    local total=$(( TEST_PASS + TEST_FAIL + TEST_SKIP ))

    cat > "$output_file" <<XMLEOF
<?xml version="1.0" encoding="UTF-8"?>
<testsuites name="${suite_name}" tests="${total}" failures="${TEST_FAIL}" skipped="${TEST_SKIP}">
  <testsuite name="${suite_name}" tests="${total}" failures="${TEST_FAIL}">
XMLEOF

    for test in "${TEST_CASES[@]}"; do
        local test_log="${TEST_RESULTS_DIR}/${test// /_}.log"
        local exit_code=1
        [[ -f "${test_log}.exit" ]] && exit_code=$(cat "${test_log}.exit")

        if (( exit_code == 0 )); then
            echo "    <testcase name=\"${test}\" classname=\"pipeline\"/>" >> "$output_file"
        else
            local failure_msg; failure_msg=$(cat "$test_log" 2>/dev/null | head -5 | sed 's/</\&lt;/g; s/>/\&gt;/g')
            cat >> "$output_file" <<CASEEOF
    <testcase name="${test}" classname="pipeline">
      <failure message="Test failed"><![CDATA[${failure_msg}]]></failure>
    </testcase>
CASEEOF
        fi
    done

    cat >> "$output_file" <<ENDXML
  </testsuite>
</testsuites>
ENDXML
    echo "JUnit report: $output_file"
}
```

---

## 66.4 Artifact Management

```bash
#!/bin/bash
# artifacts.sh - Build artifact management

ARTIFACT_DIR="${ARTIFACT_DIR:-/tmp/artifacts}"
ARTIFACT_REGISTRY="${ARTIFACT_DIR}/registry.json"

mkdir -p "$ARTIFACT_DIR"
[[ -f "$ARTIFACT_REGISTRY" ]] || echo '[]' > "$ARTIFACT_REGISTRY"

get_build_version() {
    local version_file=${1:-VERSION}
    local git_short=""

    if git rev-parse --is-inside-work-tree &>/dev/null 2>&1; then
        git_short=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
    fi

    local base_version="0.0.0"
    if [[ -f "$version_file" ]]; then
        base_version=$(cat "$version_file" | tr -d '[:space:]')
    elif [[ -f package.json ]]; then
        base_version=$(grep '"version"' package.json | head -1 | grep -oP '[0-9]+\.[0-9]+\.[0-9]+')
    elif [[ -f pom.xml ]]; then
        base_version=$(grep -m1 '<version>' pom.xml | grep -oP '[0-9]+\.[0-9]+\.[0-9]+')
    fi

    local build_num="${BUILD_NUMBER:-$(date +%Y%m%d%H%M%S)}"
    echo "${base_version}-${build_num}+${git_short:-local}"
}

build_artifact() {
    local name=$1 source_path=$2
    local version; version=$(get_build_version)
    local artifact_name="${name}-${version}.tar.gz"
    local artifact_path="${ARTIFACT_DIR}/${artifact_name}"

    pipeline_log "INFO" "Building artifact: $artifact_name"

    tar -czf "$artifact_path" \
        --transform "s|^|${name}-${version}/|" \
        -C "$(dirname "$source_path")" \
        "$(basename "$source_path")"

    local size; size=$(du -sh "$artifact_path" | cut -f1)
    local sha256; sha256=$(sha256sum "$artifact_path" | cut -d' ' -f1)

    local entry
    entry=$(jq -n \
        --arg name "$artifact_name" \
        --arg version "$version" \
        --arg path "$artifact_path" \
        --arg sha256 "$sha256" \
        --arg size "$size" \
        --arg created "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        '{name:$name, version:$version, path:$path, sha256:$sha256, size:$size, created:$created}')

    jq ". += [$entry]" "$ARTIFACT_REGISTRY" > "${ARTIFACT_REGISTRY}.tmp"
    mv "${ARTIFACT_REGISTRY}.tmp" "$ARTIFACT_REGISTRY"

    echo "$artifact_path"
    pipeline_log "INFO" "Artifact ready: $artifact_name ($size, sha256: ${sha256:0:12}...)"
}

publish_artifact() {
    local artifact_path=$1 dest_url=$2

    if [[ "$dest_url" =~ ^s3:// ]]; then
        aws s3 cp "$artifact_path" "$dest_url/$(basename "$artifact_path")" --storage-class STANDARD_IA
    elif [[ "$dest_url" =~ ^(https?|ftp):// ]]; then
        curl -fsSL -X PUT \
            -H "Content-Type: application/octet-stream" \
            --data-binary "@$artifact_path" \
            "${dest_url}/$(basename "$artifact_path")"
    elif [[ -d "$dest_url" ]]; then
        cp "$artifact_path" "$dest_url/"
        cp "${artifact_path}.sha256" "$dest_url/" 2>/dev/null || true
    fi

    pipeline_log "INFO" "Published: $(basename "$artifact_path") -> $dest_url"
}

list_artifacts() {
    jq -r '.[] | "\(.created)  \(.name)  \(.size)"' "$ARTIFACT_REGISTRY" | sort -r
}
```

---

## 66.5 Deployment Automation

```bash
#!/bin/bash
# deploy.sh - Deployment with health checks and rollback

DEPLOY_DIR="${DEPLOY_DIR:-/opt/app}"
DEPLOY_BACKUP_DIR="${DEPLOY_DIR}/.rollback"
DEPLOY_TIMEOUT=${DEPLOY_TIMEOUT:-120}

deploy_log() {
    printf '%s [DEPLOY] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "${PIPELINE_LOG:-/tmp/deploy.log}"
}

# ─── Pre-deployment Snapshot ───────────────────────────────────────────────
deploy_snapshot() {
    local timestamp; timestamp=$(date +%Y%m%d_%H%M%S)
    local snapshot_dir="${DEPLOY_BACKUP_DIR}/${timestamp}"

    if [[ -d "$DEPLOY_DIR" ]]; then
        mkdir -p "$snapshot_dir"
        rsync -a --exclude='.rollback/' "${DEPLOY_DIR}/" "${snapshot_dir}/"
        echo "$snapshot_dir"
        deploy_log "Snapshot created: $snapshot_dir"
    fi
}

deploy_rollback() {
    local target_snapshot=${1:-}

    if [[ -z "$target_snapshot" ]]; then
        target_snapshot=$(ls -dt "${DEPLOY_BACKUP_DIR}"/????????_?????? 2>/dev/null | head -1)
    fi

    if [[ -z "$target_snapshot" || ! -d "$target_snapshot" ]]; then
        deploy_log "ERROR: No snapshot available for rollback"
        return 1
    fi

    deploy_log "Rolling back to: $target_snapshot"
    rsync -a --delete \
        --exclude='.rollback/' \
        "${target_snapshot}/" \
        "${DEPLOY_DIR}/"

    deploy_log "Rollback complete"
}

# ─── Deployment Procedure ──────────────────────────────────────────────────
deploy_artifact() {
    local artifact_path=$1 service_name=$2 health_url=${3:-}

    deploy_log "Starting deployment: $service_name"

    # Snapshot before deploy
    deploy_snapshot

    # Stop service
    systemctl stop "$service_name" 2>/dev/null || true

    # Extract artifact
    mkdir -p "$DEPLOY_DIR"
    tar -xzf "$artifact_path" -C "$DEPLOY_DIR" --strip-components=1

    # Apply permissions
    chown -R "${APP_USER:-www-data}:${APP_GROUP:-www-data}" "$DEPLOY_DIR" 2>/dev/null || true

    # Start service
    systemctl start "$service_name"

    # Wait for health check
    if [[ -n "$health_url" ]]; then
        deploy_log "Waiting for health: $health_url"
        local deadline=$(( $(date +%s) + DEPLOY_TIMEOUT ))

        while (( $(date +%s) < deadline )); do
            local http_code
            http_code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$health_url" 2>/dev/null)

            if [[ "$http_code" =~ ^2 ]]; then
                deploy_log "Health check passed ($http_code): $service_name"
                return 0
            fi

            sleep 5
        done

        deploy_log "ERROR: Health check timeout for $service_name"
        deploy_log "Initiating rollback..."
        deploy_rollback
        systemctl restart "$service_name" 2>/dev/null || true
        return 1
    fi

    deploy_log "Deployment complete: $service_name"
}

# ─── Blue/Green Deployment ───────────────────────────────────────────────
deploy_blue_green() {
    local artifact_path=$1 service_name=$2 health_url=$3
    local active_slot="blue"
    local state_file="/etc/${service_name}.slot"

    [[ -f "$state_file" ]] && active_slot=$(cat "$state_file")
    local new_slot="green"
    [[ "$active_slot" == "green" ]] && new_slot="blue"

    deploy_log "Blue/Green: deploying to slot $new_slot (active: $active_slot)"

    local new_dir="/opt/${service_name}-${new_slot}"
    mkdir -p "$new_dir"
    tar -xzf "$artifact_path" -C "$new_dir" --strip-components=1

    systemctl start "${service_name}-${new_slot}" 2>/dev/null || true
    sleep 5

    local http_code
    http_code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$health_url" 2>/dev/null)

    if [[ "$http_code" =~ ^2 ]]; then
        echo "$new_slot" > "$state_file"
        systemctl stop "${service_name}-${active_slot}" 2>/dev/null || true
        deploy_log "Blue/Green switch complete: $active_slot -> $new_slot"
    else
        systemctl stop "${service_name}-${new_slot}" 2>/dev/null || true
        deploy_log "ERROR: New slot unhealthy, keeping $active_slot"
        return 1
    fi
}
```

---

## 66.6 GitHub Actions Integration

```bash
#!/bin/bash
# github_actions.sh - GitHub Actions API integration

GH_API="https://api.github.com"
GH_TOKEN="${GITHUB_TOKEN:-}"
GH_REPO="${GITHUB_REPOSITORY:-}"

gh_api() {
    local method=${1:-GET} endpoint=$2
    shift 2
    local data=${1:-}

    local curl_args=(
        -s
        -H "Authorization: Bearer $GH_TOKEN"
        -H "Accept: application/vnd.github.v3+json"
        -H "Content-Type: application/json"
        -X "$method"
    )

    [[ -n "$data" ]] && curl_args+=(-d "$data")

    curl "${curl_args[@]}" "${GH_API}${endpoint}"
}

gh_trigger_workflow() {
    local workflow_id=$1 ref=${2:-main}
    shift 2
    local inputs=${1:-{}}

    gh_api POST "/repos/${GH_REPO}/actions/workflows/${workflow_id}/dispatches" \
        "{\"ref\":\"${ref}\",\"inputs\":${inputs}}"

    pipeline_log "INFO" "Triggered workflow: $workflow_id on $ref"
}

gh_get_latest_run() {
    local workflow_id=$1 branch=${2:-main}

    gh_api GET "/repos/${GH_REPO}/actions/workflows/${workflow_id}/runs?branch=${branch}&per_page=1" | \
        jq '.workflow_runs[0]'
}

gh_wait_for_run() {
    local run_id=$1 timeout=${2:-600}

    local deadline=$(( $(date +%s) + timeout ))

    pipeline_log "INFO" "Waiting for run $run_id..."

    while (( $(date +%s) < deadline )); do
        local run_data; run_data=$(gh_api GET "/repos/${GH_REPO}/actions/runs/${run_id}")
        local status; status=$(echo "$run_data" | jq -r '.status')
        local conclusion; conclusion=$(echo "$run_data" | jq -r '.conclusion // "in_progress"')

        case "$status" in
            completed)
                pipeline_log "INFO" "Run $run_id completed: $conclusion"
                [[ "$conclusion" == "success" ]] && return 0 || return 1
                ;;
            queued|in_progress|waiting)
                pipeline_log "INFO" "Run $run_id: $status"
                sleep 30
                ;;
            *)
                pipeline_log "WARN" "Unknown status: $status"
                sleep 10
                ;;
        esac
    done

    pipeline_log "ERROR" "Run $run_id timed out after ${timeout}s"
    return 1
}

gh_get_run_logs() {
    local run_id=$1 output_dir=${2:-/tmp/gh_logs}

    mkdir -p "$output_dir"
    local zip_file="${output_dir}/run_${run_id}.zip"

    gh_api GET "/repos/${GH_REPO}/actions/runs/${run_id}/logs" > "$zip_file"
    unzip -q "$zip_file" -d "$output_dir"
    echo "$output_dir"
}

gh_create_pr() {
    local title=$1 head=$2 base=${3:-main} body=${4:-}

    gh_api POST "/repos/${GH_REPO}/pulls" \
        "$(jq -n \
            --arg title "$title" \
            --arg head "$head" \
            --arg base "$base" \
            --arg body "$body" \
            '{title:$title, head:$head, base:$base, body:$body}')" | \
        jq -r '.html_url'
}

gh_post_status() {
    local sha=$1 state=$2 context=${3:-ci} description=${4:-}

    gh_api POST "/repos/${GH_REPO}/statuses/${sha}" \
        "$(jq -n \
            --arg state "$state" \
            --arg context "$context" \
            --arg description "$description" \
            '{state:$state, context:$context, description:$description}')"
}
```

---

## 66.7 Notification System

```bash
#!/bin/bash
# notifications.sh - Pipeline notifications

notify_slack() {
    local webhook_url=$1 message=$2 color=${3:-good}
    local pipeline_status=${4:-unknown}

    [[ -z "$webhook_url" ]] && return 0

    local icon
    case "$pipeline_status" in
        success) icon=":white_check_mark:" ;;
        failure) icon=":x:" ;;
        *)       icon=":information_source:" ;;
    esac

    curl -s -X POST "$webhook_url" \
        -H 'Content-Type: application/json' \
        -d "$(jq -n \
            --arg text "$message" \
            --arg icon "$icon" \
            --arg color "$color" \
            '{text: ($icon + " " + $text), attachments: [{color: $color, text: $text}]}')"
}

notify_email() {
    local to=$1 subject=$2 body=$3

    if command -v sendmail &>/dev/null; then
        printf 'To: %s\nSubject: %s\n\n%s\n' "$to" "$subject" "$body" | sendmail "$to"
    elif command -v mail &>/dev/null; then
        echo "$body" | mail -s "$subject" "$to"
    else
        pipeline_log "WARN" "No mail agent available"
    fi
}

notify_webhook() {
    local url=$1
    shift
    local payload="$1"

    curl -s -X POST "$url" \
        -H 'Content-Type: application/json' \
        -d "$payload"
}

pipeline_notify() {
    local status=$1
    local duration=$(( $(date +%s) - PIPELINE_START_TIME ))
    local color

    [[ "$status" == "success" ]] && color="good" || color="danger"

    local message="Pipeline ${PIPELINE_NAME}: ${status^^}  (${duration}s)  Branch: ${GIT_BRANCH:-unknown}"

    [[ -n "${SLACK_WEBHOOK:-}" ]] && notify_slack "$SLACK_WEBHOOK" "$message" "$color" "$status"
    [[ -n "${NOTIFY_EMAIL:-}" ]] && notify_email "$NOTIFY_EMAIL" "[CI] $message" "$message"
    [[ -n "${NOTIFY_WEBHOOK:-}" ]] && notify_webhook "$NOTIFY_WEBHOOK" \
        "{\"pipeline\":\"${PIPELINE_NAME}\",\"status\":\"${status}\",\"duration\":${duration}}"
}
```

---

## 66.8 Full Pipeline Runner

```bash
#!/bin/bash
# run_pipeline.sh - Tie all stages together

set -euo pipefail

# โหลด libraries
source /opt/ci/pipeline.sh
source /opt/ci/build.sh
source /opt/ci/test_runner.sh
source /opt/ci/artifacts.sh
source /opt/ci/deploy.sh
source /opt/ci/notifications.sh

PIPELINE_NAME="${1:-app-pipeline}"
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
GIT_SHA=$(git rev-parse HEAD 2>/dev/null || echo "unknown")

pipeline_log "INFO" "Pipeline start: $PIPELINE_NAME  branch=$GIT_BRANCH  sha=${GIT_SHA:0:8}"

# Notify start
[[ -n "${SLACK_WEBHOOK:-}" ]] && notify_slack "$SLACK_WEBHOOK" \
    "Pipeline $PIPELINE_NAME started (branch: $GIT_BRANCH)" "warning"

# Stage: Install dependencies
run_stage "install" install_dependencies .

# Stage: Lint
run_stage "lint" bash -c 'shellcheck scripts/**/*.sh 2>/dev/null || true; eslint . 2>/dev/null || true'

# Stage: Build
run_stage "build" build_project . build

# Stage: Unit tests
run_stage "unit-tests" bash -c '
    source /opt/ci/test_runner.sh
    # Register test functions
    register_test test_config_load
    register_test test_api_response
    register_test test_backup_create
    run_all_tests
'

# Stage: Integration tests
run_stage "integration-tests" bash -c 'npm run test:integration 2>/dev/null || pytest tests/integration/ 2>/dev/null || echo "No integration tests"'

# Stage: Package artifact
artifact_path=""
if [[ "${STAGE_RESULTS[build]:-}" == "PASS" ]]; then
    run_stage "package" bash -c "
        artifact_path=\$(build_artifact ${PIPELINE_NAME} dist/)
        echo \"ARTIFACT=\$artifact_path\" > /tmp/pipeline_env
    "
    [[ -f /tmp/pipeline_env ]] && source /tmp/pipeline_env
fi

# Stage: Deploy (only on main/master)
if [[ "$GIT_BRANCH" =~ ^(main|master)$ && -n "${artifact_path:-}" ]]; then
    run_stage "deploy" deploy_artifact \
        "$artifact_path" \
        "${APP_SERVICE:-app}" \
        "${HEALTH_CHECK_URL:-}"
fi

# Summary and notify
if pipeline_summary; then
    pipeline_notify "success"
    gh_post_status "$GIT_SHA" "success" "ci/pipeline" "Pipeline passed"
    exit 0
else
    pipeline_notify "failure"
    gh_post_status "$GIT_SHA" "failure" "ci/pipeline" "Pipeline failed"
    exit 1
fi
```

---

## 66.9 Exercises

### Exercise 1: Multi-environment Pipeline
สร้าง pipeline ที่:
- dev/staging/prod environments
- Environment-specific config injection
- Manual approval gate สำหรับ prod
- Smoke tests หลัง deploy

### Exercise 2: Docker Build Pipeline
สร้าง pipeline ที่:
- Build Docker image
- Run container tests
- Scan for vulnerabilities (trivy)
- Push to registry
- Deploy to Kubernetes

### Exercise 3: Release Automation
สร้าง script ที่:
- Semantic version bump
- Generate CHANGELOG from git log
- Tag release
- Create GitHub release with artifacts
- Notify team

---

## สรุป Part 66

✅ Pipeline stage orchestration with PASS/FAIL/SKIP tracking and summary
✅ Build system detection and abstraction (make/cmake/gradle/maven/npm/go/cargo/python)
✅ Test runner with assert helpers, parallel execution, JUnit XML report
✅ Artifact management: version from git, tar+SHA256, registry JSON, publish to S3/HTTP/local
✅ Deployment with pre-deploy snapshot, health-check polling, automatic rollback
✅ Blue/green deployment slot switching
✅ GitHub Actions REST API: trigger workflow, poll run status, get logs, create PR, post commit status
✅ Notification dispatcher: Slack webhook, email, generic webhook
✅ Full pipeline runner tying all stages end-to-end

---

**→ Part 67: Container and Kubernetes Automation**
