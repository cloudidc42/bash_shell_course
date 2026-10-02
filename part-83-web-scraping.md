# Part 83: Shell Scripting for Web Scraping and Crawling
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 83.1 HTTP Client Foundation

```bash
#!/bin/bash
# scrape_http.sh - Robust HTTP client for scraping

set -euo pipefail

SCRAPE_DIR="${SCRAPE_DIR:-/tmp/scrape}"
SCRAPE_DELAY="${SCRAPE_DELAY:-1}"
SCRAPE_TIMEOUT="${SCRAPE_TIMEOUT:-30}"
SCRAPE_UA="${SCRAPE_UA:-Mozilla/5.0 (compatible; BashScraper/1.0)}"
SCRAPE_COOKIE_JAR="${SCRAPE_COOKIE_JAR:-${SCRAPE_DIR}/cookies.txt}"

mkdir -p "$SCRAPE_DIR"

_curl_base() {
    curl --silent --location --max-redirs 5 \
         --timeout "$SCRAPE_TIMEOUT" \
         --user-agent "$SCRAPE_UA" \
         --cookie-jar "$SCRAPE_COOKIE_JAR" \
         --cookie     "$SCRAPE_COOKIE_JAR" \
         --compressed \
         "$@"
}

scrape_get() {
    local url=$1; shift
    _curl_base "$@" "$url"
}

scrape_post() {
    local url=$1 data=$2; shift 2
    _curl_base --request POST --data "$data" "$@" "$url"
}

scrape_post_json() {
    local url=$1 json=$2; shift 2
    _curl_base --request POST \
               --header 'Content-Type: application/json' \
               --data "$json" "$@" "$url"
}

scrape_with_headers() {
    local url=$1; shift
    _curl_base --dump-header - "$@" "$url"
}

scrape_get_code() {
    local url=$1; shift
    _curl_base --output /dev/null --write-out '%{http_code}' "$@" "$url"
}

url_encode() {
    local raw=$1
    python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1],safe=''))" "$raw" 2>/dev/null || \
    printf '%s' "$raw" | od -An -tx1 | tr ' ' '%' | tr -d '\n' | sed 's/%$//'
}

build_query_string() {
    local -a params=("$@")
    local result=""
    for param in "${params[@]}"; do
        local key="${param%%=*}"
        local val="${param#*=}"
        [[ -n "$result" ]] && result+="&"
        result+="$(url_encode "$key")=$(url_encode "$val")"
    done
    echo "$result"
}

scrape_rate_limited() {
    local url=$1; shift
    local last_request_file="${SCRAPE_DIR}/.last_request"
    local last=0

    [[ -f "$last_request_file" ]] && last=$(cat "$last_request_file")
    local now; now=$(date +%s)
    local elapsed=$(( now - last ))
    local delay_int="${SCRAPE_DELAY%.*}"

    if (( elapsed < delay_int )); then
        sleep $(( delay_int - elapsed ))
    fi

    date +%s > "$last_request_file"
    scrape_get "$url" "$@"
}
```

---

## 83.2 HTML Parsing Helpers

