# Part 54: CI/CD Pipeline Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 54.1 Pipeline Framework

```bash
#!/bin/bash
# pipeline.sh - CI/CD pipeline framework in pure Bash

declare -A PIPELINE_STAGES=()
declare -a PIPELINE_ORDER=()
declare -A STAGE_STATUS=()
declare -A STAGE_DURATION=()
PIPELINE_START_TIME=0
PIPELINE_FAILED=false

pipeline_add_stage() {
    local name=$1 handler=$2 depends_on=${3:-}

    PIPELINE_STAGES["$name"]="$handler"
    PIPELINE_ORDER+=("$name")
    [[ -n "$depends_on" ]] && PIPELINE_STAGES["${name}__deps"]="$depends_on"
}

pipeline_run_stage() {
    local name=$1
    local handler="${PIPELINE_STAGES[$name]}"
    local start_time

    echo ""
    echo "Stage: $name"
    echo "Time:  $(date '+%H:%M:%S')"

    start_time=$(date +%s%N)
    STAGE_STATUS["$name"]="running"

    if $handler; then
        local end_time
        end_time=$(date +%s%N)
        STAGE_DURATION["$name"]=$(( (end_time - start_time) / 1000000 ))
        STAGE_STATUS["$name"]="passed"
        echo "  Stage $name: PASSED (${STAGE_DURATION[$name]}ms)"
    else
        local exit_code=$?
        STAGE_STATUS["$name"]="failed"
        echo "  Stage $name: FAILED (exit code: $exit_code)"
        PIPELINE_FAILED=true
        return 1
    fi
}

pipeline_run() {
    PIPELINE_START_TIME=$(date +%s)
    PIPELINE_FAILED=false

    echo "Pipeline started: $(date)"

    for stage_name in "${PIPELINE_ORDER[@]}"; do
        # Check dependencies
        local deps="${PIPELINE_STAGES[${stage_name}__deps]:-}"
        if [[ -n "$deps" ]]; then
            for dep in $deps; do
                if [[ "${STAGE_STATUS[$dep]:-}" != "passed" ]]; then
                    STAGE_STATUS["$stage_name"]="skipped"
                    echo "  Stage $stage_name: SKIPPED (dependency $dep not passed)"
                    continue 2
                fi
            done
        fi

        pipeline_run_stage "$stage_name" || {
            if [[ "${PIPELINE_FAIL_FAST:-true}" == "true" ]]; then
                break
            fi
        }
    done

    pipeline_summary
    $PIPELINE_FAILED && return 1 || return 0
}

pipeline_summary() {
    local end_time
    end_time=$(date +%s)
    local total=$(( end_time - PIPELINE_START_TIME ))

    echo ""
    echo "Pipeline Summary"
    for stage_name in "${PIPELINE_ORDER[@]}"; do
        local status="${STAGE_STATUS[$stage_name]:-skipped}"
        local duration="${STAGE_DURATION[$stage_name]:-0}"
        case "$status" in
            passed)  echo "  [PASS] $stage_name (${duration}ms)" ;;
            failed)  echo "  [FAIL] $stage_name (${duration}ms)" ;;
            skipped) echo "  [SKIP] $stage_name" ;;
        esac
    done
    echo ""
    echo "Total duration: ${total}s"
    echo "Status: $($PIPELINE_FAILED && echo FAILED || echo PASSED)"
}
```

---

## 54.2 Build System

