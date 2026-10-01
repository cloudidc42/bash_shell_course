# Part 37: API Integration & Webhook Automation
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 37.1 REST API Client Framework

```bash
#!/bin/bash
# api_client.sh - Generic REST API client

declare -A API_CONFIG=(
    [base_url]=""
    [token]=""
    [timeout]="30"
    [retries]="3"
    [retry_delay]="2"
)

api_init() {
    local base_url=$1
    local token=$2
    API_CONFIG[base_url]="$base_url"
    API_CONFIG[token]="$token"
}

api_call() {
    local method=$1
    local path=$2
    local data=${3:-}
    local -n response_var=${4:-_dummy_response}
    
    local url="${API_CONFIG[base_url]}${path}"
    local attempt=0
    local max_attempts="${API_CONFIG[retries]}"
    
    local args=(
        -s
        -w "\n__STATUS__%{http_code}__"
        -X "$method"
        -H "Authorization: Bearer ${API_CONFIG[token]}"
        -H "Content-Type: application/json"
        -H "Accept: application/json"
        --max-time "${API_CONFIG[timeout]}"
        --connect-timeout 10
    )
    
    [[ -n "$data" ]] && args+=(-d "$data")
    
    while (( attempt < max_attempts )); do
        (( attempt++ ))
        
        local raw_response
        raw_response=$(curl "${args[@]}" "$url" 2>/dev/null)
        
        local http_code="${raw_response##*__STATUS__}"
        http_code="${http_code%__}"
        local body="${raw_response%__STATUS__*}"
        
        case "$http_code" in
            2*)
                response_var="$body"
                return 0
                ;;
            429)
                local retry_after=60
                echo "Rate limited, waiting ${retry_after}s..." >&2
                sleep "$retry_after"
                ;;
            5*)
                if (( attempt < max_attempts )); then
                    local delay=$(( API_CONFIG[retry_delay] * attempt ))
                    echo "Server error $http_code, retry ${attempt}/${max_attempts} in ${delay}s..." >&2
                    sleep "$delay"
                fi
                ;;
            4*)
                echo "Client error $http_code: $body" >&2
                return 1
                ;;
        esac
    done
    
    echo "Max retries exceeded" >&2
    return 1
}

github_api() {
    local endpoint=$1
    local method=${2:-GET}
    local data=${3:-}
    
    local url="https://api.github.com${endpoint}"
    local args=(
        -s
        -X "$method"
        -H "Authorization: token ${GITHUB_TOKEN:?}"
        -H "Accept: application/vnd.github.v3+json"
        -H "Content-Type: application/json"
    )
    
    [[ -n "$data" ]] && args+=(-d "$data")
    curl "${args[@]}" "$url"
}

create_issue() {
    local repo=$1
    local title=$2
    local body=$3
    local labels=${4:-"bug"}
    
    local payload
    payload=$(jq -n \
        --arg title "$title" \
        --arg body "$body" \
        --argjson labels "$(echo "$labels" | jq -R 'split(",")')" \
        '{title: $title, body: $body, labels: $labels}')
    
    github_api "/repos/${repo}/issues" "POST" "$payload"
}

list_open_prs() {
    local repo=$1
    
    github_api "/repos/${repo}/pulls?state=open&per_page=100" | \
        jq -r '.[] | [.number, .title, .user.login, .created_at] | @tsv' | \
        column -t -s $'\t'
}

slack_post() {
    local channel=$1
    local text=$2
    local blocks=${3:-}
    
    local payload
    if [[ -n "$blocks" ]]; then
        payload=$(jq -n \
            --arg channel "$channel" \
            --arg text "$text" \
            --argjson blocks "$blocks" \
            '{channel: $channel, text: $text, blocks: $blocks}')
    else
        payload=$(jq -n \
            --arg channel "$channel" \
            --arg text "$text" \
            '{channel: $channel, text: $text}')
    fi
    
    curl -s -X POST \
        -H "Authorization: Bearer ${SLACK_TOKEN:?}" \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "https://slack.com/api/chat.postMessage" | \
        jq -r 'if .ok then "Posted to \(.channel)" else "Error: \(.error)" end'
}

slack_alert() {
    local channel=$1
    local severity=$2
    local title=$3
    local message=$4
    
    local color
    case "$severity" in
        critical) color="#FF0000" ;;
        warning)  color="#FF9900" ;;
        info)     color="#0099FF" ;;
        good)     color="#00CC00" ;;
        *)        color="#CCCCCC" ;;
    esac
    
    slack_post "$channel" "[$severity] $title"
}
```

