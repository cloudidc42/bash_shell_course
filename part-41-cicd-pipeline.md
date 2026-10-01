# Part 41: CI/CD Pipeline Automation
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 41.1 Pipeline Foundation

```bash
#!/bin/bash
# pipeline_core.sh - CI/CD pipeline framework

set -euo pipefail

# ─── Pipeline Context ──────────────────────────────────────────
declare -A PIPELINE=(
    [name]=""
    [build_dir]=""
    [artifact_dir]=""
    [log_dir]=""
    [start_time]=""
    [status]="pending"
)

declare -a STAGE_NAMES=()
declare -A STAGE_STATUS=()
declare -A STAGE_DURATION=()

pipeline_init() {
    local name=$1
    local build_dir=${2:-/tmp/pipeline_build}

    PIPELINE[name]="$name"
    PIPELINE[build_dir]="$build_dir"
    PIPELINE[artifact_dir]="${build_dir}/artifacts"
    PIPELINE[log_dir]="${build_dir}/logs"
    PIPELINE[start_time]=$SECONDS

    mkdir -p "${PIPELINE[artifact_dir]}" "${PIPELINE[log_dir]}"

    export PIPELINE_NAME="$name"
    export PIPELINE_BUILD_DIR="$build_dir"
    export PIPELINE_ARTIFACT_DIR="${PIPELINE[artifact_dir]}"

    echo "=== Pipeline: $name ==="
    echo "Build dir: $build_dir"
    echo "Started: $(date)"
    echo ""
}

run_stage() {
    local stage_name=$1
    local stage_func=$2
    shift 2
    local stage_args=("$@")

    STAGE_NAMES+=("$stage_name")
    local stage_start=$SECONDS
    local log_file="${PIPELINE[log_dir]}/${stage_name// /_}.log"

    echo "▶ Stage: $stage_name"

    if "$stage_func" "${stage_args[@]}" 2>&1 | tee "$log_file"; then
        local duration=$(( SECONDS - stage_start ))
        STAGE_STATUS["$stage_name"]="passed"
        STAGE_DURATION["$stage_name"]="$duration"
        echo "  ✓ ${stage_name} (${duration}s)"
    else
        local duration=$(( SECONDS - stage_start ))
        STAGE_STATUS["$stage_name"]="failed"
        STAGE_DURATION["$stage_name"]="$duration"
        echo "  ✗ ${stage_name} FAILED (${duration}s)"
        PIPELINE[status]="failed"
        pipeline_summary
        return 1
    fi
}

pipeline_summary() {
    local total_duration=$(( SECONDS - PIPELINE[start_time] ))

    echo ""
    echo "=== Pipeline Summary: ${PIPELINE[name]} ==="
    printf "%-30s %-10s %s\n" "Stage" "Status" "Duration"
    printf "%-30s %-10s %s\n" "─────────────────────────────" "──────────" "────────"

    for stage in "${STAGE_NAMES[@]}"; do
        local status="${STAGE_STATUS[$stage]:-unknown}"
        local duration="${STAGE_DURATION[$stage]:-?}"
        local icon="✓"
        [[ "$status" == "failed" ]] && icon="✗"
        [[ "$status" == "skipped" ]] && icon="⊘"
        printf "%-30s %-10s %ss\n" "$icon $stage" "$status" "$duration"
    done

    echo ""
    echo "Total: ${total_duration}s | Status: ${PIPELINE[status]:-passed}"
}
```

---

## 41.2 Build Stage Automation