```bash
#!/bin/bash
# html_parse.sh - HTML extraction without a full parser

extract_links() {
    local html=$1 base_url=${2:-}

    echo "$html" | grep -oP 'href=["\x27]\K[^"'\'']+' | while read -r link; do
        if [[ "$link" =~ ^https?:// ]]; then
            echo "$link"
        elif [[ "$link" =~ ^/ && -n "$base_url" ]]; then
            local proto_host; proto_host=$(echo "$base_url" | grep -oP '^https?://[^/]+')
            echo "${proto_host}${link}"
        elif [[ -n "$base_url" && ! "$link" =~ ^# ]]; then
            echo "${base_url%/}/${link}"
        fi
    done | sort -u
}

extract_tag_attr() {
    local html=$1 tag=$2 attr=$3
    echo "$html" | grep -oP "(?i)<${tag}[^>]+${attr}=[\"']\\K[^\"']+"
}

extract_tag_content() {
    local html=$1 tag=$2
    echo "$html" | sed -n "s/.*<${tag}[^>]*>\(.*\)<\/${tag}>.*/\1/Ip"
}

extract_meta() {
    local html=$1 name=$2
    echo "$html" | grep -oiP "(?<=<meta[^>]+name=[\"']${name}[\"'][^>]+content=[\"'])([^\"']+)" || \
    echo "$html" | grep -oiP "(?<=<meta[^>]+content=[\"'])([^\"']+)(?=[^>]+name=[\"']${name}[\"'])"
}

extract_title() {
    local html=$1
    echo "$html" | grep -oiP '(?<=<title>)[^<]+' | head -1
}

strip_html_tags() {
    echo "$1" | sed 's/<[^>]*>//g' | sed 's/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g; s/&nbsp;/ /g; s/&#[0-9]*;//g'
}

html_table_to_csv() {
    echo "$1" | \
    sed 's/<\/tr>/\n/gI' | \
    sed 's/<\/t[dh]>/,/gI' | \
    sed 's/<[^>]*>//g' | \
    sed 's/^,//; s/,$//; s/,,*/,/g' | \
    grep -v '^[[:space:]]*$'
}

extract_json_from_script() {
    local html=$1 var_name=${2:-}
    if [[ -n "$var_name" ]]; then
        echo "$html" | grep -oP "(?<=${var_name}\s*=\s*)(\{[^;]+|\[[^;]+)" | head -1
    else
        echo "$html" | grep -oP '\{[^<]{20,}\}' | head -1
    fi
}
```

---

## 83.3 BFS Web Crawler

```bash
#!/bin/bash
# crawler.sh - Breadth-first web crawler

CRAWL_MAX_PAGES="${CRAWL_MAX_PAGES:-100}"
CRAWL_DELAY="${CRAWL_DELAY:-2}"
CRAWL_SAME_DOMAIN="${CRAWL_SAME_DOMAIN:-true}"
CRAWL_OUTPUT_DIR="${CRAWL_OUTPUT_DIR:-/tmp/crawl}"

declare -A VISITED=()
declare -a QUEUE=()
declare -a CRAWL_RESULTS=()

crawl_init() {
    local start_url=$1
    mkdir -p "$CRAWL_OUTPUT_DIR"
    QUEUE=("$start_url")
    VISITED=()
    CRAWL_RESULTS=()
}

crawl_base_domain() {
    echo "$1" | grep -oP '^https?://[^/]+'
}

crawl_is_same_domain() {
    local url=$1 start_url=$2
    local base; base=$(crawl_base_domain "$start_url")
    [[ "$url" =~ ^${base} ]]
}

crawl_save_page() {
    local url=$1 html=$2
    local filename; filename=$(echo "$url" | md5sum | cut -d' ' -f1)
    echo "$html" > "${CRAWL_OUTPUT_DIR}/${filename}.html"
    echo "$url"  > "${CRAWL_OUTPUT_DIR}/${filename}.url"
}

crawl_run() {
    local start_url=$1 page_handler=${2:-}
    crawl_init "$start_url"

    local count=0

    while (( ${#QUEUE[@]} > 0 && count < CRAWL_MAX_PAGES )); do
        local url="${QUEUE[0]}"
        QUEUE=("${QUEUE[@]:1}")

        [[ -n "${VISITED[$url]:-}" ]] && continue
        VISITED["$url"]=1
        (( count++ ))

        echo "[$count/$CRAWL_MAX_PAGES] Fetching: $url"

        local html; html=$(scrape_get "$url" 2>/dev/null) || continue
        local http_code; http_code=$(scrape_get_code "$url" 2>/dev/null || echo 0)
        [[ "$http_code" == "200" ]] || continue

        crawl_save_page "$url" "$html"
        CRAWL_RESULTS+=("$url")

        [[ -n "$page_handler" ]] && "$page_handler" "$url" "$html" 2>/dev/null || true

        while IFS= read -r link; do
            [[ -n "${VISITED[$link]:-}" ]] && continue
            if $CRAWL_SAME_DOMAIN; then
                crawl_is_same_domain "$link" "$start_url" || continue
            fi
            QUEUE+=("$link")
        done < <(extract_links "$html" "$url")

        sleep "$CRAWL_DELAY"
    done

    echo ""
    echo "Crawled $count pages. Results in: $CRAWL_OUTPUT_DIR"
}

crawl_sitemap() {
    local sitemap_url=$1
    local xml; xml=$(scrape_get "$sitemap_url" 2>/dev/null)
    echo "$xml" | grep -oP '(?<=<loc>)[^<]+'
}
```

