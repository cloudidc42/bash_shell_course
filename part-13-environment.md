# Part 13: Environment Variables & Shell Configuration
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 13.1 Environment Variables

```bash
#!/bin/bash

# ดู environment ทั้งหมด
env
printenv
declare -x      # แสดงเฉพาะ exported vars

# ดู specific variable
echo $HOME
printenv PATH
echo ${MY_VAR:-"not set"}

# กำหนด environment variable
export MY_VAR="hello"
declare -x MY_VAR="hello"  # same

# Temporary environment (เฉพาะ command นั้น)
DEBUG=true ./myapp.sh
DATABASE_URL=postgres://localhost/db python3 app.py
MY_VAR=test env | grep MY_VAR

# Remove variable
unset MY_VAR
export -n MY_VAR  # ยังคงอยู่ใน shell แต่ไม่ export

# Important built-in variables
echo "HOME:     $HOME"
echo "USER:     $USER"
echo "LOGNAME:  $LOGNAME"
echo "SHELL:    $SHELL"
echo "PATH:     $PATH"
echo "PWD:      $PWD"
echo "OLDPWD:   $OLDPWD"
echo "TERM:     $TERM"
echo "LANG:     $LANG"
echo "LC_ALL:   ${LC_ALL:-not set}"
echo "EDITOR:   ${EDITOR:-not set}"
echo "PAGER:    ${PAGER:-not set}"
echo "TMPDIR:   ${TMPDIR:-/tmp}"
echo "XDG_HOME: ${XDG_CONFIG_HOME:-$HOME/.config}"
echo "HOSTNAME: $HOSTNAME"
echo "HOSTTYPE: $HOSTTYPE"
echo "OSTYPE:   $OSTYPE"
echo "MACHTYPE: $MACHTYPE"

# Bash-specific
echo "BASH:         $BASH"
echo "BASH_VERSION: $BASH_VERSION"
echo "BASH_VERSINFO: ${BASH_VERSINFO[@]}"
echo "BASHOPTS:     $BASHOPTS"
echo "SHELLOPTS:    $SHELLOPTS"
echo "BASH_SOURCE:  ${BASH_SOURCE[@]}"
echo "FUNCNAME:     ${FUNCNAME[@]}"
echo "LINENO:       $LINENO"
echo "PPID:         $PPID"
echo "RANDOM:       $RANDOM"
echo "SECONDS:      $SECONDS"

# IFS - Internal Field Separator
echo "Default IFS: [${IFS}]"
# Default: space, tab, newline

# Custom IFS
IFS=: read -r a b c <<< "one:two:three"
echo "$a $b $c"

# PATH manipulation
echo $PATH | tr ':' '\n'   # show each path on own line

# Add to PATH
export PATH="$HOME/bin:$PATH"                  # prepend
export PATH="$PATH:/usr/local/myapp/bin"       # append
export PATH="$HOME/bin:${PATH/\/old\/bin:/}"   # remove and add

# Check if in PATH
in_path() {
    local cmd=$1
    command -v "$cmd" &>/dev/null
}
```

---

## 13.2 Shell Options (set & shopt)

```bash
# set: POSIX shell options
set -e          # exit on error (errexit)
set -u          # error on unset variable (nounset)
set -o pipefail # fail on pipe error
set -x          # print commands before execution (xtrace)
set -v          # print input lines as read (verbose)
set -n          # dry run (don't execute)
set -f          # disable glob expansion (noglob)
set -b          # notify bg job completion immediately
set -C          # prevent file overwrite with >

# Combine options
set -euo pipefail   # recommended for scripts

# Unset options
set +e          # disable exit-on-error
set +x          # disable xtrace

# View current options
echo $-         # active option flags
set -o          # show all option status

# shopt: bash-specific options
shopt -s option     # set
shopt -u option     # unset
shopt option        # show status

# Useful shopt options
shopt -s autocd         # cd to directory without 'cd'
shopt -s cdspell        # fix minor cd spelling errors
shopt -s checkjobs      # check for jobs before exit
shopt -s cmdhist        # save multiline commands as one history entry
shopt -s dotglob        # include .files in glob
shopt -s extglob        # extended glob patterns
shopt -s globstar       # ** matches any directory
shopt -s histappend     # append to history file (not overwrite)
shopt -s histverify     # verify history expansion before execution
shopt -s nocasematch    # case-insensitive pattern matching
shopt -s nullglob       # glob returns empty when no match (not literal *)
shopt -s progcomp       # programmable completion
shopt -s xpg_echo       # echo interprets escape sequences

# Extended glob patterns (with shopt -s extglob)
shopt -s extglob
ls *.!(txt)             # files NOT ending in .txt
ls +(foo|bar)*          # starts with foo or bar
ls ?(optional)file      # file or optionalfile
ls @(this|that)         # exactly this or that
ls !(not_this)          # anything but not_this

# globstar for recursive matching
shopt -s globstar
echo **/*.sh            # all .sh files recursively
wc -l **/*.py           # count lines in all Python files
```

