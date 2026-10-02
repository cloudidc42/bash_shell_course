# Part 80: CI/CD Pipeline Scripting
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 80.1 Git Helpers for CI

```bash
#!/bin/bash
# ci_git.sh - Git operations for CI pipelines

set -euo pipefail

git_current_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null
}

git_current_sha() {
    git rev-parse --short HEAD 2>/dev/null
}

git_current_sha_full() {
    git rev-parse HEAD 2>/dev/null
}

git_tag_exists() {
    git tag -l "$1" | grep -q "^$1$"
}

git_latest_tag() {
    git describe --tags --abbrev=0 2>/dev/null || echo "0.0.0"
}

git_tag_create() {
    local tag=$1 message=${2:-"Release $tag"}
    git tag -a "$tag" -m "$message"
    echo "Tag created: $tag"
}

git_tag_push() {
    local tag=$1 remote=${2:-origin}
    git push "$remote" "$tag"
    echo "Tag pushed: $tag"
}

semver_bump() {
    local version=$1 part=${2:-patch}

    local major minor patch
    IFS='.' read -r major minor patch <<< "${version#v}"

    case "$part" in
        major) (( major++ )); minor=0; patch=0 ;;
        minor) (( minor++ )); patch=0 ;;
        patch) (( patch++ )) ;;
        *) echo "Unknown part: $part" >&2; return 1 ;;
    esac

    echo "${major}.${minor}.${patch}"
}

git_changelog() {
    local from=${1:-} to=${2:-HEAD} format=${3:-oneline}

    local range
    if [[ -n "$from" ]]; then
        range="${from}..${to}"
    else
        local prev_tag; prev_tag=$(git describe --tags --abbrev=0 HEAD~1 2>/dev/null || echo "")
        range="${prev_tag:+${prev_tag}..}${to}"
    fi

    case "$format" in
        oneline) git log --oneline "$range" 2>/dev/null ;;
        full)    git log --pretty=format:'- %s (%h)' "$range" 2>/dev/null ;;
        grouped)
            echo "## Changes"
            echo ""
            echo "### Features"
            git log --oneline "$range" 2>/dev/null | grep -i '^[a-f0-9]* feat' | sed 's/^[a-f0-9]* /- /' || true
            echo ""
            echo "### Bug Fixes"
            git log --oneline "$range" 2>/dev/null | grep -i '^[a-f0-9]* fix' | sed 's/^[a-f0-9]* /- /' || true
            echo ""
            echo "### Other"
            git log --oneline "$range" 2>/dev/null | grep -iv '^[a-f0-9]* \(feat\|fix\)' | sed 's/^[a-f0-9]* /- /' || true
            ;;
    esac
}

git_files_changed() {
    local from=${1:-HEAD~1} to=${2:-HEAD}
    git diff --name-only "${from}..${to}" 2>/dev/null
}

git_is_clean() {
    [[ -z "$(git status --porcelain 2>/dev/null)" ]]
}

git_ensure_clean() {
    if ! git_is_clean; then
        echo "ERROR: Working tree is not clean" >&2
        git status --short >&2
        return 1
    fi
}
```

---

## 80.2 Build Matrix

