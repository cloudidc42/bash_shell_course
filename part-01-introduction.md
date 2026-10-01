# Part 01: Introduction to Shell & Terminal
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 1.1 Shell คืออะไร?

**Shell** คือโปรแกรมที่ทำหน้าที่เป็นตัวกลางระหว่างผู้ใช้กับ Kernel ของระบบปฏิบัติการ  
เมื่อคุณพิมพ์คำสั่งใน Terminal, Shell จะรับคำสั่ง → แปลความหมาย → ส่งให้ Kernel ทำงาน

```
[User] → [Terminal] → [Shell] → [Kernel] → [Hardware]
```

### ประเภทของ Shell

| Shell | Path | คุณสมบัติ |
|-------|------|----------|
| **bash** | `/bin/bash` | Bourne Again Shell - ใช้งานแพร่หลายที่สุด |
| **sh** | `/bin/sh` | POSIX Shell - พื้นฐาน portable |
| **zsh** | `/bin/zsh` | Z Shell - features มาก, macOS default |
| **fish** | `/usr/bin/fish` | Friendly Interactive Shell |
| **dash** | `/bin/dash` | Debian Almquist Shell - เร็ว lightweight |
| **ksh** | `/bin/ksh` | KornShell - enterprise Unix |
| **tcsh** | `/bin/tcsh` | C Shell - syntax คล้าย C |

---

## 1.2 ตรวจสอบ Shell ที่ใช้งาน

```bash
# ดู Shell ปัจจุบัน
echo $SHELL
echo $0

# ดู Shell ที่ติดตั้งในระบบ
cat /etc/shells

# ดู version ของ bash
bash --version

# ดู default shell ของ user
grep "^$(whoami)" /etc/passwd
getent passwd $(whoami)
```

**ผลลัพธ์ตัวอย่าง:**
```
/bin/bash
bash, version 5.1.16(1)-release (x86_64-pc-linux-gnu)
```

---

## 1.3 Terminal vs Console vs Shell

```
Terminal Emulator (xterm, gnome-terminal, iTerm2, alacritty)
    ↓ ส่ง keystrokes / รับ output
Pseudo-Terminal (PTY - /dev/pts/0, /dev/pts/1, ...)
    ↓ เชื่อมต่อ
Shell Process (bash PID 1234)
    ↓ fork/exec
Child Processes (ls, grep, vim, ...)
```

| คำศัพท์ | ความหมาย |
|---------|----------|
| **Terminal** | อุปกรณ์หรือโปรแกรมสำหรับรับส่งข้อมูล text |
| **Console** | Physical terminal ที่ต่อตรงกับเครื่อง |
| **TTY** | Teletypewriter - ชื่อเดิมของ terminal device |
| **PTY** | Pseudo-TTY - virtual terminal |
| **Shell** | โปรแกรม interpreter รับและรัน commands |

---

## 1.4 การทำงานของ Shell

เมื่อคุณพิมพ์คำสั่ง Shell ทำงาน 7 ขั้นตอน:

```
1. Read      → อ่าน input จาก user
2. Tokenize  → แบ่ง input เป็น tokens
3. Parse     → วิเคราะห์ grammar/syntax
4. Expand    → ขยาย variables, globs, etc.
5. Redirect  → จัดการ I/O redirection
6. Execute   → รันคำสั่ง
7. Wait      → รอผลลัพธ์
```

ตัวอย่าง: `echo "Hello $USER" > output.txt`
```
Token:    [echo] ["Hello $USER"] [>] [output.txt]
Parse:    command=echo, args=["Hello $USER"], redirect=>output.txt
Expand:   args=["Hello john"]
Redirect: stdout → output.txt
Execute:  exec echo "Hello john"
```

---

## 1.5 คำสั่งพื้นฐานที่จำเป็น

### Navigation Commands