```bash
#!/bin/bash
# build_stages.sh

# ─── Code Checkout ─────────────────────────────────────────────
stage_checkout() {
    local repo_url=$1
    local branch=${2:-main}
    local target_dir="${PIPELINE_BUILD_DIR}/src"

    if [[ -d "$target_dir/.git" ]]; then
        git -C "$target_dir" fetch origin "$branch"
        git -C "$target_dir" reset --hard "origin/$branch"
    else
        git clone --depth=50 --branch "$branch" "$repo_url" "$target_dir"
    fi

    local sha
    sha=$(git -C "$target_dir" rev-parse --short HEAD)
    echo "Checked out: $branch @ $sha"

    export GIT_SHA="$sha"
    export GIT_BRANCH="$branch"
    export BUILD_NUMBER="${BUILD_NUMBER:-$(date +%Y%m%d%H%M%S)}"
    export BUILD_TAG="${GIT_BRANCH}-${GIT_SHA}-${BUILD_NUMBER}"
}

# ─── Dependency Install ────────────────────────────────────────
stage_install_deps() {
    local src_dir="${PIPELINE_BUILD_DIR}/src"
    cd "$src_dir"

    if [[ -f "package.json" ]]; then
        echo "Installing Node.js dependencies..."
        if [[ -f "package-lock.json" ]]; then
            npm ci --prefer-offline
        else
            npm install
        fi
    fi

    if [[ -f "requirements.txt" ]]; then
        echo "Installing Python dependencies..."
        python3 -m venv "${PIPELINE_BUILD_DIR}/venv"
        # shellcheck disable=SC1091
        source "${PIPELINE_BUILD_DIR}/venv/bin/activate"
        pip install -r requirements.txt -q
    fi

    if [[ -f "go.mod" ]]; then
        echo "Downloading Go modules..."
        go mod download
    fi

    if [[ -f "Cargo.toml" ]]; then
        echo "Fetching Rust dependencies..."
        cargo fetch
    fi

    if [[ -f "pom.xml" ]]; then
        echo "Resolving Maven dependencies..."
        mvn dependency:resolve -q
    fi

    echo "Dependencies installed"
}

# ─── Lint & Format Check ───────────────────────────────────────
stage_lint() {
    local src_dir="${PIPELINE_BUILD_DIR}/src"
    cd "$src_dir"

    local failed=0

    if [[ -f ".eslintrc.js" ]] || [[ -f ".eslintrc.json" ]]; then
        echo "Running ESLint..."
        npx eslint . --max-warnings=0 || (( failed++ ))
    fi

    if [[ -f "pyproject.toml" ]] || [[ -f ".flake8" ]]; then
        echo "Running flake8..."
        python3 -m flake8 . --count --statistics || (( failed++ ))
    fi

    if [[ -f ".golangci.yml" ]]; then
        echo "Running golangci-lint..."
        golangci-lint run ./... || (( failed++ ))
    fi

    find . -name "*.sh" ! -path "./.git/*" | while read -r f; do
        shellcheck "$f" || (( failed++ ))
    done

    if (( failed > 0 )); then
        echo "Lint failed: $failed checks"
        return 1
    fi

    echo "All lint checks passed"
}

# ─── Build ─────────────────────────────────────────────────────
stage_build() {
    local src_dir="${PIPELINE_BUILD_DIR}/src"
    cd "$src_dir"

    if [[ -f "package.json" ]]; then
        local build_cmd
        build_cmd=$(jq -r '.scripts.build // ""' package.json)
        if [[ -n "$build_cmd" ]]; then
            echo "Building: npm run build"
            npm run build
        fi
    fi

    if [[ -f "go.mod" ]]; then
        echo "Building Go binary..."
        local binary_name
        binary_name=$(basename "$(pwd)")
        go build -ldflags "-X main.Version=${BUILD_TAG:-dev}" \
            -o "${PIPELINE_ARTIFACT_DIR}/${binary_name}" ./...
    fi

    if [[ -f "Makefile" ]]; then
        echo "Running make..."
        make build
    fi

    echo "Build artifacts:"
    ls -lh "${PIPELINE_ARTIFACT_DIR}/" 2>/dev/null || echo "  (none)"
}
```

---

## 41.3 Test Automation

