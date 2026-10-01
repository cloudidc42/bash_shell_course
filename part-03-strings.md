# Part 03: String Manipulation
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 3.1 String Basics

```bash
#!/bin/bash

# การสร้าง string
str1='Single quotes - no expansion: $USER'
str2="Double quotes - with expansion: $USER"
str3=$'Special chars: \t tab \n newline \033[31m red \033[0m'
str4="Multi
line
string"

echo "$str1"
echo "$str2"
echo -e "$str3"   # -e เพื่อ interpret escape sequences
echo "$str4"

# String ที่มี single quote ข้างใน
str5='It'\''s a test'            # แบบ tricky
str6="It's a test"               # ใช้ double quote แทน
str7=$'It\'s a test'             # $'...' syntax

# แสดงความแตกต่าง
echo "Length of str1: ${#str1}"
echo "Length of str2: ${#str2}"
```

---

## 3.2 String Length & Character Access

```bash
str="Hello, World!"

# Length
echo "${#str}"               # 13

# Character at position (index จาก 0)
echo "${str:0:1}"            # H
echo "${str:7:1}"            # W
echo "${str: -1}"            # ! (ตัวสุดท้าย)

# Substring
echo "${str:0:5}"            # Hello
echo "${str:7}"              # World!
echo "${str:7:5}"            # World
echo "${str: -6}"            # orld!
echo "${str: -6:5}"          # orld!

# Loop ผ่านทีละ character
for ((i=0; i<${#str}; i++)); do
    printf "str[%d] = %s\n" $i "${str:$i:1}"
done

# อีกวิธี
while IFS= read -r -n1 char; do
    echo "char: $char"
done <<< "$str"
```

---

## 3.3 String Comparison

```bash
# Comparison operators ใน [[ ]]
str1="apple"
str2="banana"
str3="Apple"

# Equality
[[ "$str1" == "$str2" ]] && echo "equal" || echo "not equal"
[[ "$str1" != "$str2" ]] && echo "different"

# Lexicographic comparison
[[ "$str1" < "$str2" ]]  && echo "apple < banana"
[[ "$str2" > "$str1" ]]  && echo "banana > apple"

# Case-insensitive comparison (bash 4+)
# ด้วย nocasematch option
shopt -s nocasematch
[[ "$str1" == "$str3" ]] && echo "case-insensitive equal"
shopt -u nocasematch

# Prefix/Suffix/Contains
[[ "$str1" == app* ]]    && echo "starts with app"
[[ "$str1" == *le ]]     && echo "ends with le"
[[ "$str1" == *pp* ]]    && echo "contains pp"

# Regex comparison
[[ "$str1" =~ ^[a-z]+$ ]] && echo "all lowercase"
[[ "test123" =~ ^[a-z]+[0-9]+$ ]] && echo "letters then numbers"

# Compare length
[[ ${#str1} -gt ${#str2} ]] && echo "str1 longer"
[[ ${#str1} -lt ${#str2} ]] && echo "str1 shorter"
[[ ${#str1} -eq ${#str2} ]] && echo "same length"

# Empty/Non-empty
[[ -z "" ]]    && echo "empty"
[[ -n "hi" ]]  && echo "not empty"

# Null vs empty
var1=""
unset var2

[[ -z "$var1" ]] && echo "var1 is empty or unset"
[[ -z "${var2+x}" ]] && echo "var2 is unset"
```

---

## 3.4 String Conversion

```bash
str="Hello, World! 123"

# Upper/Lowercase (Bash 4+)
echo "${str^^}"              # HELLO, WORLD! 123
echo "${str,,}"              # hello, world! 123
echo "${str^}"               # Hello, World! 123 (first char only)
echo "${str,}"               # hELLO, WORLD! 123 (first char only)

# ใช้ tr command
echo "$str" | tr '[:lower:]' '[:upper:]'    # uppercase
echo "$str" | tr '[:upper:]' '[:lower:]'    # lowercase
echo "$str" | tr 'a-z' 'A-Z'               # alternative

# ใช้ sed
echo "$str" | sed 's/[a-z]/\u&/g'          # Capitalize each letter... ไม่ work ใน basic
echo "$str" | sed 's/./\u&/g'              # ใช้ gnu sed

# Convert spaces to underscores
echo "hello world foo" | tr ' ' '_'         # hello_world_foo

# Remove non-alphanumeric
echo "Hello, World! 123" | tr -cd '[:alnum:]'   # HelloWorld123

# Reverse string
echo "$str" | rev                           # 321 !dlroW ,olleH

# String to array of chars
IFS= read -ra chars <<< "$str"
# หรือ
chars=($(echo "$str" | fold -w1))
```

