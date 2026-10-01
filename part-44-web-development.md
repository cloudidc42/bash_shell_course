# Part 44: Web Application Development with Bash
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 44.1 HTTP Server in Pure Bash

```bash
#!/bin/bash
# http_server.sh - Minimal HTTP server using netcat/socat

# ─── Single-request handler ────────────────────────────────────
handle_request() {
    local request_line
    local method path protocol

    IFS= read -r request_line
    request_line="${request_line%$'\r'}"
    read method path protocol <<< "$request_line"

    declare -A headers=()
    while IFS= read -r header_line; do
        header_line="${header_line%$'\r'}"
        [[ -z "$header_line" ]] && break
        local key="${header_line%%:*}"
        local value="${header_line#*: }"
        headers["${key,,}"]="$value"
    done

    local body=""
    if [[ "${headers['content-length']:-0}" -gt 0 ]]; then
        local content_length="${headers['content-length']}"
        IFS= read -r -N "$content_length" body
    fi

    route_request "$method" "$path" "$body"
}

route_request() {
    local method=$1
    local path=$2
    local body=$3

    case "$method $path" in
        "GET /")
            http_response 200 "text/html" "<h1>Hello from Bash!</h1>"
            ;;
        "GET /health")
            http_response 200 "application/json" '{"status":"ok"}'
            ;;
        "GET /time")
            local now
            now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
            http_response 200 "application/json" "{\"time\":\"$now\"}"
            ;;
        "POST /echo")
            http_response 200 "application/json" "{\"echo\":$(echo "$body" | jq -c .)}"
            ;;
        *)
            http_response 404 "application/json" '{"error":"not found"}'
            ;;
    esac
}

http_response() {
    local status=$1
    local content_type=$2
    local body=$3

    local status_text
    case $status in
        200) status_text="OK" ;;
        201) status_text="Created" ;;
        400) status_text="Bad Request" ;;
        401) status_text="Unauthorized" ;;
        403) status_text="Forbidden" ;;
        404) status_text="Not Found" ;;
        500) status_text="Internal Server Error" ;;
        *) status_text="Unknown" ;;
    esac

    local body_length=${#body}

    printf "HTTP/1.1 %d %s\r\n" "$status" "$status_text"
    printf "Content-Type: %s; charset=utf-8\r\n" "$content_type"
    printf "Content-Length: %d\r\n" "$body_length"
    printf "Connection: close\r\n"
    printf "X-Powered-By: Bash\r\n"
    printf "\r\n"
    printf "%s" "$body"
}

# ─── Server Loop ───────────────────────────────────────────────
start_server() {
    local port=${1:-8080}
    local host=${2:-0.0.0.0}

    echo "Starting Bash HTTP server on ${host}:${port}"

    if command -v socat &>/dev/null; then
        socat TCP-LISTEN:"$port",reuseaddr,fork EXEC:"$0 --handle"
    elif command -v ncat &>/dev/null; then
        while true; do
            ncat -l "$port" -e "$0 --handle"
        done
    else
        echo "Error: socat or ncat required" >&2
        return 1
    fi
}

if [[ "$1" == "--handle" ]]; then
    handle_request
else
    start_server "${1:-8080}"
fi
```

---

## 44.2 REST API Client Framework

