# Part 05: Conditional Statements
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 5.1 if Statement พื้นฐาน

```bash
#!/bin/bash

# Syntax พื้นฐาน
if [ condition ]; then
    # commands
fi

# if-else
if [ condition ]; then
    # true branch
else
    # false branch
fi

# if-elif-else
if [ condition1 ]; then
    # branch 1
elif [ condition2 ]; then
    # branch 2
elif [ condition3 ]; then
    # branch 3
else
    # default branch
fi

# ตัวอย่างจริง
age=25

if [ $age -ge 18 ]; then
    echo "Adult"
fi

if [ $age -lt 0 ]; then
    echo "Invalid age"
elif [ $age -lt 13 ]; then
    echo "Child"
elif [ $age -lt 18 ]; then
    echo "Teenager"
elif [ $age -lt 65 ]; then
    echo "Adult"
else
    echo "Senior"
fi
```

---

## 5.2 Test Operators [ ] vs [[ ]] vs (( ))

```bash
# [ ] - POSIX test command (ใช้ได้ทุก shell)
[ -f file.txt ]       # file exists and is regular file
[ -d /tmp ]           # directory exists
[ "$str" = "hello" ]  # string equality (= ไม่ใช่ ==)
[ $num -eq 5 ]        # numeric equality

# [[ ]] - Bash extended test (แนะนำใน bash)
[[ -f file.txt ]]     # ปลอดภัยกว่า (ไม่ต้อง quote ทุกอย่าง)
[[ "$str" == hello ]] # ใช้ == ได้
[[ "$str" =~ ^[0-9]+$ ]]  # regex matching
[[ -f file && -r file ]]  # ใช้ && || โดยตรง

# (( )) - Arithmetic evaluation
(( 5 > 3 ))           # arithmetic comparison
(( i++ ))             # increment
(( result = 5 * 3 ))  # arithmetic assignment
if (( num > 0 )); then echo "positive"; fi

# ความแตกต่างสำคัญ
var="hello world"

# [ ] - ต้อง quote เสมอ
[ $var = "hello world" ]     # ERROR: too many arguments
[ "$var" = "hello world" ]   # OK

# [[ ]] - ไม่ต้อง quote (แต่ควร)
[[ $var = "hello world" ]]   # OK (word splitting ไม่เกิด)
[[ "$var" = "hello world" ]] # OK

# Pattern matching
[[ $var == hello* ]]         # glob pattern (ไม่ต้อง quote pattern)
[[ $var == "hello*" ]]       # literal * (ต้อง quote ถ้าต้องการ literal)
```

---

## 5.3 File Test Operators

```bash
FILE="/etc/passwd"
DIR="/tmp"
LINK="/etc/hostname"

# File existence
[ -e "$FILE" ]   && echo "exists"
[ -f "$FILE" ]   && echo "regular file"
[ -d "$DIR" ]    && echo "directory"
[ -l "$LINK" ]   && echo "symbolic link"
[ -p /tmp/fifo ] && echo "named pipe (FIFO)"
[ -S /tmp/sock ] && echo "socket"
[ -b /dev/sda ]  && echo "block device"
[ -c /dev/null ] && echo "character device"

# File permissions
[ -r "$FILE" ]   && echo "readable"
[ -w "$FILE" ]   && echo "writable"
[ -x "$FILE" ]   && echo "executable"
[ -u "$FILE" ]   && echo "setuid bit set"
[ -g "$FILE" ]   && echo "setgid bit set"
[ -k "$FILE" ]   && echo "sticky bit set"

# File content
[ -s "$FILE" ]   && echo "non-empty file"
[ ! -s "$FILE" ] && echo "empty file"

# File comparison
[ file1 -nt file2 ]  && echo "file1 newer than file2"
[ file1 -ot file2 ]  && echo "file1 older than file2"
[ file1 -ef file2 ]  && echo "same file (hardlink)"

# Ownership
[ -O "$FILE" ]   && echo "owned by current user"
[ -G "$FILE" ]   && echo "owned by current group"

# Practical example
check_file() {
    local file=$1
    echo "=== File: $file ==="
    
    if [ ! -e "$file" ]; then
        echo "  Does not exist"
        return 1
    fi
    
    [ -f "$file" ] && echo "  Type: Regular file"
    [ -d "$file" ] && echo "  Type: Directory"
    [ -l "$file" ] && echo "  Type: Symlink → $(readlink "$file")"
    
    [ -r "$file" ] && echo "  Read: yes" || echo "  Read: no"
    [ -w "$file" ] && echo "  Write: yes" || echo "  Write: no"
    [ -x "$file" ] && echo "  Execute: yes" || echo "  Execute: no"
    
    if [ -f "$file" ]; then
        [ -s "$file" ] && echo "  Size: $(stat -c%s "$file") bytes" \
                       || echo "  Size: empty"
    fi
}

check_file "/etc/passwd"
check_file "/tmp"
check_file "/nonexistent"
```

