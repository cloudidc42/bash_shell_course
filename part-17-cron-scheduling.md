# Part 17: Cron Jobs & Task Scheduling
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 17.1 Cron Basics

```bash
# Cron daemon: crond (Linux), cron (macOS)
# แต่ละ user มี crontab ของตัวเอง

# Edit crontab
crontab -e          # edit current user's crontab
crontab -l          # list current user's crontab
crontab -r          # remove crontab (careful!)

# Edit for another user (root only)
sudo crontab -e -u username
sudo crontab -l -u username

# ─── Crontab Format ───────────────────────────────────────────
# * * * * * command
# │ │ │ │ │
# │ │ │ │ └── Day of Week (0-7, 0=Sunday, 7=Sunday)
# │ │ │ └──── Month (1-12)
# │ │ └────── Day of Month (1-31)
# │ └──────── Hour (0-23)
# └────────── Minute (0-59)

# Special characters:
# *    = any/every
# ,    = list (1,3,5)
# -    = range (1-5)
# /    = step (*/5 = every 5)
# @    = special strings

# ─── Examples ─────────────────────────────────────────────────
# Every minute
* * * * * /path/to/script.sh

# Every hour at minute 0
0 * * * * /path/to/script.sh

# Every day at 2:30 AM
30 2 * * * /path/to/script.sh

# Every Monday at 9 AM
0 9 * * 1 /path/to/script.sh

# Every weekday (Mon-Fri) at 8 AM
0 8 * * 1-5 /path/to/script.sh

# Every 15 minutes
*/15 * * * * /path/to/script.sh

# First day of every month at midnight
0 0 1 * * /path/to/script.sh

# Every day at 6 AM, 12 PM, 6 PM
0 6,12,18 * * * /path/to/script.sh

# ─── Special Strings ──────────────────────────────────────────
@reboot     # at startup
@yearly     # 0 0 1 1 *
@annually   # 0 0 1 1 *
@monthly    # 0 0 1 * *
@weekly     # 0 0 * * 0
@daily      # 0 0 * * *
@midnight   # 0 0 * * *
@hourly     # 0 * * * *

# In crontab:
@daily /path/to/daily_backup.sh
@reboot /path/to/startup_script.sh
```

---

## 17.2 Crontab Best Practices

```bash
# ─── Output Handling ──────────────────────────────────────────
# By default, cron emails output to user
# Redirect to /dev/null to suppress
0 * * * * /path/script.sh > /dev/null 2>&1

# Redirect to log file
0 2 * * * /path/backup.sh >> /var/log/backup.log 2>&1

# Log with timestamp
0 2 * * * echo "[$(date)] Starting backup" >> /var/log/cron.log && \
           /path/backup.sh >> /var/log/cron.log 2>&1

# MAILTO for email alerts
MAILTO=admin@example.com
0 2 * * * /path/backup.sh   # sends output/errors to admin

# Silence successful runs, only email on error
0 2 * * * /path/backup.sh || echo "Backup FAILED at $(date)" | mail -s "Backup Error" admin@example.com

# ─── Environment in Cron ──────────────────────────────────────
# Cron runs with minimal environment
# Always use full paths!

# Good:
0 * * * * /usr/bin/python3 /home/user/script.py

# Set environment in crontab
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/home/user

0 * * * * script.py   # uses PATH above

# Or in the script itself
#!/bin/bash
export PATH=/usr/local/bin:/usr/bin:/bin
source /home/user/.bashrc

# ─── Lock Files (prevent overlap) ─────────────────────────────
# If job runs longer than interval, next run should wait/skip
*/5 * * * * flock -n /tmp/myjob.lock /path/to/job.sh

# Or in script:
LOCK_FILE="/var/run/myjob.lock"
exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Already running"; exit 1; }
PID=$$
echo $PID >&200
# ... rest of script ...
```

---

## 17.3 System Cron Files

