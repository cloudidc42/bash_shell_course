# Part 12: Process Management
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 12.1 Processes พื้นฐาน

```bash
#!/bin/bash

# ดู processes ทั้งหมด
ps aux                         # BSD style (all processes)
ps -ef                         # System V style
ps -eo pid,ppid,user,%cpu,%mem,comm  # custom format

# Process tree
ps auxf                        # forest view
pstree                         # tree format
pstree -p                      # with PIDs
pstree -u                      # with users

# Real-time monitoring
top                            # interactive
htop                           # enhanced (install first)
atop                           # advanced monitor

# ข้อมูล process เดียว
ps -p 1234 -o pid,ppid,user,comm,%cpu,%mem

# PID ของตัวเอง
echo "My PID: $$"
echo "Parent PID: $PPID"
echo "Last BG PID: $!"

# Process states
# R = Running
# S = Sleeping (interruptible)
# D = Sleeping (uninterruptible, I/O wait)
# Z = Zombie (เสร็จแล้วแต่ parent ยัง wait)
# T = Stopped
# < = High priority
# N = Low priority (nice)
# s = Session leader
# l = Multi-threaded
# + = Foreground process group

# Process info from /proc
cat /proc/$$/status           # process status
cat /proc/$$/cmdline          # command line
cat /proc/$$/environ          # environment
ls /proc/$$/fd/               # file descriptors
cat /proc/$$/maps             # memory maps
cat /proc/$$/net/tcp          # TCP connections
```

---

## 12.2 Process Creation

```bash
# Fork: bash สร้าง child process ด้วย fork()
# Exec: child process แทนที่ตัวเองด้วย new program

# ทุกครั้งที่รัน command → bash fork + exec
ls          # fork → exec ls
echo "hi"   # bash builtin (no fork needed)

# Background process
sleep 10 &          # run in background
echo "PID: $!"      # PID ของ background process

# Process substitution (creates subprocess)
wc -l <(ls /etc)    # subshell in <()
diff <(sort file1) <(sort file2)

# Command substitution (subshell)
output=$(ls /tmp)   # runs in subshell

# Subshell with ( )
(
    cd /tmp
    echo "In subshell: $PWD"
    export MY_VAR="subshell"
)
echo "Back in parent: $PWD"
echo "MY_VAR: ${MY_VAR:-unset}"  # unset (subshell can't modify parent)

# spawn subprocess
spawn_worker() {
    local id=$1
    while true; do
        echo "Worker $id: processing..."
        sleep $((RANDOM % 3 + 1))
    done
}

# Start multiple workers
for i in {1..3}; do
    spawn_worker $i &
    echo "Started worker $i (PID: $!)"
done

# Wait for all
wait
echo "All workers done"
```

---

## 12.3 Signals

```bash
# Common signals
# SIGHUP  (1)  - Hangup (terminal disconnect, or reload config)
# SIGINT  (2)  - Interrupt (Ctrl+C)
# SIGQUIT (3)  - Quit (Ctrl+\, with core dump)
# SIGKILL (9)  - Kill (cannot be caught or ignored!)
# SIGTERM (15) - Terminate (default of kill command)
# SIGSTOP (19) - Stop (cannot be caught, Ctrl+Z equivalent)
# SIGCONT (18) - Continue (resume stopped process)
# SIGUSR1 (10) - User defined
# SIGUSR2 (12) - User defined
# SIGALRM (14) - Alarm clock
# SIGCHLD (17) - Child stopped or terminated

# Show all signals
kill -l
trap -l    # in bash

# Send signals
kill -15 1234           # SIGTERM (graceful)
kill -TERM 1234         # same
kill -9 1234            # SIGKILL (force)
kill -HUP 1234          # reload config
kill -STOP 1234         # pause process
kill -CONT 1234         # resume process

# Kill by name
pkill nginx             # SIGTERM to all nginx
pkill -9 nginx          # SIGKILL to all nginx
pkill -HUP apache2      # SIGHUP to apache2
killall python3         # terminate all python3

# Signal to process group
kill -- -1234           # send to process group 1234
kill -9 -1              # kill all processes (dangerous!)

# Trap signals in script
trap 'echo "Caught SIGINT, cleaning up..."' INT
trap 'echo "Caught SIGTERM"' TERM
trap 'cleanup' EXIT      # runs on any exit

cleanup() {
    echo "Running cleanup..."
    rm -f /tmp/myapp.pid
    # ... cleanup tasks ...
}

# Ignore signals
trap '' INT TERM          # ignore Ctrl+C and SIGTERM

# Reset to default
trap - INT TERM           # restore default behavior

# Catch multiple signals
trap 'handle_signal $?' HUP INT TERM USR1 USR2

handle_signal() {
    local sig=$1
    echo "Received signal: $sig"
    case $sig in
        1)  reload_config ;;
        2|15) graceful_shutdown ;;
        10) log_status ;;
    esac
}
```