---

## 5.4 String Test Operators

```bash
# Null/Empty tests
str1=""
str2="hello"
unset str3

[ -z "$str1" ]  && echo "str1 is empty/zero length"
[ -n "$str2" ]  && echo "str2 is non-empty"
[ -z "${str3}" ] && echo "str3 is unset or empty"

# String comparison
[ "$str2" = "hello" ]   && echo "equal (POSIX)"
[[ "$str2" == "hello" ]] && echo "equal (bash)"
[ "$str2" != "world" ]  && echo "not equal"
[[ "apple" < "banana" ]] && echo "apple < banana (lexicographic)"
[[ "banana" > "apple" ]] && echo "banana > apple"

# Pattern matching (only in [[ ]])
[[ "hello123" =~ ^[a-z]+[0-9]+$ ]] && echo "letters then digits"
[[ "test.sh" == *.sh ]]             && echo "shell script"
[[ "ERROR: ..." == ERROR:* ]]       && echo "error line"

# Case insensitive
shopt -s nocasematch
[[ "Hello" == "hello" ]] && echo "case insensitive match"
shopt -u nocasematch

# Empty/unset distinction
var=""

# Is set? (even if empty)
[[ -v var ]] && echo "var is set"

# Is non-empty?
[[ -n "$var" ]] && echo "var is non-empty"

# Specific check
if [[ "${var+x}" == "x" ]]; then
    echo "var is set (possibly empty)"
else
    echo "var is unset"
fi
```

---

## 5.5 Numeric Comparison Operators

```bash
a=10
b=20

# In [ ] or test
[ $a -eq $b ]   && echo "equal"
[ $a -ne $b ]   && echo "not equal"
[ $a -lt $b ]   && echo "less than"
[ $a -le $b ]   && echo "less than or equal"
[ $a -gt $b ]   && echo "greater than"
[ $a -ge $b ]   && echo "greater than or equal"

# In (( )) - more readable
(( a == b ))    && echo "equal"
(( a != b ))    && echo "not equal"
(( a < b ))     && echo "less than"
(( a <= b ))    && echo "less than or equal"
(( a > b ))     && echo "greater than"
(( a >= b ))    && echo "greater than or equal"

# Float comparison (ต้องใช้ bc หรือ awk)
float_compare() {
    local a=$1
    local op=$2
    local b=$3
    local result=$(echo "$a $op $b" | bc -l)
    (( result == 1 ))
}

float_compare 3.14 ">" 2.71 && echo "pi > e"
float_compare 1.0  "==" 1.0 && echo "equal"
float_compare 2.5  "<=" 3.0 && echo "2.5 <= 3.0"

# awk version
awk_compare() {
    awk -v a="$1" -v b="$3" "BEGIN{exit !(a $2 b)}"
}

awk_compare 3.14 ">" 2.71 && echo "awk: pi > e"
```

---

## 5.6 Logical Operators

```bash
# AND
[ $age -ge 18 ] && [ $age -le 65 ]     # POSIX
[ $age -ge 18 -a $age -le 65 ]          # POSIX (deprecated)
[[ $age -ge 18 && $age -le 65 ]]        # Bash (preferred)

# OR
[ -f file1.txt ] || [ -f file2.txt ]    # POSIX
[ -f file1.txt -o -f file2.txt ]        # POSIX (deprecated)
[[ -f file1.txt || -f file2.txt ]]      # Bash (preferred)

# NOT
[ ! -f file.txt ]
[[ ! -d /tmp/dir ]]

# Complex conditions
name="Alice"
age=25
role="admin"

if [[ "$name" == "Alice" && ($age -ge 18 || "$role" == "admin") ]]; then
    echo "Access granted"
fi

# Short-circuit evaluation
# && : รัน right side เฉพาะ left side สำเร็จ
[ -f file.txt ] && cat file.txt

# || : รัน right side เฉพาะ left side ล้มเหลว
[ -d /backup ] || mkdir /backup
ping -c1 host &>/dev/null || echo "Host unreachable"

# Combined
mkdir -p /tmp/work && echo "Created" || echo "Failed"
[[ -f file ]] && rm file || echo "File not found"

# Practical: validate multiple conditions
validate_input() {
    local name=$1
    local age=$2
    local email=$3
    
    [[ -z "$name" ]] && { echo "Name required"; return 1; }
    [[ ${#name} -lt 2 ]] && { echo "Name too short"; return 1; }
    [[ ! "$age" =~ ^[0-9]+$ ]] && { echo "Age must be numeric"; return 1; }
    (( age < 0 || age > 150 )) && { echo "Invalid age range"; return 1; }
    [[ ! "$email" =~ ^[^@]+@[^@]+\.[^@]+$ ]] && { echo "Invalid email"; return 1; }
    
    echo "Valid input: $name, $age, $email"
    return 0
}

validate_input "Alice" "30" "alice@example.com"
validate_input "" "25" "test@test.com"
validate_input "Bob" "abc" "bob@test.com"
```

