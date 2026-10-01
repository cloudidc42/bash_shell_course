# Part 08: Input/Output & Redirection
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 8.1 Standard Streams

```bash
# Linux มี 3 standard streams:
# stdin  (0) - Standard Input  → keyboard / pipe
# stdout (1) - Standard Output → terminal screen
# stderr (2) - Standard Error  → terminal screen (error messages)

# File Descriptors:
# 0 = stdin
# 1 = stdout
# 2 = stderr
# 3+ = custom

# ดู file descriptors ของ process
ls -la /proc/$$/fd

# ตัวอย่าง
echo "This goes to stdout"
echo "This is an error" >&2
read -r input < /dev/stdin
```

---

## 8.2 Output Redirection

```bash
# Redirect stdout to file
echo "Hello" > file.txt          # overwrite
echo "World" >> file.txt         # append

# Redirect stderr to file
ls /nonexistent 2> error.txt
ls /nonexistent 2>> errors.log   # append

# Redirect both stdout and stderr
command > out.txt 2> err.txt     # separate files
command > all.txt 2>&1           # both to same file
command &> all.txt               # bash shorthand (same)
command >> all.txt 2>&1          # append both

# Redirect stdout to stderr
echo "Error message" 1>&2
echo "Error message" >&2

# Discard output (send to /dev/null)
command > /dev/null              # discard stdout
command 2> /dev/null             # discard stderr
command &> /dev/null             # discard all
command > /dev/null 2>&1         # discard all (POSIX)

# Common patterns
./build.sh 2>&1 | tee build.log  # show and save
wget url -q -O file.txt          # quiet download
grep -r pattern dir 2>/dev/null  # suppress "permission denied"
```

---

## 8.3 Input Redirection

```bash
# Redirect file to stdin
sort < unsorted.txt
wc -l < file.txt
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# Process substitution as input
diff <(command1) <(command2)
diff <(sort file1.txt) <(sort file2.txt)
wc -l <(find /etc -name "*.conf")

# Here Document (heredoc)
cat << EOF
Line 1
Line 2
Date: $(date)
EOF

# Heredoc to file
cat > config.txt << EOF
[server]
host=localhost
port=8080
EOF

# Here String
grep "pattern" <<< "text with pattern"
wc -w <<< "count these words"
base64 <<< "encode this"

# Multiple inputs with process substitution
paste <(seq 1 5) <(echo -e "a\nb\nc\nd\ne")
```

---

## 8.4 Pipes

```bash
# Basic pipe: stdout → stdin
ls -l | grep ".txt"
ps aux | grep nginx
cat file.txt | sort | uniq | wc -l

# Named pipes (FIFO)
mkfifo /tmp/mypipe

# Terminal 1: write to pipe
echo "Hello through pipe" > /tmp/mypipe

# Terminal 2: read from pipe
cat /tmp/mypipe

# Cleanup
rm /tmp/mypipe

# Bidirectional pipe with coproc (Bash 4+)
coproc myproc { python3 -c "
import sys
for line in sys.stdin:
    print(line.strip().upper())
    sys.stdout.flush()
"; }

echo "hello" >&"${myproc[1]}"
read response <&"${myproc[0]}"
echo "Response: $response"   # HELLO

# tee - split output
command | tee file.txt | next_command
echo "data" | tee /tmp/a.txt | tee /tmp/b.txt | wc -l

# Pipeline with error handling
set -o pipefail  # fail if any pipe segment fails

# Check pipeline exit codes
true | false | true
echo "${PIPESTATUS[@]}"   # 0 1 0

if ! cat file | grep "pattern" | sort > output; then
    echo "Pipeline failed"
fi
```

---

## 8.5 File Descriptors