```bash
#!/bin/bash
# rest_client.sh - Full-featured REST API client

# ─── HTTP Client Config ────────────────────────────────────────
declare -A HTTP_CONFIG=(
    [timeout]=30
    [retry_count]=3
    [retry_delay]=1
    [verbose]=false
    [follow_redirects]=true
)

declare -A HTTP_HEADERS=()
HTTP_BASE_URL=""
HTTP_AUTH_TOKEN=""

http_config() {
    local key=$1
    local value=$2
    HTTP_CONFIG["$key"]="$value"
}

http_set_base_url() { HTTP_BASE_URL="${1%/}"; }
http_set_token()    { HTTP_AUTH_TOKEN="$1"; }
http_set_header()   { HTTP_HEADERS["$1"]="$2"; }

# ─── Core Request ──────────────────────────────────────────────
http_request() {
    local method=$1
    local url=$2
    local data=${3:-}

    [[ "$url" != http* ]] && url="${HTTP_BASE_URL}${url}"

    local curl_args=(
        --silent
        --show-error
        --write-out "\n%{http_code}\n%{time_total}"
        --max-time "${HTTP_CONFIG[timeout]}"
        --request "$method"
    )

    ${HTTP_CONFIG[follow_redirects]} && curl_args+=(--location)
    ${HTTP_CONFIG[verbose]} && curl_args+=(--verbose)

    if [[ -n "$HTTP_AUTH_TOKEN" ]]; then
        curl_args+=(--header "Authorization: Bearer ${HTTP_AUTH_TOKEN}")
    fi

    for header_key in "${!HTTP_HEADERS[@]}"; do
        curl_args+=(--header "${header_key}: ${HTTP_HEADERS[$header_key]}")
    done

    curl_args+=(--header "Accept: application/json")

    if [[ -n "$data" ]]; then
        curl_args+=(
            --header "Content-Type: application/json"
            --data "$data"
        )
    fi

    local response
    response=$(curl "${curl_args[@]}" "$url" 2>&1)

    local exit_code=$?
    if (( exit_code != 0 )); then
        echo "curl error ($exit_code): $response" >&2
        return 1
    fi

    local body
    local http_code
    local time_total
    body=$(echo "$response" | head -n -2)
    http_code=$(echo "$response" | tail -n 2 | head -n 1)
    time_total=$(echo "$response" | tail -n 1)

    HTTP_LAST_STATUS="$http_code"
    HTTP_LAST_TIME="$time_total"

    if (( http_code >= 400 )); then
        echo "HTTP $http_code: $body" >&2
        return 1
    fi

    echo "$body"
}

# ─── Convenience Methods ───────────────────────────────────────
http_get()    { http_request "GET"    "$1"; }
http_delete() { http_request "DELETE" "$1"; }
http_post()   { http_request "POST"   "$1" "$2"; }
http_put()    { http_request "PUT"    "$1" "$2"; }
http_patch()  { http_request "PATCH"  "$1" "$2"; }

# ─── JSON Builder ──────────────────────────────────────────────
json_object() {
    local pairs=("$@")
    local json="{"
    local first=true

    for pair in "${pairs[@]}"; do
        local key="${pair%%=*}"
        local value="${pair#*=}"
        $first || json+=","
        first=false

        if [[ "$value" =~ ^[0-9]+(\.[0-9]+)?$ ]] || \
           [[ "$value" == "true" ]] || \
           [[ "$value" == "false" ]] || \
           [[ "$value" == "null" ]]; then
            json+="\"${key}\":${value}"
        else
            value="${value//\\/\\\\}"
            value="${value//\"/\\\"}"
            json+="\"${key}\":\"${value}\""
        fi
    done

    json+="}"
    echo "$json"
}

json_array() {
    local items=("$@")
    local json="["
    local first=true

    for item in "${items[@]}"; do
        $first || json+=","
        first=false
        if [[ "$item" =~ ^\{.*\}$ ]] || [[ "$item" =~ ^\[.*\]$ ]]; then
            json+="$item"
        else
            json+="\"$item\""
        fi
    done

    json+="]"
    echo "$json"
}

# ─── Example API Usage ─────────────────────────────────────────
github_api() {
    http_set_base_url "https://api.github.com"
    http_set_header "User-Agent" "BashScript/1.0"
    [[ -n "$GITHUB_TOKEN" ]] && http_set_token "$GITHUB_TOKEN"

    case "$1" in
        get_user)
            http_get "/users/${2}" | jq '.'
            ;;
        list_repos)
            http_get "/users/${2}/repos?per_page=10" | jq '.[].name'
            ;;
        create_issue)
            local repo=$2 title=$3 body=$4
            local payload
            payload=$(json_object "title=$title" "body=$body")
            http_post "/repos/${repo}/issues" "$payload" | jq '.number'
            ;;
    esac
}
```