```bash
#!/bin/bash
# build_matrix.sh - Parallel build across multiple dimensions

declare -a MATRIX_RESULTS=()
declare -i MATRIX_FAILED=0

build_matrix_run() {
    local matrix_json=$1 build_fn=$2 max_parallel=${3:-4}

    local -a combinations
    mapfile -t combinations < <(echo "$matrix_json" | jq -r '.[] | tojson')

    local -a pids=()
    local -a labels=()
    local -a result_files=()

    for combo in "${combinations[@]}"; do
        local label; label=$(echo "$combo" | jq -r 'to_entries | map(.key+"="+.value) | join(",")')
        local result_file; result_file=$(mktemp)
        result_files+=("$result_file")
        labels+=("$label")

        (
            export BUILD_COMBO="$combo"
            if "$build_fn" "$combo" > "$result_file" 2>&1; then
                echo "PASS" >> "$result_file"
            else
                echo "FAIL" >> "$result_file"
            fi
        ) &
        pids+=($!)

        # Throttle parallel jobs
        while (( ${#pids[@]} >= max_parallel )); do
            local new_pids=()
            for pid in "${pids[@]}"; do
                kill -0 "$pid" 2>/dev/null && new_pids+=("$pid") || wait "$pid" 2>/dev/null
            done
            pids=("${new_pids[@]}")
            (( ${#pids[@]} >= max_parallel )) && sleep 0.5
        done
    done

    # Wait for all
    for pid in "${pids[@]}"; do wait "$pid" 2>/dev/null; done

    # Collect results
    echo ""
    echo "=== Build Matrix Results ==="
    local i=0
    for result_file in "${result_files[@]}"; do
        local status; status=$(tail -1 "$result_file")
        local label="${labels[$i]}"
        printf '  %-50s %s\n' "$label" "$status"
        [[ "$status" == "FAIL" ]] && (( MATRIX_FAILED++ ))
        rm -f "$result_file"
        (( i++ ))
    done

    echo ""
    echo "Failed: $MATRIX_FAILED"
    (( MATRIX_FAILED == 0 ))
}
```

---

## 80.3 Artifact Management

```bash
#!/bin/bash
# artifacts.sh - Build artifact upload/download/sign

ARTIFACT_DIR="${ARTIFACT_DIR:-/tmp/artifacts}"
ARTIFACT_STORE="${ARTIFACT_STORE:-s3://my-artifacts}"

mkdir -p "$ARTIFACT_DIR"

artifact_save() {
    local name=$1 path=$2 build_id=${3:-${BUILD_ID:-unknown}}
    local dest="${ARTIFACT_DIR}/${build_id}/${name}"
    mkdir -p "$(dirname "$dest")"
    cp -r "$path" "$dest"
    echo "$dest"
}

artifact_load() {
    local name=$1 build_id=${2:-${BUILD_ID:-unknown}}
    echo "${ARTIFACT_DIR}/${build_id}/${name}"
}

artifact_upload_s3() {
    local local_path=$1 s3_key=$2
    aws s3 cp "$local_path" "${ARTIFACT_STORE}/${s3_key}" \
        --only-show-errors
    echo "Uploaded: $s3_key"
}

artifact_download_s3() {
    local s3_key=$1 local_path=$2
    aws s3 cp "${ARTIFACT_STORE}/${s3_key}" "$local_path" \
        --only-show-errors
    echo "Downloaded: $s3_key -> $local_path"
}

artifact_sign() {
    local file=$1 key_id=${2:-}
    local sig_file="${file}.sha256"
    sha256sum "$file" > "$sig_file"
    if [[ -n "$key_id" ]] && command -v gpg &>/dev/null; then
        gpg --detach-sign --armor --local-user "$key_id" "$file"
    fi
    echo "Signed: $sig_file"
}

artifact_verify() {
    local file=$1 sig_file="${1}.sha256"
    if [[ ! -f "$sig_file" ]]; then
        echo "No signature file: $sig_file" >&2
        return 1
    fi
    sha256sum --check "$sig_file" 2>/dev/null
}

artifact_list() {
    local build_id=${1:-}
    if [[ -n "$build_id" ]]; then
        find "${ARTIFACT_DIR}/${build_id}" -type f 2>/dev/null | sort
    else
        find "$ARTIFACT_DIR" -type f 2>/dev/null | sort
    fi
}
```

---

## 80.4 Notification Helpers