```bash
# Open file descriptors
exec 3> output.txt       # open fd 3 for writing
exec 4< input.txt        # open fd 4 for reading
exec 5<> rw_file.txt     # open fd 5 for read/write

# Write to fd 3
echo "Hello" >&3
printf "World\n" >&3

# Read from fd 4
read -r line <&4
echo "Read: $line"

# Close file descriptors
exec 3>&-    # close write fd
exec 4<&-    # close read fd
exec 5<>&-   # close read/write fd

# Redirect multiple outputs
exec 1> >(tee -a stdout.log)     # stdout to screen + log
exec 2> >(tee -a stderr.log >&2) # stderr to screen + log

# Save and restore stdout
exec 3>&1        # save stdout to fd 3
exec 1>/dev/null # redirect stdout to /dev/null
echo "This is suppressed"
exec 1>&3        # restore stdout
exec 3>&-        # close fd 3
echo "This is visible again"

# Duplicate file descriptors
exec 6>&1        # fd 6 = copy of stdout
exec 7>&2        # fd 7 = copy of stderr

# Complex redirection
{
    echo "stdout"
    echo "stderr" >&2
} 1>stdout.txt 2>stderr.txt

# Swap stdout and stderr
{
    echo "originally stdout"
    echo "originally stderr" >&2
} 3>&1 1>&2 2>&3 3>&-
```

---

## 8.6 read Command

```bash
# Basic read
read -r name
echo "You entered: $name"

# Read with prompt
read -rp "Enter your name: " name

# Read multiple variables
read -r first last <<< "John Doe"
echo "First: $first, Last: $last"

# Read from pipe
echo "hello world" | read -r word1 word2

# Read with timeout
read -rt 5 -p "Enter (5s timeout): " input || echo "Timeout!"

# Read password (no echo)
read -rsp "Password: " password
echo ""  # newline after password

# Read into array
read -ra words <<< "one two three four"
echo "${words[@]}"

# Read single character (no Enter needed)
read -rn1 -p "Press any key: " key
echo ""
echo "You pressed: $key"

# Read from file descriptor
read -r line <&3

# Read with delimiter
IFS=',' read -ra csv_fields <<< "field1,field2,field3"

# Read with custom IFS
IFS=':' read -r user _ uid _ <<< "root:x:0:0:root:/root:/bin/bash"
echo "User: $user, UID: $uid"

# Read heredoc
while IFS= read -r line; do
    echo "[$line]"
done << 'HEREDOC'
Line 1
Line 2
Line 3
HEREDOC

# Interactive menu
show_menu() {
    echo "1) Option A"
    echo "2) Option B"
    echo "3) Exit"
    read -rp "Choice: " choice
    
    case $choice in
        1) echo "Option A selected" ;;
        2) echo "Option B selected" ;;
        3) echo "Bye!"; exit 0 ;;
        *) echo "Invalid choice" ;;
    esac
}
```

---

## 8.7 printf vs echo

```bash
# echo - simple, inconsistent across systems
echo "Hello"
echo -e "Tab:\there"    # -e for escape sequences
echo -n "No newline"    # -n suppress newline

# printf - powerful, consistent, recommended
printf "Hello\n"
printf "%s\n" "Hello"
printf "Name: %-20s Age: %d\n" "Alice" 30

# printf format specifiers
printf "%s"    # string
printf "%d"    # decimal integer
printf "%f"    # float
printf "%e"    # scientific notation
printf "%x"    # hex (lowercase)
printf "%X"    # hex (uppercase)
printf "%o"    # octal
printf "%b"    # interpret backslash escapes

# Width and precision
printf "%10s"      # right-aligned, width 10
printf "%-10s"     # left-aligned, width 10
printf "%010d"     # zero-padded
printf "%.2f"      # 2 decimal places
printf "%10.2f"    # width 10, 2 decimal places

# Multiple values
printf "%s is %d years old\n" Alice 30
printf "%s is %d years old\n" Bob 25

# printf to variable
printf -v result "Hello, %s!" "World"
echo "$result"

# Useful patterns
printf "%-30s %5d bytes\n" "$filename" "$size"
printf "Progress: [%-50s] %d%%\r" "$(printf '#%.0s' $(seq 1 $progress))" $percent
printf "\033[1;31m%s\033[0m\n" "Red bold text"

# Table formatting
printf "+%-20s+%-10s+%-15s+\n" "$(printf '%.0s-' {1..20})" "$(printf '%.0s-' {1..10})" "$(printf '%.0s-' {1..15})"
printf "| %-19s| %-9s| %-14s|\n" "Name" "Age" "Email"
printf "+%-20s+%-10s+%-15s+\n" "$(printf '%.0s-' {1..20})" "$(printf '%.0s-' {1..10})" "$(printf '%.0s-' {1..15})"
printf "| %-19s| %-9d| %-14s|\n" "Alice" 30 "alice@ex.com"
```