---

## 13.3 Bash Configuration Files

```bash
# ลำดับการอ่านไฟล์ config:

# Login Shell (ssh, su -, terminal with --login):
# 1. /etc/profile
# 2. /etc/profile.d/*.sh  (sourced by /etc/profile)
# 3. ~/.bash_profile  (หรือ ~/.bash_login หรือ ~/.profile)
# 4. ~/.bashrc (usually sourced from .bash_profile)

# Interactive Non-Login Shell (new terminal tab):
# 1. /etc/bash.bashrc (or /etc/bashrc)
# 2. ~/.bashrc

# Non-Interactive Shell (scripts):
# ไม่อ่านไฟล์ใดเลย (เว้นแต่ source เอง)

# Logout:
# ~/.bash_logout

# ตรวจสอบว่าเป็น login/interactive shell
if [[ -o login ]]; then echo "login shell"; fi
if [[ $- == *i* ]]; then echo "interactive"; fi
if [[ -t 0 ]]; then echo "stdin is terminal"; fi

# ~/.bash_profile - สำหรับ login shell
cat >> ~/.bash_profile << 'EOF'
# Source .bashrc if it exists
if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
fi

# Environment variables for GUI apps
export JAVA_HOME=/usr/lib/jvm/default-java
export ANDROID_HOME=$HOME/Android/Sdk
EOF

# ~/.bashrc - สำหรับ interactive shell
cat > ~/.bashrc << 'BASHRC'
# If not running interactively, don't do anything
[[ $- != *i* ]] && return

# History settings
HISTSIZE=50000
HISTFILESIZE=100000
HISTCONTROL=ignoreboth:erasedups
HISTTIMEFORMAT="%F %T "
shopt -s histappend

# Shell options
shopt -s autocd
shopt -s cdspell
shopt -s globstar
shopt -s checkwinsize   # update LINES/COLUMNS after each command

# Prompt
PS1='\[\e[1;32m\]\u@\h\[\e[0m\]:\[\e[1;34m\]\w\[\e[0m\]\$ '

# Aliases
alias ls='ls --color=auto'
alias ll='ls -alF'
alias la='ls -A'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'
alias free='free -h'
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'

# Functions
mkcd() { mkdir -p "$1" && cd "$1"; }
extract() {
    case "$1" in
        *.tar.bz2) tar xjf "$1" ;;
        *.tar.gz)  tar xzf "$1" ;;
        *.tar.xz)  tar xJf "$1" ;;
        *.bz2)     bunzip2 "$1"  ;;
        *.gz)      gunzip "$1"   ;;
        *.tar)     tar xf "$1"   ;;
        *.zip)     unzip "$1"    ;;
        *.7z)      7z x "$1"     ;;
        *) echo "Unknown archive: $1" ;;
    esac
}

# Load local customizations
[[ -f ~/.bashrc.local ]] && source ~/.bashrc.local

BASHRC
```

---

## 13.4 Custom Prompt (PS1)