```bash
# ─── System-wide cron ─────────────────────────────────────────
# /etc/crontab (system crontab, has username field)
# Format: * * * * * user command
0 2 * * * root /usr/local/sbin/backup.sh

# /etc/cron.d/ (extra crontab files, also have username field)
cat > /etc/cron.d/myapp << 'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Run myapp cleanup daily at 3 AM
0 3 * * * myapp /opt/myapp/bin/cleanup.sh >> /var/log/myapp-cleanup.log 2>&1
EOF
chmod 644 /etc/cron.d/myapp

# Time-based directories
ls /etc/cron.hourly/    # runs hourly
ls /etc/cron.daily/     # runs daily
ls /etc/cron.weekly/    # runs weekly
ls /etc/cron.monthly/   # runs monthly

# Add script to daily cron
cat > /etc/cron.daily/my_daily_job << 'EOF'
#!/bin/bash
/opt/myapp/bin/daily_maintenance.sh >> /var/log/daily_maintenance.log 2>&1
EOF
chmod 755 /etc/cron.daily/my_daily_job

# anacron: runs missed jobs when system was off
cat /etc/anacrontab
# period  delay  job-id  command
# 1       5      cron.daily  run-parts /etc/cron.daily
# 7       10     cron.weekly run-parts /etc/cron.weekly
# 30      15     cron.monthly run-parts /etc/cron.monthly

# Check anacron log
cat /var/log/syslog | grep anacron
```

---

## 17.4 systemd Timers (Modern Alternative)

```bash
# systemd timers are more powerful than cron:
# - Dependency management
# - Logging via journald
# - Catch-up runs if missed
# - Monotonic or calendar-based

# ─── Create Timer Unit ────────────────────────────────────────
# Step 1: Create service unit
cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup Job
After=network.target

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup.sh
StandardOutput=journal
StandardError=journal
EOF

# Step 2: Create timer unit
cat > /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Run backup daily
Requires=backup.service

[Timer]
# Calendar-based (like cron)
OnCalendar=*-*-* 02:00:00     # daily at 2 AM
OnCalendar=Mon *-*-* 04:00:00 # every Monday at 4 AM
OnCalendar=*-*-1 00:00:00     # first of month

# Monotonic (relative to boot/unit activation)
# OnBootSec=10min              # 10 minutes after boot
# OnUnitActiveSec=1h           # every hour

# Randomize start time (avoid thundering herd)
RandomizedDelaySec=300        # random delay up to 5 min

# Run if missed (e.g., system was off)
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Enable and start timer
systemctl daemon-reload
systemctl enable backup.timer
systemctl start backup.timer

# ─── Manage Timers ────────────────────────────────────────────
systemctl list-timers                   # list all timers
systemctl list-timers --all             # including inactive
systemctl status backup.timer
systemctl stop backup.timer
systemctl disable backup.timer

# Run service manually
systemctl start backup.service

# View logs
journalctl -u backup.service
journalctl -u backup.service -f         # follow
journalctl -u backup.service --since today

# ─── User Timers ──────────────────────────────────────────────
# ~/.config/systemd/user/myjob.service
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/myjob.service << 'EOF'
[Unit]
Description=My User Job

[Service]
Type=oneshot
ExecStart=/home/user/bin/myjob.sh
EOF

cat > ~/.config/systemd/user/myjob.timer << 'EOF'
[Unit]
Description=Run myjob hourly

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl --user daemon-reload
systemctl --user enable myjob.timer
systemctl --user start myjob.timer
systemctl --user list-timers
```

---

## 17.5 at & batch Commands

```bash
# at: run command at specific time (one-time)
at 2:00 PM                      # interactive
at 2:00 PM tomorrow
at now + 5 minutes
at now + 2 hours
at noon
at midnight
at 5pm tuesday

# Non-interactive
echo "/path/to/command" | at 5pm friday
at -f /path/to/script.sh 5pm friday

# List pending jobs
atq
at -l

# Remove job
atrm 3                          # remove job #3
at -d 3

# View job
at -c 3                         # show commands

# batch: run when system load is low
echo "/path/to/command" | batch
batch -f /path/to/script.sh
```

---

## 17.6 Cron Job Monitoring Script