---

## 8.8 Advanced I/O Patterns

```bash
# Logging with timestamps
log_to_file() {
    local log_file=$1
    while IFS= read -r line; do
        printf "[%s] %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$line"
    done >> "$log_file"
}

# Usage: command 2>&1 | log_to_file /var/log/app.log

# Multi-output (stdout + log + stderr)
setup_logging() {
    local log_dir="/var/log/myapp"
    mkdir -p "$log_dir"
    
    # All stdout goes to stdout AND log file
    exec 1> >(tee -a "$log_dir/stdout.log")
    # All stderr goes to stderr AND error log
    exec 2> >(tee -a "$log_dir/stderr.log" >&2)
}

# Reading JSON from API
get_json_field() {
    local url=$1
    local field=$2
    
    curl -sf "$url" | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(data.get('$field', ''))
"
}

# Piped processing pipeline
process_logs() {
    local log_file=$1
    local from_date=$2
    local to_date=$3
    
    grep -E "^$from_date|^$to_date" "$log_file" | \
    grep -v "DEBUG" | \
    awk '{print $1, $2, $NF}' | \
    sort | \
    uniq -c | \
    sort -rn | \
    head -20
}

# Asynchronous I/O with coproc
start_background_processor() {
    coproc BG_PROC {
        while IFS= read -r cmd; do
            case $cmd in
                "compute:"*)
                    value="${cmd#compute:}"
                    result=$((value * value))
                    echo "result:$result"
                    ;;
                "quit") break ;;
            esac
        done
    }
    
    # Use BG_PROC[0] to read, BG_PROC[1] to write
}

# Timeout with read
read_with_timeout() {
    local timeout=$1
    local result
    
    if read -rt "$timeout" result; then
        echo "$result"
        return 0
    else
        return 1
    fi
}
```

---

## 8.9 stdin/stdout/stderr Best Practices