---

## 5.7 case Statement

```bash
# Basic case
day="Monday"
case $day in
    Monday)
        echo "Start of work week"
        ;;
    Tuesday|Wednesday|Thursday)
        echo "Mid week"
        ;;
    Friday)
        echo "TGIF!"
        ;;
    Saturday|Sunday)
        echo "Weekend!"
        ;;
    *)
        echo "Unknown day: $day"
        ;;
esac

# case with patterns
filename="image.jpg"
case "$filename" in
    *.jpg|*.jpeg|*.png|*.gif|*.webp)
        echo "Image file"
        ;;
    *.mp4|*.avi|*.mkv|*.mov)
        echo "Video file"
        ;;
    *.mp3|*.wav|*.flac|*.ogg)
        echo "Audio file"
        ;;
    *.pdf|*.doc|*.docx|*.txt)
        echo "Document file"
        ;;
    *.sh|*.bash|*.zsh)
        echo "Shell script"
        ;;
    *.py|*.rb|*.js|*.go)
        echo "Programming script"
        ;;
    *)
        echo "Unknown file type: $filename"
        ;;
esac

# case with ;;& (fall-through, Bash 4+)
char="a"
case $char in
    [aeiou])
        echo "vowel"
        ;;&    # continue matching (fall-through)
    [a-z])
        echo "lowercase letter"
        ;;&
    [[:alpha:]])
        echo "alphabetic character"
        ;;
    *)
        echo "other"
        ;;
esac

# case สำหรับ arguments
show_help() { echo "Usage: script.sh [-h] [-v] [-f file]"; }

parse_args() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)
                show_help
                exit 0
                ;;
            -v|--verbose)
                VERBOSE=true
                shift
                ;;
            -f|--file)
                FILE="$2"
                shift 2
                ;;
            -n|--name=*)
                if [[ "$1" == "--name="* ]]; then
                    NAME="${1#*=}"
                    shift
                else
                    NAME="$2"
                    shift 2
                fi
                ;;
            --)
                shift
                break
                ;;
            -*)
                echo "Unknown option: $1" >&2
                exit 1
                ;;
            *)
                ARGS+=("$1")
                shift
                ;;
        esac
    done
}
```

---

## 5.8 Conditional Expressions ขั้นสูง

```bash
# Ternary-style (short circuit)
result=$( [[ $age -ge 18 ]] && echo "adult" || echo "minor" )
echo $result

# Ternary with arithmetic
max=$(( a > b ? a : b ))
min=$(( a < b ? a : b ))
abs=$(( num >= 0 ? num : -num ))

# Select
PS3="Choose an option: "
options=("Start" "Stop" "Restart" "Status" "Quit")

select choice in "${options[@]}"; do
    case $choice in
        "Start")  echo "Starting...";  break ;;
        "Stop")   echo "Stopping...";  break ;;
        "Restart") echo "Restarting..."; break ;;
        "Status") echo "Running";      ;;
        "Quit")   echo "Bye!"; break   ;;
        *)        echo "Invalid option $REPLY" ;;
    esac
done

# Conditional assignment patterns
# Default if empty
name=${name:-"Anonymous"}

# Default and assign if unset/empty
: ${CONFIG_FILE:="/etc/app.conf"}

# Error if unset
echo "${REQUIRED_VAR:?'REQUIRED_VAR must be set'}"

# Conditional execution
[[ -f config.env ]] && source config.env
command -v docker &>/dev/null || { echo "Docker not installed"; exit 1; }

# chained conditionals
run_checks() {
    local target=$1
    
    # Check syntax: return early pattern
    [[ -z "$target" ]]     && { echo "No target"; return 1; }
    [[ ! -f "$target" ]]   && { echo "Not a file"; return 1; }
    [[ ! -r "$target" ]]   && { echo "Not readable"; return 1; }
    
    # All checks passed
    echo "Processing $target"
    return 0
}
```

