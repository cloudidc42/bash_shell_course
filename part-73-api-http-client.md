# Part 73: API Integration and HTTP Client Scripting
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 73.1 curl Wrapper Library

```bash
#!/bin/bash
# http_client.sh - Production-grade HTTP client

set -euo pipefail

HTTP_BASE_URL="${HTTP_BASE_URL:-}"
HTTP_TIMEOUT="${HTTP_TIMEOUT:-30}"
HTTP_CONNECT_TIMEOUT="${HTTP_CONNECT_TIMEOUT:-10}"
HTTP_RETRY="${HTTP_RETRY:-3}"
HTTP_VERBOSE="${HTTP_VERBOSE:-false}"
HTTP_CA_BUNDLE="${HTTP_CA_BUNDLE:-}"

# ─── Core request function ────────────────────────────────────────────────────────
http_request() {
    local method=$1 url=$2
    shift 2

    local -a curl_args=(
        --silent
        --show-error
        --location
        --max-time "$HTTP_TIMEOUT"
        --connect-timeout "$HTTP_CONNECT_TIMEOUT"
        --write-out '\n__STATUS__%{http_code}__HEADERS__%{header_json}'
        --request "$method"
    )

    [[ -n "$HTTP_CA_BUNDLE" ]] && curl_args+=(--cacert "$HTTP_CA_BUNDLE")
    $HTTP_VERBOSE && curl_args+=(--verbose)

    # Process extra args
    local body=""
    local -a headers=()
    while [[ $# -gt 0 ]]; do
        case $1 in
            --header) headers+=("$2"); shift 2 ;;
            --data)   body="$2"; shift 2 ;;
            *) curl_args+=("$1"); shift ;;
        esac
    done

    for h in "${headers[@]}"; do
        curl_args+=(--header "$h")
    done

    [[ -n "$body" ]] && curl_args+=(--data "$body")

    local response; response=$(curl "${curl_args[@]}" "$url" 2>&1)
    local exit_code=$?

    if (( exit_code != 0 )); then
        echo "HTTP_ERROR: curl failed (exit $exit_code): ${response}" >&2
        return "$exit_code"
    fi

    local body_part status_code
    body_part=$(echo "$response" | sed 's/__STATUS__[0-9]*__HEADERS__.*$//')
    status_code=$(echo "$response" | grep -oP '__STATUS__\K[0-9]+')

    HTTP_LAST_STATUS="$status_code"
    HTTP_LAST_BODY="$body_part"

    echo "$body_part"
    return 0
}

# ─── Convenience methods ─────────────────────────────────────────────────────────
http_get() {
    local path=$1; shift
    http_request GET "${HTTP_BASE_URL}${path}" "$@"
}

http_post() {
    local path=$1 body=$2; shift 2
    http_request POST "${HTTP_BASE_URL}${path}" \
        --header 'Content-Type: application/json' \
        --data "$body" "$@"
}

http_put() {
    local path=$1 body=$2; shift 2
    http_request PUT "${HTTP_BASE_URL}${path}" \
        --header 'Content-Type: application/json' \
        --data "$body" "$@"
}

http_patch() {
    local path=$1 body=$2; shift 2
    http_request PATCH "${HTTP_BASE_URL}${path}" \
        --header 'Content-Type: application/json' \
        --data "$body" "$@"
}

http_delete() {
    local path=$1; shift
    http_request DELETE "${HTTP_BASE_URL}${path}" "$@"
}

http_head() {
    local url=$1
    curl --silent --head --max-time "$HTTP_TIMEOUT" "$url" 2>/dev/null
}
```

---

## 73.2 Authentication Strategies