---

## 44.3 Web Scraper

```bash
#!/bin/bash
# web_scraper.sh - HTML parsing and web scraping

# ─── HTML Utilities ────────────────────────────────────────────
extract_tag() {
    local tag=$1
    local html=$2
    echo "$html" | grep -oP "(?<=<${tag}[^>]*>).*?(?=</${tag}>)" | head -20
}

extract_attr() {
    local attr=$1
    local html=$2
    echo "$html" | grep -oP "(?<=${attr}=\")[^\"]*" | head -20
}

extract_links() {
    local html=$1
    local base_url=${2:-}
    echo "$html" | grep -oP 'href="[^"]*"' | grep -oP '"[^"]*"' | tr -d '"' | \
        while IFS= read -r link; do
            if [[ "$link" == http* ]]; then
                echo "$link"
            elif [[ -n "$base_url" && "$link" == /* ]]; then
                echo "${base_url}${link}"
            fi
        done | sort -u
}

strip_html() {
    echo "$1" | sed 's/<[^>]*>//g' | sed 's/&nbsp;/ /g; s/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g; s/&quot;/"/g'
}

# ─── Scraper Engine ────────────────────────────────────────────
scrape_page() {
    local url=$1
    local selector=${2:-}
    local output_format=${3:-text}

    local html
    html=$(curl -sL --max-time 30 \
        -H "User-Agent: Mozilla/5.0 (compatible; BashScraper/1.0)" \
        "$url" 2>/dev/null)

    [[ -z "$html" ]] && { echo "Failed to fetch: $url" >&2; return 1; }

    if [[ -n "$selector" ]] && command -v python3 &>/dev/null; then
        echo "$html" | python3 -c "
import sys
try:
    from html.parser import HTMLParser
    import re
    content = sys.stdin.read()
    # Simple selector: tag or tag.class
    selector = '${selector}'
    if '.' in selector:
        tag, cls = selector.split('.', 1)
        pattern = r'<' + tag + r'[^>]*class=\"[^\"]*' + cls + r'[^\"]*\"[^>]*>(.*?)</' + tag + r'>'
    else:
        tag = selector
        pattern = r'<' + tag + r'[^>]*>(.*?)</' + tag + r'>'
    matches = re.findall(pattern, content, re.DOTALL | re.IGNORECASE)
    for m in matches[:20]:
        clean = re.sub(r'<[^>]+>', '', m).strip()
        if clean:
            print(clean)
except Exception as e:
    print(f'Parse error: {e}', file=sys.stderr)
"
    else
        case "$output_format" in
            text)  strip_html "$html" ;;
            links) extract_links "$html" "$url" ;;
            raw)   echo "$html" ;;
        esac
    fi
}

# ─── Crawler ───────────────────────────────────────────────────
crawl() {
    local start_url=$1
    local max_depth=${2:-2}
    local max_pages=${3:-20}
    local output_dir=${4:-./crawl_output}

    mkdir -p "$output_dir"

    declare -A visited=()
    local queue=("$start_url")
    local page_count=0
    local current_depth=0

    echo "Crawling: $start_url (depth=$max_depth, max=$max_pages)"

    while (( ${#queue[@]} > 0 && page_count < max_pages )); do
        local url="${queue[0]}"
        queue=("${queue[@]:1}")

        [[ -v "visited[$url]" ]] && continue
        visited["$url"]=1
        (( page_count++ ))

        echo "[$page_count] Fetching: $url"

        local safe_name
        safe_name=$(echo "$url" | md5sum | cut -c1-8)
        local page_file="${output_dir}/${safe_name}.html"

        curl -sL --max-time 15 "$url" > "$page_file" 2>/dev/null

        if (( current_depth < max_depth )); then
            local html
            html=$(cat "$page_file")
            local links
            links=$(extract_links "$html" "${url%%//*}//${url#*//}")

            while IFS= read -r link; do
                [[ -v "visited[$link]" ]] || queue+=("$link")
            done <<< "$links"
        fi

        sleep 0.5
    done

    echo "Crawled $page_count pages → $output_dir"
}
```