```bash
#!/bin/bash
# test_runner.sh

stage_unit_tests() {
    local src_dir="${PIPELINE_BUILD_DIR}/src"
    local report_dir="${PIPELINE_ARTIFACT_DIR}/test-reports"
    mkdir -p "$report_dir"

    cd "$src_dir"

    if [[ -f "package.json" ]]; then
        local test_cmd
        test_cmd=$(jq -r '.scripts.test // ""' package.json)
        if [[ -n "$test_cmd" ]]; then
            npm test -- \
                --coverage \
                --ci \
                --reporters=default \
                --reporters=jest-junit \
                2>&1 | tee "${report_dir}/jest.log"
        fi
    fi

    if [[ -f "go.mod" ]]; then
        go test ./... \
            -v \
            -race \
            -coverprofile="${report_dir}/coverage.out" \
            -covermode=atomic \
            2>&1 | tee "${report_dir}/go_test.log"

        go tool cover -html="${report_dir}/coverage.out" \
            -o "${report_dir}/coverage.html"

        local coverage
        coverage=$(go tool cover -func="${report_dir}/coverage.out" | \
            grep total | awk '{print $3}')
        echo "Coverage: $coverage"
    fi

    if command -v pytest &>/dev/null; then
        pytest \
            --junitxml="${report_dir}/pytest.xml" \
            --cov=. \
            --cov-report=html:"${report_dir}/htmlcov" \
            --cov-report=term-missing \
            -v \
            2>&1 | tee "${report_dir}/pytest.log"
    fi
}

stage_integration_tests() {
    local src_dir="${PIPELINE_BUILD_DIR}/src"
    local compose_file="${src_dir}/docker-compose.test.yml"

    if [[ ! -f "$compose_file" ]]; then
        echo "No integration test compose file found, skipping"
        return 0
    fi

    local stack_name="ci_test_$(date +%s)"

    trap "docker-compose -f '$compose_file' -p '$stack_name' down -v 2>/dev/null" EXIT

    docker-compose -f "$compose_file" -p "$stack_name" up -d

    echo "Waiting for services to be ready..."
    local timeout=60
    local elapsed=0
    while (( elapsed < timeout )); do
        if docker-compose -f "$compose_file" -p "$stack_name" \
            ps | grep -q "Up (healthy)"; then
            break
        fi
        sleep 5
        (( elapsed += 5 ))
    done

    echo "Running integration tests..."
    docker-compose -f "$compose_file" -p "$stack_name" \
        run --rm test

    docker-compose -f "$compose_file" -p "$stack_name" down -v
}

check_coverage_threshold() {
    local coverage_file="${PIPELINE_ARTIFACT_DIR}/test-reports/coverage.out"
    local threshold=${1:-80}

    if [[ ! -f "$coverage_file" ]]; then
        echo "No coverage file found"
        return 0
    fi

    local coverage
    coverage=$(go tool cover -func="$coverage_file" | \
        grep total | awk '{gsub(/%/,"",$3); print $3}')

    if (( $(echo "$coverage < $threshold" | bc -l) )); then
        echo "Coverage ${coverage}% is below threshold ${threshold}%"
        return 1
    fi

    echo "Coverage: ${coverage}% ≥ ${threshold}% threshold ✓"
}
```

---

## 41.4 Docker Build & Push

```bash
#!/bin/bash
# docker_build_push.sh

stage_docker_build() {
    local image_name=$1
    local registry=${2:-}
    local src_dir="${PIPELINE_BUILD_DIR}/src"

    local full_image="${image_name}"
    [[ -n "$registry" ]] && full_image="${registry}/${image_name}"

    local tags=(
        "${full_image}:${BUILD_TAG:-latest}"
        "${full_image}:${GIT_SHA:-latest}"
    )

    if [[ "${GIT_BRANCH:-}" == "main" ]] || [[ "${GIT_BRANCH:-}" == "master" ]]; then
        tags+=("${full_image}:latest")
    fi

    local tag_args=()
    for tag in "${tags[@]}"; do
        tag_args+=(-t "$tag")
    done

    echo "Building Docker image..."
    docker build \
        "${tag_args[@]}" \
        --build-arg "BUILD_TAG=${BUILD_TAG:-dev}" \
        --build-arg "GIT_SHA=${GIT_SHA:-unknown}" \
        --build-arg "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --label "git.sha=${GIT_SHA:-}" \
        --label "git.branch=${GIT_BRANCH:-}" \
        --label "build.tag=${BUILD_TAG:-}" \
        "$src_dir"

    for tag in "${tags[@]}"; do
        echo "Built: $tag"
    done

    export DOCKER_TAGS="${tags[*]}"
}

stage_docker_push() {
    local registry=${1:-}

    if [[ -z "${DOCKER_TAGS:-}" ]]; then
        echo "No tags to push" >&2
        return 1
    fi

    if [[ -n "$registry" ]]; then
        docker login "$registry" \
            -u "${REGISTRY_USER:?}" \
            -p "${REGISTRY_PASSWORD:?}"
    fi

    for tag in $DOCKER_TAGS; do
        echo "Pushing: $tag"
        docker push "$tag"
    done

    echo "All images pushed"
}

stage_docker_scan() {
    local image="${1:-}"

    if [[ -z "$image" ]]; then
        image=$(echo "$DOCKER_TAGS" | awk '{print $1}')
    fi

    if command -v trivy &>/dev/null; then
        echo "Scanning with Trivy: $image"
        trivy image \
            --exit-code 1 \
            --severity CRITICAL \
            --no-progress \
            "$image"
    elif command -v grype &>/dev/null; then
        echo "Scanning with Grype: $image"
        grype "$image" \
            --fail-on critical
    else
        echo "No vulnerability scanner available, skipping"
    fi
}
```