---

## 12.4 Job Control

```bash
# Jobs (processes started from this shell)
sleep 30 &          # start background job
sleep 60 &          # start another
jobs                # list jobs
jobs -l             # with PIDs

# Output:
# [1]  Running    sleep 30 &
# [2]- Running    sleep 60 &

# Bring to foreground
fg %1               # job 1
fg %sleep           # by name

# Send to background
bg %1               # continue job 1 in background

# Suspend current foreground job
# Ctrl+Z → sends SIGTSTP

# Wait for specific job
wait %1             # wait for job 1
wait 1234           # wait for PID
wait                # wait for all background jobs

# Disown: detach from shell
sleep 1000 &
disown %1           # shell won't send SIGHUP to job 1 on exit
disown -a           # disown all
disown -h           # mark as no-HUP (still in jobs list)

# nohup: run immune to hangups
nohup command &     # output to nohup.out
nohup command > /dev/null 2>&1 &  # redirect output

# screen/tmux for persistent sessions
screen -S mysession   # start named screen session
screen -ls            # list sessions
screen -r mysession   # reattach

tmux new -s work      # new tmux session
tmux ls               # list sessions
tmux attach -t work   # attach to session
```

---

## 12.5 Process Priority

```bash
# nice: set priority at start
# Range: -20 (highest) to 19 (lowest)
# Default: 0, only root can set negative values

nice -n 10 command       # lower priority (+10 nice)
nice -n -5 command       # higher priority (root only)
nice --19 command        # lowest priority

# renice: change running process priority
renice 10 -p 1234        # by PID
renice 5 -u username     # all processes of user
renice -5 -p 1234        # increase priority (root)

# ionice: I/O scheduling priority
ionice -c 3 command      # idle I/O class
ionice -c 2 -n 7 command # best-effort, lowest priority
ionice -p 1234           # show current class

# chrt: real-time scheduling
chrt -r 50 command       # SCHED_RR with priority 50
chrt -f 99 command       # SCHED_FIFO (highest real-time)
chrt -o 0 command        # SCHED_OTHER (normal)
chrt -p 1234             # show scheduling info

# cpulimit: limit CPU usage
cpulimit -l 50 command   # limit to 50% CPU
cpulimit -p 1234 -l 25   # limit existing process

# taskset: set CPU affinity
taskset -c 0 command             # run on CPU 0 only
taskset -c 0,2 command           # run on CPUs 0 and 2
taskset -c 0-3 command           # run on CPUs 0-3
taskset -p 1234                  # show current affinity
taskset -cp 0 1234               # set affinity of existing process
```

---

## 12.6 Process Monitoring