---

## 37.2 Webhook Server & Handler

```bash
#!/bin/bash
# webhook_handler.sh

handle_github_webhook() {
    local payload=$1
    local signature=${2:-}
    local secret="${GITHUB_WEBHOOK_SECRET:-}"
    
    if [[ -n "$secret" ]] && [[ -n "$signature" ]]; then
        local expected
        expected="sha256=$(echo -n "$payload" | \
            openssl dgst -sha256 -hmac "$secret" | \
            awk '{print $2}')"
        
        if [[ "$signature" != "$expected" ]]; then
            echo "Invalid signature" >&2
            return 1
        fi
    fi
    
    local event_type
    event_type=$(echo "$payload" | jq -r '.action // "unknown"')
    local repo
    repo=$(echo "$payload" | jq -r '.repository.full_name // ""')
    
    echo "GitHub event: $event_type on $repo"
    
    case "$event_type" in
        push)
            local branch
            branch=$(echo "$payload" | jq -r '.ref' | sed 's|refs/heads/||')
            echo "Push to $branch"
            if [[ "$branch" == "main" ]]; then
                echo "Triggering deployment..."
            fi
            ;;
        opened|synchronize)
            local pr_number
            pr_number=$(echo "$payload" | jq -r '.pull_request.number')
            echo "PR #$pr_number updated"
            ;;
        *)
            echo "Unhandled event: $event_type"
            ;;
    esac
}

enqueue_event() {
    local queue_dir="/tmp/webhook_queue"
    local event_type=$1
    local payload=$2
    
    mkdir -p "$queue_dir"
    local event_file="${queue_dir}/$(date +%s%N)_${event_type}.json"
    echo "$payload" > "$event_file"
}

process_event_queue() {
    local queue_dir="/tmp/webhook_queue"
    local handler=$1
    
    while true; do
        for event_file in "${queue_dir}"/*.json; do
            [[ -f "$event_file" ]] || continue
            
            local event_type
            event_type=$(basename "$event_file" .json | cut -d_ -f2-)
            
            local payload
            payload=$(cat "$event_file")
            
            if "$handler" "$event_type" "$payload"; then
                rm -f "$event_file"
            else
                mv "$event_file" "${queue_dir}/failed_$(basename "$event_file")"
            fi
        done
        sleep 1
    done
}
```

---

## 37.3 PagerDuty & Alerting Integration

```bash
#!/bin/bash
# alerting.sh

declare -A ALERT_ROUTING=(
    [critical]="pagerduty slack:ops-critical email:oncall@company.com"
    [warning]="slack:ops-alerts"
    [info]="slack:ops-info"
)

send_alert() {
    local severity=$1
    local title=$2
    local message=$3
    local source=${4:-$(hostname)}
    
    local channels="${ALERT_ROUTING[$severity]:-${ALERT_ROUTING[info]}}"
    
    for channel in $channels; do
        case "${channel%%:*}" in
            slack)
                local slack_channel="${channel#slack:}"
                slack_alert "#${slack_channel}" "$severity" "$title" "$message"
                ;;
            pagerduty)
                trigger_pagerduty "$severity" "$title" "$message" "$source"
                ;;
            email)
                local email="${channel#email:}"
                send_email_alert "$email" "$severity" "$title" "$message"
                ;;
        esac
    done
}

trigger_pagerduty() {
    local severity=$1
    local summary=$2
    local details=$3
    local source=${4:-$(hostname)}
    
    local pd_key="${PAGERDUTY_INTEGRATION_KEY:?}"
    local dedup_key
    dedup_key=$(echo "$summary" | md5sum | cut -d' ' -f1)
    
    local payload
    payload=$(jq -n \
        --arg routing_key "$pd_key" \
        --arg summary "$summary" \
        --arg details "$details" \
        --arg source "$source" \
        --arg severity "$severity" \
        --arg dedup_key "$dedup_key" \
        '{
            routing_key: $routing_key,
            event_action: "trigger",
            dedup_key: $dedup_key,
            payload: {
                summary: $summary,
                severity: $severity,
                source: $source,
                custom_details: {message: $details}
            }
        }')
    
    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "https://events.pagerduty.com/v2/enqueue"
}

resolve_pagerduty() {
    local summary=$1
    local dedup_key
    dedup_key=$(echo "$summary" | md5sum | cut -d' ' -f1)
    
    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "{\"routing_key\":\"${PAGERDUTY_INTEGRATION_KEY}\",\"dedup_key\":\"${dedup_key}\",\"event_action\":\"resolve\"}" \
        "https://events.pagerduty.com/v2/enqueue"
}

send_email_alert() {
    local to=$1
    local severity=$2
    local subject=$3
    local body=$4
    
    local full_subject="[$(hostname)] [$severity] $subject"
    
    if command -v mail &>/dev/null; then
        echo "$body" | mail -s "$full_subject" "$to"
    fi
}
```