---

## 41.5 Deployment Strategies

```bash
#!/bin/bash
# deploy_strategies.sh

# ─── Rolling Deployment ────────────────────────────────────────
deploy_rolling() {
    local service=$1
    local image=$2
    local namespace=${3:-default}

    echo "Rolling deploy: $service → $image"

    kubectl set image "deployment/$service" \
        "${service}=${image}" \
        -n "$namespace"

    kubectl rollout status "deployment/$service" \
        -n "$namespace" \
        --timeout=300s

    echo "Rolling deploy complete"
}

# ─── Blue/Green Deployment ─────────────────────────────────────
deploy_blue_green() {
    local service=$1
    local image=$2
    local namespace=${3:-default}

    local current_color
    current_color=$(kubectl get service "$service" -n "$namespace" \
        -o jsonpath='{.spec.selector.color}' 2>/dev/null || echo "blue")

    local new_color="green"
    [[ "$current_color" == "green" ]] && new_color="blue"

    echo "Blue/Green deploy: $current_color → $new_color"

    cat <<EOF | kubectl apply -n "$namespace" -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${service}-${new_color}
  labels:
    app: ${service}
    color: ${new_color}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${service}
      color: ${new_color}
  template:
    metadata:
      labels:
        app: ${service}
        color: ${new_color}
    spec:
      containers:
      - name: ${service}
        image: ${image}
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

    kubectl rollout status "deployment/${service}-${new_color}" \
        -n "$namespace" --timeout=300s

    kubectl patch service "$service" -n "$namespace" \
        -p "{\"spec\":{\"selector\":{\"color\":\"${new_color}\"}}}" 

    sleep 10
    kubectl delete deployment "${service}-${current_color}" \
        -n "$namespace" --ignore-not-found

    echo "Blue/Green deploy complete: now serving $new_color"
}

# ─── Canary Deployment ─────────────────────────────────────────
deploy_canary() {
    local service=$1
    local image=$2
    local namespace=${3:-default}
    local canary_weight=${4:-10}

    echo "Canary deploy: $service @ ${canary_weight}%"

    local stable_replicas=9
    local canary_replicas=$(( stable_replicas * canary_weight / 100 ))
    [[ $canary_replicas -lt 1 ]] && canary_replicas=1

    cat <<EOF | kubectl apply -n "$namespace" -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${service}-canary
  labels:
    app: ${service}
    track: canary
spec:
  replicas: ${canary_replicas}
  selector:
    matchLabels:
      app: ${service}
      track: canary
  template:
    metadata:
      labels:
        app: ${service}
        track: canary
    spec:
      containers:
      - name: ${service}
        image: ${image}
EOF

    kubectl rollout status "deployment/${service}-canary" \
        -n "$namespace" --timeout=180s

    echo "Canary live: ${canary_weight}% traffic"
    echo "Monitoring for 5 minutes..."

    local check_interval=30
    local monitor_duration=300
    local elapsed=0
    local errors=0

    while (( elapsed < monitor_duration )); do
        sleep "$check_interval"
        (( elapsed += check_interval ))

        local error_rate
        error_rate=$(kubectl logs \
            -l "app=${service},track=canary" \
            -n "$namespace" \
            --since="${check_interval}s" 2>/dev/null | \
            grep -c "ERROR" || true)

        if (( error_rate > 5 )); then
            (( errors++ ))
            echo "  ⚠ High error rate at ${elapsed}s: $error_rate errors"
        else
            echo "  ✓ ${elapsed}s: error_rate=$error_rate"
        fi

        if (( errors >= 3 )); then
            echo "Too many errors, rolling back canary..."
            kubectl delete deployment "${service}-canary" \
                -n "$namespace" --ignore-not-found
            return 1
        fi
    done

    echo "Canary healthy, promoting..."
    kubectl set image "deployment/$service" \
        "${service}=${image}" -n "$namespace"
    kubectl delete deployment "${service}-canary" \
        -n "$namespace" --ignore-not-found

    echo "Canary promotion complete"
}
```

---

## 41.6 Pipeline Notifications