```bash
# ดู directory ปัจจุบัน (Print Working Directory)
pwd

# เปลี่ยน directory
cd /home/user          # ไปที่ absolute path
cd Documents           # ไปที่ relative path
cd ..                  # ไปที่ parent directory
cd -                   # กลับไป directory ก่อนหน้า
cd ~                   # กลับ home directory
cd ~/Downloads         # ไปที่ Downloads ใน home

# แสดงไฟล์ใน directory (List)
ls                     # แบบพื้นฐาน
ls -l                  # Long format (permissions, size, date)
ls -la                 # รวม hidden files
ls -lh                 # Human readable size (KB, MB, GB)
ls -lt                 # เรียงตาม time modified
ls -lr                 # Reverse order
ls -R                  # Recursive (subdirectories ด้วย)
ls --color=auto        # สีตามประเภทไฟล์
```

### File Operations

```bash
# สร้างไฟล์เปล่า
touch file.txt
touch file1.txt file2.txt file3.txt

# สร้าง directory
mkdir mydir
mkdir -p path/to/nested/dir    # สร้าง parent dirs ด้วย
mkdir -m 755 secured_dir       # สร้างพร้อมกำหนด permissions

# คัดลอกไฟล์
cp source.txt destination.txt
cp -r sourcedir/ destdir/      # คัดลอก directory (recursive)
cp -p file1 file2              # preserve permissions, timestamps
cp -i file1 file2              # interactive (ถามก่อน overwrite)
cp -v file1 file2              # verbose output

# ย้ายหรือ rename ไฟล์
mv oldname.txt newname.txt
mv file.txt /tmp/
mv -i src dst                  # interactive

# ลบไฟล์
rm file.txt
rm -r directory/               # ลบ directory (recursive)
rm -f file.txt                 # force (ไม่ถาม)
rm -rf directory/              # ⚠️ ระวัง! ลบทุกอย่างโดยไม่ถาม

# ดูเนื้อหาไฟล์
cat file.txt                   # แสดงทั้งหมด
cat -n file.txt                # แสดงพร้อม line numbers
head file.txt                  # แสดง 10 บรรทัดแรก
head -n 20 file.txt            # แสดง 20 บรรทัดแรก
tail file.txt                  # แสดง 10 บรรทัดสุดท้าย
tail -n 30 file.txt            # แสดง 30 บรรทัดสุดท้าย
tail -f /var/log/syslog        # follow (real-time)
less file.txt                  # pager (q=quit, /=search)
more file.txt                  # pager แบบเก่า
```

### System Information

```bash
# ข้อมูลระบบ
uname -a                       # ข้อมูล kernel ทั้งหมด
uname -r                       # kernel version
uname -m                       # machine hardware (x86_64, arm64)
hostname                       # ชื่อ host
hostname -I                    # IP addresses
whoami                         # username ปัจจุบัน
id                             # user id, group id
id username                    # id ของ user อื่น

# ข้อมูล hardware/OS
lscpu                          # CPU information
lsmem                          # Memory information
free -h                        # RAM usage (human readable)
df -h                          # Disk usage
df -h /                        # Disk usage ของ root partition
lsblk                          # Block devices
lspci                          # PCI devices
lsusb                          # USB devices

# Uptime และ load
uptime                         # เวลาทำงาน + load average
w                              # users logged in + load
last                           # login history
lastlog                        # last login ของทุก user
```

### Process Management

```bash
# ดู processes
ps                             # processes ของ session นี้
ps aux                         # ทุก process ใน system
ps -ef                         # POSIX format
ps -p 1234                     # process ด้วย PID
pstree                         # แสดงเป็น tree

# top/htop
top                            # interactive process viewer
htop                           # enhanced version (ต้องติดตั้ง)

# Kill process
kill 1234                      # ส่ง SIGTERM ไปที่ PID
kill -9 1234                   # SIGKILL (force kill)
kill -l                        # แสดง signal ทั้งหมด
killall nginx                  # kill โดยใช้ชื่อ
pkill -f "python script.py"    # kill ด้วย pattern

# Background/Foreground
command &                      # run in background
jobs                           # ดู background jobs
fg %1                          # bring job 1 to foreground
bg %1                          # ส่ง job 1 ไป background
ctrl+z                         # suspend current process
ctrl+c                         # interrupt (SIGINT)
nohup command &                # รันต่อแม้ terminal ปิด
disown %1                      # detach job จาก shell
```

---

## 1.6 Help System

