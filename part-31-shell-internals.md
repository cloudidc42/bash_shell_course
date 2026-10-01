# Part 31: Shell Internals & Advanced Bash Features
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 31.1 Bash Execution Model

```bash
# ─── How Bash Executes Commands ────────────────────────────────
# 1. Read input (line by line or script)
# 2. Tokenize (split into tokens)
# 3. Parse (build command tree)
# 4. Expand (variables, globs, etc.)
# 5. Execute

# ─── Subshells vs Current Shell ────────────────────────────────
# Subshell: new process, inherits environment but changes don't propagate back

# This runs in subshell — x not modified in parent
(x=100; echo "In subshell: $x")
echo "In parent: ${x:-unset}"  # unset

# This runs in current shell
{ x=100; echo "In group: $x"; }
echo "In parent: $x"  # 100

# Pipelines also create subshells!
echo "hello" | read word
echo "word: ${word:-empty}"  # empty! read ran in subshell

# Fix: use lastpipe (bash 4.2+)
shopt -s lastpipe
echo "hello" | read word
echo "word: $word"  # hello

# Or use process substitution
read word < <(echo "hello")
echo "word: $word"  # hello

# ─── Fork and Exec ─────────────────────────────────────────────
# Every external command: fork() + exec()
# Builtins: no fork (cd, echo, read, etc.)

# Check if builtin
type -t echo     # builtin
type -t ls       # file
type -t cd       # builtin

# Avoid unnecessary forks
# Bad: external command for each iteration
for i in {1..100}; do
    echo "$i" >> /tmp/numbers.txt
done

# Good: single fork
printf '%s\n' {1..100} > /tmp/numbers.txt

# ─── Command Lookup Order ──────────────────────────────────────
# 1. Aliases
# 2. Functions
# 3. Builtins
# 4. External commands (PATH)

# Override command lookup
command ls      # skip functions/aliases, use external or builtin
builtin cd      # force builtin
\ls             # skip aliases only

# ─── IFS (Internal Field Separator) ───────────────────────────
# Default: space, tab, newline
echo "$IFS" | cat -A  # show invisible chars

# Split string
data="one:two:three"
IFS=: read -ra parts <<< "$data"
echo "${parts[@]}"  # one two three

# Read CSV
IFS=, read -r f1 f2 f3 <<< "a,b,c"
echo "$f1 $f2 $f3"  # a b c

# Change IFS for for loop
IFS=$'\n'
for line in $(cat /etc/hosts); do
    echo "Line: $line"
done
IFS=$' \t\n'  # restore
```

---

## 31.2 Variable Expansion Internals

