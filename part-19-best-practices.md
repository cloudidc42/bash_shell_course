# Part 19: Script Best Practices & Production-Ready Scripts
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 19.1 Script Structure Template

```bash
#!/usr/bin/env bash
# =============================================================================
# script_name.sh - Brief description
#
# Usage: script_name.sh [OPTIONS] <argument>
#
# Options:
#   -h, --help      Show this help
#   -v, --verbose   Verbose output
#   -n, --dry-run   Dry run (don't make changes)
#   -c, --config    Config file path
#
# Examples:
#   script_name.sh -v input.txt
#   script_name.sh --config /etc/myapp.conf deploy
# =============================================================================

set -euo pipefail

# ─── Constants ────────────────────────────────────────────────
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
readonly SCRIPT_VERSION="1.0.0"
readonly TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# ─── Defaults ─────────────────────────────────────────────────
VERBOSE=false
DRY_RUN=false
CONFIG_FILE="${HOME}/.config/myapp/config.conf"
LOG_FILE="/var/log/myapp/${SCRIPT_NAME%.sh}.log"

# ─── Colors (only in terminal) ────────────────────────────────
if [[ -t 2 ]]; then
    RED='\033[0;31m'
    GREEN='\033[0;32m'
    YELLOW='\033[1;33m'
    BLUE='\033[0;34m'
    NC='\033[0m'
else
    RED='' GREEN='' YELLOW='' BLUE='' NC=''
fi

# ─── Logging ──────────────────────────────────────────────────
log_info()  { echo -e "${GREEN}[INFO]${NC}  $(date '+%T') $*" >&2; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC}  $(date '+%T') $*" >&2; }
log_error() { echo -e "${RED}[ERROR]${NC} $(date '+%T') $*" >&2; }
log_debug() { $VERBOSE && echo -e "${BLUE}[DEBUG]${NC} $(date '+%T') $*" >&2 || true; }
die()       { log_error "$*"; exit 1; }

# ─── Help ─────────────────────────────────────────────────────
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] <argument>

Brief description of what this script does.

OPTIONS:
    -h, --help          Show this help message
    -v, --verbose       Enable verbose output
    -n, --dry-run       Dry run mode (no changes)
    -c, --config FILE   Config file (default: $CONFIG_FILE)

ARGUMENTS:
    argument            Description of required argument

EXAMPLES:
    $SCRIPT_NAME -v input.txt
    $SCRIPT_NAME --dry-run --config /etc/app.conf deploy

VERSION: $SCRIPT_VERSION
EOF
}

# ─── Argument Parsing ─────────────────────────────────────────
parse_args() {
    local args=()
    
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)
                usage
                exit 0
                ;;
            -v|--verbose)
                VERBOSE=true
                ;;
            -n|--dry-run)
                DRY_RUN=true
                log_warn "Dry run mode enabled"
                ;;
            -c|--config)
                CONFIG_FILE="${2:?Config file required}"
                shift
                ;;
            --)
                shift
                args+=("$@")
                break
                ;;
            -*)
                die "Unknown option: $1"
                ;;
            *)
                args+=("$1")
                ;;
        esac
        shift
    done
    
    ARGS=("${args[@]}")
}

# ─── Validation ───────────────────────────────────────────────
validate() {
    # Check required arguments
    [[ ${#ARGS[@]} -gt 0 ]] || die "Missing required argument. Use --help for usage."
    
    # Check config file
    [[ -f "$CONFIG_FILE" ]] || die "Config file not found: $CONFIG_FILE"
    
    # Check required commands
    local cmds=(curl jq python3)
    for cmd in "${cmds[@]}"; do
        command -v "$cmd" &>/dev/null || die "Required command not found: $cmd"
    done
    
    # Check permissions
    [[ -w /var/log/myapp ]] || die "Cannot write to log directory"
}

# ─── Cleanup ──────────────────────────────────────────────────
TEMP_FILES=()

cleanup() {
    for f in "${TEMP_FILES[@]:-}"; do
        [[ -f "$f" ]] && rm -f "$f"
    done
    log_debug "Cleanup complete"
}

trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

# ─── Main ─────────────────────────────────────────────────────
main() {
    log_info "Starting $SCRIPT_NAME v$SCRIPT_VERSION"
    log_debug "Arguments: ${ARGS[*]:-none}"
    
    # Load config
    if [[ -f "$CONFIG_FILE" ]]; then
        # shellcheck source=/dev/null
        source "$CONFIG_FILE"
        log_debug "Loaded config: $CONFIG_FILE"
    fi
    
    # Create temp file (tracked for cleanup)
    local tmp_file
    tmp_file=$(mktemp)
    TEMP_FILES+=("$tmp_file")
    
    # Main logic
    if $DRY_RUN; then
        log_info "[DRY RUN] Would process: ${ARGS[0]}"
    else
        log_info "Processing: ${ARGS[0]}"
        # ... actual work ...
    fi
    
    log_info "Done"
}

# ─── Entry point ──────────────────────────────────────────────
parse_args "$@"
validate
main
```

