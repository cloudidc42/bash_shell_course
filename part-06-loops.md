# Part 06: Loops (for, while, until)
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 6.1 for Loop

```bash
#!/bin/bash

# 1. for in - iterate over list
for item in apple banana cherry; do
    echo "$item"
done

# 2. for in - iterate over array
fruits=("apple" "banana" "cherry")
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# 3. C-style for loop
for ((i=0; i<10; i++)); do
    echo "i = $i"
done

# 4. for with range (bash 4+)
for i in {1..10}; do
    echo "$i"
done

# Range with step
for i in {0..20..2}; do   # 0, 2, 4, 6... 20
    echo "$i"
done

for i in {10..1..-2}; do  # 10, 8, 6... 2
    echo "$i"
done

# 5. for over files
for file in /etc/*.conf; do
    echo "Config: $file"
done

# 6. for over command output
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    echo "User: $user"
done

# Better: mapfile
mapfile -t users < <(awk -F: '$3 >= 1000 {print $1}' /etc/passwd)
for user in "${users[@]}"; do
    echo "User: $user"
done

# 7. Nested for loops
for i in {1..3}; do
    for j in {1..3}; do
        printf "%3d" $((i * j))
    done
    echo
done
```

---

## 6.2 while Loop

```bash
# Basic while
i=0
while [ $i -lt 5 ]; do
    echo "i = $i"
    ((i++))
done

# while with [[ ]]
count=10
while [[ $count -gt 0 ]]; do
    echo "Count: $count"
    ((count--))
done

# Infinite loop
while true; do
    echo "Running..."
    sleep 1
    break   # ออกด้วย break
done

# while : (same as while true)
while :; do
    read -p "Enter 'quit' to exit: " input
    [[ "$input" == "quit" ]] && break
    echo "You entered: $input"
done

# while read - อ่านไฟล์บรรทัดต่อบรรทัด
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/passwd

# while read with pipe
ps aux | while IFS= read -r line; do
    echo "Process: $line"
done

# while read from command
while IFS= read -r filename; do
    echo "Processing: $filename"
done < <(find /tmp -name "*.log" 2>/dev/null)

# Read multiple fields
while IFS=: read -r username _ uid gid gecos home shell; do
    echo "User: $username, UID: $uid, Shell: $shell"
done < /etc/passwd

# while with multiple conditions
x=0
y=10
while [[ $x -lt 5 && $y -gt 0 ]]; do
    echo "x=$x, y=$y"
    ((x++))
    ((y -= 2))
done
```

---

## 6.3 until Loop

```bash
# until: รันจนกว่า condition จะเป็น true (ตรงข้าม while)
i=0
until [ $i -ge 5 ]; do
    echo "i = $i"
    ((i++))
done

# รอจนกว่า service จะ start
until systemctl is-active nginx &>/dev/null; do
    echo "Waiting for nginx..."
    sleep 2
done
echo "Nginx is up!"

# รอจนกว่า file จะมีอยู่
until [ -f /tmp/done.flag ]; do
    echo "Waiting for process to complete..."
    sleep 5
done
echo "Process completed!"

# until with counter (max attempts)
MAX_TRIES=10
attempt=0
until ping -c1 google.com &>/dev/null || (( attempt >= MAX_TRIES )); do
    echo "Attempt $((attempt+1)) failed, retrying..."
    ((attempt++))
    sleep 2
done

if (( attempt >= MAX_TRIES )); then
    echo "Failed after $MAX_TRIES attempts"
else
    echo "Connected!"
fi
```

---

## 6.4 Loop Control (break, continue)

```bash
# break - ออกจาก loop
for i in {1..10}; do
    [[ $i -eq 5 ]] && break
    echo "$i"
done
# Output: 1 2 3 4

# continue - ข้ามไป iteration ถัดไป
for i in {1..10}; do
    [[ $((i % 2)) -eq 0 ]] && continue
    echo "$i"   # เฉพาะเลขคี่
done

# break N - ออกจาก N levels ของ nested loops
for i in {1..3}; do
    for j in {1..3}; do
        [[ $j -eq 2 ]] && break 2   # ออกจากทั้ง 2 loops
        echo "i=$i j=$j"
    done
done

# continue N - continue ที่ level N
for i in {1..3}; do
    for j in {1..3}; do
        [[ $j -eq 2 ]] && continue 2   # continue outer loop
        echo "i=$i j=$j"
    done
done

# Loop with exit code
while read -r line; do
    [[ "$line" == "STOP" ]] && break
    process_line "$line"
done < input.txt

# Break on error
for file in *.txt; do
    if ! process "$file"; then
        echo "Error processing $file, stopping"
        break
    fi
done
```