```bash
#!/bin/bash
# build_system.sh - Build automation

BUILD_DIR="${BUILD_DIR:-./build}"
DIST_DIR="${DIST_DIR:-./dist}"
VERSION_FILE="${VERSION_FILE:-VERSION}"

get_version() {
    if [[ -f "$VERSION_FILE" ]]; then
        cat "$VERSION_FILE"
    elif git describe --tags --exact-match 2>/dev/null; then
        git describe --tags --exact-match
    elif git describe --tags 2>/dev/null; then
        git describe --tags
    else
        echo "0.0.0-$(git rev-parse --short HEAD 2>/dev/null || echo 'unknown')"
    fi
}

bump_version() {
    local bump_type=${1:-patch}
    local current
    current=$(get_version)
    local major minor patch

    IFS='.' read -r major minor patch <<< "${current%-*}"

    case "$bump_type" in
        major) ((major++)); minor=0; patch=0 ;;
        minor) ((minor++)); patch=0 ;;
        patch) ((patch++)) ;;
    esac

    local new_version="${major}.${minor}.${patch}"
    echo "$new_version" > "$VERSION_FILE"
    echo "Version bumped: $current -> $new_version"
    echo "$new_version"
}

tag_release() {
    local version=${1:-$(get_version)}
    local message=${2:-"Release $version"}

    git tag -a "v${version}" -m "$message"
    echo "Tagged: v${version}"
}

stage_clean() {
    echo "Cleaning build artifacts..."
    rm -rf "$BUILD_DIR" "$DIST_DIR"
    mkdir -p "$BUILD_DIR" "$DIST_DIR"
}

stage_lint() {
    echo "Running linters..."
    local failed=0

    if command -v shellcheck &>/dev/null; then
        while IFS= read -r script; do
            if ! shellcheck -S warning "$script" 2>/dev/null; then
                echo "  ShellCheck failed: $script"
                ((failed++))
            fi
        done < <(find . -name "*.sh" -not -path "./.git/*" -not -path "./vendor/*")
    fi

    while IFS= read -r script; do
        if ! bash -n "$script" 2>/dev/null; then
            echo "  Syntax error: $script"
            ((failed++))
        fi
    done < <(find . -name "*.sh" -not -path "./.git/*")

    [[ $failed -eq 0 ]] && echo "  All lint checks passed"
    return $failed
}

stage_test() {
    echo "Running tests..."
    local test_dir="${TEST_DIR:-./tests}"
    local failed=0 passed=0

    if [[ ! -d "$test_dir" ]]; then
        echo "  No tests directory found, skipping"
        return 0
    fi

    while IFS= read -r test_file; do
        local test_name
        test_name=$(basename "$test_file" .sh)

        if bash "$test_file" 2>/dev/null; then
            ((passed++))
            echo "  PASS: $test_name"
        else
            ((failed++))
            echo "  FAIL: $test_name"
        fi
    done < <(find "$test_dir" -name "test_*.sh" -o -name "*_test.sh")

    echo "  Tests: $passed passed, $failed failed"
    return $failed
}

stage_build() {
    local version
    version=$(get_version)
    echo "Building version $version..."

    mkdir -p "$BUILD_DIR"

    local tarball="$DIST_DIR/app-${version}.tar.gz"
    tar -czf "$tarball" \
        --exclude=".git" \
        --exclude="$BUILD_DIR" \
        --exclude="$DIST_DIR" \
        --exclude="*.tmp" \
        . 2>/dev/null
    echo "  Archive: $tarball"
}

stage_docker_build() {
    local version
    version=$(get_version)
    local image_name="${DOCKER_IMAGE:-app}"
    local full_tag="${image_name}:${version}"

    echo "Building Docker image: $full_tag..."

    docker build \
        --build-arg VERSION="$version" \
        --build-arg BUILD_DATE="$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
        --label "version=$version" \
        -t "$full_tag" \
        -t "${image_name}:latest" \
        .

    echo "  Image: $full_tag"
}
```

---

## 54.3 Deployment Automation