---

## 83.4 robots.txt Compliance

```bash
#!/bin/bash
# robots.sh - robots.txt parser and compliance checker

ROBOTS_CACHE_DIR="${SCRAPE_DIR:-/tmp/scrape}/robots"
mkdir -p "$ROBOTS_CACHE_DIR"

robots_fetch() {
    local base_url=$1
    local cache_file="${ROBOTS_CACHE_DIR}/$(echo "$base_url" | md5sum | cut -d' ' -f1).txt"

    if [[ ! -f "$cache_file" ]] || \
       [[ $(( $(date +%s) - $(stat -c '%Y' "$cache_file" 2>/dev/null || echo 0) )) -gt 86400 ]]; then
        scrape_get "${base_url}/robots.txt" > "$cache_file" 2>/dev/null || echo "" > "$cache_file"
    fi

    cat "$cache_file"
}

robots_is_allowed() {
    local url=$1 user_agent=${2:-*}
    local base_url; base_url=$(crawl_base_domain "$url")
    local path="${url#${base_url}}"
    [[ -z "$path" ]] && path="/"

    local robots; robots=$(robots_fetch "$base_url")
    local in_agent=false
    local allowed=true

    while IFS= read -r line; do
        line="${line%%#*}"
        line="${line%"${line##*[![:space:]]}"}"
        [[ -z "$line" ]] && continue

        local directive="${line%%:*}"
        local value="${line#*: }"
        value="${value# }"

        case "${directive,,}" in
            user-agent)
                [[ "$value" == "$user_agent" || "$value" == "*" ]] && in_agent=true || in_agent=false
                ;;
            disallow)
                $in_agent && [[ -n "$value" ]] && [[ "$path" =~ ^${value} ]] && allowed=false
                ;;
            allow)
                $in_agent && [[ "$path" =~ ^${value} ]] && allowed=true
                ;;
        esac
    done <<< "$robots"

    $allowed
}

robots_get_crawl_delay() {
    local base_url=$1
    local robots; robots=$(robots_fetch "$base_url")
    echo "$robots" | grep -i '^Crawl-delay:' | head -1 | awk '{print $2}' || echo "$SCRAPE_DELAY"
}
```

---

## 83.5 Data Extraction Patterns

```bash
#!/bin/bash
# extract_data.sh - Common scraping extraction patterns

scrape_price() {
    local html=$1
    echo "$html" | grep -oP '[\$\€\£]\s*[0-9,]+\.?[0-9]*' | head -5
}

scrape_emails() {
    local html=$1
    strip_html_tags "$html" | grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' | sort -u
}

scrape_phone_numbers() {
    local html=$1
    strip_html_tags "$html" | grep -oP '\+?[\d\s\-\(\)]{10,20}' | \
        grep -P '\d{3}.*\d{4}' | head -10
}

scrape_og_metadata() {
    local html=$1
    local result="{}"
    local -a props=(title description image url type site_name)

    for prop in "${props[@]}"; do
        local val; val=$(echo "$html" | \
            grep -oiP "(?<=<meta[^>]+property=[\"']og:${prop}[\"'][^>]+content=[\"'])[^\"']+" | head -1)
        [[ -n "$val" ]] && \
            result=$(echo "$result" | jq --arg k "$prop" --arg v "$val" '. + {($k): $v}' 2>/dev/null || echo "$result")
    done
    echo "$result"
}

scrape_structured_data() {
    local html=$1
    echo "$html" | \
        grep -oP '(?<=<script type=["\x27]application/ld\+json["\x27]>)[\s\S]*?(?=</script>)' | \
        head -1 | jq '.' 2>/dev/null || echo "{}"
}

scrape_multiple_pages() {
    local url_template=$1 start=${2:-1} end=${3:-10} field=${4:-page}

    for i in $(seq "$start" "$end"); do
        local url="${url_template/\{${field}\}/$i}"
        echo "Page $i: $url" >&2
        scrape_rate_limited "$url"
        echo ""
    done
}
```