```bash
#!/bin/bash
# auth.sh - HTTP authentication handlers

auth_bearer() {
    local token=$1
    echo "Authorization: Bearer $token"
}

auth_basic() {
    local user=$1 password=$2
    local encoded; encoded=$(echo -n "${user}:${password}" | base64 -w0)
    echo "Authorization: Basic $encoded"
}

auth_api_key() {
    local key_name=${1:-X-API-Key} key_value=$2
    echo "${key_name}: ${key_value}"
}

# OAuth2 Client Credentials
oauth2_token() {
    local token_url=$1 client_id=$2 client_secret=$3
    local scope=${4:-}

    local data="grant_type=client_credentials&client_id=${client_id}&client_secret=${client_secret}"
    [[ -n "$scope" ]] && data+="&scope=${scope}"

    local response; response=$(curl --silent \
        --request POST "$token_url" \
        --header 'Content-Type: application/x-www-form-urlencoded' \
        --data "$data" 2>/dev/null)

    local token; token=$(echo "$response" | jq -r '.access_token // empty')
    local expires_in; expires_in=$(echo "$response" | jq -r '.expires_in // 3600')

    if [[ -z "$token" ]]; then
        echo "OAuth2 error: $(echo "$response" | jq -r '.error // "unknown"')" >&2
        return 1
    fi

    OAUTH2_TOKEN="$token"
    OAUTH2_EXPIRES_AT=$(( $(date +%s) + expires_in - 60 ))
    echo "$token"
}

oauth2_token_valid() {
    [[ -n "${OAUTH2_TOKEN:-}" ]] && (( $(date +%s) < ${OAUTH2_EXPIRES_AT:-0} ))
}

oauth2_get_or_refresh() {
    local token_url=$1 client_id=$2 client_secret=$3 scope=${4:-}

    if oauth2_token_valid; then
        echo "$OAUTH2_TOKEN"
    else
        oauth2_token "$token_url" "$client_id" "$client_secret" "$scope"
    fi
}
```

---

## 73.3 Retry with Exponential Backoff

```bash
#!/bin/bash
# retry.sh - Retry logic with backoff and jitter

retry_with_backoff() {
    local max_attempts=${1:-5}
    local base_delay=${2:-1}
    local max_delay=${3:-60}
    shift 3
    local cmd=("$@")

    local attempt=1
    local delay=$base_delay

    while (( attempt <= max_attempts )); do
        if "${cmd[@]}"; then
            return 0
        fi

        if (( attempt == max_attempts )); then
            echo "All $max_attempts attempts failed: ${cmd[*]}" >&2
            return 1
        fi

        # Add jitter (0-50% of delay)
        local jitter=$(( RANDOM % (delay / 2 + 1) ))
        local sleep_time=$(( delay + jitter ))
        (( sleep_time > max_delay )) && sleep_time=$max_delay

        echo "Attempt $attempt failed, retrying in ${sleep_time}s..." >&2
        sleep "$sleep_time"

        delay=$(( delay * 2 ))
        (( delay > max_delay )) && delay=$max_delay
        (( attempt++ ))
    done
}

retry_http() {
    local url=$1
    local max_attempts=${2:-3}
    local acceptable_codes=${3:-200,201,204}

    local attempt=1 delay=1

    while (( attempt <= max_attempts )); do
        local response; response=$(curl --silent --write-out '\n%{http_code}' \
            --max-time 30 "$url" 2>/dev/null)
        local http_code; http_code=$(echo "$response" | tail -1)
        local body; body=$(echo "$response" | head -n -1)

        if echo "$acceptable_codes" | grep -qw "$http_code"; then
            echo "$body"
            return 0
        fi

        if (( http_code >= 400 && http_code < 500 )); then
            echo "Client error $http_code, not retrying" >&2
            return 1
        fi

        echo "HTTP $http_code, retry $attempt/${max_attempts}" >&2
        sleep $(( delay + RANDOM % delay + 1 ))
        delay=$(( delay * 2 ))
        (( attempt++ ))
    done

    return 1
}
```

---

## 73.4 Response Caching