```bash
#!/usr/bin/env bash
# Template สำหรับ script ที่จัดการ I/O อย่างถูกต้อง

# Strict mode
set -euo pipefail

# Output functions
info()    { printf "[INFO]  %s\n" "$*" >&1; }
success() { printf "[OK]    %s\n" "$*" >&1; }
warn()    { printf "[WARN]  %s\n" "$*" >&2; }
error()   { printf "[ERROR] %s\n" "$*" >&2; }
debug()   { [[ "${DEBUG:-}" == "true" ]] && printf "[DEBUG] %s\n" "$*" >&2; }

# Usage message (always to stderr)
usage() {
    cat >&2 << EOF
Usage: $(basename "$0") [OPTIONS] FILE

Process input FILE and produce output.

Options:
  -h, --help          Show this help
  -v, --verbose       Enable verbose output
  -o, --output FILE   Write output to FILE (default: stdout)
  -e, --errors FILE   Write errors to FILE (default: stderr)

Examples:
  $(basename "$0") input.txt
  $(basename "$0") -o output.txt input.txt
  cat input.txt | $(basename "$0") -
EOF
}

# Parse arguments
OUTPUT=""
ERROR_LOG=""
VERBOSE=false
INPUT=""

while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)    usage; exit 0 ;;
        -v|--verbose) VERBOSE=true; shift ;;
        -o|--output)  OUTPUT="$2"; shift 2 ;;
        -e|--errors)  ERROR_LOG="$2"; shift 2 ;;
        -)            INPUT="-"; shift ;;
        -*)           error "Unknown option: $1"; usage; exit 1 ;;
        *)            INPUT="$1"; shift ;;
    esac
done

# Setup output redirection
if [[ -n "$OUTPUT" ]]; then
    exec 1>"$OUTPUT"
fi
if [[ -n "$ERROR_LOG" ]]; then
    exec 2>"$ERROR_LOG"
fi

# Determine input source
if [[ -z "$INPUT" ]]; then
    if [[ -t 0 ]]; then
        error "No input provided (stdin is a terminal)"
        usage
        exit 1
    fi
    INPUT="-"
fi

# Process input
process_input() {
    local source=$1
    local line_num=0
    
    while IFS= read -r line; do
        ((line_num++))
        debug "Processing line $line_num: $line"
        
        # Your processing here
        echo "${line^^}"
    done < <(
        if [[ "$source" == "-" ]]; then
            cat
        else
            cat "$source"
        fi
    )
}

process_input "$INPUT"
info "Processing complete"
```

---

## 8.10 /dev/null, /dev/zero, /dev/urandom

```bash
# /dev/null - blackhole (discard)
echo "discard this" > /dev/null
command > /dev/null 2>&1

# /dev/zero - infinite zeros
dd if=/dev/zero of=zero_file bs=1M count=10    # create 10MB file of zeros
dd if=/dev/zero bs=1M | head -c 100M > big_file

# /dev/urandom - random bytes
# Generate random password
cat /dev/urandom | tr -dc 'A-Za-z0-9!@#$%^&*' | head -c 16
openssl rand -base64 24

# Random number
head -c 4 /dev/urandom | od -An -tu4 | tr -d ' '

# Generate random file
dd if=/dev/urandom of=random_file bs=1k count=1

# /dev/stdin, /dev/stdout, /dev/stderr
cat /dev/stdin < file.txt    # same as cat file.txt
echo "test" > /dev/stderr    # write to stderr

# /dev/tcp, /dev/udp (bash built-in networking)
# HTTP request
exec 3<>/dev/tcp/httpbin.org/80
echo -e "GET /ip HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n" >&3
cat <&3
exec 3>&-

# Check if port is open
check_port() {
    local host=$1
    local port=$2
    (echo >/dev/tcp/$host/$port) &>/dev/null && echo "open" || echo "closed"
}

check_port google.com 443
check_port localhost 22
```

---

## 8.11 Logging Framework