```bash
#!/bin/bash
# ci_notify.sh - CI/CD notification helpers

slack_notify() {
    local message=$1 color=${2:-good} webhook=${SLACK_WEBHOOK:-}

    [[ -z "$webhook" ]] && { echo "SLACK_WEBHOOK not set"; return 1; }

    local payload
    payload=$(jq -n \
        --arg text "$message" \
        --arg color "$color" \
        '{attachments:[{color:$color, text:$text, ts: now|todate}]}')

    curl -s -X POST \
        --header 'Content-Type: application/json' \
        --data "$payload" \
        "$webhook" > /dev/null
}

slack_build_status() {
    local status=$1 app=$2 version=${3:-} branch=${4:-} sha=${5:-}
    local color; [[ "$status" == "success" ]] && color="good" || color="danger"
    local icon; [[ "$status" == "success" ]] && icon=":white_check_mark:" || icon=":x:"
    local msg="${icon} *${app}* build ${status}"
    [[ -n "$version" ]] && msg+="  version: ${version}"
    [[ -n "$branch" ]]  && msg+="  branch: ${branch}"
    [[ -n "$sha" ]]     && msg+="  commit: ${sha}"
    slack_notify "$msg" "$color"
}

github_set_commit_status() {
    local repo=$1 sha=$2 state=$3 description=$4 context=${5:-ci}
    local token=${GITHUB_TOKEN:-}

    [[ -z "$token" ]] && { echo "GITHUB_TOKEN not set"; return 1; }
    [[ "$sha" =~ ^[0-9a-f]{40}$ ]] || sha=$(git rev-parse "$sha" 2>/dev/null || echo "$sha")

    curl -s -X POST \
        --header "Authorization: token $token" \
        --header 'Content-Type: application/json' \
        --data "{\"state\":\"$state\",\"description\":\"$description\",\"context\":\"$context\"}" \
        "https://api.github.com/repos/${repo}/statuses/${sha}" > /dev/null
    echo "GitHub status: $state ($context)"
}

create_github_release() {
    local repo=$1 tag=$2 title=$3 body=$4 draft=${5:-false}
    local token=${GITHUB_TOKEN:-}

    [[ -z "$token" ]] && { echo "GITHUB_TOKEN not set"; return 1; }

    local payload; payload=$(jq -n \
        --arg tag "$tag" \
        --arg name "$title" \
        --arg body "$body" \
        --argjson draft "$draft" \
        '{tag_name:$tag, name:$name, body:$body, draft:$draft}')

    curl -s -X POST \
        --header "Authorization: token $token" \
        --header 'Content-Type: application/json' \
        --data "$payload" \
        "https://api.github.com/repos/${repo}/releases" | \
        jq -r '.html_url // "error"'
}
```

---

## 80.5 Quality Gates

```bash
#!/bin/bash
# quality_gates.sh - CI quality gate enforcement

QG_RESULTS=()
QG_FAILED=0

qg_check() {
    local name=$1 fn=$2
    shift 2
    local -a args=("$@")

    echo -n "Gate: $name ... "
    if "$fn" "${args[@]}" 2>/dev/null; then
        echo "PASS"
        QG_RESULTS+=("PASS:$name")
    else
        echo "FAIL"
        QG_RESULTS+=("FAIL:$name")
        (( QG_FAILED++ ))
    fi
}

qg_coverage_threshold() {
    local coverage_file=$1 threshold=${2:-80}
    local coverage; coverage=$(grep -oP 'Total.*?\K[0-9.]+(?=%)' "$coverage_file" 2>/dev/null | tail -1)
    [[ -z "$coverage" ]] && { echo "Cannot read coverage from: $coverage_file" >&2; return 1; }
    awk "BEGIN{ exit ($coverage < $threshold) }"
}

qg_test_results() {
    local junit_xml=$1
    local failures; failures=$(grep -c 'failure\|error' "$junit_xml" 2>/dev/null || echo 0)
    (( failures == 0 ))
}

qg_lint_pass() {
    local lint_log=$1
    ! grep -q 'error\|Error' "$lint_log" 2>/dev/null
}

qg_docker_scan() {
    local image=$1 threshold=${2:-high}
    if command -v trivy &>/dev/null; then
        trivy image --exit-code 1 \
            --severity "${threshold^^}" \
            --quiet "$image" 2>/dev/null
    else
        echo "trivy not installed, skipping" >&2
        return 0
    fi
}

qg_summary() {
    echo ""
    echo "=== Quality Gate Summary ==="
    for result in "${QG_RESULTS[@]}"; do
        local status="${result%%:*}" name="${result#*:}"
        printf '  %-10s %s\n' "$status" "$name"
    done
    echo ""
    echo "Failed gates: $QG_FAILED"
    (( QG_FAILED == 0 ))
}
```

---

## 80.6 Full CI Pipeline Example

