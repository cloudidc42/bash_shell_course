# Part 27: Git Automation & GitHub API
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 27.1 Git Automation

```bash
# ─── Repository Setup ─────────────────────────────────────────
init_repo() {
    local dir=${1:-.}
    
    git init "$dir"
    cd "$dir"
    
    # Setup .gitignore
    cat > .gitignore << 'EOF'
# Dependencies
node_modules/
vendor/
.venv/

# Build output
dist/
build/
*.pyc
__pycache__/

# Environment
.env
.env.local
*.env

# Editor
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/
EOF

    # Initial commit
    git add .gitignore
    git commit -m "Initial commit: add .gitignore"
    
    echo "Repository initialized in $dir"
}

# ─── Branch Management ────────────────────────────────────────
# Create feature branch
new_feature() {
    local feature_name=$1
    local base_branch=${2:-main}
    
    git checkout "$base_branch"
    git pull origin "$base_branch"
    git checkout -b "feature/$feature_name"
    
    echo "Created: feature/$feature_name from $base_branch"
}

# Merge and cleanup
finish_feature() {
    local branch=$(git branch --show-current)
    local target=${1:-main}
    
    [[ "$branch" == "$target" ]] && echo "Already on $target" && return 1
    
    # Ensure clean
    git diff --exit-code &>/dev/null || die "Uncommitted changes"
    
    # Rebase onto target
    git fetch origin "$target"
    git rebase "origin/$target"
    
    # Merge
    git checkout "$target"
    git pull origin "$target"
    git merge --no-ff "$branch" -m "Merge $branch into $target"
    git push origin "$target"
    
    # Cleanup
    git branch -d "$branch"
    git push origin --delete "$branch"
    
    echo "Merged $branch into $target"
}

# ─── Commit Helpers ───────────────────────────────────────────
# Conventional commits
commit() {
    local type=$1
    local scope=${2:-}
    local message=$3
    
    local subject="${type}${scope:+(${scope})}: ${message}"
    
    git add -A
    git commit -m "$subject"
}

# Quick commit with status
qcommit() {
    local message=$1
    
    git status --short
    read -rp "Commit? [y/N] " confirm
    [[ "$confirm" =~ ^[Yy] ]] || return 0
    
    git add -A
    git commit -m "$message"
}

# Amend last commit (before push)
amend() {
    git add -A
    git commit --amend --no-edit
}

# ─── Status & Info ────────────────────────────────────────────
# Enhanced git status
gst() {
    echo "=== Branch: $(git branch --show-current) ==="
    git status --short
    echo ""
    
    # Show stash if any
    local stash_count
    stash_count=$(git stash list | wc -l)
    (( stash_count > 0 )) && echo "Stashes: $stash_count"
    
    # Show ahead/behind
    local upstream
    upstream=$(git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "")
    if [[ -n "$upstream" ]]; then
        local ahead behind
        ahead=$(git rev-list --count "${upstream}..HEAD")
        behind=$(git rev-list --count "HEAD..${upstream}")
        echo "Ahead: $ahead, Behind: $behind"
    fi
}

# Pretty log
glog() {
    git log \
        --oneline \
        --graph \
        --decorate \
        --color \
        "${@:---20}"
}

# Search commit history
gsearch() {
    local query=$1
    git log --all --grep="$query" --oneline
}

# Show changes between branches
gdiff() {
    local from=${1:-main}
    local to=${2:-HEAD}
    git diff "$from...$to" --stat
}
```

---

## 27.2 Git Hooks

```bash
# ─── Pre-commit Hook ──────────────────────────────────────────
cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/bash
set -e

echo "Running pre-commit checks..."

# Run linting
if command -v eslint &>/dev/null; then
    git diff --cached --name-only --diff-filter=ACM | \
        grep '\.js$' | xargs eslint --max-warnings=0 2>/dev/null || \
        { echo "ESLint failed"; exit 1; }
fi

# Run shellcheck on shell scripts
if command -v shellcheck &>/dev/null; then
    git diff --cached --name-only --diff-filter=ACM | \
        grep '\.sh$' | xargs -r shellcheck 2>/dev/null || \
        { echo "ShellCheck failed"; exit 1; }
fi

# Prevent committing secrets
if git diff --cached | grep -E '(api_key|secret_key|password)\s*=\s*["|\x27][^"\x27]+' 2>/dev/null; then
    echo "Possible secret in commit! Aborting."
    exit 1
fi

# Run tests
if [[ -f package.json ]]; then
    npm test --silent 2>/dev/null || { echo "Tests failed"; exit 1; }
fi

echo "Pre-commit checks passed!"
HOOK
chmod +x .git/hooks/pre-commit

# ─── Commit-msg Hook ──────────────────────────────────────────
cat > .git/hooks/commit-msg << 'HOOK'
#!/bin/bash

commit_regex='^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,72}$'
commit_msg=$(cat "$1")

if ! echo "$commit_msg" | grep -qE "$commit_regex"; then
    echo "Invalid commit message format!"
    echo "Expected: type(scope): description"
    echo "Types: feat, fix, docs, style, refactor, test, chore"
    echo "Example: feat(auth): add JWT authentication"
    exit 1
fi
HOOK
chmod +x .git/hooks/commit-msg

# ─── Pre-push Hook ────────────────────────────────────────────
cat > .git/hooks/pre-push << 'HOOK'
#!/bin/bash

# Prevent force push to main/master
protected_branches=(main master production)
current_branch=$(git branch --show-current)

for branch in "${protected_branches[@]}"; do
    if [[ "$current_branch" == "$branch" ]]; then
        while read -r local_ref local_sha remote_ref remote_sha; do
            if [[ "$local_sha" == "$(git rev-parse HEAD)" ]] && git log --oneline "$remote_sha..$local_sha" | grep -q "force"; then
                echo "Force push to $branch is not allowed!"
                exit 1
            fi
        done
    fi
done

# Run full test suite
if [[ -f package.json ]]; then
    npm run test:all || { echo "Tests failed, push aborted"; exit 1; }
fi
HOOK
chmod +x .git/hooks/pre-push

# Install hooks for entire team (using git config)
cat > install_hooks.sh << 'EOF'
#!/bin/bash
for hook in .githooks/*; do
    ln -sf "../../$hook" ".git/hooks/$(basename $hook)"
done
echo "Hooks installed"
EOF
git config core.hooksPath .githooks
```

