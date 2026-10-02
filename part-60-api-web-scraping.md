# Part 60: API Development and Web Scraping
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 60.1 HTTP Client Framework

```bash
#!/bin/bash
# http_client.sh - Full-featured HTTP client

HTTP_TIMEOUT="${HTTP_TIMEOUT:-30}"
HTTP_RETRIES="${HTTP_RETRIES:-3}"
HTTP_USER_AGENT="${HTTP_USER_AGENT:-bash-http-client/1.0}"

# ─── Base Request ──────────────────────────────────────────────
http_request() {
    local method=$1 url=$2
    shift 2

    local -A opts=(
        [content_type]="application/json"
        [timeout]="$HTTP_TIMEOUT"
        [retries]="$HTTP_RETRIES"
    )

    local headers=()
    local data=""
    local output_file=""

    while [[ $# -gt 0 ]]; do
        case "$1" in
            --header=*)    headers+=("${1#--header=}") ;;
            --data=*)      data="${1#--data=}" ;;
            --output=*)    output_file="${1#--output=}" ;;
            --timeout=*)   opts[timeout]="${1#--timeout=}" ;;
            --content-type=*) opts[content_type]="${1#--content-type=}" ;;
            --bearer=*)    headers+=("Authorization: Bearer ${1#--bearer=}") ;;
            --basic=*)     headers+=("Authorization: Basic $(echo -n "${1#--basic=}" | base64)") ;;
        esac
        shift
    done

    local curl_args=(
        --silent
        --show-error
        --location
        --max-time "${opts[timeout]}"
        --user-agent "$HTTP_USER_AGENT"
        --write-out '\n__STATUS__%{http_code}__TOTAL__%{time_total}__'
        --request "$method"
    )

    for header in "${headers[@]}"; do
        curl_args+=(--header "$header")
    done

    [[ -n "$data" ]] && curl_args+=(
        --header "Content-Type: ${opts[content_type]}"
        --data "$data"
    )

    [[ -n "$output_file" ]] && curl_args+=(--output "$output_file")

    local attempt=1 response exit_code
    while (( attempt <= opts[retries] )); do
        response=$(curl "${curl_args[@]}" "$url" 2>&1)
        exit_code=$?

        if (( exit_code == 0 )); then
            break
        fi

        (( attempt++ ))
        sleep $(( attempt * 2 ))
    done

    # Parse status and timing from write-out
    local body status timing
    if [[ "$response" =~ (.*)$'\n'__STATUS__([0-9]+)__TOTAL__([0-9.]+)__ ]]; then
        body="${BASH_REMATCH[1]}"
        status="${BASH_REMATCH[2]}"
        timing="${BASH_REMATCH[3]}"
    else
        body="$response"
        status=0
        timing=0
    fi

    echo "$body"
    return $(( status >= 400 ))
}

http_get()    { http_request GET    "$@"; }
http_post()   { http_request POST   "$@"; }
http_put()    { http_request PUT    "$@"; }
http_delete() { http_request DELETE "$@"; }
http_patch()  { http_request PATCH  "$@"; }

# ─── Response Helpers ────────────────────────────────────────────
http_get_status() {
    local url=$1
    curl -s -o /dev/null -w "%{http_code}" --max-time "$HTTP_TIMEOUT" "$url" 2>/dev/null
}

http_download() {
    local url=$1 dest=$2

    local dir; dir=$(dirname "$dest")
    mkdir -p "$dir"

    curl --silent --location --progress-bar \
        --max-time "$HTTP_TIMEOUT" \
        --output "$dest" \
        "$url"

    echo "Downloaded: $dest ($(du -sh "$dest" | cut -f1))"
}

http_download_parallel() {
    local -n urls_ref=$1
    local dest_dir=$2
    local max_jobs=${3:-5}

    mkdir -p "$dest_dir"

    local job_count=0
    for url in "${urls_ref[@]}"; do
        local filename; filename=$(basename "$url")
        http_download "$url" "$dest_dir/$filename" &

        (( job_count++ ))
        if (( job_count >= max_jobs )); then
            wait
            job_count=0
        fi
    done
    wait
}
```

---

## 60.2 REST API Client