---

## 44.4 Webhook Server

```bash
#!/bin/bash
# webhook_server.sh - Webhook receiver and processor

WEBHOOK_SECRET="${WEBHOOK_SECRET:-}"
WEBHOOK_LOG="/var/log/webhooks.log"
declare -A WEBHOOK_HANDLERS=()

webhook_register() {
    local event=$1
    local handler=$2
    WEBHOOK_HANDLERS["$event"]="$handler"
}

verify_hmac_signature() {
    local payload=$1
    local signature=$2
    local secret=$3

    [[ -z "$secret" ]] && return 0

    local expected
    expected="sha256=$(echo -n "$payload" | openssl dgst -sha256 -hmac "$secret" | awk '{print $2}')"

    if [[ "$signature" != "$expected" ]]; then
        echo "Signature mismatch" >&2
        return 1
    fi
}

process_webhook() {
    local method path_info
    local content_type content_length signature event_type
    local body=""

    IFS= read -r -d $'\r' request_line
    read method path_info _ <<< "$request_line"

    declare -A req_headers=()
    while IFS= read -r header; do
        header="${header%$'\r'}"
        [[ -z "$header" ]] && break
        local key="${header%%:*}"
        local value="${header#*: }"
        req_headers["${key,,}"]="$value"
    done

    content_length="${req_headers['content-length']:-0}"
    content_type="${req_headers['content-type']:-}"
    signature="${req_headers['x-hub-signature-256']:-${req_headers['x-signature']:-}}"
    event_type="${req_headers['x-github-event']:-${req_headers['x-event-type']:-generic}}"

    if (( content_length > 0 )); then
        IFS= read -r -N "$content_length" body
    fi

    echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) EVENT=$event_type PATH=$path_info" >> "$WEBHOOK_LOG"

    if [[ -n "$WEBHOOK_SECRET" ]]; then
        if ! verify_hmac_signature "$body" "$signature" "$WEBHOOK_SECRET"; then
            http_response 401 "application/json" '{"error":"invalid signature"}'
            return
        fi
    fi

    if [[ -v "WEBHOOK_HANDLERS[$event_type]" ]]; then
        local result
        result=$(echo "$body" | "${WEBHOOK_HANDLERS[$event_type]}" 2>&1)
        local exit_code=$?

        if (( exit_code == 0 )); then
            http_response 200 "application/json" '{"status":"processed"}'
        else
            http_response 500 "application/json" "{\"error\":\"handler failed\"}"
        fi
    else
        http_response 200 "application/json" '{"status":"ignored"}'
    fi
}

# ─── Example handlers ──────────────────────────────────────────
handle_push_event() {
    local payload
    payload=$(cat)

    local repo branch pusher
    repo=$(echo "$payload" | jq -r '.repository.full_name')
    branch=$(echo "$payload" | jq -r '.ref' | sed 's|refs/heads/||')
    pusher=$(echo "$payload" | jq -r '.pusher.name')

    echo "Push to $repo ($branch) by $pusher"

    if [[ "$branch" == "main" ]]; then
        echo "Triggering deployment..."
        /opt/deploy/deploy.sh "$repo" "$branch" &
    fi
}

handle_pr_event() {
    local payload
    payload=$(cat)

    local action pr_number title
    action=$(echo "$payload" | jq -r '.action')
    pr_number=$(echo "$payload" | jq -r '.number')
    title=$(echo "$payload" | jq -r '.pull_request.title')

    echo "PR #$pr_number [$action]: $title"
}

webhook_register "push" handle_push_event
webhook_register "pull_request" handle_pr_event
```

---

## 44.5 Static Site Generator