```bash
#!/bin/bash
# http_cache.sh - File-based HTTP response cache

HTTP_CACHE_DIR="${HTTP_CACHE_DIR:-/tmp/http_cache}"
HTTP_CACHE_TTL="${HTTP_CACHE_TTL:-300}"

mkdir -p "$HTTP_CACHE_DIR"

cache_key() {
    local url=$1
    echo -n "$url" | sha256sum | cut -d' ' -f1
}

cache_get() {
    local url=$1
    local key; key=$(cache_key "$url")
    local cache_file="${HTTP_CACHE_DIR}/${key}"
    local meta_file="${cache_file}.meta"

    [[ -f "$cache_file" && -f "$meta_file" ]] || return 1

    local cached_at; cached_at=$(cat "$meta_file" 2>/dev/null)
    local age=$(( $(date +%s) - cached_at ))

    if (( age < HTTP_CACHE_TTL )); then
        cat "$cache_file"
        return 0
    fi

    rm -f "$cache_file" "$meta_file"
    return 1
}

cache_set() {
    local url=$1 body=$2
    local key; key=$(cache_key "$url")
    local cache_file="${HTTP_CACHE_DIR}/${key}"
    local meta_file="${cache_file}.meta"

    echo -n "$body" > "$cache_file"
    date +%s > "$meta_file"
}

cache_invalidate() {
    local url=$1
    local key; key=$(cache_key "$url")
    rm -f "${HTTP_CACHE_DIR}/${key}" "${HTTP_CACHE_DIR}/${key}.meta"
}

cache_clear_expired() {
    local now; now=$(date +%s)
    for meta_file in "$HTTP_CACHE_DIR"/*.meta; do
        [[ -f "$meta_file" ]] || continue
        local cached_at; cached_at=$(cat "$meta_file")
        local age=$(( now - cached_at ))
        if (( age >= HTTP_CACHE_TTL )); then
            local base="${meta_file%.meta}"
            rm -f "$base" "$meta_file"
        fi
    done
}

cached_http_get() {
    local url=$1

    if cache_get "$url"; then
        return 0
    fi

    local response; response=$(curl --silent --fail --max-time 30 "$url" 2>/dev/null) || return 1
    cache_set "$url" "$response"
    echo "$response"
}
```

---

## 73.5 Pagination Handling

```bash
#!/bin/bash
# pagination.sh - Handle paginated API responses

fetch_all_pages_link_header() {
    local url=$1
    local -a results=()
    local current_url="$url"
    local page=1

    while [[ -n "$current_url" ]]; do
        echo "Fetching page $page: $current_url" >&2

        local response headers body
        response=$(curl --silent --dump-header /tmp/http_headers \
            --max-time 30 "$current_url" 2>/dev/null)
        headers=$(cat /tmp/http_headers 2>/dev/null)

        results+=("$response")

        # Parse Link header: <url>; rel="next"
        local next_url
        next_url=$(echo "$headers" | grep -i '^Link:' | \
            grep -oP '<([^>]+)>;\s*rel="next"' | \
            grep -oP '(?<=<)[^>]+')

        current_url="$next_url"
        (( page++ ))
    done

    # Merge JSON arrays
    printf '%s\n' "${results[@]}" | jq -s '[.[] | if type=="array" then .[] else . end]' 2>/dev/null
}

fetch_all_pages_cursor() {
    local url_template=$1 cursor_field=${2:-cursor} items_field=${3:-items}
    local cursor=""
    local -a all_items=()
    local page=1

    while true; do
        local url="$url_template"
        [[ -n "$cursor" ]] && url+="&${cursor_field}=${cursor}"

        echo "Fetching page $page" >&2
        local response; response=$(curl --silent --fail --max-time 30 "$url" 2>/dev/null) || break

        local items; items=$(echo "$response" | jq -c ".${items_field}[]" 2>/dev/null)
        [[ -z "$items" ]] && break
        all_items+=("$items")

        cursor=$(echo "$response" | jq -r ".next_${cursor_field} // empty" 2>/dev/null)
        [[ -z "$cursor" || "$cursor" == "null" ]] && break

        (( page++ ))
    done

    printf '%s\n' "${all_items[@]}" | jq -s .
}

fetch_all_pages_offset() {
    local url_template=$1 page_size=${2:-100}
    local offset=0
    local -a all_items=()

    while true; do
        local url="${url_template}&limit=${page_size}&offset=${offset}"
        local response; response=$(curl --silent --fail --max-time 30 "$url" 2>/dev/null) || break

        local count; count=$(echo "$response" | jq 'if type=="array" then length else .items|length end' 2>/dev/null || echo 0)
        (( count == 0 )) && break

        all_items+=("$response")
        offset=$(( offset + page_size ))
        (( count < page_size )) && break
    done

    printf '%s\n' "${all_items[@]}" | jq -s '[.[] | if type=="array" then .[] else .items[] end]' 2>/dev/null
}
```

---

## 73.6 GraphQL Client