---

## 6.5 Loop Patterns

### Retry Pattern

```bash
retry() {
    local max=$1
    local delay=$2
    shift 2
    local cmd=("$@")
    local attempt=1
    
    while (( attempt <= max )); do
        echo "Attempt $attempt/$max: ${cmd[*]}"
        if "${cmd[@]}"; then
            return 0
        fi
        (( attempt++ ))
        if (( attempt <= max )); then
            echo "Failed, retrying in ${delay}s..."
            sleep "$delay"
        fi
    done
    
    echo "All $max attempts failed" >&2
    return 1
}

# Usage
retry 3 5 curl -f https://api.example.com/health
retry 5 2 ping -c1 8.8.8.8
```

### Progress Bar Pattern

```bash
progress_bar() {
    local current=$1
    local total=$2
    local width=50
    
    local percent=$((current * 100 / total))
    local filled=$((current * width / total))
    local empty=$((width - filled))
    
    printf "\r["
    printf "%${filled}s" | tr ' ' '█'
    printf "%${empty}s" | tr ' ' '░'
    printf "] %3d%% (%d/%d)" $percent $current $total
    
    [[ $current -eq $total ]] && echo
}

# Usage
total=100
for ((i=1; i<=total; i++)); do
    progress_bar $i $total
    sleep 0.05
done
```

### Spinner Pattern

```bash
spinner() {
    local pid=$1
    local message=${2:-"Processing..."}
    local frames=('⠋' '⠙' '⠹' '⠸' '⠼' '⠴' '⠦' '⠧' '⠇' '⠏')
    local i=0
    
    tput civis  # hide cursor
    while kill -0 "$pid" 2>/dev/null; do
        printf "\r%s %s" "${frames[((i++ % ${#frames[@]}))]}" "$message"
        sleep 0.1
    done
    printf "\r✓ %s Done\n" "$message"
    tput cnorm  # show cursor
}

# Usage
sleep 3 &
spinner $! "Loading data"
```

### Parallel Loop Pattern

```bash
# Run commands in parallel with limited concurrency
parallel_run() {
    local max_jobs=$1
    shift
    local items=("$@")
    local pids=()
    
    for item in "${items[@]}"; do
        # Wait if we have too many jobs
        while (( ${#pids[@]} >= max_jobs )); do
            # Check for finished jobs
            local new_pids=()
            for pid in "${pids[@]}"; do
                kill -0 "$pid" 2>/dev/null && new_pids+=("$pid")
            done
            pids=("${new_pids[@]}")
            sleep 0.1
        done
        
        # Start new job
        process_item "$item" &
        pids+=($!)
    done
    
    # Wait for remaining jobs
    wait "${pids[@]}"
}

# Process in batches
process_in_batches() {
    local batch_size=$1
    shift
    local items=("$@")
    local count=${#items[@]}
    
    for ((i=0; i<count; i+=batch_size)); do
        local batch=("${items[@]:$i:$batch_size}")
        echo "Processing batch $((i/batch_size + 1)): ${batch[*]}"
        
        # Process each item in batch in parallel
        for item in "${batch[@]}"; do
            echo "  Processing: $item" &
        done
        wait  # รอ batch นี้เสร็จก่อน
    done
}

process_in_batches 3 file1 file2 file3 file4 file5 file6 file7
```

---

## 6.6 Loop over Files & Directories

```bash
# Loop over files (safe - handles spaces in names)
find /path -name "*.txt" -print0 | while IFS= read -r -d '' file; do
    echo "Processing: $file"
done

# Process files recursively
process_dir() {
    local dir=$1
    
    while IFS= read -r -d '' file; do
        case "$file" in
            *.sh)  echo "Shell: $file" ;;
            *.py)  echo "Python: $file" ;;
            *.txt) echo "Text: $file" ;;
        esac
    done < <(find "$dir" -type f -print0)
}

# Loop with file info
for file in /var/log/*.log; do
    [[ -f "$file" ]] || continue
    size=$(stat -c%s "$file" 2>/dev/null || echo 0)
    lines=$(wc -l < "$file")
    echo "$(printf '%-40s' "$file") ${size}B, ${lines} lines"
done

# Find and process old files
find /tmp -mtime +7 -type f | while IFS= read -r file; do
    echo "Old file: $file"
    # rm -f "$file"  # uncomment to delete
done

# Create multiple directories
for dir in logs backups temp cache; do
    mkdir -p "/var/app/$dir"
    chmod 755 "/var/app/$dir"
done

# Batch rename files
for file in *.jpg; do
    [[ -f "$file" ]] || continue
    newname="${file/ /_}"       # spaces to underscores
    newname="${newname,,}"      # lowercase
    [[ "$file" != "$newname" ]] && mv "$file" "$newname"
done
```