```bash
#!/bin/bash
# ssg.sh - Static site generator

SSG_CONTENT_DIR="content"
SSG_TEMPLATES_DIR="templates"
SSG_OUTPUT_DIR="public"
SSG_BASE_URL="${BASE_URL:-http://localhost}"

ssg_init() {
    mkdir -p "$SSG_CONTENT_DIR/posts" \
             "$SSG_TEMPLATES_DIR" \
             "$SSG_OUTPUT_DIR" \
             "$SSG_OUTPUT_DIR/posts"

    cat > "${SSG_TEMPLATES_DIR}/base.html" << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{title}} | My Blog</title>
    <link rel="stylesheet" href="/style.css">
</head>
<body>
<nav><a href="/">Home</a> | <a href="/posts">Posts</a></nav>
<main>
{{content}}
</main>
<footer>Generated by BashSSG</footer>
</body>
</html>
EOF

    cat > "${SSG_OUTPUT_DIR}/style.css" << 'EOF'
body { font-family: sans-serif; max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
nav { border-bottom: 1px solid #ccc; padding-bottom: 0.5rem; margin-bottom: 1rem; }
code { background: #f4f4f4; padding: 0.2em 0.4em; border-radius: 3px; }
pre code { display: block; padding: 1em; overflow-x: auto; }
EOF

    echo "Initialized SSG structure"
}

parse_frontmatter() {
    local file=$1
    local field=$2

    awk '/^---$/{if(started){exit}else{started=1;next}} started{print}' "$file" | \
        grep "^${field}:" | head -1 | sed "s/^${field}: *//"
}

extract_content() {
    local file=$1
    awk '/^---$/{count++; if(count==2){found=1;next}} found{print}' "$file"
}

markdown_to_html() {
    if command -v pandoc &>/dev/null; then
        pandoc -f markdown -t html
    elif command -v python3 &>/dev/null; then
        python3 -c "
import sys
try:
    import markdown
    content = sys.stdin.read()
    print(markdown.markdown(content, extensions=['fenced_code', 'tables']))
except ImportError:
    import re
    content = sys.stdin.read()
    content = re.sub(r'^# (.+)$', r'<h1>\1</h1>', content, flags=re.MULTILINE)
    content = re.sub(r'^## (.+)$', r'<h2>\1</h2>', content, flags=re.MULTILINE)
    content = re.sub(r'^### (.+)$', r'<h3>\1</h3>', content, flags=re.MULTILINE)
    content = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', content)
    content = re.sub(r'\*(.+?)\*', r'<em>\1</em>', content)
    content = re.sub(r'\`(.+?)\`', r'<code>\1</code>', content)
    paragraphs = content.split('\n\n')
    result = []
    for p in paragraphs:
        p = p.strip()
        if p and not p.startswith('<h'):
            p = '<p>' + p + '</p>'
        result.append(p)
    print('\n'.join(result))
"
    else
        cat
    fi
}

apply_template() {
    local template_file=$1
    local title=$2
    local content=$3

    local template
    template=$(cat "$template_file")
    template="${template//\{\{title\}\}/$title}"
    template="${template//\{\{content\}\}/$content}"
    echo "$template"
}

build_post() {
    local post_file=$1

    local title date slug tags
    title=$(parse_frontmatter "$post_file" "title")
    date=$(parse_frontmatter "$post_file" "date")
    slug=$(parse_frontmatter "$post_file" "slug")
    [[ -z "$slug" ]] && slug=$(basename "$post_file" .md)

    local md_content
    md_content=$(extract_content "$post_file")

    local html_content
    html_content=$(echo "$md_content" | markdown_to_html)

    local full_content="<h1>${title}</h1><time>${date}</time>${html_content}"

    local output_file="${SSG_OUTPUT_DIR}/posts/${slug}.html"
    apply_template "${SSG_TEMPLATES_DIR}/base.html" "$title" "$full_content" > "$output_file"

    echo "Built: $output_file"
    echo "${date}|${slug}|${title}"
}

build_index() {
    local posts_index=$1

    local posts_html="<h1>Posts</h1><ul>"
    while IFS='|' read -r date slug title; do
        posts_html+="<li><a href=\"/posts/${slug}.html\">${title}</a> <small>${date}</small></li>"
    done < <(echo "$posts_index" | sort -r)
    posts_html+="</ul>"

    apply_template "${SSG_TEMPLATES_DIR}/base.html" "Home" "$posts_html" > "${SSG_OUTPUT_DIR}/index.html"
    echo "Built: ${SSG_OUTPUT_DIR}/index.html"
}

build_site() {
    echo "Building site..."
    local posts_index=""

    for post_file in "${SSG_CONTENT_DIR}"/posts/*.md; do
        [[ -f "$post_file" ]] || continue
        local entry
        entry=$(build_post "$post_file")
        posts_index+="$entry"$'\n'
    done

    build_index "$posts_index"
    echo "Done. Output: $SSG_OUTPUT_DIR/"
}

case "${1:-build}" in
    init)  ssg_init ;;
    build) build_site ;;
    *)     echo "Usage: $0 [init|build]" ;;
esac
```