```bash
#!/bin/bash
# graphql.sh - GraphQL client

gql_query() {
    local endpoint=$1 query=$2
    local variables=${3:-{}}
    local token=${4:-${GRAPHQL_TOKEN:-}}

    local payload; payload=$(jq -n \
        --arg q "$query" \
        --argjson v "$variables" \
        '{query: $q, variables: $v}' 2>/dev/null)

    local -a headers=('Content-Type: application/json')
    [[ -n "$token" ]] && headers+=("Authorization: Bearer $token")

    local curl_args=(--silent --fail --max-time 30 --request POST)
    for h in "${headers[@]}"; do
        curl_args+=(--header "$h")
    done
    curl_args+=(--data "$payload" "$endpoint")

    local response; response=$(curl "${curl_args[@]}" 2>/dev/null)

    # Check for GraphQL errors
    local errors; errors=$(echo "$response" | jq -r '.errors // empty' 2>/dev/null)
    if [[ -n "$errors" && "$errors" != "null" ]]; then
        echo "GraphQL Errors: $errors" >&2
        return 1
    fi

    echo "$response" | jq '.data' 2>/dev/null
}

gql_mutation() {
    local endpoint=$1 mutation=$2 variables=${3:-{}} token=${4:-${GRAPHQL_TOKEN:-}}
    gql_query "$endpoint" "$mutation" "$variables" "$token"
}

gql_introspect() {
    local endpoint=$1 token=${2:-${GRAPHQL_TOKEN:-}}

    local introspection_query='{
        __schema {
            types { name kind description fields { name type { name kind } } }
        }
    }'

    gql_query "$endpoint" "$introspection_query" '{}' "$token"
}
```

---

## 73.7 Webhook Server

```bash
#!/bin/bash
# webhook_server.sh - Simple webhook receiver

WEBHOOK_PORT="${WEBHOOK_PORT:-8888}"
WEBHOOK_SECRET="${WEBHOOK_SECRET:-}"
WEBHOOK_LOG="${WEBHOOK_LOG:-/tmp/webhooks.log}"

webhook_verify_hmac() {
    local body=$1 signature=$2 secret=${3:-$WEBHOOK_SECRET}

    local expected; expected=$(echo -n "$body" | \
        openssl dgst -sha256 -hmac "$secret" | \
        grep -oP '(?<= )\S+')

    [[ "$signature" == "sha256=${expected}" ]]
}

webhook_parse_request() {
    local raw_request=$1
    local headers body in_body=false

    while IFS= read -r line; do
        line="${line%$'\r'}"  # strip CR
        if $in_body; then
            body+="$line"
        elif [[ -z "$line" ]]; then
            in_body=true
        else
            headers+="$line\n"
        fi
    done <<< "$raw_request"

    WEBHOOK_HEADERS="$headers"
    WEBHOOK_BODY="$body"
}

webhook_handle() {
    local body=$1 headers=$2

    # Extract signature header
    local sig; sig=$(echo "$headers" | grep -i 'X-Hub-Signature-256:' | awk '{print $2}' | tr -d $'\r')

    # Verify if secret is configured
    if [[ -n "$WEBHOOK_SECRET" ]]; then
        if ! webhook_verify_hmac "$body" "$sig"; then
            echo "401 Unauthorized: Invalid signature" >&2
            return 1
        fi
    fi

    # Parse event type
    local event; event=$(echo "$headers" | grep -i 'X-GitHub-Event:' | awk '{print $2}' | tr -d $'\r')

    # Log
    printf '%s event=%s body_len=%d\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$event" "${#body}" >> "$WEBHOOK_LOG"

    # Dispatch
    local handler="webhook_on_${event//-/_}"
    if declare -f "$handler" &>/dev/null; then
        "$handler" "$body"
    else
        webhook_on_default "$event" "$body"
    fi
}

webhook_on_default() {
    local event=$1 body=$2
    echo "Unhandled event: $event"
    echo "$body" | jq . 2>/dev/null || echo "$body"
}

webhook_serve() {
    echo "Webhook server listening on :${WEBHOOK_PORT}"

    while true; do
        local raw_request
        raw_request=$(nc -l -p "$WEBHOOK_PORT" 2>/dev/null)

        webhook_parse_request "$raw_request"

        if webhook_handle "$WEBHOOK_BODY" "$WEBHOOK_HEADERS"; then
            printf 'HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK'
        else
            printf 'HTTP/1.1 401 Unauthorized\r\nContent-Length: 12\r\n\r\nUnauthorized'
        fi | nc -l -p "$WEBHOOK_PORT" > /dev/null 2>&1 &
    done
}
```

---

## 73.8 Rate Limiter