---

## 27.3 GitHub API Integration

```bash
#!/bin/bash
# github_api.sh - GitHub API client

GITHUB_TOKEN="${GITHUB_TOKEN:?GITHUB_TOKEN required}"
GITHUB_API="https://api.github.com"
REPO="${GITHUB_REPO:?GITHUB_REPO required}"  # format: owner/repo

gh_api() {
    local method=${1:-GET}
    local endpoint=$2
    local data=${3:-}
    
    local args=(
        -s -f
        -X "$method"
        -H "Authorization: Bearer $GITHUB_TOKEN"
        -H "Accept: application/vnd.github.v3+json"
        -H "Content-Type: application/json"
        -H "X-GitHub-Api-Version: 2022-11-28"
    )
    
    [[ -n "$data" ]] && args+=(-d "$data")
    
    curl "${args[@]}" "${GITHUB_API}${endpoint}"
}

# ─── Repositories ─────────────────────────────────────────────
list_repos() {
    local org=$1
    gh_api GET "/orgs/$org/repos?per_page=100&type=all" | \
        jq '.[] | {name, private, language, pushed_at}' | \
        jq -r '"\(.pushed_at) \(.name) (\(.language // "N/A")) \(if .private then "[private]" else "" end)"' | \
        sort -r
}

get_repo_info() {
    gh_api GET "/repos/$REPO" | \
        jq '{name, description, default_branch, stars: .stargazers_count, forks: .forks_count}'
}

# ─── Issues ───────────────────────────────────────────────────
create_issue() {
    local title=$1
    local body=${2:-}
    local labels=${3:-}
    
    local data
    data=$(jq -n \
        --arg title "$title" \
        --arg body "$body" \
        --argjson labels "$(echo "$labels" | jq -R 'split(",")')" \
        '{title: $title, body: $body, labels: $labels}')
    
    gh_api POST "/repos/$REPO/issues" "$data" | jq '{number, html_url}'
}

list_issues() {
    local state=${1:-open}
    gh_api GET "/repos/$REPO/issues?state=$state&per_page=50" | \
        jq -r '.[] | "#\(.number) \(.title) [\(.state)]"'
}

close_issue() {
    local number=$1
    gh_api PATCH "/repos/$REPO/issues/$number" '{"state":"closed"}' | \
        jq '{number, state}'
}

# ─── Pull Requests ────────────────────────────────────────────
create_pr() {
    local title=$1
    local body=$2
    local head=${3:-$(git branch --show-current)}
    local base=${4:-main}
    
    local data
    data=$(jq -n \
        --arg title "$title" \
        --arg body "$body" \
        --arg head "$head" \
        --arg base "$base" \
        '{title: $title, body: $body, head: $head, base: $base}')
    
    gh_api POST "/repos/$REPO/pulls" "$data" | jq '{number, html_url}'
}

list_prs() {
    local state=${1:-open}
    gh_api GET "/repos/$REPO/pulls?state=$state" | \
        jq -r '.[] | "#\(.number) \(.title) [\(.head.ref) → \(.base.ref)]"'
}

merge_pr() {
    local pr_number=$1
    local method=${2:-squash}
    
    gh_api PUT "/repos/$REPO/pulls/$pr_number/merge" \
        "{\"merge_method\":\"$method\"}" | \
        jq '{merged, message}'
}

# ─── Releases ─────────────────────────────────────────────────
create_release() {
    local tag=$1
    local name=${2:-$tag}
    local body=${3:-}
    local prerelease=${4:-false}
    
    local data
    data=$(jq -n \
        --arg tag_name "$tag" \
        --arg name "$name" \
        --arg body "$body" \
        --argjson prerelease "$prerelease" \
        '{tag_name: $tag_name, name: $name, body: $body, prerelease: $prerelease}')
    
    gh_api POST "/repos/$REPO/releases" "$data" | jq '{id, tag_name, html_url}'
}

upload_release_asset() {
    local release_id=$1
    local file=$2
    
    local filename
    filename=$(basename "$file")
    local mime_type
    mime_type=$(file -b --mime-type "$file")
    
    curl -s \
        -H "Authorization: Bearer $GITHUB_TOKEN" \
        -H "Content-Type: $mime_type" \
        --data-binary "@$file" \
        "https://uploads.github.com/repos/$REPO/releases/$release_id/assets?name=$filename" | \
        jq '{name, browser_download_url}'
}

# ─── GitHub Actions ───────────────────────────────────────────
trigger_workflow() {
    local workflow=$1
    local ref=${2:-main}
    local inputs=${3:-{}}
    
    local data
    data=$(jq -n \
        --arg ref "$ref" \
        --argjson inputs "$inputs" \
        '{ref: $ref, inputs: $inputs}')
    
    gh_api POST "/repos/$REPO/actions/workflows/$workflow/dispatches" "$data"
    echo "Workflow triggered: $workflow"
}

list_workflow_runs() {
    local workflow=${1:-}
    local endpoint="/repos/$REPO/actions/runs"
    [[ -n "$workflow" ]] && endpoint="/repos/$REPO/actions/workflows/$workflow/runs"
    
    gh_api GET "$endpoint?per_page=10" | \
        jq -r '.workflow_runs[] | "\(.id) \(.status) \(.conclusion // "running") \(.head_commit.message | split("\n")[0])"'
}

# ─── Complete release workflow ────────────────────────────────
release_workflow() {
    local version=$1
    
    echo "Releasing v$version..."
    
    # Ensure clean
    git diff --exit-code || die "Uncommitted changes"
    
    # Update version in package.json
    npm version "$version" --no-git-tag-version
    
    # Commit
    git add package.json package-lock.json
    git commit -m "chore(release): v$version"
    
    # Tag
    git tag -a "v$version" -m "Release v$version"
    
    # Push
    git push origin main
    git push origin "v$version"
    
    # Generate changelog
    local changelog
    changelog=$(git log "$(git describe --tags --abbrev=0 HEAD^)..HEAD" \
        --oneline --no-merges 2>/dev/null || echo "Initial release")
    
    # Create GitHub release
    create_release "v$version" "v$version" "$changelog"
    
    echo "Released v$version!"
}
```