---

## 19.2 Idiomatic Bash

```bash
# ─── Prefer [[ ]] over [ ] ────────────────────────────────────
# [ ] = POSIX, limited
# [[ ]] = bash-only, better

# Bad: quoting often required in [ ]
[ "$var" = "value" ]

# Good: safer, regex support, && || work correctly
[[ "$var" = "value" ]]
[[ "$var" =~ ^[0-9]+$ ]]
[[ -f "$file" && -r "$file" ]]

# ─── Use $() not backticks ────────────────────────────────────
# Old (backticks - hard to nest)
result=`command`

# Modern (preferred)
result=$(command)
result=$(outer "$(inner arg)")    # easy to nest

# ─── Printf over echo ─────────────────────────────────────────
# echo behavior varies by system/flag
echo -e "text"   # might not work everywhere

# printf is portable and predictable
printf "text\n"
printf "%s\n" "$var"
printf "%-20s %5d\n" "label" "$num"

# ─── Read safely ──────────────────────────────────────────────
# Process file lines safely
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Process output of command safely
while IFS= read -r line; do
    echo "$line"
done < <(command)

# ─── Arrays over strings ──────────────────────────────────────
# Bad: fragile with spaces
files="file1.txt file2.txt file with spaces.txt"
for f in $files; do echo "$f"; done  # breaks on "file with spaces"

# Good: use arrays
files=("file1.txt" "file2.txt" "file with spaces.txt")
for f in "${files[@]}"; do echo "$f"; done  # works correctly

# ─── Quote variables ──────────────────────────────────────────
# Bad: word splitting and glob expansion
cp $source $destination
rm $files

# Good: always quote
cp "$source" "$destination"
rm "${files[@]}"
rm -- "$file"   # -- prevents flags-as-filenames

# ─── Local variables in functions ─────────────────────────────
process() {
    local result        # local scope
    local -i count=0    # local integer
    local -a items=()   # local array
    
    result=$(compute)
    echo "$result"
}

# ─── Nameref for output parameters ───────────────────────────
get_value() {
    local -n result_ref=$1   # nameref
    result_ref="computed value"
}

get_value my_variable
echo "$my_variable"  # "computed value"

# ─── Default values ───────────────────────────────────────────
name=${1:-"default"}          # default if unset or empty
name=${1:?"Name required"}    # error if unset or empty
name=${1-"default"}           # default only if unset (not empty)

# ─── Glob safety ──────────────────────────────────────────────
# Be careful: ls *.txt fails if no .txt files
# Use nullglob or check first
shopt -s nullglob
files=(*.txt)
[[ ${#files[@]} -gt 0 ]] && echo "${files[@]}"

# Or use find
while IFS= read -r -d '' file; do
    echo "$file"
done < <(find . -name "*.txt" -print0)
```

---

## 19.3 Security Best Practices