```bash
# Manual pages
man ls                         # man page ของ ls
man -k keyword                 # ค้นหาจาก keyword
man 5 passwd                   # section 5 ของ passwd
man -a printf                  # ทุก section ของ printf

# Built-in help
help cd                        # help สำหรับ bash builtins
help if
help for

# --help flag
ls --help
grep --help
find --help | head -50

# info pages (GNU programs)
info coreutils
info bash

# whatis
whatis ls                      # คำอธิบายสั้น
whatis grep
apropos network                # เหมือน man -k
```

---

## 1.7 Shell Prompt คืออะไร?

```bash
# Default prompt (PS1)
user@hostname:~$               # regular user
root@hostname:~#               # root user

# ดู PS1 variable
echo $PS1

# Components ใน prompt:
# \u = username
# \h = hostname (short)
# \H = hostname (full)
# \w = current directory (full path)
# \W = current directory (basename only)
# \$ = $ for user, # for root
# \t = time HH:MM:SS
# \d = date
# \n = newline
# \! = history number

# ตัวอย่างกำหนด prompt แบบง่าย
PS1='\u@\h:\w\$ '

# Prompt สีสวยงาม
PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

---

## 1.8 History & Keyboard Shortcuts

### History Commands

```bash
# ดู command history
history
history 20                     # 20 คำสั่งล่าสุด
history | grep ssh             # ค้นหาใน history

# รัน command จาก history
!!                             # รัน command ล่าสุด
!50                            # รัน command หมายเลข 50
!ssh                           # รัน command ล่าสุดที่ขึ้นด้วย ssh
!?config                       # รัน command ล่าสุดที่มี "config"

# History expansion
!$                             # argument สุดท้ายของ command ก่อนหน้า
!*                             # arguments ทั้งหมดของ command ก่อนหน้า
^old^new                       # แก้ไข command ก่อนหน้า

# จัดการ history
history -c                     # ล้าง history ทั้งหมด
history -d 50                  # ลบ command หมายเลข 50
unset HISTFILE                 # ไม่บันทึก history (session นี้)

# History settings
HISTSIZE=10000                 # จำนวน commands ใน memory
HISTFILESIZE=20000             # จำนวน commands ในไฟล์
HISTCONTROL=ignoredups         # ไม่บันทึก duplicates
HISTCONTROL=ignorespace        # ไม่บันทึก commands ที่ขึ้นด้วย space
HISTCONTROL=ignoreboth         # ทั้งสอง
```

### Keyboard Shortcuts (Readline)

```
Navigation:
Ctrl+A          ไปต้นบรรทัด
Ctrl+E          ไปท้ายบรรทัด
Ctrl+F / →      ไปข้างหน้า 1 character
Ctrl+B / ←      ไปข้างหลัง 1 character
Alt+F           ไปข้างหน้า 1 word
Alt+B           ไปข้างหลัง 1 word

Editing:
Ctrl+D          ลบ character ข้างหน้า cursor / EOF
Ctrl+H / Backspace  ลบ character หลัง cursor
Ctrl+K          ลบจาก cursor ถึงท้ายบรรทัด
Ctrl+U          ลบจาก cursor ถึงต้นบรรทัด
Ctrl+W          ลบ word ข้างหลัง cursor
Alt+D           ลบ word ข้างหน้า cursor
Ctrl+Y          Paste (yank) ที่ถูก kill ล่าสุด
Ctrl+_          Undo

History:
Ctrl+P / ↑      command ก่อนหน้า
Ctrl+N / ↓      command ถัดไป
Ctrl+R          Reverse search (พิมพ์เพื่อค้นหา)
Ctrl+S          Forward search
Ctrl+G          ยกเลิก search

Process:
Ctrl+C          Interrupt (SIGINT)
Ctrl+Z          Suspend (SIGTSTP)
Ctrl+D          EOF / ออกจาก shell
Ctrl+L          Clear screen
Ctrl+\          Quit (SIGQUIT)