```bash
# ─── Expansion Order ───────────────────────────────────────────
# 1. Brace expansion:          {a,b,c}
# 2. Tilde expansion:          ~, ~user
# 3. Parameter expansion:      $var, ${var}
# 4. Command substitution:     $(cmd), `cmd`
# 5. Arithmetic expansion:     $(( expr ))
# 6. Word splitting:           IFS
# 7. Pathname expansion (glob): *, ?, [...]
# 8. Quote removal

# ─── Parameter Expansion Reference ────────────────────────────
var="Hello World"

# Basic
echo "${var}"           # Hello World
echo "${#var}"          # 11 (length)

# Default values
echo "${undefined:-default}"    # default (if unset or empty)
echo "${undefined-default}"     # default (only if unset)
echo "${undefined:+alt}"        # "" (empty if unset)
echo "${var:+alt}"              # alt (if set and non-empty)

# Assignment
echo "${undefined:=newval}"     # newval (assigns if unset)
echo "$undefined"               # newval

# Error if unset
# echo "${required:?'required is not set'}"

# Substring
echo "${var:0:5}"     # Hello
echo "${var:6}"       # World
echo "${var: -5}"     # World (negative index)

# Pattern removal
path="/home/user/file.txt"
echo "${path#*/}"     # home/user/file.txt (shortest match from start)
echo "${path##*/}"    # file.txt (longest match from start)
echo "${path%.*}"     # /home/user/file (shortest match from end)
echo "${path%%.*}"    # /home/user/file (longest match from end)

# Pattern replacement
echo "${var/World/Bash}"   # Hello Bash
echo "${var//l/L}"         # HeLLo WorLd (all occurrences)
echo "${var/#H/h}"         # hello World (start anchor)
echo "${var/%d/D}"         # Hello WorlD (end anchor)

# Case conversion (bash 4+)
echo "${var,,}"   # hello world (lowercase)
echo "${var^^}"   # HELLO WORLD (uppercase)
echo "${var,}"    # hELLO WORLD (first char lower)
echo "${var^}"    # Hello World (first char upper)

# ─── Indirect Variable Reference ───────────────────────────────
name="myvar"
myvar="hello"

# Old way (unsafe with eval)
echo "${!name}"   # hello (indirect reference)

# Nameref (bash 4.3+)
declare -n ref=myvar
echo "$ref"    # hello
ref="world"    # modifies myvar
echo "$myvar"  # world

# ─── Array Expansions ──────────────────────────────────────────
arr=(a b c d e)

echo "${arr[@]}"     # all elements
echo "${arr[*]}"     # all elements as one word
echo "${#arr[@]}"    # number of elements
echo "${!arr[@]}"    # indices
echo "${arr[@]:1:3}" # slice: elements 1,2,3

# Associative array
declare -A map
map[key1]="val1"
map[key2]="val2"

echo "${!map[@]}"    # all keys
echo "${map[@]}"     # all values
echo "${map[key1]}"  # val1
```

---

## 31.3 Process Substitution & Coprocesses

```bash
# ─── Process Substitution ──────────────────────────────────────
# <(cmd) — file descriptor for output of cmd
# >(cmd) — file descriptor for input to cmd

# Compare two command outputs
diff <(ls dir1) <(ls dir2)

# Read from command
while read -r line; do
    echo "Got: $line"
done < <(find /etc -name "*.conf" 2>/dev/null)

# Multiple inputs
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f3 /etc/passwd)

# Write to multiple destinations
tee >(gzip > file.gz) >(sha256sum > file.sha256) > /dev/null < file

# ─── Coprocesses ───────────────────────────────────────────────
# Bidirectional pipe with background process

# Start coprocess
coproc bc_proc { bc; }

# Write to coprocess
echo "2 + 2" >&"${bc_proc[1]}"

# Read from coprocess
read -r result <&"${bc_proc[0]}"
echo "Result: $result"  # 4

# Named coprocess
coproc CALC { awk '{ print $1 * 2 }'; }

echo "5" >&"${CALC[1]}"
read -r doubled <&"${CALC[0]}"
echo "Doubled: $doubled"  # 10

# Kill coprocess
kill "$CALC_PID"

# ─── Here-string and Here-doc ──────────────────────────────────
# Here-string
read -r word <<< "hello"
grep -c "pattern" <<< "$variable"

# Here-doc
cat << 'EOF'
No expansion here: $HOME
EOF

cat << EOF
Expansion here: $HOME
EOF

# Indented heredoc (bash 4+)
cat <<- EOF
	This text has leading tabs stripped
	Useful for indented code
	EOF

# ─── Named Pipes (FIFOs) ───────────────────────────────────────
# Create named pipe
mkfifo /tmp/mypipe

# Producer (background)
(echo "data from producer" > /tmp/mypipe) &

# Consumer
read -r data < /tmp/mypipe
echo "Received: $data"

# Cleanup
rm /tmp/mypipe

# ─── File Descriptors ──────────────────────────────────────────
# Open FD for reading
exec 3< /etc/hostname
read -r hostname <&3
exec 3<&-   # close
echo "Hostname: $hostname"

# Open FD for writing
exec 4> /tmp/output.txt
echo "Line 1" >&4
echo "Line 2" >&4
exec 4>&-   # close

# Read/write FD
exec 5<>/tmp/rw.txt
echo "hello" >&5
# Seek back to start (not portable)
# Re-open for reading
exec 6< /tmp/rw.txt
read -r line <&6
echo "$line"
exec 5>&- 6<&-
```