```bash
# Prompt variables
# \u = username
# \h = hostname (short)
# \H = hostname (full)
# \w = working directory (full)
# \W = working directory (basename)
# \$ = $ or # (based on UID)
# \t = time HH:MM:SS
# \T = time HH:MM:SS (12h)
# \@ = time AM/PM
# \d = date "Weekday Month DD"
# \n = newline
# \l = terminal device
# \j = number of jobs
# \! = history number
# \# = command number
# \[ \] = begin/end non-printing chars

# Colors
BLACK='\[\033[0;30m\]'
RED='\[\033[0;31m\]'
GREEN='\[\033[0;32m\]'
YELLOW='\[\033[0;33m\]'
BLUE='\[\033[0;34m\]'
PURPLE='\[\033[0;35m\]'
CYAN='\[\033[0;36m\]'
WHITE='\[\033[0;37m\]'
BOLD_RED='\[\033[1;31m\]'
BOLD_GREEN='\[\033[1;32m\]'
BOLD_BLUE='\[\033[1;34m\]'
NC='\[\033[0m\]'   # No Color

# Simple colored prompt
PS1="${BOLD_GREEN}\u@\h${NC}:${BOLD_BLUE}\w${NC}\$ "

# Show git branch in prompt
git_branch() {
    local branch
    branch=$(git branch 2>/dev/null | grep '^*' | sed 's/\* //')
    [[ -n "$branch" ]] && echo " ($branch)"
}

PS1="${BOLD_GREEN}\u@\h${NC}:${BOLD_BLUE}\w${YELLOW}\$(git_branch)${NC}\$ "

# Two-line prompt with timestamp
PS1="${CYAN}[\t]${NC} ${BOLD_GREEN}\u@\h${NC}:${BOLD_BLUE}\w${NC}\n\$ "

# Show exit code of last command
show_exit() {
    local code=$?
    [[ $code -ne 0 ]] && echo " [${code}]"
}

PS1="${BOLD_GREEN}\u@\h${NC}:${BOLD_BLUE}\w${BOLD_RED}\$(show_exit)${NC}\$ "

# PS2: continuation prompt (after line continuation with \)
PS2="${CYAN}... ${NC}"

# PS3: prompt for select command
PS3="Choose option: "

# PS4: debug prompt (set -x)
PS4="${CYAN}+${BASH_SOURCE}:${LINENO}:${FUNCNAME[0]:+${FUNCNAME[0]}:} ${NC}"

# PROMPT_COMMAND: runs before each prompt
PROMPT_COMMAND='history -a'  # save history after each command
PROMPT_COMMAND='printf "\033]0;%s@%s:%s\007" "$USER" "$HOSTNAME" "${PWD/#$HOME/~}"'  # window title
```

---

## 13.5 Aliases & Functions

```bash
# Aliases - simple command shortcuts
alias update='sudo apt-get update && sudo apt-get upgrade -y'
alias ports='netstat -tulanp'
alias myip='curl -s https://api.ipify.org && echo'
alias weather='curl wttr.in'
alias c='clear'
alias h='history'
alias j='jobs -l'

# Navigation aliases
alias home='cd ~'
alias docs='cd ~/Documents'
alias dl='cd ~/Downloads'
alias dev='cd ~/Development'

# Safety aliases
alias rm='rm -I'          # ask before removing 3+ files
alias mv='mv -i'          # ask before overwrite
alias cp='cp -i'          # ask before overwrite
alias ln='ln -i'          # ask before overwrite

# Enhanced commands
alias grep='grep --color=auto -n'
alias ls='ls --color=auto --group-directories-first'
alias diff='diff --color=auto'
alias ping='ping -c 5'

# Alias for sudo (allow aliases after sudo)
alias sudo='sudo '

# View aliases
alias              # list all
alias ls           # show specific
type ls            # show full definition

# Functions (more powerful than aliases)
# สามารถรับ arguments, มี local variables, return value

# Go up N directories
up() {
    local levels=${1:-1}
    local path=""
    for ((i=0; i<levels; i++)); do
        path="../$path"
    done
    cd "${path:-.}"
}

# Create and enter directory
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Find and kill by name
fk() {
    local name=$1
    local pids
    pids=$(pgrep -f "$name")
    [[ -z "$pids" ]] && echo "No process found: $name" && return 1
    echo "Killing: $pids"
    kill -9 $pids
}

# Quick HTTP server
serve() {
    local port=${1:-8000}
    local dir=${2:-.}
    echo "Serving $dir on http://localhost:$port"
    python3 -m http.server "$port" -d "$dir"
}

# Extract various archive types
extract() {
    if [[ ! -f "$1" ]]; then
        echo "'$1' is not a valid file"
        return 1
    fi
    
    case "$1" in
        *.tar.bz2|*.tbz2) tar xjvf "$1" ;;
        *.tar.gz|*.tgz)   tar xzvf "$1" ;;
        *.tar.xz|*.txz)   tar xJvf "$1" ;;
        *.bz2)            bunzip2 "$1" ;;
        *.rar)            unrar x "$1" ;;
        *.gz)             gunzip "$1" ;;
        *.tar)            tar xvf "$1" ;;
        *.zip)            unzip "$1" ;;
        *.Z)              uncompress "$1" ;;
        *.7z)             7z x "$1" ;;
        *.deb)            ar x "$1" ;;
        *.xz)             unxz "$1" ;;
        *)                echo "Cannot extract: $1" ;;
    esac
}

# Backup file before editing
bak() {
    cp "$1" "${1}.bak.$(date +%Y%m%d_%H%M%S)"
    echo "Backup: ${1}.bak.$(date +%Y%m%d_%H%M%S)"
}
```