---

## 83.6 Parallel Fetcher

```bash
#!/bin/bash
# parallel_fetch.sh - Rate-limited parallel URL fetching

parallel_fetch() {
    local urls_file=$1 output_dir=${2:-/tmp/fetch} workers=${3:-4}

    mkdir -p "$output_dir"

    local -a pids=()
    local count=0

    while IFS= read -r url; do
        [[ -z "$url" || "$url" =~ ^# ]] && continue

        local filename; filename=$(echo "$url" | md5sum | cut -d' ' -f1)
        local output="${output_dir}/${filename}"

        (
            local code; code=$(curl --silent --location \
                --timeout "$SCRAPE_TIMEOUT" \
                --user-agent "$SCRAPE_UA" \
                --output "$output" \
                --write-out '%{http_code}' \
                "$url" 2>/dev/null)
            echo "$url|$code|$output" >> "${output_dir}/manifest.txt"
        ) &
        pids+=($!)
        (( count++ ))

        while (( ${#pids[@]} >= workers )); do
            local new_pids=()
            for pid in "${pids[@]}"; do
                kill -0 "$pid" 2>/dev/null && new_pids+=("$pid") || wait "$pid" 2>/dev/null
            done
            pids=("${new_pids[@]}")
            (( ${#pids[@]} >= workers )) && sleep 0.2
        done

        sleep "$(echo "$SCRAPE_DELAY / $workers" | awk '{printf "%.2f", $1}')" 2>/dev/null || sleep 1
    done < "$urls_file"

    for pid in "${pids[@]}"; do wait "$pid" 2>/dev/null; done

    echo "Fetched $count URLs to: $output_dir"
    sort "${output_dir}/manifest.txt" 2>/dev/null || true
}
```

---

## 83.7 Exercises

### Exercise 1: Product Price Monitor
สร้าง tool ที่:
- Scrape product pages ทุก 1 ชั่วโมง
- Track price history ใน SQLite
- Alert เมื่อ price drops > 20%
- Export to CSV

### Exercise 2: Sitemap Crawler
สร้าง crawler ที่:
- Parse sitemap.xml
- Fetch ทุก URL แบบ parallel
- Check HTTP status
- Report broken links

### Exercise 3: Content Aggregator
สร้าง aggregator ที่:
- Fetch multiple RSS/Atom feeds
- Parse XML หรือ HTML
- Deduplicate by URL
- Store ใน SQLite + export JSON

---

## สรุป Part 83

✅ _curl_base wrapper: cookie jar, UA, timeout, compressed, redirect follow
┅ url_encode, build_query_string: percent-encoding helpers
┅ scrape_rate_limited: last-request file-based delay enforcement
┅ extract_links: absolute + relative URL resolution
┅ extract_tag_attr/content, extract_meta, extract_title, strip_html_tags
┅ html_table_to_csv: pure sed pipeline
┅ BFS crawler: visited map, queue, same-domain filter, page handler hook
┅ robots.txt: fetch+cache, user-agent/disallow/allow parse, crawl_delay
┅ Data extractors: price, email, phone, OG metadata, ld+json, pagination links
┅ parallel_fetch: worker-throttled URL batch with manifest output

---

**→ Part 84: Advanced Process Management and Job Control**