---

## 31.4 Advanced Pattern Matching

```bash
# ─── Extended Glob (extglob) ───────────────────────────────────
shopt -s extglob

# Patterns:
# ?(pattern) — zero or one
# *(pattern) — zero or more
# +(pattern) — one or more
# @(pattern) — exactly one
# !(pattern) — anything except

# Match files
ls *.+(jpg|png|gif)      # files ending in jpg, png, or gif
ls !(*.txt)              # all files except .txt

# In conditionals
file="script.sh"
[[ "$file" == +(*.sh|*.bash) ]] && echo "Shell script"

# Trim prefix/suffix with extglob
path="/usr/local/bin/myapp"
echo "${path##/*/}"      # bin/myapp
echo "${path//@(\/)/}"   # remove all slashes (wrong, example only)

# ─── Case Statement Patterns ───────────────────────────────────
value="hello"
case "$value" in
    h*)          echo "starts with h" ;;
    *[0-9]*)     echo "contains digit" ;;
    [[:upper:]]*)echo "starts with uppercase" ;;
    ?(pre)fix)   echo "fix or prefix" ;;  # requires extglob
    *)           echo "no match" ;;
esac

# ─── Globstar ──────────────────────────────────────────────────
shopt -s globstar

# Recursive glob
for f in **/*.sh; do
    echo "Found: $f"
done

# Count files
shopt -s globstar nullglob
files=(**/*.py)
echo "Python files: ${#files[@]}"

# ─── Nullglob and Failglob ─────────────────────────────────────
shopt -s nullglob
# Pattern with no match expands to nothing (not kept as literal)
for f in /nonexistent/*.txt; do
    echo "$f"  # never executes
done

shopt -s failglob
# Pattern with no match causes error
# ls /nonexistent/*.txt  # would error

# ─── dotglob ───────────────────────────────────────────────────
shopt -s dotglob
# Include hidden files in glob
for f in *; do
    echo "$f"  # includes .hidden files
done
```

---

## 31.5 Bash Options & Shell Configuration

```bash
# ─── set options ───────────────────────────────────────────────
set -e          # exit on error
set -u          # treat unset vars as errors
set -x          # trace execution
set -o pipefail # pipe fails if any command fails
set -o nounset  # same as -u
set -o errexit  # same as -e
set -o xtrace   # same as -x

# Combining
set -euo pipefail

# In script header
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'

# Check current settings
set -o   # show all options

# ─── shopt options ─────────────────────────────────────────────
shopt -s autocd      # type dirname to cd
shopt -s cdspell     # autocorrect cd typos
shopt -s checkjobs   # warn about running jobs on exit
shopt -s cmdhist     # save multi-line cmds as single history entry
shopt -s dirspell    # correct directory spelling
shopt -s dotglob     # include dotfiles in glob
shopt -s extglob     # extended patterns
shopt -s globstar    # ** recursive glob
shopt -s histappend  # append, not overwrite, history
shopt -s nocaseglob  # case-insensitive glob
shopt -s nullglob    # no-match glob → empty

# ─── POSIX mode ────────────────────────────────────────────────
set -o posix   # enable POSIX mode (more portable)
set +o posix   # disable

# ─── Restricted shell ──────────────────────────────────────────
# bash -r or bash --restricted
# Prevents: cd, setting PATH, redirecting to files, exec

# ─── Startup files ─────────────────────────────────────────────
# Login shell:     /etc/profile → ~/.bash_profile or ~/.bash_login or ~/.profile
# Interactive:     /etc/bash.bashrc → ~/.bashrc
# Non-interactive: $BASH_ENV
# Logout:          ~/.bash_logout

# Force read of ~/.bashrc in login shell
# Add to ~/.bash_profile:
[[ -f ~/.bashrc ]] && . ~/.bashrc

# ─── READLINE configuration ────────────────────────────────────
# ~/.inputrc
cat > ~/.inputrc << 'EOF'
# Case-insensitive completion
set completion-ignore-case on

# Show all completions on first tab
set show-all-if-ambiguous on

# Colored completion
set colored-completion-prefix on
set colored-stats on

# History search with arrows
"\e[A": history-search-backward
"\e[B": history-search-forward

# Skip over words with ctrl+arrow
"\e[1;5C": forward-word
"\e[1;5D": backward-word
EOF
```