---

## 5.9 Compound Commands

```bash
# Grouping with { } - runs in current shell
{
    echo "command 1"
    echo "command 2"
    echo "command 3"
} > output.txt

# Grouping with ( ) - runs in subshell
(
    cd /tmp
    ls
    # Changes to dir don't affect parent
)
echo "Still in: $PWD"

# Compound conditions
{
    check1 &&
    check2 &&
    check3
} || handle_error

# if/else in one line
if [[ $x -gt 0 ]]; then echo "positive"; else echo "non-positive"; fi

# Multiple statements after then
if [[ $debug == true ]]; then
    set -x
    log_level="DEBUG"
    verbose=true
fi

# Nested if
check_system() {
    if [[ $(id -u) -eq 0 ]]; then
        echo "Running as root"
        if [[ -f /etc/debian_version ]]; then
            echo "Debian/Ubuntu system"
        elif [[ -f /etc/redhat-release ]]; then
            echo "RHEL/CentOS system"
        else
            echo "Unknown Linux distribution"
        fi
    else
        echo "Not root - some operations may fail"
        if ! sudo -n true 2>/dev/null; then
            echo "No sudo access"
            return 1
        fi
    fi
}
```

---

## 5.10 Error Handling ใน Conditionals

```bash
#!/bin/bash

# Exit on error pattern
set -e    # ออกเมื่อ command ล้มเหลว

# But check specific commands
if ! command_that_might_fail; then
    echo "Command failed, handling error"
    # don't exit
fi

# Trap errors
trap 'echo "Error at line $LINENO"' ERR

# Guard clause pattern
validate_args() {
    local file=$1
    
    # Guard clauses - return early on failure
    if [[ $# -ne 1 ]]; then
        echo "Error: Exactly one argument required" >&2
        return 1
    fi
    
    if [[ ! -f "$file" ]]; then
        echo "Error: File not found: $file" >&2
        return 1
    fi
    
    if [[ ! -r "$file" ]]; then
        echo "Error: File not readable: $file" >&2
        return 1
    fi
    
    # Main logic
    echo "File is valid: $file"
    return 0
}

# Die on error helper
die() {
    echo "FATAL: $*" >&2
    exit 1
}

# Usage
[ -f /etc/config ] || die "Config file missing"
[ -d /var/log ]    || die "Log directory missing"

# Verbose checking
check_requirement() {
    local name=$1
    local check_cmd=$2
    
    printf "Checking %s... " "$name"
    if eval "$check_cmd" &>/dev/null; then
        echo "OK"
        return 0
    else
        echo "MISSING"
        return 1
    fi
}

check_requirement "bash 4+" "[[ ${BASH_VERSINFO[0]} -ge 4 ]]"
check_requirement "curl"    "command -v curl"
check_requirement "jq"      "command -v jq"
check_requirement "docker"  "command -v docker"
```

---

## 5.11 Conditional Script Template