```bash
#!/bin/bash
# deploy.sh - Deployment automation

DEPLOY_LOG="/var/log/deploy.log"
DEPLOY_LOCK="/var/run/deploy.lock"
ROLLBACK_DIR="/var/deploy/rollbacks"

acquire_deploy_lock() {
    local timeout=${1:-300}
    local elapsed=0

    while [[ -f "$DEPLOY_LOCK" ]]; do
        local lock_pid
        lock_pid=$(cat "$DEPLOY_LOCK" 2>/dev/null)
        if ! kill -0 "$lock_pid" 2>/dev/null; then
            echo "Stale lock removed"
            rm -f "$DEPLOY_LOCK"
            break
        fi
        if (( elapsed >= timeout )); then
            echo "ERROR: Deploy lock timeout after ${timeout}s"
            return 1
        fi
        sleep 5
        ((elapsed += 5))
    done

    echo $$ > "$DEPLOY_LOCK"
    trap 'rm -f "$DEPLOY_LOCK"' EXIT
}

deploy_log() {
    local level=$1 message=$2
    echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') [$level] $message" | tee -a "$DEPLOY_LOG"
}

pre_deploy_checks() {
    local environment=$1
    local version=$2

    deploy_log INFO "Running pre-deployment checks for $version -> $environment"

    local disk_free
    disk_free=$(df / | awk 'NR==2{print $5}' | tr -d '%')
    if (( disk_free > 80 )); then
        deploy_log ERROR "Insufficient disk space (${disk_free}% used)"
        return 1
    fi

    deploy_log INFO "Pre-deployment checks passed"
}

blue_green_deploy() {
    local version=$1
    local environment=${2:-production}
    local health_check_url=${3:-"http://localhost/health"}

    acquire_deploy_lock || return 1

    deploy_log INFO "Starting blue/green deployment: $version"

    local current_slot
    current_slot=$(cat /var/deploy/active_slot 2>/dev/null || echo "blue")
    local new_slot
    new_slot=$([[ "$current_slot" == "blue" ]] && echo "green" || echo "blue")
    local new_port
    new_port=$([[ "$new_slot" == "blue" ]] && echo "8080" || echo "8081")

    deploy_log INFO "Deploying to $new_slot slot (port $new_port)"

    deploy_to_slot "$new_slot" "$new_port" "$version" || {
        deploy_log ERROR "Deployment to $new_slot failed"
        return 1
    }

    local retries=0
    while (( retries < 12 )); do
        if curl -sf "${health_check_url}:${new_port}/health" &>/dev/null; then
            deploy_log INFO "Health check passed on $new_slot"
            break
        fi
        ((retries++))
        sleep 5
    done

    if (( retries >= 12 )); then
        deploy_log ERROR "Health check failed after 60s"
        return 1
    fi

    switch_traffic "$new_slot" "$new_port"
    echo "$new_slot" > /var/deploy/active_slot
    echo "$version" > /var/deploy/current_version

    deploy_log INFO "Traffic switched to $new_slot"
    sleep 30
    stop_slot "$current_slot"
    deploy_log INFO "Blue/green deployment complete: $version"
}

deploy_to_slot() {
    local slot=$1 port=$2 version=$3
    rsync -a "/opt/releases/${version}/" "/opt/slots/${slot}/"
    sed -i "s/PORT=.*/PORT=${port}/" "/opt/slots/${slot}/.env"
    systemctl restart "app@${slot}" 2>/dev/null
}

switch_traffic() {
    local slot=$1 port=$2
    cat > /etc/nginx/conf.d/upstream.conf << EOF
upstream app_backend {
    server 127.0.0.1:${port};
    keepalive 32;
}
EOF
    nginx -t && nginx -s reload
}

stop_slot() {
    local slot=$1
    systemctl stop "app@${slot}" 2>/dev/null || true
    deploy_log INFO "Stopped slot: $slot"
}

rollback() {
    local target_version=${1:-}
    local environment=${2:-production}

    deploy_log WARN "Initiating rollback (environment: $environment)"

    if [[ -z "$target_version" ]]; then
        target_version=$(ls -t "$ROLLBACK_DIR" 2>/dev/null | head -2 | tail -1)
        [[ -z "$target_version" ]] && { deploy_log ERROR "No rollback version found"; return 1; }
    fi

    deploy_log INFO "Rolling back to: $target_version"
    blue_green_deploy "$target_version" "$environment"
    deploy_log INFO "Rollback complete"
}
```

---

## 54.4 GitOps Integration

```bash
#!/bin/bash
# gitops.sh - GitOps workflow automation

GITOPS_REPO="${GITOPS_REPO:-./}"
GITOPS_BRANCH="${GITOPS_BRANCH:-main}"
APPS_DIR="${GITOPS_REPO}/apps"

gitops_sync() {
    local app=${1:-}

    echo "Syncing GitOps state..."
    git -C "$GITOPS_REPO" fetch origin "$GITOPS_BRANCH"
    git -C "$GITOPS_REPO" reset --hard "origin/$GITOPS_BRANCH"

    if [[ -n "$app" ]]; then
        apply_app_config "$app"
    else
        while IFS= read -r app_dir; do
            local app_name
            app_name=$(basename "$app_dir")
            apply_app_config "$app_name"
        done < <(find "$APPS_DIR" -mindepth 1 -maxdepth 1 -type d 2>/dev/null)
    fi
}

apply_app_config() {
    local app=$1
    local app_dir="$APPS_DIR/$app"

    [[ ! -d "$app_dir" ]] && { echo "App not found: $app"; return 1; }

    echo "Applying config for: $app"

    [[ -d "$app_dir/k8s" ]] && kubectl apply -f "$app_dir/k8s/" --recursive 2>/dev/null
    [[ -f "$app_dir/docker-compose.yml" ]] && docker compose -f "$app_dir/docker-compose.yml" up -d
    [[ -f "$app_dir/apply.sh" ]] && bash "$app_dir/apply.sh"
}

update_image_tag() {
    local app=$1 image=$2 new_tag=$3 branch="${4:-main}"
    local app_dir="$APPS_DIR/$app"

    echo "Updating $app image to $image:$new_tag..."

    local deployment_file="$app_dir/k8s/deployment.yaml"
    [[ -f "$deployment_file" ]] && \
        sed -i "s|image: ${image}:.*|image: ${image}:${new_tag}|g" "$deployment_file"

    local compose_file="$app_dir/docker-compose.yml"
    [[ -f "$compose_file" ]] && \
        sed -i "s|image: ${image}:.*|image: ${image}:${new_tag}|g" "$compose_file"

    git -C "$GITOPS_REPO" add "$app_dir"
    git -C "$GITOPS_REPO" commit -m "chore: update $app image to $image:$new_tag"
    git -C "$GITOPS_REPO" push origin "$branch"

    echo "Image tag updated and pushed"
}

promote_to_environment() {
    local app=$1 source_env=$2 target_env=$3
    local source_dir="$APPS_DIR/$app/envs/$source_env"
    local target_dir="$APPS_DIR/$app/envs/$target_env"

    local current_image
    current_image=$(grep "image:" "$source_dir/deployment.yaml" | awk '{print $2}' | head -1)

    sed -i "s|image: .*|image: $current_image|g" "$target_dir/deployment.yaml"

    git -C "$GITOPS_REPO" add "$target_dir"
    git -C "$GITOPS_REPO" commit -m "chore: promote $app to $target_env ($current_image)"
    git -C "$GITOPS_REPO" push origin "$GITOPS_BRANCH"

    echo "Promoted: $app -> $target_env ($current_image)"
}
```