---

## 3.5 Search & Replace

```bash
text="The quick brown fox jumps over the lazy dog"

# Parameter expansion (limited)
echo "${text/fox/cat}"           # แทนครั้งแรก
echo "${text//the/a}"            # แทนทั้งหมด (case sensitive)
echo "${text/#The/A}"            # แทน prefix
echo "${text/%dog/cat}"          # แทน suffix

# sed - Stream Editor (powerful)
echo "$text" | sed 's/fox/cat/'              # แทนครั้งแรก
echo "$text" | sed 's/the/a/g'              # แทนทั้งหมด
echo "$text" | sed 's/the/a/gi'             # case insensitive
echo "$text" | sed 's/\bthe\b/a/g'          # whole word only
echo "$text" | sed 's/[aeiou]/*/g'          # แทน vowels
echo "$text" | sed 's/\(fox\)/[\1]/g'       # เพิ่ม brackets รอบ fox
echo "$text" | sed 's/\b\w/\u&/g'           # Capitalize first letter of each word

# Multiple replacements
echo "$text" | sed -e 's/fox/cat/g' -e 's/dog/fish/g'

# In-place replacement (ระวัง!)
# sed -i 's/old/new/g' file.txt           # แก้ไขไฟล์โดยตรง
# sed -i.bak 's/old/new/g' file.txt       # backup ก่อน

# awk
echo "$text" | awk '{gsub(/fox/, "cat"); print}'
echo "$text" | awk '{gsub(/the/, "a"); print}'

# Python (complex patterns)
echo "$text" | python3 -c "
import sys, re
text = sys.stdin.read().rstrip()
result = re.sub(r'\bfox\b', 'cat', text, flags=re.IGNORECASE)
print(result)
"
```

---

## 3.6 Split & Join

```bash
# Split by delimiter
csv="apple,banana,cherry,date"

# Method 1: IFS
IFS=',' read -ra fruits <<< "$csv"
for fruit in "${fruits[@]}"; do
    echo "Fruit: $fruit"
done

# Method 2: tr + read
while IFS= read -r fruit; do
    echo "Fruit: $fruit"
done < <(echo "$csv" | tr ',' '\n')

# Method 3: parameter expansion
while [[ "$csv" == *,* ]]; do
    part="${csv%%,*}"
    echo "Part: $part"
    csv="${csv#*,}"
done
echo "Part: $csv"

# Split into array
IFS='/' read -ra path_parts <<< "/home/user/documents"
echo "Parts: ${path_parts[@]}"
echo "Count: ${#path_parts[@]}"

# Split by multi-char delimiter
data="one::two::three"
IFS=':' read -ra parts <<< "$data"
# ได้ ["one", "", "two", "", "three"]

# Better for multi-char:
readarray -d '::' -t parts < <(echo -n "$data" | sed 's/::/\n/g')

# Join array elements
fruits=("apple" "banana" "cherry")

# Join with comma
joined=$(IFS=','; echo "${fruits[*]}")
echo $joined   # apple,banana,cherry

# Join with custom separator
join_array() {
    local sep=$1
    shift
    local arr=("$@")
    local result=""
    for ((i=0; i<${#arr[@]}; i++)); do
        [[ $i -gt 0 ]] && result+="$sep"
        result+="${arr[$i]}"
    done
    echo "$result"
}

join_array " | " "${fruits[@]}"   # apple | banana | cherry
join_array $'\n' "${fruits[@]}"   # แต่ละอันบรรทัดใหม่
```

---

## 3.7 Trim & Clean