```bash
#!/usr/bin/env bash
# =================================================================
# Script: system_check.sh
# Description: ตรวจสอบความพร้อมของระบบก่อนติดตั้ง application
# =================================================================

set -euo pipefail

# Colors
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'

pass()  { printf "${GREEN}✓${NC} %s\n" "$*"; }
fail()  { printf "${RED}✗${NC} %s\n" "$*"; }
warn()  { printf "${YELLOW}!${NC} %s\n" "$*"; }

# Check results tracker
ERRORS=0
WARNINGS=0

check() {
    local name=$1
    shift
    if eval "$@" &>/dev/null; then
        pass "$name"
    else
        fail "$name"
        ((ERRORS++))
    fi
}

soft_check() {
    local name=$1
    shift
    if eval "$@" &>/dev/null; then
        pass "$name"
    else
        warn "$name (optional)"
        ((WARNINGS++))
    fi
}

echo "System Requirements Check"
echo "========================="
echo ""

# OS Check
echo "--- Operating System ---"
if [[ -f /etc/os-release ]]; then
    . /etc/os-release
    pass "OS: $PRETTY_NAME"
else
    fail "Cannot determine OS"
    ((ERRORS++))
fi

# Architecture
ARCH=$(uname -m)
case $ARCH in
    x86_64)  pass "Architecture: x86_64 (64-bit)" ;;
    aarch64) pass "Architecture: ARM64" ;;
    armv7l)  warn "Architecture: ARM32 (limited support)" ;;
    *)       fail "Unsupported architecture: $ARCH"; ((ERRORS++)) ;;
esac

# Kernel version
KERNEL=$(uname -r | cut -d. -f1-2)
MIN_KERNEL="4.19"
if awk "BEGIN{exit !($KERNEL >= $MIN_KERNEL)}"; then
    pass "Kernel: $(uname -r)"
else
    fail "Kernel too old: $(uname -r) (minimum: $MIN_KERNEL)"
    ((ERRORS++))
fi

echo ""
echo "--- Hardware ---"

# Memory check
TOTAL_MEM=$(free -m | awk '/^Mem:/{print $2}')
if (( TOTAL_MEM >= 2048 )); then
    pass "RAM: ${TOTAL_MEM}MB (≥ 2GB)"
elif (( TOTAL_MEM >= 1024 )); then
    warn "RAM: ${TOTAL_MEM}MB (1GB minimum, 2GB recommended)"
    ((WARNINGS++))
else
    fail "RAM: ${TOTAL_MEM}MB (insufficient, minimum 1GB)"
    ((ERRORS++))
fi

# Disk space
FREE_DISK=$(df -BG / | awk 'NR==2{gsub(/G/,"",$4); print $4}')
if (( FREE_DISK >= 20 )); then
    pass "Disk: ${FREE_DISK}GB free (≥ 20GB)"
elif (( FREE_DISK >= 10 )); then
    warn "Disk: ${FREE_DISK}GB free (recommend 20GB+)"
    ((WARNINGS++))
else
    fail "Disk: ${FREE_DISK}GB free (insufficient, need 10GB+)"
    ((ERRORS++))
fi

echo ""
echo "--- Required Tools ---"
for tool in bash curl wget git tar gzip; do
    check "$tool" "command -v $tool"
done

echo ""
echo "--- Optional Tools ---"
for tool in jq vim tmux htop docker python3 node; do
    soft_check "$tool" "command -v $tool"
done

echo ""
echo "--- Network ---"
check "Internet connectivity" "ping -c1 -W3 8.8.8.8"
check "DNS resolution" "host google.com"
soft_check "HTTPS to GitHub" "curl -s --max-time 5 https://github.com"

echo ""
echo "========================="
echo "Results: $((ERRORS)) errors, $WARNINGS warnings"

if (( ERRORS > 0 )); then
    echo -e "${RED}FAILED: System does not meet requirements${NC}"
    exit 1
elif (( WARNINGS > 0 )); then
    echo -e "${YELLOW}PASSED WITH WARNINGS: Some optional requirements missing${NC}"
    exit 0
else
    echo -e "${GREEN}PASSED: All requirements met${NC}"
    exit 0
fi
```

---

## 5.12 Exercises

### Exercise 1: Number Validator
เขียนฟังก์ชัน `validate_number` ที่:
- ตรวจสอบว่าเป็นตัวเลข
- ตรวจสอบ range (min, max)
- ตรวจสอบว่าเป็น integer หรือ float

### Exercise 2: Menu System
สร้าง interactive menu ด้วย select ที่มีตัวเลือก:
1. Show system info
2. Show disk usage
3. Show network info
4. Show running processes
5. Exit

### Exercise 3: File Validator
เขียน script ที่ validate file:
- ตรวจสอบ existence
- ตรวจสอบ permissions
- ตรวจสอบ size (warning ถ้าใหญ่กว่า 100MB)
- ตรวจสอบ format ตาม extension

### Exercise 4: Port Checker
เขียน script ที่รับ host:port แล้ว:
- ตรวจสอบว่า host reachable
- ตรวจสอบว่า port open
- รายงานผลด้วย pass/fail

---

## สรุป Part 05

✅ if/elif/else syntax  
✅ Test operators [ ], [[ ]], (( ))  
✅ File tests (-f, -d, -r, -w, -x)  
✅ String tests (-z, -n, ==, =~)  
✅ Numeric comparisons (-eq, -gt, etc.)  
✅ Logical operators (&&, ||, !)  
✅ case statement (patterns, fall-through)  
✅ Conditional expressions ขั้นสูง  
✅ Guard clauses & error handling  

---

**→ Part 06: Loops (for, while, until)**