---

## 37.4 JSON Processing Patterns

```bash
# ─── jq Recipes ─────────────────────────────────────────────────
# Extract nested values
echo '{"user": {"name": "Alice", "age": 30}}' | jq '.user.name'

# Filter array
echo '[{"name":"Alice","admin":true},{"name":"Bob","admin":false}]' | \
    jq '[.[] | select(.admin == true)]'

# Transform array
echo '[1,2,3,4,5]' | jq '[.[] * 2]'

# Group and aggregate
echo '[{"dept":"eng","salary":100},{"dept":"eng","salary":120},{"dept":"hr","salary":90}]' | \
    jq 'group_by(.dept) | map({dept: .[0].dept, avg: (map(.salary) | add / length)})'

# Merge objects
jq -n '{"a":1} + {"b":2}'

# Build from environment
jq -n --arg host "$(hostname)" --arg time "$(date -u +%s)" \
    '{host: $host, timestamp: $time|tonumber}'

# Recursive descent
echo '{"id":1,"nested":{"id":2,"arr":[{"id":3}]}}' | \
    jq '[.. | .id? | numbers]'

# ─── CSV to JSON ─────────────────────────────────────────────────
csv_to_json() {
    local csv_file=$1
    local delimiter=${2:-,}
    
    awk -F"$delimiter" '
    NR == 1 {
        for (i=1; i<=NF; i++) headers[i] = $i
        next
    }
    {
        printf "{"
        for (i=1; i<=NF; i++) {
            if (i > 1) printf ","
            gsub(/"/, "\\\"", $i)
            printf "\"" headers[i] "\":\"" $i "\""
        }
        print "}"
    }' "$csv_file" | jq -s '.'
}

validate_json_fields() {
    local json=$1
    shift
    local required_fields=("$@")
    
    for field in "${required_fields[@]}"; do
        local value
        value=$(echo "$json" | jq -r ".${field} // empty")
        
        if [[ -z "$value" ]]; then
            echo "Missing required field: $field" >&2
            return 1
        fi
    done
    
    return 0
}
```

---

## 37.5 Exercises

### Exercise 1: GitHub Actions Trigger
สร้าง tool trigger GitHub Actions:
- POST to workflow dispatch endpoint
- Poll for completion
- Stream logs
- Report status

### Exercise 2: Multi-Provider Alert System
สร้าง alerting system:
- Support Slack, PagerDuty, Teams
- Dedup alerts
- Escalation policies
- Auto-resolve

### Exercise 3: Webhook Dashboard
สร้าง webhook monitoring:
- Receive webhooks
- Log all events
- Replay failed events
- Statistics

---

## สรุป Part 37

✅ REST API client framework  
✅ Retry with exponential backoff  
✅ GitHub API integration  
✅ Slack rich messages  
✅ Webhook server & handler  
✅ Event queue processing  
✅ PagerDuty integration  
✅ Multi-channel alert routing  
✅ Advanced jq patterns  

---

**→ Part 38: Database Operations & Data Pipelines**