---

## 6.7 Loop over Lines & CSV

```bash
# Read CSV file
while IFS=',' read -r name age email city; do
    # Skip header
    [[ "$name" == "name" ]] && continue
    echo "Name: $name, Age: $age, City: $city"
done < users.csv

# Process with field validation
while IFS=',' read -r -a fields; do
    [[ ${#fields[@]} -ne 4 ]] && echo "Invalid line: ${fields[*]}" && continue
    
    name="${fields[0]}"
    age="${fields[1]}"
    email="${fields[2]}"
    city="${fields[3]}"
    
    # Trim whitespace
    name=$(echo "$name" | xargs)
    age=$(echo "$age" | xargs)
    
    # Validate
    [[ ! "$age" =~ ^[0-9]+$ ]] && echo "Invalid age for $name" && continue
    
    echo "Valid: $name ($age) - $email"
done < data.csv

# Generate CSV
generate_report() {
    echo "Name,Age,Email,Status"
    
    while IFS= read -r user; do
        local info
        info=$(getent passwd "$user")
        [[ -z "$info" ]] && continue
        
        local age=$((RANDOM % 40 + 20))
        local email="${user}@example.com"
        local status=$( (( RANDOM % 3 == 0 )) && echo "inactive" || echo "active" )
        
        echo "$user,$age,$email,$status"
    done < <(awk -F: '$3 >= 1000 {print $1}' /etc/passwd)
}
```

---

## 6.8 Loop over Network Data

```bash
# Ping multiple hosts
hosts=("8.8.8.8" "8.8.4.4" "1.1.1.1" "192.168.1.1")

echo "Network Connectivity Test:"
for host in "${hosts[@]}"; do
    if ping -c1 -W1 "$host" &>/dev/null; then
        rtt=$(ping -c1 "$host" 2>/dev/null | grep -oP 'time=\K[0-9.]+')
        printf "  %-15s ✓ ONLINE  (%.1f ms)\n" "$host" "$rtt"
    else
        printf "  %-15s ✗ OFFLINE\n" "$host"
    fi
done

# Port scan loop
scan_ports() {
    local host=$1
    local start=${2:-1}
    local end=${3:-1024}
    
    echo "Scanning $host ports $start-$end..."
    local open_ports=()
    
    for ((port=start; port<=end; port++)); do
        if timeout 0.5 bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null; then
            open_ports+=($port)
            printf "\r  Open port: %-6d" $port
        fi
    done
    
    echo ""
    echo "Open ports: ${open_ports[*]:-none}"
}

# Download multiple files
download_files() {
    local urls=("$@")
    local success=0
    local failed=0
    
    for url in "${urls[@]}"; do
        local filename="${url##*/}"
        echo -n "Downloading $filename... "
        
        if curl -s -o "$filename" --max-time 30 "$url"; then
            echo "OK"
            ((success++))
        else
            echo "FAILED"
            ((failed++))
        fi
    done
    
    echo "Done: $success success, $failed failed"
}
```

---

## 6.9 Loop Performance

```bash
# สร้าง large array
time for i in $(seq 1 10000); do arr+=($i); done
time arr=($(seq 1 10000))   # ← เร็วกว่ามาก
time readarray -t arr < <(seq 1 10000)  # ← เร็วที่สุด

# String concatenation ใน loop
# ช้า:
result=""
for i in {1..1000}; do
    result="${result}${i} "
done

# เร็วกว่า:
result=$(seq 1 1000 | tr '\n' ' ')

# File processing ใน loop
# ช้า: รัน cat ทุก iteration
while read line; do
    echo "$line"
done < <(cat file.txt)

# เร็วกว่า: redirect โดยตรง
while read line; do
    echo "$line"
done < file.txt

# External commands ใน loop (ช้า)
for file in *.txt; do
    size=$(ls -l "$file" | awk '{print $5}')  # fork ทุก iteration
done

# ดีกว่า: ใช้ bash builtins
for file in *.txt; do
    size=$(stat -c%s "$file")  # ยังช้า แต่ดีกว่า
done

# ดีที่สุด: process once
while IFS= read -r line; do
    # parse output of ls -l once
    file="${line##* }"
    size="${line%% *}"
done < <(ls -l *.txt | awk '{print $5, $9}')
```

---

## 6.10 Real-world Loop Examples

### Log Monitor Script