---

## 44.6 API Rate Limiter & Cache

```bash
#!/bin/bash
# api_cache.sh - HTTP response caching

CACHE_DIR="${XDG_CACHE_HOME:-$HOME/.cache}/bash_http"
CACHE_TTL=${CACHE_TTL:-300}

mkdir -p "$CACHE_DIR"

cache_key() {
    local url=$1
    echo "$url" | md5sum | cut -c1-32
}

cache_get() {
    local url=$1
    local key
    key=$(cache_key "$url")
    local cache_file="${CACHE_DIR}/${key}"

    [[ ! -f "$cache_file" ]] && return 1

    local mtime age
    mtime=$(stat -c %Y "$cache_file" 2>/dev/null || stat -f %m "$cache_file")
    age=$(( $(date +%s) - mtime ))

    if (( age > CACHE_TTL )); then
        rm -f "$cache_file"
        return 1
    fi

    cat "$cache_file"
}

cache_set() {
    local url=$1
    local data=$2
    local key
    key=$(cache_key "$url")
    echo "$data" > "${CACHE_DIR}/${key}"
}

cache_clear() {
    local url=${1:-}
    if [[ -n "$url" ]]; then
        local key
        key=$(cache_key "$url")
        rm -f "${CACHE_DIR}/${key}"
    else
        rm -f "${CACHE_DIR}"/*
        echo "Cache cleared"
    fi
}

cached_http_get() {
    local url=$1
    local cached

    cached=$(cache_get "$url") && {
        echo "$cached"
        return 0
    }

    local response
    response=$(curl -sf --max-time 30 "$url") || return 1
    cache_set "$url" "$response"
    echo "$response"
}

# ─── Rate Limiter ──────────────────────────────────────────────
RATE_LIMIT_DIR="${XDG_RUNTIME_DIR:-/tmp}/rate_limits"
mkdir -p "$RATE_LIMIT_DIR"

rate_limit_check() {
    local key=$1
    local max_requests=${2:-10}
    local window_secs=${3:-60}

    local rate_file="${RATE_LIMIT_DIR}/${key}"
    local now
    now=$(date +%s)
    local window_start=$(( now - window_secs ))

    touch "$rate_file"

    local count
    count=$(awk -v cutoff="$window_start" '$1 > cutoff' "$rate_file" | wc -l)

    if (( count >= max_requests )); then
        local oldest
        oldest=$(head -1 "$rate_file" | awk '{print $1}')
        local wait_time=$(( window_secs - (now - oldest) ))
        echo "Rate limit exceeded. Wait ${wait_time}s" >&2
        return 1
    fi

    echo "$now" >> "$rate_file"
    tail -n "$max_requests" "$rate_file" > "${rate_file}.tmp"
    mv "${rate_file}.tmp" "$rate_file"
    return 0
}
```

---

## 44.7 JSON API Server with Authentication

