# Part 23: JSON & Data Formats Processing
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 23.1 jq - JSON Processor

```bash
# ─── Installation ─────────────────────────────────────────────
apt-get install jq
brew install jq

# ─── Basic syntax ─────────────────────────────────────────────
# . = identity (pretty print)
echo '{"name":"Alice","age":30}' | jq '.'

# .field = access field
echo '{"name":"Alice","age":30}' | jq '.name'       # "Alice"
echo '{"name":"Alice","age":30}' | jq '.name,.age'  # "Alice"\n30

# -r = raw output (no quotes)
echo '{"name":"Alice"}' | jq -r '.name'  # Alice (no quotes)

# ─── Arrays ───────────────────────────────────────────────────
echo '[1,2,3,4,5]' | jq '.[]'           # iterate: 1 2 3 4 5
echo '[1,2,3,4,5]' | jq '.[2]'          # index: 3
echo '[1,2,3,4,5]' | jq '.[2:4]'        # slice: [3,4]
echo '[1,2,3,4,5]' | jq 'length'        # 5

# Array of objects
cat > data.json << 'EOF'
[
  {"id":1,"name":"Alice","role":"admin","salary":90000},
  {"id":2,"name":"Bob","role":"user","salary":60000},
  {"id":3,"name":"Carol","role":"admin","salary":85000},
  {"id":4,"name":"Dave","role":"user","salary":55000}
]
EOF

jq '.[0]' data.json                     # first item
jq '.[] | .name' data.json              # all names
jq '.[] | select(.role=="admin")' data.json   # filter
jq '.[] | select(.salary > 70000) | .name' data.json  # conditional
jq 'length' data.json                   # count items
jq 'map(.salary) | add' data.json       # sum salaries
jq 'map(.salary) | add / length' data.json  # average
jq 'map(select(.role=="admin")) | length' data.json  # count admins

# ─── Transformations ──────────────────────────────────────────
# Create new object
jq '.[] | {name: .name, isAdmin: (.role == "admin")}' data.json

# Sort
jq 'sort_by(.salary)' data.json
jq 'sort_by(.name) | reverse' data.json

# Group by
jq 'group_by(.role)' data.json

# Unique
jq '[.[].role] | unique' data.json

# Select + transform
jq '.[] | select(.salary > 70000) | {name, salary}' data.json

# ─── String operations ────────────────────────────────────────
jq '.[] | .name | ascii_upcase' data.json
jq '.[] | .name | split(" ")[0]' data.json   # first word
jq '.[] | "\(.name) earns \(.salary)"' data.json  # interpolation

# ─── Numbers ──────────────────────────────────────────────────
jq '.[] | .salary * 1.1' data.json          # 10% raise
jq '.[] | .salary | floor' data.json
jq '.[] | (.salary / 1000 | round) * 1000' data.json  # round to 1000

# ─── Object manipulation ──────────────────────────────────────
# Add field
jq '.[] | . + {dept: "Engineering"}' data.json

# Remove field
jq '.[] | del(.role)' data.json

# Rename field
jq '.[] | {id, username: .name, role, salary}' data.json

# Update nested
echo '{"user":{"name":"Alice","prefs":{"theme":"dark"}}}' | \
    jq '.user.prefs.theme = "light"'

# Merge objects
jq -n '{"a":1} + {"b":2}'   # {"a":1,"b":2}
```

---

## 23.2 jq Advanced

```bash
# ─── Filters and functions ────────────────────────────────────
# map
jq 'map(.salary * 1.1)' data.json

# reduce
jq 'reduce .[] as $item (0; . + $item.salary)' data.json

# any/all
jq 'any(.[]; .role == "admin")' data.json   # true if any admin
jq 'all(.[]; .salary > 50000)' data.json    # true if all > 50k

# env (access environment variables)
export MY_VAR="hello"
jq -n 'env.MY_VAR'

# $ENV
jq -n '$ENV.HOME'

# ─── Input/Output formats ─────────────────────────────────────
# Compact output
jq -c '.' data.json

# Tab indentation
jq --tab '.' data.json

# Indent 4 spaces
jq --indent 4 '.' data.json

# Raw input (treat input as strings, not JSON)
echo '{"key":"value"}' | jq -Rs '.'   # escape as JSON string

# Null input (for building from scratch)
jq -n '{"generated": now | todate, "version": "1.0"}'

# Multiple inputs
jq -s '.' file1.json file2.json   # slurp: combine as array
jq -n '[inputs]' file1.json file2.json  # same

# ─── Variables ────────────────────────────────────────────────
# --arg: string variable
jq --arg name "Alice" '.[] | select(.name == $name)' data.json

# --argjson: JSON variable
jq --argjson min 70000 '.[] | select(.salary >= $min)' data.json

# --slurpfile: load JSON file into variable
jq --slurpfile config config.json '.[0].threshold as $t | .[] | select(.val > $t)' data.json

# ─── Defining functions ───────────────────────────────────────
jq '
def salary_grade:
  if . >= 90000 then "Senior"
  elif . >= 70000 then "Mid"
  else "Junior"
  end;

.[] | {name, grade: (.salary | salary_grade)}
' data.json

# ─── Error handling ───────────────────────────────────────────
# try-catch
jq '.[] | try .missing.path catch "N/A"' data.json

# Default value with //
jq '.[] | .missing // "default"' data.json

# ─── Streaming ────────────────────────────────────────────────
# Process large JSON streams
jq -cn --stream '
  fromstream(1|truncate_stream(tostream))
' large.json
```