```bash
# Trim whitespace (leading/trailing)
str="   Hello, World!   "

# Trim leading spaces
trimmed_left="${str#"${str%%[![:space:]]*}"}"
echo "'$trimmed_left'"          # 'Hello, World!   '

# Trim trailing spaces
trimmed_right="${str%"${str##*[![:space:]]}"}"
echo "'$trimmed_right'"         # '   Hello, World!'

# Trim both (simple)
trimmed=$(echo "$str" | xargs)
echo "'$trimmed'"               # 'Hello, World!'

# Trim both (bash only)
trim() {
    local var="$1"
    # leading
    var="${var#"${var%%[![:space:]]*}"}"
    # trailing
    var="${var%"${var##*[![:space:]]}"}"
    echo "$var"
}
echo "$(trim "$str")"

# Remove all whitespace
echo "$str" | tr -d ' \t'
echo "${str// /}"               # ลบ space

# Squeeze multiple spaces to one
echo "hello   world   foo" | tr -s ' '   # hello world foo

# Remove special characters
echo "Hello! @#$%^&*()" | tr -cd '[:alnum:] '    # Hello 
echo "hello\nworld" | tr -d '\n'                  # helloworld

# Normalize line endings
sed 's/\r$//' windows_file.txt    # Convert CRLF to LF
sed -i 's/\r//' file.txt          # In-place

# Remove blank lines
sed '/^[[:space:]]*$/d' file.txt

# Remove comments
sed '/^#/d' config.txt
sed 's/#.*//' config.txt    # ลบ inline comments ด้วย
```

---

## 3.8 Pattern Matching & Extraction

```bash
# Glob patterns ใน [[ ]]
filename="report_2024_01_final.pdf"

[[ "$filename" == *.pdf ]]        && echo "PDF file"
[[ "$filename" == report_* ]]     && echo "Report file"
[[ "$filename" == *2024* ]]       && echo "Year 2024"
[[ "$filename" == *_final.* ]]    && echo "Final version"

# Regex ใน [[ =~ ]]
email="user@example.com"
phone="555-123-4567"
ip="192.168.1.100"
date="2024-01-15"

# Email validation
[[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]] \
    && echo "Valid email"

# Phone validation
[[ "$phone" =~ ^[0-9]{3}-[0-9]{3}-[0-9]{4}$ ]] \
    && echo "Valid US phone"

# IP validation
[[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] \
    && echo "Valid IP format"

# Date validation
[[ "$date" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]] \
    && echo "Valid date format"

# Extract matches with BASH_REMATCH
text="Error: Connection timeout at 14:32:55"
if [[ "$text" =~ ([0-9]{2}):([0-9]{2}):([0-9]{2}) ]]; then
    echo "Full match: ${BASH_REMATCH[0]}"   # 14:32:55
    echo "Hour:   ${BASH_REMATCH[1]}"       # 14
    echo "Minute: ${BASH_REMATCH[2]}"       # 32
    echo "Second: ${BASH_REMATCH[3]}"       # 55
fi

# grep for extraction
echo "Error code: 404 - Not Found" | grep -oP '\d+'    # 404
echo "Name: John Doe" | grep -oP '(?<=Name: ).*'       # John Doe

# sed for extraction
echo "IP: 192.168.1.1 found" | sed -n 's/.*IP: \([0-9.]*\).*/\1/p'
```

---

## 3.9 Advanced String Processing

```bash
# String padding
printf "%-20s%s\n" "Name:" "John Doe"
printf "%-20s%d\n" "Age:" 25
printf "%20s\n" "right aligned"

# Number formatting
printf "%05d\n" 42            # 00042
printf "%.2f\n" 3.14159       # 3.14
printf "%+d\n" 42             # +42
printf "%e\n" 12345.678       # 1.234568e+04
printf "%x\n" 255             # ff (hex)
printf "%o\n" 8               # 10 (octal)
printf "%b\n" 10              # 2 (interpret octal escape)

# String repetition
printf '%*s\n' 20 '' | tr ' ' '-'   # --------------------
printf '=%.0s' {1..40}; echo        # ========================================

# Word count / char count
text="The quick brown fox"
echo "Words: $(echo "$text" | wc -w)"
echo "Chars: $(echo "$text" | wc -c)"
echo "Lines: $(echo "$text" | wc -l)"

# Unique words
echo "$text" | tr ' ' '\n' | sort -u

# Count occurrences
echo "banana" | grep -o "a" | wc -l    # count 'a' in banana = 3

# Anagram check
is_anagram() {
    local word1=$(echo "$1" | grep -o . | sort | tr -d '\n')
    local word2=$(echo "$2" | grep -o . | sort | tr -d '\n')
    [[ "$word1" == "$word2" ]] && echo "Anagram!" || echo "Not anagram"
}
is_anagram "listen" "silent"    # Anagram!
is_anagram "hello" "world"      # Not anagram

# Palindrome check
is_palindrome() {
    local str=$(echo "$1" | tr '[:upper:]' '[:lower:]' | tr -cd '[:alnum:]')
    [[ "$str" == "$(echo "$str" | rev)" ]] && echo "Palindrome" || echo "Not palindrome"
}
is_palindrome "racecar"         # Palindrome
is_palindrome "hello"           # Not palindrome
is_palindrome "A man a plan a canal Panama"  # Palindrome
```