---

## 13.6 Completion & Tab Completion

```bash
# Enable programmable completion
# ต้องมี bash-completion package

# Load completion
if [[ -f /etc/bash_completion ]]; then
    . /etc/bash_completion
fi

# Custom completion function
_my_command_completion() {
    local cur prev words cword
    _init_completion || return
    
    # Available subcommands
    local subcommands="start stop status restart list"
    
    case $prev in
        start|stop|restart)
            # Complete with running services
            COMPREPLY=($(compgen -W "$(systemctl list-units --type=service --state=running --no-legend | awk '{print $1}' | sed 's/.service//')" -- "$cur"))
            ;;
        *)
            COMPREPLY=($(compgen -W "$subcommands" -- "$cur"))
            ;;
    esac
}

complete -F _my_command_completion my_command

# File completion for specific commands
complete -f -X '!*.gz' zcat    # only .gz files for zcat
complete -d cd                  # only directories for cd

# Custom word completion
_colors=(red green blue yellow purple orange)
complete -W "${_colors[*]}" my_color_command

# Dynamic completion
complete -C my_completion_generator my_command
```

---

## 13.7 .env Files & Secret Management

```bash
#!/bin/bash

# Load .env file
load_env() {
    local env_file="${1:-.env}"
    
    [[ -f "$env_file" ]] || { echo "No .env file found"; return 1; }
    
    # Source with safety checks
    while IFS= read -r line || [[ -n "$line" ]]; do
        # Skip comments and empty lines
        [[ "$line" =~ ^[[:space:]]*# ]] && continue
        [[ -z "${line// /}" ]] && continue
        
        # Validate format
        [[ "$line" =~ ^[A-Z_][A-Z0-9_]*= ]] || continue
        
        # Export
        export "$line"
    done < "$env_file"
}

# Create .env template
create_env_template() {
    cat > .env.example << 'EOF'
# Database Configuration
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=myapp
DATABASE_USER=myuser
DATABASE_PASSWORD=

# API Keys
API_KEY=
SECRET_KEY=

# Application
APP_ENV=development
APP_PORT=8080
DEBUG=false
LOG_LEVEL=INFO
EOF
    echo "Created .env.example"
}

# Check required variables
require_env() {
    local missing=()
    
    for var in "$@"; do
        [[ -z "${!var}" ]] && missing+=("$var")
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "Error: Missing required environment variables:"
        printf "  - %s\n" "${missing[@]}"
        return 1
    fi
}

# Usage
load_env .env
require_env DATABASE_HOST DATABASE_USER API_KEY || exit 1

# Never echo sensitive values
# echo "$API_KEY"  # ❌ bad

# Safe way to check if set
[[ -n "$API_KEY" ]] && echo "API_KEY is set" || echo "API_KEY is missing"

# Mask sensitive in logs
mask_secret() {
    local value=$1
    local visible=${2:-4}
    local len=${#value}
    
    if (( len <= visible )); then
        printf '%*s' $len | tr ' ' '*'
    else
        echo "${value:0:$visible}$(printf '%*s' $((len-visible)) | tr ' ' '*')"
    fi
}

echo "API Key: $(mask_secret "$API_KEY")"

# Using pass (password manager)
# SECRET=$(pass show myapp/api_key)

# Using gopass
# SECRET=$(gopass show -o myapp/api_key)
```