```bash
#!/bin/bash
# process_monitor.sh - Monitor processes

# Monitor specific process
monitor_process() {
    local name=$1
    local interval=${2:-5}
    
    echo "Monitoring: $name (every ${interval}s)"
    
    while true; do
        local pids
        pids=$(pgrep -f "$name" 2>/dev/null)
        
        if [[ -z "$pids" ]]; then
            echo "[$(date +%H:%M:%S)] Process '$name' NOT FOUND"
        else
            for pid in $pids; do
                if [[ -d /proc/$pid ]]; then
                    local cpu mem vsz rss
                    read -r _ _ _ _ _ _ cpu mem vsz rss < <(
                        ps -o pid,ppid,user,%cpu,%mem,vsz,rss,comm -p "$pid" | tail -1
                    )
                    printf "[%s] PID:%-7d CPU:%-6s MEM:%-6s VSZ:%-8s RSS:%-8s\n" \
                        "$(date +%H:%M:%S)" "$pid" "$cpu%" "$mem%" "${vsz}K" "${rss}K"
                fi
            done
        fi
        
        sleep "$interval"
    done
}

# Watch for high CPU processes
watch_cpu() {
    local threshold=${1:-80}
    
    while true; do
        ps aux --no-headers | \
            awk -v thresh="$threshold" '$3 > thresh {
                printf "HIGH CPU: PID=%s USER=%s CPU=%.1f%% CMD=%s\n",
                       $2, $1, $3, $11
            }'
        sleep 10
    done
}

# Alert on zombie processes
watch_zombies() {
    local prev_count=0
    
    while true; do
        local count
        count=$(ps aux | grep -c "^.*Z.*" || echo 0)
        
        if (( count > prev_count )); then
            echo "WARNING: $count zombie processes detected!"
            ps aux | awk '$8=="Z" {print "ZOMBIE:", $2, $11}'
        fi
        
        prev_count=$count
        sleep 30
    done
}

# Resource usage summary
resource_summary() {
    echo "=== System Resources ==="
    echo ""
    
    # CPU
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d% -f1)
    echo "CPU Usage: ${cpu_usage}%"
    
    # Memory
    read -r total used free <<< $(free -m | awk '/^Mem:/{print $2, $3, $4}')
    local mem_pct=$((used * 100 / total))
    echo "Memory: ${used}MB / ${total}MB (${mem_pct}%)"
    
    # Disk I/O
    if command -v iostat &>/dev/null; then
        echo ""
        echo "Disk I/O:"
        iostat -d 1 1 | awk 'NR>3 && /[a-z]/{printf "  %-10s read:%8.1f KB/s write:%8.1f KB/s\n", $1, $3, $4}'
    fi
    
    # Load average
    read -r one five fifteen < <(cat /proc/loadavg | awk '{print $1, $2, $3}')
    local cpus
    cpus=$(nproc)
    echo ""
    echo "Load Average: $one (1m), $five (5m), $fifteen (15m) [${cpus} CPUs]"
    
    # Top 5 by CPU
    echo ""
    echo "Top 5 Processes (CPU):"
    ps aux --sort=-%cpu | head -6 | tail -5 | \
        awk '{printf "  %-12s PID:%-8s CPU:%-6s MEM:%-6s %s\n", $1, $2, $3"%", $4"%", $11}'
    
    # Top 5 by Memory
    echo ""
    echo "Top 5 Processes (Memory):"
    ps aux --sort=-%mem | head -6 | tail -5 | \
        awk '{printf "  %-12s PID:%-8s CPU:%-6s MEM:%-6s %s\n", $1, $2, $3"%", $4"%", $11}'
}
```

---

## 12.7 Process Communication