---

## 31.6 Trap & Signal Handling (Advanced)

```bash
#!/bin/bash
# Advanced signal handling

# ─── Signal list ───────────────────────────────────────────────
# SIGHUP  (1)  — terminal closed
# SIGINT  (2)  — Ctrl+C
# SIGQUIT (3)  — Ctrl+\
# SIGTERM (15) — graceful termination
# SIGKILL (9)  — force kill (cannot be trapped)
# SIGUSR1 (10) — user-defined
# SIGUSR2 (12) — user-defined
# SIGCHLD (17) — child process changed state
# SIGWINCH(28) — terminal window resized

# ─── Cleanup trap ──────────────────────────────────────────────
TMPDIR=$(mktemp -d)
PIDFILE="/tmp/myapp.pid"
RUNNING=true

cleanup() {
    local exit_code=$?
    echo "Cleaning up..."
    rm -rf "$TMPDIR"
    rm -f "$PIDFILE"
    
    # Restore terminal if needed
    tput cnorm 2>/dev/null  # show cursor
    
    exit "$exit_code"
}

trap cleanup EXIT
trap 'echo "Interrupted"; RUNNING=false' INT TERM

# ─── Graceful shutdown ─────────────────────────────────────────
shutdown_handler() {
    echo "SIGTERM received, shutting down gracefully..."
    RUNNING=false
    
    # Wait for current operation to finish
    local timeout=30
    while [[ -n "${CURRENT_OP:-}" ]] && (( timeout > 0 )); do
        echo "Waiting for $CURRENT_OP to finish..."
        sleep 1
        (( timeout-- ))
    done
    
    echo "Shutdown complete"
    exit 0
}

trap shutdown_handler TERM

# ─── Signal forwarding to children ────────────────────────────
forward_signal() {
    local sig=$1
    echo "Forwarding $sig to children..."
    kill -"$sig" 0 2>/dev/null || true
}

trap 'forward_signal TERM; exit 0' TERM
trap 'forward_signal INT; exit 130' INT

# ─── SIGUSR for runtime control ────────────────────────────────
VERBOSE=false
RELOAD_CONFIG=false

handle_sigusr1() {
    echo "SIGUSR1 received: toggling verbose mode"
    $VERBOSE && VERBOSE=false || VERBOSE=true
}

handle_sigusr2() {
    echo "SIGUSR2 received: reloading config"
    RELOAD_CONFIG=true
}

trap handle_sigusr1 USR1
trap handle_sigusr2 USR2

# Send signal:
# kill -USR1 $PID
# kill -USR2 $PID

# ─── ERR trap ──────────────────────────────────────────────────
handle_error() {
    local exit_code=$?
    local line_number=$1
    local command=$2
    
    echo "Error $exit_code at line $line_number: $command" >&2
    echo "Stack trace:" >&2
    
    local frame=0
    while caller $frame; do
        (( frame++ ))
    done >&2
}

trap 'handle_error $LINENO "$BASH_COMMAND"' ERR

# ─── DEBUG trap ────────────────────────────────────────────────
debug_hook() {
    echo "About to run: $BASH_COMMAND" >&2
}

# Enable selectively
# trap debug_hook DEBUG

# ─── RETURN trap ───────────────────────────────────────────────
myfunc() {
    trap 'echo "Function returning"' RETURN
    echo "In function"
}
```