---

## 13.8 Shell Startup Profiling

```bash
#!/bin/bash
# profile_startup.sh - Profile shell startup time

# Method 1: time the startup
time bash -i -c exit 2>&1

# Method 2: xtrace with timestamps
# Add to ~/.bashrc temporarily:
# PS4='+ $(date "+%s.%N")\011 '
# exec 3>&2 2>/tmp/bashstart.$$.log
# set -x
# ... your .bashrc ...
# set +x
# exec 2>&3 3>&-

# Method 3: zsh-like startup profiler for bash
profile_bash() {
    local log_file=$(mktemp)
    
    bash --rcfile <(
        echo "PS4='+\$(date +%s%N)\t\${BASH_SOURCE}\t\${LINENO}\t'"
        echo "exec 2>$log_file"
        echo "set -x"
        cat ~/.bashrc
        echo "set +x"
    ) -i -c exit 2>/dev/null
    
    # Analyze
    awk '{
        if (NR > 1) {
            delta = $1 - prev_time
            if (delta > 10000000) {  # > 10ms
                printf "%6.0f ms: %s:%s %s\n", delta/1000000, $2, $3, $4
            }
        }
        prev_time = $1
    }' "$log_file" | sort -rn | head -20
    
    rm -f "$log_file"
}

# Lazy loading for slow commands
# Instead of loading completion immediately:
lazy_load() {
    local cmd=$1
    local init_cmd=$2
    
    eval "${cmd}() {
        unset -f ${cmd}
        ${init_cmd}
        ${cmd} \"\$@\"
    }"
}

# Example: lazy load nvm
lazy_load nvm 'export NVM_DIR="$HOME/.nvm"; [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"'
```

---

## 13.9 Complete .bashrc Example