```bash
# Pipes: one-directional
producer | consumer

# Named Pipes (FIFO)
mkfifo /tmp/mypipe
echo "data" > /tmp/mypipe &     # write (blocks until reader)
cat /tmp/mypipe                  # read

# Two-way with mkfifo
mkfifo /tmp/req /tmp/resp

# Server
(
    while IFS= read -r request < /tmp/req; do
        echo "Processing: $request" >&2
        echo "Response to: $request" > /tmp/resp
    done
) &

# Client
echo "query 1" > /tmp/req
read response < /tmp/resp
echo "Got: $response"

# Coproc for bidirectional communication
coproc WORKER {
    while IFS= read -r line; do
        echo "PROCESSED: $line"
    done
}

echo "hello" >&"${WORKER[1]}"
read result <&"${WORKER[0]}"
echo "Result: $result"

# Shared memory via /dev/shm
SHMEM=/dev/shm/myapp_$$

# Write
echo "shared_data" > "$SHMEM"

# Read from another process
cat "$SHMEM"

# Cleanup
rm -f "$SHMEM"

# Process synchronization with semaphore (file lock)
sem_wait() {
    local lock=$1
    exec {fd}>"$lock"
    flock -x "$fd"
    eval "_SEM_FD_${lock//\//_}=$fd"
}

sem_post() {
    local lock=$1
    local fd_var="_SEM_FD_${lock//\//_}"
    flock -u "${!fd_var}"
    eval "exec ${!fd_var}>&-"
}
```

---

## 12.8 Daemon Processes

```bash
#!/usr/bin/env bash
# daemon.sh - Template สำหรับ daemon process

DAEMON_NAME="myapp"
PID_FILE="/var/run/${DAEMON_NAME}.pid"
LOG_FILE="/var/log/${DAEMON_NAME}.log"
LOCK_FILE="/var/lock/${DAEMON_NAME}.lock"

# Main daemon function (runs as background process)
daemon_main() {
    echo "Daemon started (PID: $$)" >> "$LOG_FILE"
    
    while true; do
        # Main work here
        echo "$(date): Working..." >> "$LOG_FILE"
        
        # Handle SIGHUP for reload
        if [[ -f /tmp/${DAEMON_NAME}.reload ]]; then
            rm -f /tmp/${DAEMON_NAME}.reload
            echo "$(date): Reloading config..." >> "$LOG_FILE"
            load_config
        fi
        
        sleep 60
    done
}

start() {
    if [[ -f "$PID_FILE" ]] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
        echo "$DAEMON_NAME already running (PID: $(cat "$PID_FILE"))"
        return 1
    fi
    
    echo "Starting $DAEMON_NAME..."
    
    # Daemonize
    (
        # Detach from terminal
        setsid 2>/dev/null || true
        
        # Close stdin, redirect stdout/stderr
        exec < /dev/null
        exec >> "$LOG_FILE" 2>&1
        
        # Set umask
        umask 022
        
        # Change to safe directory
        cd / || exit 1
        
        # Run main function
        daemon_main
    ) &
    
    local pid=$!
    echo $pid > "$PID_FILE"
    echo "$DAEMON_NAME started (PID: $pid)"
}

stop() {
    if [[ ! -f "$PID_FILE" ]]; then
        echo "$DAEMON_NAME not running"
        return 1
    fi
    
    local pid
    pid=$(cat "$PID_FILE")
    
    if kill -0 "$pid" 2>/dev/null; then
        echo "Stopping $DAEMON_NAME (PID: $pid)..."
        kill -TERM "$pid"
        
        # Wait up to 10 seconds
        for ((i=0; i<10; i++)); do
            kill -0 "$pid" 2>/dev/null || break
            sleep 1
        done
        
        if kill -0 "$pid" 2>/dev/null; then
            echo "Force killing..."
            kill -9 "$pid"
        fi
    fi
    
    rm -f "$PID_FILE"
    echo "$DAEMON_NAME stopped"
}

status() {
    if [[ -f "$PID_FILE" ]]; then
        local pid
        pid=$(cat "$PID_FILE")
        if kill -0 "$pid" 2>/dev/null; then
            echo "$DAEMON_NAME running (PID: $pid)"
            return 0
        else
            echo "$DAEMON_NAME dead (stale PID file)"
            return 1
        fi
    else
        echo "$DAEMON_NAME not running"
        return 1
    fi
}

case "${1:-}" in
    start)   start   ;;
    stop)    stop    ;;
    restart) stop; start ;;
    status)  status  ;;
    *)       echo "Usage: $0 {start|stop|restart|status}"; exit 1 ;;
esac
```