```bash
#!/bin/bash
# rate_limiter.sh - Token bucket rate limiter

RATE_LIMIT_DIR="${RATE_LIMIT_DIR:-/tmp/rate_limits}"
mkdir -p "$RATE_LIMIT_DIR"

rate_limit_check() {
    local key=$1
    local max_tokens=${2:-60}
    local refill_rate=${3:-1}  # tokens per second
    local window=${4:-60}      # refill window in seconds

    local state_file="${RATE_LIMIT_DIR}/${key//[^a-zA-Z0-9_-]/_}"
    local lock_file="${state_file}.lock"

    (
        flock -x 9

        local now; now=$(date +%s)
        local tokens=$max_tokens last_refill=$now

        if [[ -f "$state_file" ]]; then
            IFS=' ' read -r tokens last_refill < "$state_file"
        fi

        # Refill tokens based on elapsed time
        local elapsed=$(( now - last_refill ))
        local new_tokens=$(( tokens + elapsed * refill_rate ))
        (( new_tokens > max_tokens )) && new_tokens=$max_tokens

        if (( new_tokens < 1 )); then
            echo "$new_tokens $now" > "$state_file"
            echo "RATE_LIMITED: key=$key tokens=$new_tokens" >&2
            exit 1
        fi

        (( new_tokens-- ))
        echo "$new_tokens $now" > "$state_file"
        exit 0

    ) 9>"$lock_file"
}

rate_limited_call() {
    local key=$1 max_tokens=${2:-10} refill_rate=${3:-1}
    shift 3
    local cmd=("$@")

    if rate_limit_check "$key" "$max_tokens" "$refill_rate"; then
        "${cmd[@]}"
    else
        echo "Rate limit exceeded for: $key" >&2
        return 429
    fi
}

rate_limit_status() {
    local key=$1
    local state_file="${RATE_LIMIT_DIR}/${key//[^a-zA-Z0-9_-]/_}"

    if [[ -f "$state_file" ]]; then
        local tokens last_refill
        IFS=' ' read -r tokens last_refill < "$state_file"
        echo "Key: $key  Tokens: $tokens  Last refill: $(date -d @$last_refill 2>/dev/null || date -r $last_refill 2>/dev/null)"
    else
        echo "Key: $key  No state (full bucket)"
    fi
}
```

---

## 73.9 API Health Checker

```bash
#!/bin/bash
# api_health.sh - API endpoint health monitoring

declare -A API_SLA_TIMES=()
declare -A API_SLA_FAILURES=()

check_api_endpoint() {
    local name=$1 url=$2
    local expected_status=${3:-200}
    local expected_body_pattern=${4:-}
    local max_response_ms=${5:-2000}

    local start_ns; start_ns=$(date +%s%N)
    local response http_code body

    response=$(curl --silent \
        --write-out '\n%{http_code}' \
        --max-time $(( max_response_ms / 1000 + 1 )) \
        --connect-timeout 5 \
        "$url" 2>/dev/null)

    local exit_code=$?
    local end_ns; end_ns=$(date +%s%N)
    local response_ms=$(( (end_ns - start_ns) / 1000000 ))

    http_code=$(echo "$response" | tail -1)
    body=$(echo "$response" | head -n -1)

    local status="OK"
    local reason=""

    if (( exit_code != 0 )); then
        status="FAIL" reason="Connection failed (curl exit $exit_code)"
    elif [[ "$http_code" != "$expected_status" ]]; then
        status="FAIL" reason="HTTP $http_code (expected $expected_status)"
    elif [[ -n "$expected_body_pattern" ]] && ! echo "$body" | grep -q "$expected_body_pattern"; then
        status="FAIL" reason="Body pattern not found: $expected_body_pattern"
    elif (( response_ms > max_response_ms )); then
        status="SLOW" reason="${response_ms}ms > ${max_response_ms}ms SLA"
    fi

    API_SLA_TIMES["$name"]=$response_ms
    [[ "$status" != "OK" ]] && API_SLA_FAILURES["$name"]=$(( ${API_SLA_FAILURES[$name]:-0} + 1 ))

    printf '%-30s %-5s %5dms  %s\n' "$name" "$status" "$response_ms" "$reason"

    [[ "$status" == "OK" ]]
}

run_api_health_checks() {
    local config_file=${1:-api_health.conf}
    local passed=0 failed=0

    echo "=== API Health Check ==="
    echo ""

    while IFS='|' read -r name url expected_status pattern max_ms; do
        [[ "$name" =~ ^# || -z "$name" ]] && continue
        check_api_endpoint "$name" "$url" "${expected_status:-200}" "$pattern" "${max_ms:-2000}" && \
            (( passed++ )) || (( failed++ ))
    done < "$config_file"

    echo ""
    echo "Results: PASS=$passed FAIL=$failed"
    (( failed == 0 ))
}

# config format: name|url|expected_status|body_pattern|max_ms
# Example config (api_health.conf):
# homepage|https://example.com|200|Welcome|1000
# api_health|https://api.example.com/health|200|{"status":"ok"}|500
```