```bash
# ─── Input Validation ─────────────────────────────────────────
validate_input() {
    local input=$1
    local type=$2
    
    case $type in
        filename)
            # Only allow safe chars in filenames
            [[ "$input" =~ ^[a-zA-Z0-9._-]+$ ]] || die "Invalid filename: $input"
            [[ "$input" != *..* ]] || die "Path traversal not allowed"
            ;;
        integer)
            [[ "$input" =~ ^-?[0-9]+$ ]] || die "Not an integer: $input"
            ;;
        alphanumeric)
            [[ "$input" =~ ^[a-zA-Z0-9]+$ ]] || die "Not alphanumeric: $input"
            ;;
        email)
            [[ "$input" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]] || \
                die "Invalid email: $input"
            ;;
    esac
}

# ─── Avoid eval ───────────────────────────────────────────────
# Never do this:
user_input="rm -rf /"
eval "$user_input"   # DANGEROUS

# Use arrays instead
cmd=(git commit -m "$message")
"${cmd[@]}"

# ─── Temp file security ───────────────────────────────────────
# Bad: predictable filename
tmp="/tmp/myapp.tmp"

# Good: use mktemp
tmp=$(mktemp)
tmp_dir=$(mktemp -d)
trap "rm -f '$tmp'; rm -rf '$tmp_dir'" EXIT

# ─── Avoid /tmp for secrets ───────────────────────────────────
# Use in-memory tmpfs if available
if [[ -d /dev/shm ]]; then
    secret_dir=$(mktemp -d /dev/shm/XXXXXX)
else
    secret_dir=$(mktemp -d)
fi
chmod 700 "$secret_dir"
trap "rm -rf '$secret_dir'" EXIT

# ─── SUID script considerations ───────────────────────────────
# Never write setuid shell scripts (they don't work securely anyway)
# Use setuid C wrapper if absolutely necessary

# ─── Avoid command injection ──────────────────────────────────
# Bad: variable in command string
ssh user@"$host" "ls $dir"    # dir could be "; rm -rf /"

# Good: use arrays, quote properly
ssh "user@${host}" "ls -- ${dir@Q}"

# Or validate first
[[ "$dir" =~ ^[/a-zA-Z0-9._-]+$ ]] || die "Invalid directory"
ssh "user@${host}" "ls -- $dir"

# ─── Secure file permissions ──────────────────────────────────
# Scripts with sensitive operations
chmod 700 sensitive_script.sh
chmod 600 config_with_passwords.conf

# Create private temp dir
create_private_dir() {
    local dir=$1
    mkdir -p "$dir"
    chmod 700 "$dir"
}

# ─── Handle sensitive output ──────────────────────────────────
# Don't log passwords or tokens
log_info "Connecting to database..."
# NOT: log_info "Password: $DB_PASSWORD"

# Mask in output
mask() { printf '%*s' "${#1}" | tr ' ' '*'; }
echo "Token: $(mask "$API_TOKEN")"
```

---

## 19.4 Performance Optimization

```bash
# ─── Avoid forks ──────────────────────────────────────────────
# Bad: fork for each iteration
while read -r line; do
    length=$(echo "$line" | wc -c)   # 2 forks per line!
done < file.txt

# Good: use bash built-ins
while IFS= read -r line; do
    length=${#line}   # no fork
done < file.txt

# ─── Batch operations ─────────────────────────────────────────
# Bad: process one at a time
for file in *.log; do
    gzip "$file"     # separate gzip call for each
done

# Better: batch
gzip *.log          # one call

# Even better: parallel
printf '%s\0' *.log | xargs -0 -P4 gzip

# ─── Use built-ins over external commands ─────────────────────
# Bad (external command)
path=$(dirname "$file")
base=$(basename "$file")

# Good (built-in parameter expansion)
path="${file%/*}"
base="${file##*/}"

# String operations
# Bad
length=$(echo "$str" | wc -c)
upper=$(echo "$str" | tr '[:lower:]' '[:upper:]')

# Good
length=${#str}
upper="${str^^}"    # bash 4+

# ─── Parallel execution ───────────────────────────────────────
# Simple parallel with &
for host in "${hosts[@]}"; do
    check_host "$host" &
done
wait   # wait for all

# With concurrency limit
MAX_JOBS=4
for task in "${tasks[@]}"; do
    process "$task" &
    while (( $(jobs -r | wc -l) >= MAX_JOBS )); do
        wait -n 2>/dev/null || sleep 0.1
    done
done
wait

# ─── String concatenation ─────────────────────────────────────
# Bad: O(n²) due to string copying
result=""
for i in {1..1000}; do
    result+="$i "   # append is actually ok in bash
done

# Better: use array and join
items=()
for i in {1..1000}; do
    items+=("$i")
done
result="${items[*]}"    # join with IFS
```

---

## 19.5 Documentation & Maintainability