```bash
#!/bin/bash
# cron_monitor.sh - Monitor and report cron job status

set -euo pipefail

LOG_DIR="/var/log/cron_jobs"
REPORT_FILE="$LOG_DIR/weekly_report.txt"
ALERT_EMAIL="${ALERT_EMAIL:-admin@localhost}"

mkdir -p "$LOG_DIR"

# Wrapper function for cron jobs
run_job() {
    local job_name=$1
    local command=$2
    local log_file="$LOG_DIR/${job_name}.log"
    local status_file="$LOG_DIR/${job_name}.status"
    
    local start_time
    start_time=$(date +%s)
    local date_str
    date_str=$(date '+%Y-%m-%d %H:%M:%S')
    
    echo "[$date_str] Starting: $job_name" >> "$log_file"
    
    if $command >> "$log_file" 2>&1; then
        local status="SUCCESS"
        local exit_code=0
    else
        local status="FAILED"
        local exit_code=$?
    fi
    
    local end_time
    end_time=$(date +%s)
    local duration=$(( end_time - start_time ))
    
    echo "[$date_str] $status (exit: $exit_code, duration: ${duration}s)" >> "$log_file"
    
    # Save status
    cat > "$status_file" << EOF
job=$job_name
status=$status
exit_code=$exit_code
duration=${duration}s
last_run=$date_str
EOF
    
    # Alert on failure
    if [[ "$status" == "FAILED" ]]; then
        echo "Cron job FAILED: $job_name (exit: $exit_code, duration: ${duration}s)" | \
            mail -s "[CRON ALERT] $job_name Failed" "$ALERT_EMAIL" 2>/dev/null || true
    fi
    
    return $exit_code
}

# Generate weekly report
generate_report() {
    {
        echo "=== Cron Job Weekly Report ==="
        echo "Generated: $(date)"
        echo ""
        
        for status_file in "$LOG_DIR"/*.status; do
            [[ -f "$status_file" ]] || continue
            
            echo "--- $(basename "$status_file" .status) ---"
            cat "$status_file"
            echo ""
        done
    } > "$REPORT_FILE"
    
    mail -s "Cron Job Weekly Report" "$ALERT_EMAIL" < "$REPORT_FILE" 2>/dev/null || true
}

# Usage example
# In crontab:
# */5 * * * * /usr/local/bin/cron_monitor.sh run_job "data_sync" "/opt/app/sync.sh"
# @weekly /usr/local/bin/cron_monitor.sh generate_report

case "${1:-}" in
    run_job) run_job "$2" "$3" ;;
    generate_report) generate_report ;;
    *) echo "Usage: $0 [run_job <name> <command>|generate_report]" ;;
esac
```

---

## 17.7 Complete Cron Schedule Template

```bash
#!/bin/bash
# setup_crontab.sh - Setup crontab programmatically

# Function to add cron job without duplication
add_cron_job() {
    local schedule=$1
    local command=$2
    local comment=${3:-""}
    
    # Check if already exists
    if crontab -l 2>/dev/null | grep -qF "$command"; then
        echo "Job already exists: $command"
        return 0
    fi
    
    # Add new job
    (crontab -l 2>/dev/null; echo "# $comment"; echo "$schedule $command") | crontab -
    echo "Added: $schedule $command"
}

# Function to remove cron job
remove_cron_job() {
    local command=$1
    crontab -l 2>/dev/null | grep -vF "$command" | crontab -
    echo "Removed: $command"
}

# Setup jobs
add_cron_job "0 2 * * *" "/usr/local/bin/backup.sh" "Daily backup at 2 AM"
add_cron_job "*/15 * * * *" "/usr/local/bin/health_check.sh" "Health check every 15 min"
add_cron_job "0 0 * * 0" "/usr/local/bin/weekly_report.sh" "Weekly report on Sunday"
add_cron_job "@reboot" "/usr/local/bin/startup.sh" "Run on startup"

echo "Current crontab:"
crontab -l
```

---

## 17.8 Exercises

### Exercise 1: Backup Scheduler
สร้าง complete backup system:
- Daily incremental backup
- Weekly full backup
- Retention policy (keep 30 days)
- Email report

### Exercise 2: systemd Timer Migration
แปลง cron jobs เป็น systemd timers:
- Create service units
- Create timer units
- Enable and test
- Monitor with journald

### Exercise 3: Job Queue System
สร้าง simple job queue:
- Add jobs to queue
- Process jobs sequentially
- Track status
- Retry failed jobs

---

## สรุป Part 17

✅ Cron format and syntax  
✅ Crontab management  
✅ System-wide cron files  
✅ anacron for offline systems  
✅ systemd timers (modern alternative)  
✅ at & batch commands  
✅ Cron job monitoring  
✅ Programmatic crontab management  

---

**→ Part 18: Debugging & Error Handling**