---

## 73.10 File Upload and Download

```bash
#!/bin/bash
# http_transfer.sh - File upload/download with progress

http_download() {
    local url=$1 output_file=${2:-} show_progress=${3:-true}

    [[ -z "$output_file" ]] && output_file=$(basename "$url" | cut -d? -f1)

    local -a curl_args=(--location --fail --retry 3 --retry-delay 2)
    $show_progress && curl_args+=(--progress-bar) || curl_args+=(--silent)
    curl_args+=(--output "$output_file")

    curl "${curl_args[@]}" "$url"
    local exit_code=$?

    if (( exit_code == 0 )); then
        local size; size=$(du -h "$output_file" | cut -f1)
        echo "Downloaded: $output_file ($size)"
    else
        echo "Download failed: $url" >&2
        rm -f "$output_file"
        return "$exit_code"
    fi
}

http_upload_file() {
    local url=$1 file=$2 field_name=${3:-file}
    local token=${4:-${UPLOAD_TOKEN:-}}

    local -a curl_args=(--silent --fail --request POST)
    [[ -n "$token" ]] && curl_args+=(--header "Authorization: Bearer $token")
    curl_args+=(--form "${field_name}=@${file}")

    curl "${curl_args[@]}" "$url"
}

http_upload_multipart() {
    local url=$1
    shift
    local -a fields=("$@")

    local -a curl_args=(--silent --fail --request POST)
    for field in "${fields[@]}"; do
        local key="${field%%=*}" val="${field#*=}"
        if [[ "$val" == @* ]]; then
            curl_args+=(--form "${key}=${val}")
        else
            curl_args+=(--form "${key}=${val}")
        fi
    done

    curl "${curl_args[@]}" "$url"
}

http_download_parallel() {
    local -a urls=("$@")
    local output_dir="${DOWNLOAD_DIR:-/tmp/downloads}"
    mkdir -p "$output_dir"

    local -a pids=()
    for url in "${urls[@]}"; do
        local filename; filename=$(basename "$url" | cut -d? -f1)
        (
            curl --silent --fail --location \
                --output "${output_dir}/${filename}" \
                "$url" && \
            echo "OK: $filename" || \
            echo "FAIL: $url"
        ) &
        pids+=($!)
    done

    for pid in "${pids[@]}"; do wait "$pid"; done
    echo "Downloads complete: $output_dir"
}
```

---

## 73.11 Exercises

### Exercise 1: REST API Wrapper
สร้าง wrapper สำหรับ GitHub API ที่:
- List repos, issues, PRs
- Create/close issues
- Trigger workflow
- Rate limit awareness

### Exercise 2: API Aggregator
สร้าง aggregator ที่:
- Fan-out พร้อมกัน N API
- Merge responses
- Timeout individual calls
- Return partial results

### Exercise 3: Webhook Relay
สร้าง relay ที่:
- Receive webhook
- Validate signature
- Transform payload
- Forward to multiple targets
- Retry on failure

---

## สรุป Part 73

✅ curl wrapper: GET/POST/PUT/PATCH/DELETE with header injection and status capture
✅ Auth: Bearer, Basic, API key, OAuth2 client_credentials with token refresh
┅ Retry: exponential backoff + jitter, HTTP-aware retry (skip 4xx)
┅ File-based response cache with TTL, SHA256 key, auto-expire
┅ Pagination: Link header, cursor-based, offset-based traversal
┅ GraphQL: query/mutation execution, variable injection, introspection
┅ Webhook server: HMAC-SHA256 signature verification, event dispatch
┅ Rate limiter: token bucket via file-lock state, flock-based atomicity
┅ API health checker: multi-endpoint battery, SLA timing, config file format
┅ File upload/download: multipart, parallel download with fan-out

---

**→ Part 74: System Monitoring and Observability Dashboards**