---

## 23.3 YAML Processing

```bash
# ─── yq - YAML processor (similar to jq) ─────────────────────
# Install
apt-get install yq   # or
pip install yq       # or
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O yq
chmod +x yq

# Basic usage (mikefarah/yq - preferred)
yq '.name' config.yaml
yq '.servers[0].host' config.yaml
yq '.servers[] | .host' config.yaml
yq '.servers | length' config.yaml

# Update
yq '.replicas = 3' deployment.yaml

# In-place edit
yq -i '.replicas = 3' deployment.yaml
yq -i '.env[] |= select(.name == "PORT").value = "8080"' app.yaml

# Convert YAML to JSON
yq -o=json '.' config.yaml

# Convert JSON to YAML
cat data.json | yq -P '.'

# Multi-document YAML
yq '.[0].name' multi-doc.yaml   # first document
yq '.name' multi-doc.yaml       # all documents

# ─── Python for YAML ──────────────────────────────────────────
# When yq not available
yaml_get() {
    python3 -c "
import yaml, sys
data = yaml.safe_load(open('$1'))
print(eval('data$2'))
"
}

yaml_set() {
    python3 << EOF
import yaml
with open('$1') as f:
    data = yaml.safe_load(f)
# Set value: data['key'] = 'value'
exec("data$2 = '$3'")
with open('$1', 'w') as f:
    yaml.dump(data, f, default_flow_style=False)
EOF
}

# Process Kubernetes YAML
process_k8s_yaml() {
    local file=$1
    
    # Get all resource types
    yq '.kind' "$file"
    
    # Get container images
    yq '.spec.containers[].image' "$file"
    
    # Update image tag
    yq -i ".spec.containers[0].image = \"myapp:v2.0\"" "$file"
    
    # Get all env vars
    yq '.spec.containers[].env[] | "\(.name)=\(.value)"' "$file"
}
```

---

## 23.4 CSV Processing

```bash
# ─── Basic CSV with awk ───────────────────────────────────────
awk -F, '{print $1,$3}' data.csv              # extract columns
awk -F, 'NR>1 {sum+=$3} END {print sum}' data.csv  # sum column 3

# Handle quoted fields with csvkit
pip install csvkit

csvcut -c 1,3 data.csv          # select columns 1 and 3
csvgrep -c 2 -m "Alice" data.csv  # filter where column 2 = Alice
csvsort -c 3 data.csv           # sort by column 3
csvjoin -c id users.csv orders.csv   # JOIN two files
csvstat data.csv                # statistics
csv2json data.csv               # convert to JSON

# ─── Pure bash CSV parser ─────────────────────────────────────
# Simple (doesn't handle quoted fields with commas)
while IFS=, read -r field1 field2 field3; do
    echo "Field1=$field1 Field2=$field2 Field3=$field3"
done < data.csv

# Better: skip header
{
    IFS=, read -r _ # skip header
    while IFS=, read -r id name email; do
        echo "Processing: $name ($email)"
    done
} < users.csv

# ─── Generate CSV ─────────────────────────────────────────────
generate_csv() {
    local output=$1
    
    {
        echo "id,name,email,created_at"
        for i in {1..100}; do
            echo "$i,User_$i,user_$i@example.com,$(date +%Y-%m-%d)"
        done
    } > "$output"
}

# ─── CSV to JSON ──────────────────────────────────────────────
csv_to_json() {
    python3 << 'EOF'
import csv, json, sys

filename = sys.argv[1] if len(sys.argv) > 1 else '/dev/stdin'
with open(filename) as f:
    reader = csv.DictReader(f)
    data = list(reader)
print(json.dumps(data, indent=2))
EOF
}

# ─── Process large CSV in parallel ────────────────────────────
process_large_csv() {
    local file=$1
    local workers=4
    local total_lines
    total_lines=$(wc -l < "$file")
    local chunk=$(( total_lines / workers ))
    
    # Split and process in parallel
    for ((i=0; i<workers; i++)); do
        local start=$(( i * chunk + 2 ))   # +2 to skip header
        local end=$(( (i+1) * chunk + 1 ))
        (( i == workers-1 )) && end="$"  # last worker gets rest
        
        sed -n "${start},${end}p" "$file" | \
            awk -F, '{print $1, $2}' > "output_${i}.tmp" &
    done
    wait
    
    cat output_*.tmp > output.txt
    rm output_*.tmp
}
```

---

## 23.5 XML Processing