---

## 3.10 Here Documents & Here Strings

```bash
# Here Document (heredoc)
cat << 'EOF'
This text will not expand:
$HOME $USER ${PATH}
EOF

cat << EOF
This text WILL expand:
Home: $HOME
User: $USER
Date: $(date)
EOF

# Indented heredoc (Bash 4+, ใช้ dash -)
cat <<-EOF
    This line has leading tabs stripped
    But spaces are kept
    $(date)
EOF

# Heredoc ใน function
generate_config() {
    local host=$1
    local port=$2
    cat << EOF
server {
    listen $port;
    server_name $host;
    root /var/www/${host};
    
    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOF
}

generate_config "example.com" 80

# Heredoc to file
cat > /tmp/config.txt << 'EOF'
# Configuration file
KEY1=value1
KEY2=value2
EOF

# Here String
read var <<< "hello world"
echo $var

wc -w <<< "count these words"   # 3

grep "pattern" <<< "text with pattern inside"

# Process heredoc line by line
while IFS= read -r line; do
    echo "Line: $line"
done << 'EOF'
First line
Second line
Third line
EOF
```

---

## 3.11 String Encoding & Escaping

```bash
# URL Encoding
url_encode() {
    local string=$1
    local encoded=""
    local pos c o
    for (( pos=0 ; pos<${#string} ; pos++ )); do
        c=${string:$pos:1}
        case "$c" in
            [-_.~a-zA-Z0-9]) o="$c" ;;
            *) printf -v o '%%%02x' "'$c" ;;
        esac
        encoded+="$o"
    done
    echo "$encoded"
}

# URL Decoding
url_decode() {
    local url_encoded="${1//+/ }"
    printf '%b' "${url_encoded//%/\\x}"
}

encoded=$(url_encode "Hello, World! This is a test@email.com")
echo "Encoded: $encoded"
echo "Decoded: $(url_decode "$encoded")"

# Base64
echo "Hello, World!" | base64                    # Encode
echo "SGVsbG8sIFdvcmxkIQo=" | base64 -d         # Decode

# Hex encoding
echo "hello" | xxd                               # Hex dump
echo "hello" | xxd -p                           # Plain hex
echo "68656c6c6f" | xxd -r -p                   # Hex to ascii

# HTML Entities
html_encode() {
    echo "$1" | sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g; s/"/\&quot;/g'
}
html_encode "<script>alert('XSS')</script>"

# Escape for various contexts
# Shell escape
printf '%q' "hello world 'quotes'"

# JSON string escape
echo "Hello \"World\"" | python3 -c "import json,sys; print(json.dumps(sys.stdin.read().rstrip()))"

# Regex escape
escape_regex() {
    echo "$1" | sed 's/[.[\*^$()+?{}|]/\\&/g'
}
escaped=$(escape_regex "file.txt (version 1.0)")
echo "$escaped"
```

---

## 3.12 Multi-line String Processing