```bash
#!/bin/bash
# api_auth.sh - JWT-style authentication for Bash API

SECRET_KEY="${API_SECRET:-$(openssl rand -hex 32)}"

# ─── Token Generation (simplified JWT-like) ────────────────────
base64url_encode() {
    echo -n "$1" | base64 | tr '+/' '-_' | tr -d '='
}

create_token() {
    local user_id=$1
    local role=${2:-user}
    local expires_at=$(( $(date +%s) + 3600 ))

    local header='{"alg":"HS256","typ":"JWT"}'
    local payload="{\"sub\":\"${user_id}\",\"role\":\"${role}\",\"exp\":${expires_at}}"

    local header_b64 payload_b64 signature
    header_b64=$(base64url_encode "$header")
    payload_b64=$(base64url_encode "$payload")

    signature=$(echo -n "${header_b64}.${payload_b64}" | \
        openssl dgst -sha256 -hmac "$SECRET_KEY" -binary | \
        base64 | tr '+/' '-_' | tr -d '=')

    echo "${header_b64}.${payload_b64}.${signature}"
}

verify_token() {
    local token=$1

    IFS='.' read -r header_b64 payload_b64 sig <<< "$token"

    local expected_sig
    expected_sig=$(echo -n "${header_b64}.${payload_b64}" | \
        openssl dgst -sha256 -hmac "$SECRET_KEY" -binary | \
        base64 | tr '+/' '-_' | tr -d '=')

    if [[ "$sig" != "$expected_sig" ]]; then
        echo "Invalid token signature" >&2
        return 1
    fi

    local payload
    payload=$(echo "$payload_b64" | base64 -d 2>/dev/null || \
              echo "$payload_b64" | base64 --decode 2>/dev/null)

    local exp
    exp=$(echo "$payload" | python3 -c "import sys,json; print(json.load(sys.stdin)['exp'])")
    local now
    now=$(date +%s)

    if (( now > exp )); then
        echo "Token expired" >&2
        return 1
    fi

    echo "$payload"
}

# ─── Middleware ────────────────────────────────────────────────
require_auth() {
    local auth_header=$1

    if [[ -z "$auth_header" ]]; then
        http_response 401 "application/json" '{"error":"missing authorization"}'
        return 1
    fi

    local token="${auth_header#Bearer }"
    local payload

    payload=$(verify_token "$token") || {
        http_response 401 "application/json" '{"error":"invalid token"}'
        return 1
    }

    TOKEN_USER_ID=$(echo "$payload" | python3 -c "import sys,json; print(json.load(sys.stdin)['sub'])")
    TOKEN_ROLE=$(echo "$payload" | python3 -c "import sys,json; print(json.load(sys.stdin)['role'])")
    return 0
}

require_role() {
    local required_role=$1
    if [[ "$TOKEN_ROLE" != "$required_role" && "$TOKEN_ROLE" != "admin" ]]; then
        http_response 403 "application/json" '{"error":"insufficient permissions"}'
        return 1
    fi
}
```

---

## 44.8 Exercises

### Exercise 1: REST CRUD API
สร้าง REST API สำหรับ todo list:
- GET /todos — list all
- POST /todos — create
- PUT /todos/:id — update
- DELETE /todos/:id — delete
- Persistence ด้วย JSON file

### Exercise 2: Static Blog Generator
สร้าง SSG ที่:
- อ่าน Markdown posts
- Generate HTML with table of contents
- RSS feed generation
- Tag pages

### Exercise 3: Webhook Processor
สร้าง GitHub webhook processor ที่:
- รับ push events
- Auto-deploy เมื่อ push ไป main
- Notify Slack
- Log deployment history

---

## สรุป Part 44

✅ HTTP server in pure Bash (socat/ncat)
✅ REST API client framework
✅ Web scraper with link extraction
✅ Webhook server with HMAC verification
✅ Static site generator (Markdown → HTML)
✅ HTTP response caching
✅ Rate limiting
✅ JWT-style token auth

---

**→ Part 45: System Administration Automation**