---

## 12.9 Process Management Script

```bash
#!/usr/bin/env bash
# procman.sh - Process manager

# Kill process tree (including children)
kill_tree() {
    local pid=$1
    local signal=${2:-15}
    
    # Get all descendant PIDs
    local pids=($pid)
    local i=0
    
    while (( i < ${#pids[@]} )); do
        local children
        children=$(pgrep -P "${pids[$i]}" 2>/dev/null)
        for child in $children; do
            pids+=($child)
        done
        ((i++))
    done
    
    # Kill in reverse order (children first)
    for ((i=${#pids[@]}-1; i>=0; i--)); do
        kill -"$signal" "${pids[$i]}" 2>/dev/null
    done
}

# Graceful shutdown with timeout
graceful_shutdown() {
    local pid=$1
    local timeout=${2:-30}
    
    echo "Sending SIGTERM to PID $pid..."
    kill -TERM "$pid" 2>/dev/null || return 1
    
    # Wait for graceful exit
    local waited=0
    while kill -0 "$pid" 2>/dev/null; do
        (( waited >= timeout )) && break
        sleep 1
        ((waited++))
    done
    
    if kill -0 "$pid" 2>/dev/null; then
        echo "Force killing PID $pid after ${timeout}s..."
        kill -9 "$pid" 2>/dev/null
    fi
}

# Find and kill by port
kill_by_port() {
    local port=$1
    local pids
    pids=$(lsof -t -i:"$port" 2>/dev/null)
    
    if [[ -z "$pids" ]]; then
        echo "No process on port $port"
        return 1
    fi
    
    echo "Killing processes on port $port: $pids"
    kill -9 $pids
}

# Resource limits with ulimit
set_limits() {
    ulimit -n 65536       # max file descriptors
    ulimit -u 512         # max user processes
    ulimit -v 1048576     # max virtual memory (KB)
    ulimit -c unlimited   # core dump size
}

# Process watchdog
watchdog() {
    local cmd=$1
    local max_restarts=${2:-5}
    local restart_delay=${3:-5}
    local restarts=0
    
    while (( restarts <= max_restarts )); do
        echo "Starting: $cmd"
        eval "$cmd"
        local exit_code=$?
        
        if (( exit_code == 0 )); then
            echo "Process exited normally"
            return 0
        fi
        
        ((restarts++))
        echo "Process crashed (exit: $exit_code), restart $restarts/$max_restarts"
        sleep "$restart_delay"
    done
    
    echo "Max restarts reached, giving up"
    return 1
}
```

---

## 12.10 Exercises

### Exercise 1: Process Monitor
สร้าง script ที่:
- Monitor ทุก process ทุก 5 วินาที
- แจ้งเตือนถ้า CPU > 80% หรือ RAM > 90%
- บันทึก history ใน CSV
- แสดง real-time dashboard

### Exercise 2: Service Manager
สร้าง simple service manager:
- Start/Stop/Status/Restart service
- Auto-restart on crash (watchdog)
- Log rotation
- Health check endpoint

### Exercise 3: Parallel Processor
สร้าง script ที่:
- รับรายการ tasks จากไฟล์
- รัน parallel ด้วย worker pool (N workers)
- Progress bar
- Error handling

### Exercise 4: Process Profiler
สร้าง script ที่:
- Record CPU และ memory usage ทุกวินาที
- สร้างกราฟ ASCII
- สรุป statistics

---

## สรุป Part 12

✅ Process basics (ps, top, pstree)  
✅ Process creation (fork, exec, subshell)  
✅ Signals (kill, trap, pkill)  
✅ Job control (fg, bg, jobs, nohup)  
✅ Process priority (nice, renice, ionice)  
✅ Process monitoring  
✅ IPC (pipes, FIFO, coproc)  
✅ Daemon processes  
✅ Process management utilities  

---

**→ Part 13: Environment Variables & Shell Config**