Completion:
Tab             Auto-complete
Tab Tab         แสดงตัวเลือก
Alt+?           แสดงตัวเลือก completion
```

---

## 1.9 Shell Configuration Files

```bash
# Login Shell (เมื่อ login เข้าระบบ)
/etc/profile              # system-wide, รันก่อน
/etc/profile.d/*.sh       # files ใน profile.d/
~/.bash_profile           # user's login config
~/.bash_login             # ถ้าไม่มี .bash_profile
~/.profile                # ถ้าไม่มีทั้งสอง

# Interactive Non-Login Shell (เปิด terminal ใหม่)
/etc/bash.bashrc          # system-wide bashrc
~/.bashrc                 # user's interactive config

# Logout
~/.bash_logout            # รันเมื่อ logout

# ลำดับการอ่านไฟล์ (Login Shell):
# /etc/profile → ~/.bash_profile → ~/.bashrc
```

```bash
# ตัวอย่าง ~/.bashrc ที่มีประโยชน์
# -----------------------------------------------

# Aliases
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'
alias free='free -h'

# Custom PS1 Prompt
PS1='\[\e[1;32m\]\u\[\e[0m\]@\[\e[1;34m\]\h\[\e[0m\]:\[\e[1;33m\]\w\[\e[0m\]\$ '

# Path additions
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# History settings
export HISTSIZE=50000
export HISTFILESIZE=100000
export HISTCONTROL=ignoreboth:erasedups
export HISTTIMEFORMAT="%F %T "

# Editor
export EDITOR=vim
export VISUAL=vim

# Colors for ls
export LS_COLORS='di=1;34:ln=1;36:ex=1;32'

# Functions
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Reload bashrc
alias reload='source ~/.bashrc'
```

---

## 1.10 การเขียน Shell Script แรก

### 1. สร้างไฟล์ script

```bash
# สร้างไฟล์ใหม่
touch hello.sh

# หรือใช้ editor
nano hello.sh
vim hello.sh
```

### 2. โครงสร้าง Script พื้นฐาน

```bash
#!/bin/bash
# ===================================================
# Script Name: hello.sh
# Description: My first bash script
# Author: Your Name
# Date: 2024-01-01
# Version: 1.0
# ===================================================

# แสดงข้อความ
echo "Hello, World!"
echo "สวัสดีชาวโลก!"

# แสดงข้อมูลระบบ
echo "Username: $(whoami)"
echo "Hostname: $(hostname)"
echo "Date: $(date)"
echo "Directory: $(pwd)"
```

### 3. Shebang Line (`#!/bin/bash`)

```bash
#!/bin/bash        # ใช้ bash โดยตรง
#!/usr/bin/env bash # ค้นหา bash ใน PATH (portable)
#!/bin/sh          # POSIX sh
#!/usr/bin/python3 # Python script
#!/usr/bin/perl    # Perl script

# ทำไม env bash ดีกว่า?
# ระบบต่างๆ อาจมี bash ที่ path ต่างกัน
# env จะค้นหา bash ใน PATH อัตโนมัติ
```

### 4. กำหนดสิทธิ์รัน

```bash
# ดู permissions ปัจจุบัน
ls -l hello.sh
# -rw-r--r-- 1 user user 150 Jan 1 hello.sh

# เพิ่ม execute permission
chmod +x hello.sh
chmod 755 hello.sh

# ดูอีกครั้ง
ls -l hello.sh
# -rwxr-xr-x 1 user user 150 Jan 1 hello.sh
```

### 5. รัน Script

```bash
# วิธีที่ 1: ระบุ path เต็ม
/home/user/hello.sh

# วิธีที่ 2: relative path (ต้องอยู่ใน directory เดียวกัน)
./hello.sh

# วิธีที่ 3: ผ่าน bash (ไม่ต้อง chmod +x)
bash hello.sh
sh hello.sh

# วิธีที่ 4: source (รันใน shell ปัจจุบัน - ไม่ fork)
source hello.sh
. hello.sh
```

---

## 1.11 Script ตัวอย่างสมบูรณ์: System Info Script

```bash
#!/usr/bin/env bash
# =============================================================
# Script: system_info.sh
# Description: แสดงข้อมูลระบบแบบครบถ้วน
# =============================================================

# สีสำหรับ output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'  # No Color

# ฟังก์ชันแสดง header
print_header() {
    echo -e "\n${BOLD}${BLUE}═══════════════════════════════════════════${NC}"
    echo -e "${BOLD}${CYAN}  $1${NC}"
    echo -e "${BOLD}${BLUE}═══════════════════════════════════════════${NC}"
}

# ฟังก์ชันแสดง info
print_info() {
    printf "${GREEN}  %-20s${NC}: ${YELLOW}%s${NC}\n" "$1" "$2"
}

# ========== SYSTEM ==========
print_header "SYSTEM INFORMATION"
print_info "Hostname" "$(hostname)"
print_info "OS" "$(grep PRETTY_NAME /etc/os-release 2>/dev/null | cut -d= -f2 | tr -d '"')"
print_info "Kernel" "$(uname -r)"
print_info "Architecture" "$(uname -m)"
print_info "Uptime" "$(uptime -p 2>/dev/null || uptime | awk '{print $3,$4}' | sed 's/,//')"

# ========== USER ==========
print_header "USER INFORMATION"
print_info "Username" "$(whoami)"
print_info "UID" "$(id -u)"
print_info "Groups" "$(id -Gn | tr ' ' ',')"
print_info "Home Dir" "$HOME"
print_info "Shell" "$SHELL"

# ========== CPU ==========
print_header "CPU INFORMATION"
CPU_MODEL=$(grep "model name" /proc/cpuinfo 2>/dev/null | head -1 | cut -d: -f2 | xargs)
CPU_CORES=$(nproc 2>/dev/null || grep -c "processor" /proc/cpuinfo)
CPU_LOAD=$(uptime | awk -F'load average:' '{print $2}' | xargs)
print_info "CPU Model" "${CPU_MODEL:-N/A}"
print_info "CPU Cores" "$CPU_CORES"
print_info "Load Average" "$CPU_LOAD"

# ========== MEMORY ==========
print_header "MEMORY INFORMATION"
if command -v free &>/dev/null; then
    TOTAL_MEM=$(free -h | awk '/^Mem:/ {print $2}')
    USED_MEM=$(free -h | awk '/^Mem:/ {print $3}')
    FREE_MEM=$(free -h | awk '/^Mem:/ {print $4}')
    TOTAL_SWAP=$(free -h | awk '/^Swap:/ {print $2}')
    USED_SWAP=$(free -h | awk '/^Swap:/ {print $3}')
    print_info "Total RAM" "$TOTAL_MEM"
    print_info "Used RAM" "$USED_MEM"
    print_info "Free RAM" "$FREE_MEM"
    print_info "Total Swap" "$TOTAL_SWAP"
    print_info "Used Swap" "$USED_SWAP"
fi

# ========== DISK ==========
print_header "DISK INFORMATION"
echo -e "${GREEN}  Filesystem Usage:${NC}"
df -h 2>/dev/null | awk 'NR==1{printf "  %-20s %-8s %-8s %-8s %s\n",$1,$2,$3,$4,$6} NR>1 && /^\// {printf "  %-20s %-8s %-8s %-8s %s\n",$1,$2,$3,$4,$6}'

# ========== NETWORK ==========
print_header "NETWORK INFORMATION"
print_info "Hostname" "$(hostname -f 2>/dev/null || hostname)"
if command -v ip &>/dev/null; then
    echo -e "${GREEN}  Network Interfaces:${NC}"
    ip -br addr 2>/dev/null | awk '{printf "    %-12s %-10s %s\n", $1, $2, $3}'
elif command -v ifconfig &>/dev/null; then
    ifconfig 2>/dev/null | grep -E "^[a-z]|inet " | awk '/^[a-z]/{iface=$1} /inet /{print "    "iface": "$2}'
fi

# ========== TOP PROCESSES ==========
print_header "TOP 5 PROCESSES (by CPU)"
ps aux --sort=-%cpu 2>/dev/null | awk 'NR<=6 {printf "  %-8s %-6s %-6s %s\n",$1,$2,$3,$11}' | head -6

# ========== LAST LOGINS ==========
print_header "LAST 5 LOGINS"
last 2>/dev/null | head -5 | awk '{printf "  %-12s %-10s %-20s %s\n",$1,$2,$4" "$5,$6}'

# ========== SECURITY ==========
print_header "SECURITY OVERVIEW"
print_info "SELinux" "$(getenforce 2>/dev/null || echo 'N/A')"
print_info "Firewall" "$(systemctl is-active ufw 2>/dev/null || systemctl is-active firewalld 2>/dev/null || echo 'unknown')"
print_info "Sudo" "$(sudo -l -n 2>/dev/null | grep -c NOPASSWD || echo 'restricted') NOPASSWD entries"
WORLD_WRITABLE=$(find / -maxdepth 3 -perm -o+w -type f 2>/dev/null | wc -l)
print_info "World-Writable" "$WORLD_WRITABLE files (top 3 dirs)"

echo -e "\n${BOLD}${GREEN}  Script completed at: $(date)${NC}\n"
```

### บันทึกและรัน:

```bash
chmod +x system_info.sh
./system_info.sh
```

---

## 1.12 Debugging Scripts

```bash
# เปิด debug mode (แสดงทุก command ที่รัน)
bash -x script.sh

# Verbose mode (แสดง input ที่อ่าน)
bash -v script.sh

# ทั้ง verbose + debug
bash -xv script.sh

# ใน script เอง
#!/bin/bash
set -x        # เปิด debug
set -v        # เปิด verbose
set +x        # ปิด debug
set +v        # ปิด verbose

# Strict mode (แนะนำให้ใช้เสมอ)
set -e        # exit ทันทีเมื่อ error
set -u        # error เมื่อใช้ unset variable
set -o pipefail # error เมื่อ pipe ล้มเหลว
set -euo pipefail # ทั้งหมดรวมกัน
```

---

## 1.13 Exit Codes

```bash
# ทุก command มี exit code
# 0 = สำเร็จ
# 1-255 = error (convention: 1=general error, 2=misuse, 126=no permission, 127=not found)

ls /tmp          # รัน command
echo $?          # ดู exit code ล่าสุด → 0

ls /nonexistent  # command ที่ fail
echo $?          # → 2

# ใช้ใน conditions
if ls /tmp; then
    echo "Directory exists"
fi

# Exit script
exit 0           # สำเร็จ
exit 1           # error

# Common exit codes
# 0   = Success
# 1   = General error
# 2   = Misuse of shell builtin
# 126 = Permission denied
# 127 = Command not found
# 128 = Invalid argument to exit
# 128+n = Fatal error signal n (e.g., 130 = SIGINT Ctrl+C)
# 255 = Exit status out of range
```

---

## 1.14 Exercises

### Exercise 1: สำรวจ Shell
```bash
# รันคำสั่งเหล่านี้และจดบันทึกผลลัพธ์:
echo $SHELL
echo $BASH_VERSION
type ls
type cd
which bash
file /bin/bash
```

### Exercise 2: Script แรก
สร้างไฟล์ `my_info.sh` ที่แสดง:
- ชื่อ username
- วันที่และเวลาปัจจุบัน
- directory ปัจจุบัน
- จำนวนไฟล์ใน home directory

```bash
#!/bin/bash
# แนวทาง (ลองทำเองก่อน!)
echo "User: ???"
echo "Date: ???"
echo "Dir: ???"
echo "Files: ???"
```

### Exercise 3: Keyboard Shortcuts
ฝึกใช้ shortcuts ต่อไปนี้ใน terminal:
1. `Ctrl+R` → ค้นหา history
2. `Ctrl+A` → ไปต้นบรรทัด
3. `Ctrl+E` → ไปท้ายบรรทัด
4. `!!` → รัน command ล่าสุด
5. `!$` → ใช้ argument สุดท้าย

---

## 1.15 สรุป Part 01

ในบทนี้คุณได้เรียนรู้:

✅ Shell คืออะไร และมีกี่ประเภท  
✅ วิธีการทำงานของ Shell (tokenize → parse → expand → execute)  
✅ คำสั่งพื้นฐาน: navigation, file ops, system info  
✅ History และ keyboard shortcuts  
✅ Shell configuration files  
✅ การเขียน script แรก  
✅ Debug mode และ exit codes  

---

## สิ่งที่ต้องรู้ก่อนไป Part 02

- [x] รู้จัก shell คืออะไร
- [x] รัน basic commands ได้
- [x] เขียน script เบื้องต้นได้
- [x] รู้จัก exit codes
- [x] ใช้ keyboard shortcuts ได้

---

**→ Part 02: Variables, Data Types & Arithmetic**