```bash
# ─── xmllint ──────────────────────────────────────────────────
# Validate and format XML
xmllint --format input.xml
xmllint --valid schema.xml
xmllint --xpath "//user/@name" users.xml

# ─── Python xml.etree ─────────────────────────────────────────
xml_query() {
    local file=$1
    local xpath=$2
    
    python3 << EOF
import xml.etree.ElementTree as ET
tree = ET.parse('$file')
root = tree.getroot()
for elem in root.findall('$xpath'):
    print(elem.text or ET.tostring(elem, encoding='unicode'))
EOF
}

# ─── xmlstarlet ───────────────────────────────────────────────
# Install: apt-get install xmlstarlet
xmlstarlet sel -t -v "//user/name" users.xml    # select
xmlstarlet ed --update "//config/port" -v "8080" config.xml  # edit
xmlstarlet val -e schema.xsd data.xml           # validate
```

---

## 23.6 API Integration

```bash
#!/bin/bash
# api_client.sh - REST API client

BASE_URL="https://api.example.com"
API_KEY="${API_KEY:?API_KEY required}"

# ─── Helper functions ─────────────────────────────────────────
api_get() {
    local endpoint=$1
    local params=${2:-}
    
    curl -sf \
        -H "Authorization: Bearer $API_KEY" \
        -H "Content-Type: application/json" \
        "${BASE_URL}${endpoint}${params:+?$params}"
}

api_post() {
    local endpoint=$1
    local data=$2
    
    curl -sf -X POST \
        -H "Authorization: Bearer $API_KEY" \
        -H "Content-Type: application/json" \
        -d "$data" \
        "${BASE_URL}${endpoint}"
}

api_put() {
    local endpoint=$1
    local data=$2
    
    curl -sf -X PUT \
        -H "Authorization: Bearer $API_KEY" \
        -H "Content-Type: application/json" \
        -d "$data" \
        "${BASE_URL}${endpoint}"
}

api_delete() {
    local endpoint=$1
    
    curl -sf -X DELETE \
        -H "Authorization: Bearer $API_KEY" \
        "${BASE_URL}${endpoint}"
}

# ─── Usage examples ───────────────────────────────────────────
# List users
list_users() {
    api_get "/users" "page=1&limit=20" | jq '.data[] | {id, name, email}'
}

# Create user
create_user() {
    local name=$1
    local email=$2
    
    local payload
    payload=$(jq -n \
        --arg name "$name" \
        --arg email "$email" \
        '{name: $name, email: $email}')
    
    api_post "/users" "$payload" | jq '.id'
}

# Paginated fetch (get all pages)
fetch_all() {
    local endpoint=$1
    local page=1
    local all_items=()
    
    while true; do
        local response
        response=$(api_get "$endpoint" "page=$page&limit=100")
        
        local items count
        items=$(echo "$response" | jq -c '.data[]')
        count=$(echo "$response" | jq '.data | length')
        
        [[ $count -eq 0 ]] && break
        
        while IFS= read -r item; do
            all_items+=("$item")
        done <<< "$items"
        
        (( page++ ))
        
        # Check if last page
        local total_pages
        total_pages=$(echo "$response" | jq '.meta.total_pages // 1')
        (( page > total_pages )) && break
    done
    
    printf '%s\n' "${all_items[@]}" | jq -s '.'
}

# ─── GitHub API example ───────────────────────────────────────
github_list_repos() {
    local user=$1
    
    curl -sf \
        -H "Accept: application/vnd.github.v3+json" \
        "https://api.github.com/users/$user/repos?per_page=100&sort=updated" | \
        jq '.[] | {name, description, language, stars: .stargazers_count}' | \
        jq -s 'sort_by(-.stars)'
}

github_create_issue() {
    local repo=$1
    local title=$2
    local body=$3
    local token=$4
    
    curl -sf -X POST \
        -H "Authorization: token $token" \
        -H "Accept: application/vnd.github.v3+json" \
        -d "$(jq -n --arg t "$title" --arg b "$body" '{title:$t, body:$b}')" \
        "https://api.github.com/repos/$repo/issues"
}
```

---

## 23.7 Exercises

### Exercise 1: JSON Config Manager
สร้าง tool สำหรับ manage JSON configs:
- Get/set nested values
- Merge config files
- Validate against schema
- Track changes

### Exercise 2: API Aggregator
สร้าง script ที่:
- Fetch data จากหลาย APIs
- Merge และ transform data
- Cache responses
- Export เป็น CSV/JSON

### Exercise 3: YAML Deployer
สร้าง tool สำหรับ Kubernetes:
- Validate YAML syntax
- Substitute environment variables
- Apply to cluster
- Track deployment history

---

## สรุป Part 23

✅ jq basics and advanced usage  
✅ jq transformations and functions  
✅ YAML processing with yq  
✅ CSV processing (awk, csvkit, python)  
✅ XML with xmllint and xmlstarlet  
✅ REST API client  
✅ Pagination handling  
✅ GitHub API integration  

---

**→ Part 24: Database Operations from Shell**