```bash
#!/bin/bash
# rest_client.sh - REST API client framework

# ─── API Client Factory ────────────────────────────────────────────
api_client_create() {
    local name=$1 base_url=$2 auth_type=${3:-none} auth_value=${4:-}

    declare -gA "API_CLIENT_${name}=(
        [base_url]='$base_url'
        [auth_type]='$auth_type'
        [auth_value]='$auth_value'
    )"
}

api_call() {
    local client_name=$1 method=$2 path=$3
    shift 3

    local base_url_var="API_CLIENT_${client_name}[base_url]"
    local auth_type_var="API_CLIENT_${client_name}[auth_type]"
    local auth_value_var="API_CLIENT_${client_name}[auth_value]"

    local base_url="${!base_url_var}"
    local auth_type="${!auth_type_var}"
    local auth_value="${!auth_value_var}"
    local url="${base_url}${path}"

    local extra_args=()

    case "$auth_type" in
        bearer) extra_args+=("--bearer=$auth_value") ;;
        basic)  extra_args+=("--basic=$auth_value") ;;
        apikey) extra_args+=("--header=X-API-Key: $auth_value") ;;
    esac

    http_request "$method" "$url" "${extra_args[@]}" "$@"
}

# ─── Pagination Handler ────────────────────────────────────────────
api_paginate() {
    local client_name=$1 path=$2
    local page=1 per_page=${3:-100}
    local results_key=${4:-results}
    local all_results="[]"

    while true; do
        local response
        response=$(api_call "$client_name" GET "${path}?page=${page}&per_page=${per_page}")

        local items count
        items=$(echo "$response" | jq -c ".${results_key}" 2>/dev/null)
        count=$(echo "$items" | jq 'length' 2>/dev/null || echo 0)

        (( count == 0 )) && break

        all_results=$(echo "$all_results $items" | jq -s 'add')

        (( count < per_page )) && break
        (( page++ ))
    done

    echo "$all_results"
}

# ─── GitHub API ──────────────────────────────────────────────────
github_api_setup() {
    local token=${1:-$GITHUB_TOKEN}
    api_client_create "github" "https://api.github.com" "bearer" "$token"
}

github_list_repos() {
    local org=$1
    api_call "github" GET "/orgs/${org}/repos" | \
        jq -r '.[] | "\(.name)\t\(.language)\t\(.stargazers_count) stars"'
}

github_create_issue() {
    local repo=$1 title=$2 body=$3 labels=${4:-}

    local payload
    payload=$(jq -n \
        --arg title "$title" \
        --arg body "$body" \
        '{"title": $title, "body": $body}')

    if [[ -n "$labels" ]]; then
        local labels_json
        labels_json=$(echo "$labels" | tr ',' '\n' | jq -R . | jq -s .)
        payload=$(echo "$payload" | jq --argjson labels "$labels_json" '. + {labels: $labels}')
    fi

    api_call "github" POST "/repos/${repo}/issues" --data="$payload"
}

github_get_workflow_runs() {
    local repo=$1 workflow=${2:-} status=${3:-}

    local path="/repos/${repo}/actions/runs"
    local query_params="per_page=10"
    [[ -n "$workflow" ]] && query_params+="&workflow_id=$workflow"
    [[ -n "$status" ]] && query_params+="&status=$status"

    api_call "github" GET "${path}?${query_params}" | \
        jq -r '.workflow_runs[] | "\(.id)\t\(.status)\t\(.conclusion)\t\(.created_at)\t\(.name)"'
}
```

---

## 60.3 Web Scraping