```bash
#!/bin/bash
# ci_pipeline.sh - Complete CI/CD pipeline

pipeline_main() {
    local app=${1:-myapp}
    local version; version=$(git_latest_tag)
    local sha; sha=$(git_current_sha)
    local branch; branch=$(git_current_branch)
    local build_id="${app}-${sha}"

    echo "=== CI Pipeline: $app ==="
    echo "Version: $version  SHA: $sha  Branch: $branch"
    echo ""

    # Set pending GitHub status
    github_set_commit_status "${GITHUB_REPO:-org/repo}" "$sha" pending "Pipeline running" "ci/pipeline"

    local exit_code=0

    # Step 1: Build
    echo "--- Build ---"
    if docker_build "${app}:${sha}" . Dockerfile; then
        echo "Build OK"
    else
        echo "Build FAILED" >&2
        github_set_commit_status "${GITHUB_REPO:-org/repo}" "$sha" failure "Build failed" "ci/pipeline"
        slack_build_status failure "$app" "$sha" "$branch"
        exit 1
    fi

    # Step 2: Tests
    echo ""
    echo "--- Tests ---"
    if docker run --rm "${app}:${sha}" sh -c 'npm test 2>&1'; then
        echo "Tests OK"
    else
        echo "Tests FAILED" >&2
        exit_code=1
    fi

    # Step 3: Quality gates
    echo ""
    echo "--- Quality Gates ---"
    if (( exit_code == 0 )); then
        qg_check "test_results" qg_test_results "/tmp/junit.xml" || (( exit_code++ ))
    fi

    # Step 4: Push image (if on main/master)
    if [[ "$branch" =~ ^(main|master)$ ]] && (( exit_code == 0 )); then
        echo ""
        echo "--- Push Image ---"
        local new_version; new_version=$(semver_bump "$version" patch)
        docker tag "${app}:${sha}" "${REGISTRY:-registry.example.com}/${app}:${new_version}"
        docker push "${REGISTRY:-registry.example.com}/${app}:${new_version}"

        git_tag_create "v${new_version}" "Release $new_version"
        git_tag_push "v${new_version}"

        # Create GitHub release
        local changelog; changelog=$(git_changelog "" "HEAD" grouped)
        create_github_release "${GITHUB_REPO:-org/repo}" "v${new_version}" \
            "Release v${new_version}" "$changelog" false

        echo "Released: v${new_version}"
    fi

    # Final status
    if (( exit_code == 0 )); then
        github_set_commit_status "${GITHUB_REPO:-org/repo}" "$sha" success "Pipeline passed" "ci/pipeline"
        slack_build_status success "$app" "$sha" "$branch"
    else
        github_set_commit_status "${GITHUB_REPO:-org/repo}" "$sha" failure "Pipeline failed" "ci/pipeline"
        slack_build_status failure "$app" "$sha" "$branch"
    fi

    return $exit_code
}
```

---

## 80.7 Exercises

### Exercise 1: Multi-stage Build Pipeline
สร้าง pipeline ที่:
- Parallel: lint, test, security scan
- Sequential: build image → push → deploy
- Quality gates ก่อนแต่ละขั้น
- Artifact เซ็นโดย build number

### Exercise 2: Auto Release
สร้าง script สำหรับ:
- Parse commit messages for feat/fix/breaking
- Bump semver accordingly
- Generate CHANGELOG.md
- Tag, push, create GitHub release

### Exercise 3: Rollback System
สร้าง rollback system ที่:
- Track last good deploy (version + sha)
- Detect failures ผ่าน health check
- Auto-rollback Helm release
- Notify on rollback

---

## สรุป Part 80

✅ Git helpers: branch, sha, tag, semver_bump, changelog (grouped/oneline/full)
┅ Build matrix: parallel N-dim, throttle, result collection
┅ Artifacts: save/load locally, S3 upload/download, sign (sha256+gpg), verify
┅ Notifications: Slack (color+attachment), GitHub commit status, create release
┅ Quality gates: coverage threshold, JUnit results, lint log, trivy Docker scan
┅ Full pipeline example: build→test→gates→push→tag→release→notify
┅ semver_bump: major/minor/patch increment
┅ git_changelog: grouped by feat/fix/other เหมาะกับ PR body / CHANGELOG.md

---

**→ Part 81: Infrastructure as Code with Bash**