```bash
#!/bin/bash
# pipeline_notify.sh

notify_slack() {
    local status=$1
    local pipeline_name=$2
    local duration=$3
    local build_url=${4:-}

    local color emoji
    case "$status" in
        passed)  color="#00CC00"; emoji=":white_check_mark:" ;;
        failed)  color="#FF0000"; emoji=":x:" ;;
        warning) color="#FF9900"; emoji=":warning:" ;;
        *)       color="#CCCCCC"; emoji=":information_source:" ;;
    esac

    local payload
    payload=$(jq -n \
        --arg pipeline "$pipeline_name" \
        --arg status "$status" \
        --arg duration "$duration" \
        --arg color "$color" \
        --arg emoji "$emoji" \
        --arg branch "${GIT_BRANCH:-unknown}" \
        --arg sha "${GIT_SHA:-unknown}" \
        --arg url "${build_url}" \
        '{
            attachments: [{
                color: $color,
                title: ($emoji + " Pipeline: " + $pipeline),
                fields: [
                    {title: "Status", value: $status, short: true},
                    {title: "Duration", value: $duration, short: true},
                    {title: "Branch", value: $branch, short: true},
                    {title: "Commit", value: $sha, short: true}
                ],
                footer: "CI/CD Pipeline",
                ts: now|floor
            }]
        }')

    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "${SLACK_WEBHOOK_URL:?}" > /dev/null

    echo "Slack notification sent: $status"
}

notify_github_status() {
    local state=$1
    local description=$2
    local context=${3:-"CI/CD Pipeline"}

    local gh_state
    case "$state" in
        passed)  gh_state="success" ;;
        failed)  gh_state="failure" ;;
        running) gh_state="pending" ;;
        *)       gh_state="error" ;;
    esac

    curl -s -X POST \
        -H "Authorization: token ${GITHUB_TOKEN:?}" \
        -H "Content-Type: application/json" \
        -d "$(jq -n \
            --arg state "$gh_state" \
            --arg desc "$description" \
            --arg context "$context" \
            '{state: $state, description: $desc, context: $context}')" \
        "https://api.github.com/repos/${GITHUB_REPOSITORY:?}/statuses/${GIT_SHA:?}" > /dev/null

    echo "GitHub status: $gh_state - $description"
}
```

---

## 41.7 Artifact Management

```bash
#!/bin/bash
# artifact_manager.sh

save_artifact() {
    local source_path=$1
    local artifact_name=$2
    local artifact_dir="${PIPELINE_ARTIFACT_DIR}"

    mkdir -p "$artifact_dir"

    if [[ -d "$source_path" ]]; then
        tar -czf "${artifact_dir}/${artifact_name}.tar.gz" \
            -C "$(dirname "$source_path")" \
            "$(basename "$source_path")"
        echo "Saved artifact: ${artifact_name}.tar.gz"
    else
        cp "$source_path" "${artifact_dir}/${artifact_name}"
        echo "Saved artifact: $artifact_name"
    fi
}

upload_artifacts_s3() {
    local bucket="${S3_BUCKET:?}"
    local prefix="${BUILD_TAG:-$(date +%Y%m%d%H%M%S)}"

    echo "Uploading artifacts to s3://${bucket}/${prefix}/"

    aws s3 cp "${PIPELINE_ARTIFACT_DIR}/" \
        "s3://${bucket}/${prefix}/" \
        --recursive \
        --storage-class STANDARD_IA

    echo "Artifacts uploaded: s3://${bucket}/${prefix}/"
}

cleanup_old_artifacts() {
    local bucket="${S3_BUCKET:?}"
    local retention_days=${1:-30}

    local cutoff_date
    cutoff_date=$(date -d "${retention_days} days ago" +%Y-%m-%d 2>/dev/null || \
                  date -v-"${retention_days}"d +%Y-%m-%d)

    aws s3api list-objects-v2 \
        --bucket "$bucket" \
        --query "Contents[?LastModified<='${cutoff_date}T00:00:00'].Key" \
        --output text | \
        tr '\t' '\n' | \
        while read -r key; do
            [[ -z "$key" ]] && continue
            aws s3 rm "s3://${bucket}/${key}"
            echo "Removed: $key"
        done
}
```

---

## 41.8 Complete Pipeline Example