```bash
#!/bin/bash
# web_scraper.sh - HTML parsing and data extraction

# ─── HTML Parsing ────────────────────────────────────────────────
html_extract_tag() {
    local html=$1 tag=$2 attribute=${3:-}

    if [[ -n "$attribute" ]]; then
        echo "$html" | grep -oP "<${tag}[^>]*\s${attribute}=['\"]?\K[^'\"\>\s]+" | head -20
    else
        echo "$html" | sed -n "s/.*<${tag}[^>]*>\(.*\)<\/${tag}>.*/\1/p" | \
            sed 's/<[^>]*>//g' | grep -v '^[[:space:]]*$'
    fi
}

html_extract_links() {
    local html=$1 base_url=${2:-}

    echo "$html" | grep -oP 'href=["\x27]\K[^"\x27]+' | sort -u | while read -r link; do
        if [[ "$link" =~ ^https?:// ]]; then
            echo "$link"
        elif [[ "$link" =~ ^/ ]]; then
            echo "${base_url}${link}"
        elif [[ -n "$base_url" ]]; then
            echo "${base_url}/${link}"
        else
            echo "$link"
        fi
    done
}

html_extract_table() {
    local html=$1 table_index=${2:-0}

    echo "$html" | awk '
    BEGIN { in_table=0; row=0; table_count=0 }
    /<table[^>]*>/ { if (table_count++ == target_table) in_table=1 }
    /<\/table>/ { if (in_table) { in_table=0; exit } }
    in_table && /<tr[^>]*>/ { row++; col=0 }
    in_table && /<t[dh][^>]*>/ {
        col++
        match($0, /<t[dh][^>]*>(.*)<\/t[dh]>/, arr)
        gsub(/<[^>]+>/, "", arr[1])
        printf "%s\t", arr[1]
    }
    in_table && /<\/tr>/ { print "" }
    ' target_table="$table_index"
}

html_strip_tags() {
    local html=$1
    echo "$html" | sed 's/<[^>]*>//g' | \
        sed "s/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g; s/&quot;/\"/g; s/&#39;/'/g" | \
        grep -v '^[[:space:]]*$'
}

html_extract_json_ld() {
    local html=$1

    echo "$html" | grep -oP '(?<=<script type="application/ld\+json">).*?(?=</script>)' | \
        while IFS= read -r json; do
            echo "$json" | jq . 2>/dev/null
        done
}

# ─── Page Crawler ──────────────────────────────────────────────────
declare -A CRAWLED_URLS=()

crawl_page() {
    local url=$1 depth=${2:-0} max_depth=${3:-2}

    [[ "${CRAWLED_URLS[$url]:-}" ]] && return
    CRAWLED_URLS["$url"]=1

    local html
    html=$(curl -s -L --max-time 10 --user-agent "Mozilla/5.0" "$url" 2>/dev/null)
    [[ -z "$html" ]] && return

    echo "$url"

    (( depth < max_depth )) || return

    local base_url="${url%/*}"
    html_extract_links "$html" "$base_url" | while read -r link; do
        if [[ "$link" == *"${url%%//*//*/}"* ]] || [[ "$link" =~ ^/ ]]; then
            crawl_page "$link" $(( depth + 1 )) "$max_depth"
        fi
    done
}

# ─── Structured Data Extraction ──────────────────────────────────────────
scrape_structured() {
    local url=$1
    shift
    local -A selectors=()

    while [[ $# -gt 0 ]]; do
        selectors["${1%%=*}"]="${1#*=}"
        shift
    done

    local html
    html=$(curl -s -L --max-time 15 "$url" 2>/dev/null)

    local result="{"
    local first=true

    for field in "${!selectors[@]}"; do
        $first || result+=","
        first=false

        local selector="${selectors[$field]}"
        local value=""

        local tag="${selector%%[.#@]*}"
        value=$(html_extract_tag "$html" "$tag" | head -1 | xargs)
        value="${value//\"/\\\"}"
        result+="\"${field}\": \"${value}\""
    done

    result+="}"
    echo "$result" | jq . 2>/dev/null || echo "$result"
}
```

---

## 60.4 Rate Limiting and Politeness

```bash
#!/bin/bash
# rate_limiter.sh - Request rate limiting for scrapers

declare -A RATE_LIMIT_COUNTERS=()
declare -A RATE_LIMIT_WINDOWS=()

# ─── Rate Limiter ──────────────────────────────────────────────────
rate_limit_init() {
    local name=$1 requests_per_second=${2:-1}

    RATE_LIMIT_COUNTERS["$name"]=0
    RATE_LIMIT_WINDOWS["$name"]=$(date +%s%N)

    declare -g "RATE_LIMIT_${name}_RPS=$requests_per_second"
}

rate_limit_wait() {
    local name=$1

    local rps_var="RATE_LIMIT_${name}_RPS"
    local rps="${!rps_var:-1}"

    local now_ns; now_ns=$(date +%s%N)
    local window_start="${RATE_LIMIT_WINDOWS[$name]}"
    local count="${RATE_LIMIT_COUNTERS[$name]}"

    local elapsed_ms=$(( (now_ns - window_start) / 1000000 ))

    if (( elapsed_ms >= 1000 )); then
        RATE_LIMIT_COUNTERS["$name"]=0
        RATE_LIMIT_WINDOWS["$name"]=$now_ns
        return 0
    fi

    if (( count >= rps )); then
        local wait_ms=$(( 1000 - elapsed_ms ))
        sleep "0.${wait_ms}"
        RATE_LIMIT_COUNTERS["$name"]=0
        RATE_LIMIT_WINDOWS["$name"]=$(date +%s%N)
        return 0
    fi

    (( RATE_LIMIT_COUNTERS["$name"]++ ))
}

polite_fetch() {
    local url=$1 rate_name=${2:-default}

    rate_limit_wait "$rate_name"

    local jitter=$(( RANDOM % 500 ))
    sleep "0.${jitter}"

    curl -s -L \
        --max-time 15 \
        --user-agent "Mozilla/5.0 (compatible; MyBot/1.0)" \
        --header "Accept: text/html,application/xhtml+xml" \
        --header "Accept-Language: en-US,en;q=0.9" \
        --compressed \
        "$url"
}

check_robots_txt() {
    local base_url=$1 path=${2:-/}

    local robots
    robots=$(curl -s --max-time 5 "${base_url}/robots.txt" 2>/dev/null)

    local is_allowed=true

    echo "$robots" | awk -v path="$path" '
    /^User-agent: \*/ { check=1; next }
    /^User-agent:/ && !/\*/ { check=0 }
    check && /^Disallow:/ {
        disallow = $2
        if (index(path, disallow) == 1) {
            print "DISALLOWED"
            exit
        }
    }
    ' | grep -q "DISALLOWED" && is_allowed=false

    $is_allowed && echo "allowed" || echo "disallowed"
}
```