```bash
#!/bin/bash
# Library documentation format

# ─── Function Documentation ───────────────────────────────────

# format_bytes - Format bytes to human-readable
# Arguments:
#   $1 - number of bytes
# Returns:
#   Formatted string like "1.5 GB"
# Example:
#   format_bytes 1536000000  # outputs "1.5 GB"
format_bytes() {
    local bytes=$1
    
    if (( bytes < 1024 )); then
        printf "%d B" "$bytes"
    elif (( bytes < 1048576 )); then
        printf "%.1f KB" "$(bc <<< "scale=1; $bytes/1024")"
    elif (( bytes < 1073741824 )); then
        printf "%.1f MB" "$(bc <<< "scale=1; $bytes/1048576")"
    else
        printf "%.1f GB" "$(bc <<< "scale=1; $bytes/1073741824")"
    fi
}

# ─── Version Control Friendliness ─────────────────────────────
# One statement per line for cleaner diffs
declare -A config=(
    [host]="localhost"
    [port]="5432"
    [name]="mydb"
)

# Use heredoc for multiline strings
read -r -d '' SQL << 'SQL'
SELECT id, name, email
FROM users
WHERE active = true
ORDER BY name
LIMIT 100
SQL

# ─── Configuration as data ────────────────────────────────────
# Separate config from logic

# config.conf
cat > config.conf << 'EOF'
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp

# Server  
SERVER_PORT=8080
SERVER_WORKERS=4

# Features
FEATURE_NEW_UI=false
FEATURE_BETA=false
EOF

# Load in script
# shellcheck source=config.conf
source config.conf
```

---

## 19.6 Checklist for Production Scripts

```bash
#!/bin/bash
# Production script checklist checker

check_script() {
    local script=$1
    local issues=()
    
    echo "Checking: $script"
    echo "─────────────────────────────────────"
    
    # Check shebang
    local first_line
    first_line=$(head -1 "$script")
    if [[ "$first_line" == "#!/bin/bash" ]] || [[ "$first_line" == "#!/usr/bin/env bash" ]]; then
        echo "✓ Has proper shebang"
    else
        issues+=("Missing or improper shebang")
    fi
    
    # Check set options
    if grep -q 'set -e\|set -euo pipefail' "$script"; then
        echo "✓ Has error handling (set -e)"
    else
        issues+=("Missing 'set -e' or 'set -euo pipefail'")
    fi
    
    # Check for unquoted variables
    if grep -qP '\$\w+(?!["])\b' "$script" 2>/dev/null; then
        echo "! Potential unquoted variables (check manually)"
    fi
    
    # Check shellcheck availability
    if command -v shellcheck &>/dev/null; then
        local sc_output
        sc_output=$(shellcheck "$script" 2>&1)
        if [[ -z "$sc_output" ]]; then
            echo "✓ Passes shellcheck"
        else
            issues+=("Has shellcheck warnings")
            echo "! Shellcheck warnings found"
        fi
    fi
    
    # Check for trap
    if grep -q 'trap' "$script"; then
        echo "✓ Has signal handling (trap)"
    else
        echo "! Consider adding trap for cleanup"
    fi
    
    # Check for logging
    if grep -qP 'log_|echo.*\[INFO\]' "$script"; then
        echo "✓ Has logging"
    else
        echo "! Consider adding structured logging"
    fi
    
    # Print issues
    if [[ ${#issues[@]} -gt 0 ]]; then
        echo ""
        echo "Issues to fix:"
        for issue in "${issues[@]}"; do
            echo "  ✗ $issue"
        done
    else
        echo ""
        echo "✓ Script passes basic checks"
    fi
}

check_script "${1:-$0}"
```

---

## 19.7 Exercises

### Exercise 1: Production Script Template
สร้าง template generator ที่:
- Generate script skeleton จาก template
- Include best practices by default
- Customizable ด้วย parameters

### Exercise 2: Code Review Tool
สร้าง tool ที่ review bash scripts:
- Run shellcheck
- Check naming conventions
- Find common antipatterns
- Generate report

### Exercise 3: Script Library
สร้าง reusable library:
- Logging
- Error handling
- Configuration
- Common utilities

---

## สรุป Part 19

✅ Script structure template  
✅ Idiomatic bash practices  
✅ Security best practices  
✅ Performance optimization  
✅ Documentation standards  
✅ Production script checklist  
✅ ShellCheck integration  

---

**→ Part 20: Mini Project - System Monitor**