```bash
#!/bin/bash
# full_pipeline.sh - Complete CI/CD example

set -euo pipefail

source ./pipeline_core.sh
source ./build_stages.sh
source ./test_runner.sh
source ./docker_build_push.sh
source ./pipeline_notify.sh

main() {
    local repo_url="${REPO_URL:?Set REPO_URL}"
    local branch="${GIT_BRANCH:-main}"
    local image_name="${IMAGE_NAME:?Set IMAGE_NAME}"
    local registry="${REGISTRY:-}"
    local deploy_env="${DEPLOY_ENV:-staging}"

    pipeline_init "build-${branch}" "/tmp/ci_build_$$"

    trap 'pipeline_summary; notify_slack "${PIPELINE[status]}" "${PIPELINE[name]}" "$(( SECONDS - PIPELINE[start_time] ))s"' EXIT

    notify_github_status "running" "Pipeline started"

    run_stage "Checkout"           stage_checkout     "$repo_url" "$branch"
    run_stage "Install Deps"       stage_install_deps
    run_stage "Lint"               stage_lint
    run_stage "Unit Tests"         stage_unit_tests
    run_stage "Build"              stage_build
    run_stage "Docker Build"       stage_docker_build "$image_name" "$registry"
    run_stage "Security Scan"      stage_docker_scan
    run_stage "Integration Tests"  stage_integration_tests
    run_stage "Docker Push"        stage_docker_push  "$registry"

    if [[ "$branch" == "main" ]] && [[ "$deploy_env" != "none" ]]; then
        local image="${registry:+${registry}/}${image_name}:${BUILD_TAG}"
        run_stage "Deploy to ${deploy_env}" deploy_rolling \
            "$image_name" "$image" "$deploy_env"
    fi

    PIPELINE[status]="passed"
    notify_github_status "passed" "Pipeline passed"

    pipeline_summary
}

main "$@"
```

---

## 41.9 GitHub Actions Integration

```bash
#!/bin/bash
# gh_actions_trigger.sh

trigger_workflow() {
    local owner=$1
    local repo=$2
    local workflow_id=$3
    local ref=${4:-main}
    local inputs=${5:-{}}

    echo "Triggering: ${owner}/${repo} / $workflow_id @ $ref"

    curl -s -X POST \
        -H "Authorization: token ${GITHUB_TOKEN:?}" \
        -H "Accept: application/vnd.github.v3+json" \
        -d "$(jq -n \
            --arg ref "$ref" \
            --argjson inputs "$inputs" \
            '{ref: $ref, inputs: $inputs}')" \
        "https://api.github.com/repos/${owner}/${repo}/actions/workflows/${workflow_id}/dispatches"

    echo "Workflow triggered"
}

wait_for_workflow() {
    local owner=$1
    local repo=$2
    local workflow_id=$3
    local ref=${4:-main}
    local timeout=${5:-600}

    echo "Waiting for workflow (timeout: ${timeout}s)..."

    local elapsed=0 poll_interval=15 run_id=""

    while (( elapsed < timeout )); do
        sleep $poll_interval
        (( elapsed += poll_interval ))

        local response
        response=$(curl -s \
            -H "Authorization: token ${GITHUB_TOKEN:?}" \
            -H "Accept: application/vnd.github.v3+json" \
            "https://api.github.com/repos/${owner}/${repo}/actions/workflows/${workflow_id}/runs?branch=${ref}&per_page=1")

        [[ -z "$run_id" ]] && run_id=$(echo "$response" | jq -r '.workflow_runs[0].id // empty')
        [[ -n "$run_id" ]] || continue

        local status conclusion
        status=$(echo "$response" | jq -r '.workflow_runs[0].status')
        conclusion=$(echo "$response" | jq -r '.workflow_runs[0].conclusion // ""')

        echo "  [${elapsed}s] $status ${conclusion:+(${conclusion})}"

        if [[ "$status" == "completed" ]]; then
            [[ "$conclusion" == "success" ]] && return 0 || return 1
        fi
    done

    echo "Timeout"; return 1
}
```

---

## 41.10 Exercises

### Exercise 1: Self-Hosted Runner Setup
สร้าง script จัดการ GitHub Actions runner:
- Register runner with token
- Run as systemd service
- Auto-update runner binary
- Monitor runner health

### Exercise 2: Multi-Environment Promotion
สร้าง pipeline:
- Build once, deploy many
- dev → staging → prod
- Approval gates
- Rollback automation

### Exercise 3: Pipeline Metrics Dashboard
สร้าง dashboard แสดง:
- Build success rate trend
- Average build duration
- Flaky test detection

---

## สรุป Part 41

✅ Pipeline framework (stages, status tracking)  
✅ Build automation (checkout, deps, lint, build)  
✅ Test runner (unit, integration, coverage)  
✅ Docker build & push pipeline  
✅ Blue/Green & Canary deployment  
✅ Pipeline notifications (Slack, GitHub)  
✅ Artifact management (S3)  
✅ GitHub Actions trigger & monitor  

---

**→ Part 42: Infrastructure as Code with Bash**