---

## 60.5 API Testing

```bash
#!/bin/bash
# api_test.sh - REST API testing framework

declare -A TEST_RESULTS=()
declare -i TEST_PASS=0 TEST_FAIL=0

# ─── Test Framework ────────────────────────────────────────────────
api_test() {
    local name=$1 method=$2 url=$3
    shift 3

    local expected_status=${EXPECTED_STATUS:-200}
    local expected_body=${EXPECTED_BODY:-}
    local timeout=${TEST_TIMEOUT:-10}

    local response http_status
    response=$(curl -s -o /tmp/api_test_body \
        -w "%{http_code}" \
        --max-time "$timeout" \
        --request "$method" \
        "$@" \
        "$url" 2>/dev/null)

    http_status="$response"
    local body; body=$(cat /tmp/api_test_body 2>/dev/null)

    local passed=true
    local failures=()

    if [[ "$http_status" != "$expected_status" ]]; then
        passed=false
        failures+=("Expected status $expected_status, got $http_status")
    fi

    if [[ -n "$expected_body" ]]; then
        if ! echo "$body" | grep -q "$expected_body"; then
            passed=false
            failures+=("Body does not contain: $expected_body")
        fi
    fi

    if $passed; then
        printf "  [PASS] %s\n" "$name"
        (( TEST_PASS++ ))
        TEST_RESULTS["$name"]="PASS"
    else
        printf "  [FAIL] %s\n" "$name"
        for f in "${failures[@]}"; do
            printf "         %s\n" "$f"
        done
        (( TEST_FAIL++ ))
        TEST_RESULTS["$name"]="FAIL"
    fi
}

api_test_suite_summary() {
    local total=$(( TEST_PASS + TEST_FAIL ))
    echo ""
    echo "=== Test Summary ==="
    printf "  Passed: %d/%d\n" "$TEST_PASS" "$total"
    printf "  Failed: %d/%d\n" "$TEST_FAIL" "$total"

    if (( TEST_FAIL > 0 )); then
        echo "  Failed tests:"
        for name in "${!TEST_RESULTS[@]}"; do
            [[ "${TEST_RESULTS[$name]}" == "FAIL" ]] && printf "    - %s\n" "$name"
        done
    fi

    return $TEST_FAIL
}
```

---

## 60.6 Exercises

### Exercise 1: REST API Wrapper
สร้าง generic REST API wrapper ที่:
- Support all HTTP methods
- Handle OAuth2 token refresh
- Auto-retry on 5xx errors
- Request/response logging

### Exercise 2: News Aggregator
สร้าง scraper ที่:
- Fetch RSS/Atom feeds
- Parse HTML news pages
- Deduplicate articles
- Export to JSON/CSV

### Exercise 3: API Health Testing Suite
สร้าง test suite ที่:
- Test all API endpoints
- Validate response schemas
- Performance benchmarks
- CI/CD integration

---

## สรุป Part 60

✅ HTTP client with retry, auth (bearer/basic/apikey), write-out parsing
✅ REST API client factory with named clients
✅ API pagination with jq-based result accumulation
✅ GitHub API: list repos, create issues, workflow run status
✅ HTML parsing: tag extraction, link extraction, table parsing
✅ Web crawler with visited URL tracking and depth limit
✅ Rate limiter with RPS control and random jitter
✅ Robots.txt compliance checker
✅ API testing framework with status and body assertions

---

**→ Part 61: Performance Optimization and Profiling**