---

## 31.7 Advanced I/O Patterns

```bash
#!/bin/bash
# Advanced I/O techniques

# ─── Multiplexing output ───────────────────────────────────────
tee_output() {
    local cmd=$1
    shift
    local outputs=("$@")
    
    local tee_args=()
    for out in "${outputs[@]}"; do
        tee_args+=("$out")
    done
    
    eval "$cmd" | tee "${tee_args[@]}" > /dev/null
}

# Example: log to file AND stdout
exec > >(tee -a /var/log/myapp.log) 2>&1

# ─── Async I/O with FDs ────────────────────────────────────────
read_nonblock() {
    local fd=$1
    local result_var=$2
    
    if IFS= read -r -t 0.1 line <&"$fd"; then
        printf -v "$result_var" '%s' "$line"
        return 0
    fi
    return 1  # no data available
}

# Set up pipe
exec 7< <(while true; do echo "data $(date)"; sleep 1; done)

# Poll
while true; do
    if read_nonblock 7 line; then
        echo "Got: $line"
    fi
    sleep 0.1
done
exec 7<&-

# ─── Progress to stderr ────────────────────────────────────────
show_progress() {
    local current=$1
    local total=$2
    local width=40
    
    local pct=$(( current * 100 / total ))
    local filled=$(( current * width / total ))
    local bar
    bar=$(printf '%*s' "$filled" | tr ' ' '#')
    printf '\r[%-*s] %3d%%' "$width" "$bar" "$pct" >&2
    
    (( current >= total )) && echo "" >&2
}

# ─── Logging levels ────────────────────────────────────────────
declare -A LOG_LEVELS=([DEBUG]=0 [INFO]=1 [WARN]=2 [ERROR]=3)
LOG_LEVEL=${LOG_LEVEL:-INFO}

log() {
    local level=$1
    shift
    local message="$*"
    
    local current_level=${LOG_LEVELS[$level]:-1}
    local min_level=${LOG_LEVELS[$LOG_LEVEL]:-1}
    
    (( current_level < min_level )) && return
    
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    local color
    case "$level" in
        DEBUG) color='\033[36m' ;;
        INFO)  color='\033[32m' ;;
        WARN)  color='\033[33m' ;;
        ERROR) color='\033[31m' ;;
        *)     color='\033[0m'  ;;
    esac
    
    local nc='\033[0m'
    
    printf "${color}[%s] [%s]${nc} %s\n" "$timestamp" "$level" "$message" >&2
    
    if [[ -n "${LOGFILE:-}" ]]; then
        printf "[%s] [%s] %s\n" "$timestamp" "$level" "$message" >> "$LOGFILE"
    fi
}

log DEBUG "Debug message (only shown if LOG_LEVEL=DEBUG)"
log INFO  "Process started"
log WARN  "Low disk space"
log ERROR "Connection failed"
```

---

## 31.8 Exercises

### Exercise 1: Shell Profiler
สร้าง script profiler:
- Measure execution time of functions
- Track number of function calls
- Count forks (external commands)
- Report top slow functions

### Exercise 2: IFS Manipulator
สร้าง CSV parser ที่ใช้แต่ bash builtins:
- Parse CSV with quoted fields
- Handle escaped commas
- Output as array
- Zero external commands

### Exercise 3: Signal-based IPC
สร้าง inter-process communication:
- Master/worker with SIGUSR1/USR2
- Status reporting via signals
- Graceful reload without restart
- Coordination lock-free

---

## สรุป Part 31

✅ Bash execution model (subshells, fork/exec)  
✅ Variable expansion internals (all forms)  
✅ Process substitution & coprocesses  
✅ Extended glob (extglob, globstar)  
✅ Bash options (set, shopt)  
✅ Advanced trap & signal handling  
✅ Advanced I/O patterns (FDs, async, logging)  

---

**→ Part 32: Performance Optimization & Benchmarking**