```bash
#!/bin/bash
# ~/.bashrc - Complete example configuration

# Exit if not interactive
[[ $- != *i* ]] && return

# ─── HISTORY ──────────────────────────────────────────────────
HISTSIZE=100000
HISTFILESIZE=200000
HISTCONTROL=ignoreboth:erasedups
HISTTIMEFORMAT="%Y-%m-%d %T "
shopt -s histappend
shopt -s cmdhist
PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"

# ─── SHELL OPTIONS ────────────────────────────────────────────
shopt -s autocd          # cd without cd command
shopt -s cdspell         # auto-correct cd typos
shopt -s checkjobs       # check jobs before exit
shopt -s checkwinsize    # update LINES/COLUMNS
shopt -s dirspell        # correct dir name spelling
shopt -s globstar        # ** for recursive glob
shopt -s extglob         # extended glob
shopt -s nocaseglob      # case-insensitive glob
shopt -s nullglob        # no error on no match

# ─── EDITOR ───────────────────────────────────────────────────
export EDITOR='vim'
export VISUAL='vim'
export PAGER='less'
export LESS='-R -F -X'

# ─── PATH ─────────────────────────────────────────────────────
# Helper to add to PATH only if not already there
pathadd() {
    [[ -d "$1" && ":$PATH:" != *":$1:"* ]] && PATH="$1:$PATH"
}

pathadd "$HOME/bin"
pathadd "$HOME/.local/bin"
pathadd "/usr/local/bin"
pathadd "$HOME/.cargo/bin"
pathadd "$HOME/go/bin"
export PATH

# ─── COLORS ───────────────────────────────────────────────────
if command -v dircolors &>/dev/null; then
    eval "$(dircolors -b)"
fi
export GCC_COLORS='error=01;31:warning=01;35:note=01;36:caret=01;32:locus=01:quote=01'

# ─── PROMPT ───────────────────────────────────────────────────
_git_branch() {
    local branch
    branch=$(git symbolic-ref --short HEAD 2>/dev/null) || \
    branch=$(git rev-parse --short HEAD 2>/dev/null)
    [[ -n "$branch" ]] && echo " \[\033[33m\](${branch})\[\033[0m\]"
}

_exit_code() {
    local code=$?
    [[ $code -ne 0 ]] && echo " \[\033[31m\][${code}]\[\033[0m\]"
}

PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]$(_git_branch)$(_exit_code)\$ '

# ─── ALIASES ──────────────────────────────────────────────────
# Navigation
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias ~='cd ~'
alias -- -='cd -'

# ls
alias ls='ls --color=auto --group-directories-first'
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias lh='ls -alFh'
alias lt='ls -alFt'

# grep
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# Safety
alias rm='rm -I --preserve-root'
alias mv='mv -i'
alias cp='cp -i'
alias ln='ln -i'
alias chown='chown --preserve-root'
alias chmod='chmod --preserve-root'
alias chgrp='chgrp --preserve-root'

# Shortcuts
alias c='clear'
alias h='history'
alias j='jobs -l'
alias reload='source ~/.bashrc'
alias edit='$EDITOR ~/.bashrc'
alias ports='ss -tulpn'
alias myip='curl -s https://api.ipify.org && echo'
alias update='sudo apt-get update && sudo apt-get upgrade -y'

# ─── FUNCTIONS ────────────────────────────────────────────────
mkcd() { mkdir -p "$1" && cd "$1"; }
bak() { cp "$1" "${1}.bak.$(date +%Y%m%d_%H%M%S)"; }
up() { local p=""; for ((i=0;i<${1:-1};i++)); do p="../$p"; done; cd "${p:-.}"; }

man() {
    LESS_TERMCAP_md=$'\e[01;31m' \
    LESS_TERMCAP_me=$'\e[0m' \
    LESS_TERMCAP_us=$'\e[01;36m' \
    LESS_TERMCAP_ue=$'\e[0m' \
    command man "$@"
}

# ─── COMPLETION ───────────────────────────────────────────────
if [[ -f /etc/bash_completion ]]; then
    . /etc/bash_completion
elif [[ -f /usr/share/bash-completion/bash_completion ]]; then
    . /usr/share/bash-completion/bash_completion
fi

# ─── LOCAL CONFIG ─────────────────────────────────────────────
[[ -f ~/.bashrc.local ]] && . ~/.bashrc.local
[[ -f ~/.bash_aliases ]] && . ~/.bash_aliases
```

---

## 13.10 Exercises

### Exercise 1: dotfiles Manager
สร้าง script ที่:
- Install/update dotfiles จาก git repo
- สร้าง symlinks ใน $HOME
- Backup existing files
- Works on Ubuntu/macOS

### Exercise 2: Environment Switcher
สร้าง script ที่ switch environment:
- development, staging, production
- Load ตัวแปรที่ตรงกัน
- Show current environment
- Prevent accidents ใน production

### Exercise 3: Profile Optimizer
วิเคราะห์ .bashrc/.zshrc:
- หา aliases ที่ไม่ได้ใช้
- วัด startup time
- แนะนำ optimizations

### Exercise 4: Secret Vault
สร้าง simple secret manager:
- Encrypt ด้วย GPG
- Store ใน ~/.secrets/
- Load เฉพาะ secrets ที่ต้องการ
- Never echo to terminal

---

## สรุป Part 13

✅ Environment variables (export, unset, printenv)  
✅ Built-in shell variables  
✅ Shell options (set, shopt)  
✅ Configuration files (.bashrc, .bash_profile)  
✅ Custom prompts (PS1, PS2, PS4)  
✅ Aliases & functions  
✅ Tab completion  
✅ .env file management  
✅ Secret handling  
✅ Complete .bashrc example  

---

**→ Part 14: Permissions & Ownership**