```bash
# Count lines
text="line1
line2
line3"
echo "${text}" | wc -l

# Get specific line
echo "${text}" | sed -n '2p'           # line 2
echo "${text}" | awk 'NR==2'           # same
mapfile -t lines <<< "$text"           # to array
echo "${lines[1]}"                     # index 1 = line 2

# Reverse lines
echo "$text" | tac
echo "$text" | sed -n '1!G;h;$p'      # POSIX version

# Sort lines
echo "$text" | sort
echo "$text" | sort -r                 # reverse
echo "$text" | sort -u                 # unique
echo "$text" | sort -n                 # numeric

# Filter lines
echo "$text" | grep "pattern"
echo "$text" | grep -v "pattern"       # exclude
echo "$text" | grep -n "pattern"       # with line numbers

# Process each line
while IFS= read -r line; do
    echo "Processing: $line"
done <<< "$text"

# Add line numbers
nl "$text"
cat -n <<< "$text"
awk '{print NR": "$0}' <<< "$text"

# Remove duplicate lines
echo "$text" | sort | uniq
echo "$text" | awk '!seen[$0]++'    # ไม่ต้อง sort

# Get lines between patterns
echo "$text" | sed -n '/START/,/END/p'
awk '/START/,/END/' <<< "$text"
```

---

## 3.13 String Utilities Script

```bash
#!/bin/bash
# string_utils.sh - Collection of string utility functions

# Repeat a string N times
repeat_str() {
    local str=$1
    local n=$2
    printf "%${n}s" | tr ' ' "$str"
}

# Center text in given width
center_text() {
    local text=$1
    local width=${2:-80}
    local len=${#text}
    local pad=$(( (width - len) / 2 ))
    printf "%${pad}s%s%${pad}s\n" "" "$text" ""
}

# Box a string
box_text() {
    local text=$1
    local len=${#text}
    local border=$(repeat_str "─" $((len + 2)))
    echo "┌${border}┐"
    echo "│ ${text} │"
    echo "└${border}┘"
}

# Title case
title_case() {
    echo "$1" | awk '{for(i=1;i<=NF;i++) $i=toupper(substr($i,1,1)) substr($i,2); print}'
}

# Camel case to snake case
camel_to_snake() {
    echo "$1" | sed 's/\([A-Z]\)/_\L\1/g' | sed 's/^_//'
}

# Snake case to camel case
snake_to_camel() {
    echo "$1" | awk -F_ '{for(i=2;i<=NF;i++) $i=toupper(substr($i,1,1)) substr($i,2); print}' OFS=""
}

# Count occurrences of substring
count_occurrences() {
    local str=$1
    local sub=$2
    echo "${str}" | grep -o "$sub" | wc -l
}

# Truncate with ellipsis
truncate_str() {
    local str=$1
    local max=$2
    if [[ ${#str} -gt $max ]]; then
        echo "${str:0:$((max-3))}..."
    else
        echo "$str"
    fi
}

# String contains any of
contains_any() {
    local str=$1
    shift
    for pattern in "$@"; do
        [[ "$str" == *"$pattern"* ]] && return 0
    done
    return 1
}

# Demo
echo "=== String Utilities Demo ==="
echo ""
echo "Repeat: $(repeat_str '*' 20)"
center_text "Hello, World!"
box_text "Important Message"
echo "Title Case: $(title_case "hello world foo bar")"
echo "Camel→Snake: $(camel_to_snake "myVariableName")"
echo "Snake→Camel: $(snake_to_camel "my_variable_name")"
echo "Count 'a' in banana: $(count_occurrences "banana" "a")"
echo "Truncate: $(truncate_str "This is a very long string that should be truncated" 30)"

if contains_any "Hello World" "World" "Planet" "Universe"; then
    echo "Contains a match!"
fi
```

---

## 3.14 Real-world Examples

### Log Parser