---

## 54.5 Notification Integration

```bash
#!/bin/bash
# notifications.sh - CI/CD notification integrations

slack_notify() {
    local webhook_url=$1
    local channel=${2:-"#deployments"}
    local status=$3
    local title=$4
    local message=$5

    local color
    case "$status" in
        success) color="#36a64f" ;;
        failure) color="#ff0000" ;;
        warning) color="#ffcc00" ;;
        *)       color="#cccccc" ;;
    esac

    local payload
    payload=$(printf '{"channel":"%s","attachments":[{"color":"%s","title":"%s","text":"%s","ts":%s}]}' \
        "$channel" "$color" "$title" "$message" "$(date +%s)")

    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "$webhook_url" &>/dev/null
}

github_set_status() {
    local token=$1 repo=$2 sha=$3 state=$4 context=$5 description=$6

    local payload
    payload=$(printf '{"state":"%s","description":"%s","context":"%s"}' \
        "$state" "$description" "$context")

    curl -s -X POST \
        -H "Authorization: token $token" \
        -H "Accept: application/vnd.github.v3+json" \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "https://api.github.com/repos/${repo}/statuses/${sha}"
}

send_deploy_email() {
    local to=$1 subject=$2 body=$3

    if command -v sendmail &>/dev/null; then
        printf 'To: %s\nSubject: %s\n\n%s' "$to" "$subject" "$body" | sendmail -v "$to" 2>/dev/null
    elif command -v mail &>/dev/null; then
        echo "$body" | mail -s "$subject" "$to"
    fi
}

notify_pipeline_result() {
    local pipeline_name=$1 status=$2 duration=$3 details=$4

    local message="Pipeline: $pipeline_name | Status: $status | Duration: ${duration}s"

    [[ -n "${SLACK_WEBHOOK:-}" ]] && \
        slack_notify "$SLACK_WEBHOOK" "#deployments" \
            "$([[ $status == 'PASSED' ]] && echo success || echo failure)" \
            "Pipeline $status: $pipeline_name" "$message"

    [[ -n "${NOTIFY_EMAIL:-}" ]] && \
        send_deploy_email "$NOTIFY_EMAIL" "[CI/CD] $status: $pipeline_name" "$message"
}
```

---

## 54.6 Exercises

### Exercise 1: Full CI/CD Pipeline
สร้าง pipeline สำหรับ application ที่:
- Stage: lint → test → build → docker → deploy
- Parallel test execution
- Blue/green deployment
- Slack notifications ทุก stage

### Exercise 2: GitOps Reconciler
สร้าง reconciler ที่:
- Poll git repository ทุก 30 วินาที
- Detect drift จาก desired state
- Auto-apply changes
- Alert เมื่อ sync ล้มเหลว

### Exercise 3: Release Automation
สร้าง release workflow ที่:
- Semantic version bumping
- Changelog generation จาก commits
- Git tagging
- GitHub release creation

---

## สรุป Part 54

✅ Pipeline framework with stage dependencies and fail-fast
✅ Build system: version management, lint, test, bundle, Docker
✅ Blue/green deployment with health checks and traffic switching
✅ Automatic rollback to previous version
✅ GitOps sync: apply manifests, update image tags
✅ Environment promotion workflow
✅ Slack, GitHub Status, and email notifications

---

**→ Part 55: Monitoring and Alerting Systems**