```bash
#!/bin/bash
# log_monitor.sh - Monitor log file for patterns

LOG_FILE="${1:-/var/log/syslog}"
PATTERN="${2:-ERROR}"
INTERVAL="${3:-5}"
ALERT_THRESHOLD=5

declare -A error_counts
last_pos=0

monitor_log() {
    echo "Monitoring $LOG_FILE for '$PATTERN' (checking every ${INTERVAL}s)"
    echo "Press Ctrl+C to stop"
    
    trap 'echo "Monitoring stopped"; exit 0' INT TERM
    
    while true; do
        # Get new lines since last check
        local new_lines
        new_lines=$(tail -n +$((last_pos + 1)) "$LOG_FILE" 2>/dev/null)
        last_pos=$(wc -l < "$LOG_FILE")
        
        # Count pattern occurrences
        local count=0
        while IFS= read -r line; do
            [[ "$line" == *"$PATTERN"* ]] || continue
            
            ((count++))
            
            # Extract timestamp (if present)
            local timestamp
            timestamp=$(echo "$line" | grep -oP '^\S+ \S+ \S+')
            
            echo "[$timestamp] MATCH: ${line:0:100}"
        done <<< "$new_lines"
        
        # Alert if too many errors
        if (( count >= ALERT_THRESHOLD )); then
            echo "⚠️  ALERT: $count occurrences of '$PATTERN' in last ${INTERVAL}s!"
        fi
        
        sleep "$INTERVAL"
    done
}

monitor_log
```

### Backup Script with Loop

```bash
#!/bin/bash
# rotating_backup.sh - Backup with rotation

BACKUP_DIR="/backup"
SOURCE_DIRS=("/etc" "/home" "/var/www")
MAX_BACKUPS=7
DATE=$(date +%Y%m%d_%H%M%S)

create_backup() {
    local source=$1
    local dirname="${source//\//_}"
    local backup_name="${dirname}_${DATE}.tar.gz"
    local backup_path="${BACKUP_DIR}/${backup_name}"
    
    echo "Backing up $source..."
    
    if tar -czf "$backup_path" "$source" 2>/dev/null; then
        local size=$(du -sh "$backup_path" | cut -f1)
        echo "  Created: $backup_name ($size)"
        return 0
    else
        echo "  FAILED: $source"
        return 1
    fi
}

rotate_backups() {
    local pattern=$1
    
    # Get sorted list of backups
    mapfile -t backups < <(ls -t "${BACKUP_DIR}/${pattern}" 2>/dev/null)
    
    # Remove old backups beyond max
    local count=${#backups[@]}
    if (( count > MAX_BACKUPS )); then
        for ((i=MAX_BACKUPS; i<count; i++)); do
            echo "Removing old backup: ${backups[$i]}"
            rm -f "${backups[$i]}"
        done
    fi
}

# Main
mkdir -p "$BACKUP_DIR"

success=0
failed=0

for source in "${SOURCE_DIRS[@]}"; do
    if [[ -d "$source" ]]; then
        if create_backup "$source"; then
            ((success++))
        else
            ((failed++))
        fi
        
        # Rotate old backups for this source
        dirname="${source//\//_}"
        rotate_backups "${dirname}_*.tar.gz"
    else
        echo "Skipping $source: directory not found"
    fi
done

echo ""
echo "Backup summary: $success succeeded, $failed failed"
echo "Backup location: $BACKUP_DIR"
ls -lh "$BACKUP_DIR/"
```

---

## 6.11 Exercises

### Exercise 1: FizzBuzz
เขียน FizzBuzz 1-100 โดยใช้:
- for loop แบบ C-style
- while loop
- until loop

### Exercise 2: Fibonacci
สร้าง script แสดง Fibonacci sequence ด้วย while loop:
- รับ n เป็น argument
- แสดง n ตัวแรก

### Exercise 3: Process Monitor
สร้าง script ที่:
- รัน loop ทุก 5 วินาที
- แสดง top 5 processes by CPU
- บันทึก log
- หยุดเมื่อกด Ctrl+C อย่าง graceful

### Exercise 4: File Watcher
สร้าง script ที่ watch directory:
- ตรวจสอบไฟล์ใหม่ทุก 2 วินาที
- แสดง timestamp เมื่อมีไฟล์ใหม่
- เก็บ history ใน array

---

## สรุป Part 06

✅ for loop (list, array, C-style, range, files)  
✅ while loop (condition, infinite, read)  
✅ until loop  
✅ break, continue (with levels)  
✅ Retry, progress bar, spinner patterns  
✅ Parallel loop execution  
✅ File/CSV processing loops  
✅ Network loops  
✅ Performance considerations  

---

**→ Part 07: Functions**