```bash
#!/bin/bash
# log_parser.sh - Parse Apache/Nginx access logs

# Format: IP - - [date] "METHOD path HTTP/ver" status bytes
LOG_FILE="${1:-/var/log/nginx/access.log}"

if [[ ! -f "$LOG_FILE" ]]; then
    echo "Log file not found: $LOG_FILE"
    exit 1
fi

echo "=== Log Analysis: $LOG_FILE ==="
echo ""

# Total requests
total=$(wc -l < "$LOG_FILE")
echo "Total requests: $total"

# Status code breakdown
echo ""
echo "Status Code Breakdown:"
awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -rn | \
    awk '{printf "  HTTP %-5s: %d requests\n", $2, $1}'

# Top 10 IPs
echo ""
echo "Top 10 IP Addresses:"
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10 | \
    awk '{printf "  %-15s: %d requests\n", $2, $1}'

# Top 10 URLs
echo ""
echo "Top 10 Requested URLs:"
awk '{print $7}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10 | \
    awk '{printf "  %d\t%s\n", $1, $2}'

# Error requests (4xx, 5xx)
errors=$(awk '$9 ~ /^[45]/' "$LOG_FILE" | wc -l)
echo ""
echo "Error requests (4xx/5xx): $errors ($(( errors * 100 / total ))%)"

# Bandwidth
total_bytes=$(awk '{sum += $10} END {print sum}' "$LOG_FILE")
echo "Total bandwidth: $(echo "scale=2; $total_bytes/1048576" | bc) MB"
```

### Config File Parser

```bash
#!/bin/bash
# ini_parser.sh - Parse INI-style config files

declare -A CONFIG

parse_ini() {
    local file=$1
    local section=""
    
    while IFS= read -r line; do
        # Skip comments and empty lines
        [[ "$line" =~ ^[[:space:]]*[#;] ]] && continue
        [[ -z "${line// /}" ]] && continue
        
        # Section header
        if [[ "$line" =~ ^\[([^\]]+)\]$ ]]; then
            section="${BASH_REMATCH[1]}"
            continue
        fi
        
        # Key=Value
        if [[ "$line" =~ ^([^=]+)=(.*)$ ]]; then
            local key="${BASH_REMATCH[1]// /}"
            local value="${BASH_REMATCH[2]}"
            # Trim value
            value="${value#"${value%%[![:space:]]*}"}"
            value="${value%"${value##*[![:space:]]}"}"
            # Remove quotes
            value="${value#\"}"
            value="${value%\"}"
            value="${value#\'}"
            value="${value%\'}"
            
            if [[ -n "$section" ]]; then
                CONFIG["${section}.${key}"]="$value"
            else
                CONFIG["$key"]="$value"
            fi
        fi
    done < "$file"
}

# Create test config
cat > /tmp/test.ini << 'EOF'
# Application Config

[database]
host = localhost
port = 5432
name = myapp_db
user = dbuser
password = secret123

[server]
host = 0.0.0.0
port = 8080
debug = false
workers = 4

[logging]
level = INFO
file = /var/log/app.log
EOF

parse_ini /tmp/test.ini

echo "Database config:"
echo "  Host: ${CONFIG[database.host]}"
echo "  Port: ${CONFIG[database.port]}"
echo "  Name: ${CONFIG[database.name]}"

echo ""
echo "Server config:"
echo "  Host: ${CONFIG[server.host]}"
echo "  Port: ${CONFIG[server.port]}"
echo "  Debug: ${CONFIG[server.debug]}"
```

---

## 3.15 Exercises

### Exercise 1: String Reverse
เขียนฟังก์ชันที่ reverse string โดยไม่ใช้ `rev` command

### Exercise 2: Password Validator
เขียน script ตรวจสอบ password ว่า:
- ยาวอย่างน้อย 8 ตัวอักษร
- มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
- มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว
- มีตัวเลขอย่างน้อย 1 ตัว
- มี special character อย่างน้อย 1 ตัว

### Exercise 3: CSV Parser
เขียน script ที่ parse CSV file และแสดงข้อมูลเป็นตาราง

### Exercise 4: Text Statistics
สร้าง script วิเคราะห์ text file แสดง:
- จำนวนบรรทัด, คำ, ตัวอักษร
- คำที่ซ้ำมากที่สุด 10 อันดับ
- ความยาวคำเฉลี่ย
- ประโยคที่ยาวที่สุด

---

## สรุป Part 03

✅ String basics (single/double quotes)  
✅ Length, substring, character access  
✅ Comparison (==, <, >, =~)  
✅ Conversion (case, encoding)  
✅ Search & Replace (sed, parameter expansion)  
✅ Split & Join  
✅ Trim & Clean  
✅ Pattern matching & regex  
✅ Here documents & strings  
✅ Encoding (URL, Base64, Hex)  

---

**→ Part 04: Arrays & Associative Arrays**