```bash
#!/usr/bin/env bash
# Advanced logging framework

# =====================================================================
# Configuration
# =====================================================================
declare -r LOG_DIR="${LOG_DIR:-/tmp/bash_logs}"
declare -r LOG_FILE="${LOG_DIR}/app.log"
declare -r MAX_LOG_SIZE=$((10 * 1024 * 1024))  # 10MB
declare -r MAX_LOG_BACKUPS=5

# Log levels
declare -rA LOG_LEVELS=(
    [TRACE]=0
    [DEBUG]=1
    [INFO]=2
    [WARN]=3
    [ERROR]=4
    [FATAL]=5
)

declare -r CURRENT_LOG_LEVEL="${LOG_LEVEL:-INFO}"

# =====================================================================
# Internal functions
# =====================================================================
_log_rotate() {
    [[ -f "$LOG_FILE" ]] || return
    
    local size
    size=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)
    
    (( size < MAX_LOG_SIZE )) && return
    
    for ((i=MAX_LOG_BACKUPS-1; i>=1; i--)); do
        [[ -f "${LOG_FILE}.$i" ]] && mv "${LOG_FILE}.$i" "${LOG_FILE}.$((i+1))"
    done
    
    mv "$LOG_FILE" "${LOG_FILE}.1"
    gzip "${LOG_FILE}.1" &
}

_log_write() {
    local level=$1
    local message=$2
    local caller="${BASH_SOURCE[2]}:${BASH_LINENO[1]}"
    
    # Check log level
    local current_level=${LOG_LEVELS[$CURRENT_LOG_LEVEL]:-2}
    local msg_level=${LOG_LEVELS[$level]:-2}
    
    (( msg_level < current_level )) && return
    
    # Create log entry
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S.%3N')
    local pid=$$
    local entry="$timestamp [$level] [pid:$pid] [$caller] $message"
    
    # Ensure log directory exists
    mkdir -p "$LOG_DIR"
    
    # Check rotation
    _log_rotate
    
    # Write to file
    echo "$entry" >> "$LOG_FILE"
    
    # Also print to stderr for ERROR and FATAL
    if (( msg_level >= ${LOG_LEVELS[ERROR]} )); then
        echo "$entry" >&2
    fi
}

# =====================================================================
# Public API
# =====================================================================
log_trace() { _log_write "TRACE" "$*"; }
log_debug() { _log_write "DEBUG" "$*"; }
log_info()  { _log_write "INFO"  "$*"; }
log_warn()  { _log_write "WARN"  "$*"; }
log_error() { _log_write "ERROR" "$*"; }
log_fatal() { _log_write "FATAL" "$*"; exit 1; }

# Structured logging
log_json() {
    local level=$1
    shift
    local message=""
    
    # Build JSON
    local json="{"
    json+="\"timestamp\":\"$(date -Iseconds)\","
    json+="\"level\":\"$level\","
    
    while [[ $# -gt 0 ]]; do
        local key=$1
        local val=$2
        shift 2
        json+="\"$key\":\"$val\","
    done
    
    json="${json%,}}"
    
    echo "$json" >> "${LOG_DIR}/app.json.log"
}

# Usage example
mkdir -p "$LOG_DIR"

log_info "Application starting"
log_debug "Config loaded from /etc/app.conf"
log_warn "Memory usage above 80%"
log_error "Failed to connect to database"
log_json "INFO" "event" "user_login" "user" "alice" "ip" "192.168.1.1"
```

---

## 8.12 Exercises

### Exercise 1: Log Parser
สร้าง script ที่:
- อ่าน nginx access log จาก stdin หรือ file
- แสดงสถิติ: total requests, status codes, top IPs
- Output ไปที่ file ถ้าระบุ -o flag

### Exercise 2: Multi-pipe Chain
สร้าง pipeline ที่:
- อ่าน /etc/passwd
- กรองเฉพาะ users ที่ UID >= 1000
- แสดง username, UID, home directory
- เรียงตาม UID
- บันทึกทั้งหมดและแสดงด้วย tee

### Exercise 3: Interactive Input
สร้าง script สำหรับสร้าง user profile:
- ถามชื่อ, อายุ, email, password (hidden)
- Validate ทุก input
- แสดง summary
- บันทึกเป็น JSON

### Exercise 4: Logger
สร้าง logging system ที่:
- บันทึก log พร้อม timestamp
- Rotate เมื่อ > 1MB
- แสดงสีตาม log level
- รองรับ LOG_LEVEL environment variable

---

## สรุป Part 08

✅ Standard streams (stdin, stdout, stderr)  
✅ Output redirection (>, >>, 2>, &>)  
✅ Input redirection (<, <<, <<<)  
✅ Pipes (|, tee, named pipes)  
✅ File descriptors (exec, >&, <&)  
✅ read command (options, timeout, password)  
✅ printf vs echo  
✅ /dev/null, /dev/zero, /dev/urandom  
✅ /dev/tcp networking  
✅ Logging framework  

---

**→ Part 09: File Operations**