---

## 27.4 Git Statistics

```bash
#!/bin/bash
# git_stats.sh - Repository statistics

# Commits per author
commits_by_author() {
    git shortlog -sn --all | head -10
}

# Activity by time
activity_by_hour() {
    git log --format="%ad" --date=format:"%H" | sort | uniq -c
}

activity_by_day() {
    git log --format="%ad" --date=format:"%A" | sort | uniq -c
}

# Files changed most
hottest_files() {
    git log --format="" --name-only | \
        grep -v '^$' | \
        sort | uniq -c | \
        sort -rn | head -20
}

# Largest files
largest_files() {
    git ls-tree -r -l HEAD | \
        awk '{print $4, $5}' | \
        sort -rn | head -10 | \
        awk '{printf "%-10s %s\n", $1, $2}'
}

# Lines of code
lines_of_code() {
    git ls-files | \
        grep -E '\.(sh|bash|py|js|ts|go|rs|java|c|cpp)$' | \
        xargs wc -l 2>/dev/null | \
        sort -rn | head -20
}

# Contribution over time
contributions_over_time() {
    git log --format="%ad %an" --date=format:"%Y-%m" | \
        awk '{count[$2" "$3]++} END{for(k in count) print k, count[k]}' | \
        sort
}

echo "=== Repository Statistics ==="
echo ""
echo "── Commits by Author ──"
commits_by_author
echo ""
echo "── Hottest Files (most changed) ──"
hottest_files
echo ""
echo "── Commit Activity by Hour ──"
activity_by_hour
```

---

## 27.5 Exercises

### Exercise 1: Git Flow Automator
สร้าง tool ที่ implement git-flow:
- feature start/finish
- hotfix start/finish
- release start/finish
- Automatic version bumping

### Exercise 2: PR Monitor
สร้าง script ที่:
- Watch for new PRs
- Auto-assign reviewers
- Check CI status
- Merge when approved

### Exercise 3: Changelog Generator
สร้าง changelog จาก git history:
- Group by type (feat, fix, etc.)
- Link to issues/PRs
- Format as Markdown
- Auto-update CHANGELOG.md

---

## สรุป Part 27

✅ Git repository automation  
✅ Branch management (feature, merge, cleanup)  
✅ Git hooks (pre-commit, commit-msg, pre-push)  
✅ GitHub API (issues, PRs, releases, workflows)  
✅ Automated release workflow  
✅ Repository statistics  

---

**→ Part 28: Log Analysis & Monitoring**
