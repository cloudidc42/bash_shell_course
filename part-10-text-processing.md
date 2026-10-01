# Part 10: Text Processing (grep, sed, awk)
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 10.1 grep - Global Regular Expression Print

```bash
#!/bin/bash

# Basic grep
grep "pattern" file.txt
grep "error" /var/log/syslog
grep "root" /etc/passwd

# Options
grep -i "pattern" file      # case insensitive
grep -r "pattern" dir/      # recursive
grep -l "pattern" *.txt     # list matching files
grep -L "pattern" *.txt     # list non-matching files
grep -n "pattern" file      # line numbers
grep -c "pattern" file      # count matches
grep -v "pattern" file      # invert (lines NOT matching)
grep -w "word" file         # whole word only
grep -x "exact line" file   # whole line match
grep -q "pattern" file      # quiet (just exit code)
grep -m 5 "pattern" file    # max 5 matches
grep -A 3 "pattern" file    # 3 lines after
grep -B 3 "pattern" file    # 3 lines before
grep -C 3 "pattern" file    # 3 lines context (before+after)
grep -o "pattern" file      # only matching part
grep -h "pattern" *.txt     # suppress filename
grep -H "pattern" file      # force filename
grep -e "p1" -e "p2" file   # multiple patterns

# Regex types
grep "pattern" file         # BRE (Basic Regular Expression)
grep -E "p1|p2" file        # ERE (Extended)
grep -P "(?<=:)\d+" file    # PCRE (Perl-compatible)
egrep "p1|p2" file          # same as grep -E

# Practical examples

# Find failed SSH logins
grep "Failed password" /var/log/auth.log | head -20

# Get IP addresses from log
grep -oP '\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b' access.log | sort | uniq -c | sort -rn

# Find config lines (not comments)
grep -v "^[[:space:]]*#" /etc/ssh/sshd_config | grep -v "^$"

# Count specific HTTP status codes
grep -c " 200 " access.log
grep -cP " (4\d\d|5\d\d) " access.log  # 4xx or 5xx errors

# Multiline grep
grep -A1 "^Host" ~/.ssh/config | grep -v "^--"

# File with matches + line numbers
grep -rn "TODO\|FIXME\|HACK" src/ 2>/dev/null

# Search compressed files
zgrep "pattern" file.gz
bzgrep "pattern" file.bz2
```

---

## 10.2 sed - Stream Editor

```bash
# Basic substitution: s/pattern/replacement/flags
echo "hello world" | sed 's/hello/hi/'         # replace first
echo "hello hello" | sed 's/hello/hi/g'         # replace all (global)
echo "HELLO" | sed 's/hello/hi/I'              # case insensitive
echo "hello" | sed 's/hello/hi/2'              # replace 2nd occurrence

# Flags
# g = global (replace all)
# I = case insensitive (GNU extension)
# p = print (use with -n)
# i = in-place (modify file)
# N = Nth occurrence

# Print specific lines
sed -n '5p' file.txt                # line 5
sed -n '5,10p' file.txt            # lines 5-10
sed -n '$p' file.txt               # last line
sed -n '/pattern/p' file.txt       # lines matching pattern
sed -n '/start/,/end/p' file.txt   # between patterns

# Delete lines
sed '5d' file.txt                  # delete line 5
sed '5,10d' file.txt               # delete lines 5-10
sed '/pattern/d' file.txt          # delete matching lines
sed '/^$/d' file.txt               # delete empty lines
sed '/^[[:space:]]*$/d' file.txt   # delete blank lines
sed '/^#/d' file.txt               # delete comment lines

# Insert/Append/Change
sed '3i\New line before 3' file.txt    # insert before line 3
sed '3a\New line after 3' file.txt     # append after line 3
sed '3c\Replace line 3' file.txt       # change line 3
sed '/pattern/a\Append after pattern' file.txt

# Multiple operations
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
# or
sed 's/foo/bar/g
s/baz/qux/g' file.txt

# In-place editing
sed -i 's/old/new/g' file.txt          # modify file directly
sed -i.bak 's/old/new/g' file.txt      # with backup (.bak)
sed -i'' 's/old/new/g' file.txt        # macOS compatible

# Advanced substitution with groups
echo "John Smith" | sed 's/\([^ ]*\) \([^ ]*\)/\2, \1/'    # Smith, John
echo "2024-01-15" | sed 's/\([0-9]*\)-\([0-9]*\)-\([0-9]*\)/\3\/\2\/\1/'  # 15/01/2024

# GNU sed: \u \l \U \L for case
echo "hello world" | sed 's/\b\w/\u&/g'   # Capitalize each word
echo "HELLO" | sed 's/.*/\L&/'             # lowercase all

# Address ranges with regex
sed '/START/,/END/s/foo/bar/g' file.txt    # only between START and END
sed '/pattern/q' file.txt                  # quit after first match

# Multiline
sed 'N;s/\n/ /' file.txt                  # join pairs of lines
sed ':a;N;$!ba;s/\n/ /g' file.txt         # join all lines

# Practical examples

# Remove HTML tags
echo "<b>Hello</b> <i>World</i>" | sed 's/<[^>]*>//g'

# Extract between markers
sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' file.pem

# Number lines
sed = file.txt | sed 'N;s/\n/\t/'

# Double-space a file
sed 'G' file.txt

# Remove trailing whitespace
sed 's/[[:space:]]*$//' file.txt
sed -i 's/[[:space:]]*$//' file.txt   # in-place
```

---

## 10.3 awk - Pattern Scanning Language

```bash
# awk syntax: awk 'pattern { action }' file

# Basic printing
awk '{print}' file.txt              # print all lines (same as cat)
awk '{print $1}' file.txt           # print field 1
awk '{print $1, $3}' file.txt       # print fields 1 and 3
awk '{print $NF}' file.txt          # print last field
awk '{print NR, $0}' file.txt       # print line number + line

# Field separator
awk -F: '{print $1}' /etc/passwd    # colon delimiter
awk -F, '{print $2}' data.csv       # comma delimiter
awk -F'\t' '{print $3}' data.tsv   # tab delimiter
awk 'BEGIN{FS=":"} {print $1}' /etc/passwd

# Output separator
awk -F: 'OFS="|" {print $1,$3,$7}' /etc/passwd

# Built-in variables
# NR  = current record number (line number)
# NF  = number of fields in current record
# FS  = field separator (default: whitespace)
# RS  = record separator (default: newline)
# OFS = output field separator (default: space)
# ORS = output record separator (default: newline)
# $0  = entire current record
# $1..$NF = individual fields

# Patterns
awk '/pattern/' file.txt             # print matching lines
awk '!/pattern/' file.txt            # print non-matching
awk '$3 > 100' file.txt              # field 3 > 100
awk 'NR==5' file.txt                 # only line 5
awk 'NR>=5 && NR<=10' file.txt       # lines 5-10
awk '/start/,/end/' file.txt         # between patterns

# BEGIN and END blocks
awk 'BEGIN{print "Start"} {print} END{print "End"}' file.txt

# Arithmetic
awk '{sum += $3} END {print "Total:", sum}' data.txt
awk '{count++} END {print "Lines:", count}' file.txt
awk '{sum += $3} END {print "Avg:", sum/NR}' data.txt

# Conditions
awk '$3 > 100 {print $1, "high"}' file.txt
awk '{if ($3 > 100) print $1, "high"; else print $1, "low"}' file.txt

# String functions
awk '{print length($0)}' file.txt          # line length
awk '{print toupper($1)}' file.txt         # uppercase
awk '{print tolower($1)}' file.txt         # lowercase
awk '{print substr($1, 2, 3)}' file.txt    # substring
awk '{gsub(/pattern/, "replace")}1' file  # global replace
awk '{sub(/pattern/, "replace")}1' file   # first replace
awk '{split($0, a, ":")} {print a[1]}' file  # split field

# Array operations
awk '{count[$1]++} END {for (k in count) print k, count[k]}' file.txt

# Practical awk recipes

# Sum a column
awk '{sum += $1} END {print sum}' numbers.txt

# Average
awk '{sum += $1; count++} END {if (count > 0) printf "%.2f\n", sum/count}' numbers.txt

# Count unique values in field 1
awk '{seen[$1]++} END {print length(seen)}' file.txt

# Print fields in reverse order
awk '{for(i=NF;i>=1;i--) printf "%s ", $i; print ""}' file.txt

# Skip header
awk 'NR>1 {print}' file.csv

# Process /etc/passwd
awk -F: '$3 >= 1000 {printf "%-15s UID=%-6d Shell=%s\n", $1, $3, $7}' /etc/passwd

# Two-file join simulation
awk 'NR==FNR{a[$1]=$2; next} {print $1, a[$1], $2}' file1.txt file2.txt

# Statistics from log
awk '
    /GET/ { gets++ }
    /POST/ { posts++ }
    /404/ { errors++ }
    END {
        print "GETs:", gets+0
        print "POSTs:", posts+0
        print "404s:", errors+0
    }
' access.log

# Remove duplicate lines (preserve order)
awk '!seen[$0]++' file.txt

# Column alignment
awk '{printf "%-20s %10s\n", $1, $2}' data.txt
```

---

## 10.4 Advanced awk Programming

```bash
# Functions in awk
awk '
function max(a, b) {
    return (a > b) ? a : b
}
function min(a, b) {
    return (a < b) ? a : b
}
{
    if (NR == 1) {
        maximum = $1
        minimum = $1
    } else {
        maximum = max(maximum, $1)
        minimum = min(minimum, $1)
    }
    sum += $1
}
END {
    print "Max:", maximum
    print "Min:", minimum
    print "Sum:", sum
    print "Avg:", sum/NR
}
' numbers.txt

# Multi-file processing
awk '
FNR == 1 {
    print "=== File:", FILENAME, "==="
}
{
    print NR, $0
}
' file1.txt file2.txt

# Complex report generation
awk -F, '
BEGIN {
    print "SALES REPORT"
    print "============"
    OFS = "\t"
}
NR > 1 {
    region[$1] += $3
    count[$1]++
    total += $3
}
END {
    printf "%-15s %10s %8s\n", "Region", "Revenue", "Count"
    printf "%-15s %10s %8s\n", "------", "-------", "-----"
    for (r in region) {
        printf "%-15s %10.2f %8d\n", r, region[r], count[r]
    }
    printf "%-15s %10.2f %8d\n", "TOTAL", total, NR-1
}
' sales.csv

# awk with piped input
ps aux | awk '
NR > 1 {
    cpu[$1] += $3
    mem[$1] += $4
}
END {
    for (u in cpu) {
        printf "%-12s CPU: %6.1f%% MEM: %6.1f%%\n", u, cpu[u], mem[u]
    }
}
' | sort -t: -k2 -rn

# Network log analysis
awk '
{
    # Apache log format: IP - - [date] "method url version" status bytes
    match($0, /^([0-9.]+)/, ip)
    match($0, /"[A-Z]+ ([^ ]+)/, url)
    status = $(NF-1)
    bytes = $NF
    
    requests[ip[1]]++
    if (status >= 400) errors[ip[1]]++
    total_bytes[ip[1]] += (bytes == "-") ? 0 : bytes
}
END {
    printf "%-18s %8s %8s %12s\n", "IP", "Reqs", "Errors", "Bytes"
    for (ip in requests) {
        printf "%-18s %8d %8d %12d\n", ip, requests[ip], errors[ip]+0, total_bytes[ip]
    }
}
' /var/log/nginx/access.log | sort -k2 -rn | head -20
```

---

## 10.5 Combining grep, sed, awk

```bash
# Real-world pipeline examples

# Extract IPs with error rate > 5%
grep "HTTP" access.log | \
    awk '{ip=$1; status=$(NF-1); reqs[ip]++; if(status>=400)errs[ip]++}
         END{for(ip in reqs) if(errs[ip]/reqs[ip]>0.05) print ip, errs[ip], reqs[ip]}' | \
    sort -k2 -rn

# Parse config file to variables
eval $(grep "^[A-Z_]*=" config.env | sed 's/^/export /')

# Transform CSV to SQL INSERT
awk -F, 'NR>1 {
    printf "INSERT INTO users VALUES (%s, '"'"'%s'"'"', %s);\n", $1, $2, $3
}' users.csv

# Log summary report
grep "$(date +%Y-%m-%d)" /var/log/nginx/access.log | \
    awk '{status=$(NF-1)} /200/{ok++} /404/{nf++} /500/{err++} END{
        print "200:", ok+0, "404:", nf+0, "500:", err+0
    }'

# Find duplicate lines with counts
sort file.txt | uniq -c | sort -rn | awk '$1>1{print $1, $2}'

# Extract specific fields from JSON-like logs
grep "user_login" app.log | \
    sed 's/.*user=\([^ ]*\).*/\1/' | \
    sort | uniq -c | sort -rn | head -10

# CSV column statistics
awk -F, 'NR>1 {
    n++
    sum += $3
    if (n==1 || $3>max) max=$3
    if (n==1 || $3<min) min=$3
}
END {
    printf "Count: %d\nSum: %.2f\nAvg: %.2f\nMax: %.2f\nMin: %.2f\n",
           n, sum, sum/n, max, min
}' data.csv
```

---

## 10.6 Other Text Tools

```bash
# cut - extract columns
cut -d: -f1,3 /etc/passwd           # fields 1 and 3
cut -d, -f2- data.csv               # field 2 to end
cut -c1-80 long_file.txt            # first 80 characters
cut -c -80 long_file.txt            # same

# paste - merge files column-wise
paste file1.txt file2.txt           # tab-separated
paste -d, file1.txt file2.txt       # comma-separated
paste -s file.txt                    # serialize (all on one line)
seq 5 | paste - - -                  # 3 columns from sequence

# join - join files on common field
sort -k1 file1.txt > sorted1.txt
sort -k1 file2.txt > sorted2.txt
join sorted1.txt sorted2.txt        # inner join on field 1
join -a1 sorted1.txt sorted2.txt    # left join
join -a2 sorted1.txt sorted2.txt    # right join
join -1 2 -2 3 f1.txt f2.txt       # join on f1 field2, f2 field3

# tr - translate characters
echo "Hello World" | tr 'a-z' 'A-Z'      # uppercase
echo "Hello World" | tr -d 'aeiou'       # delete vowels
echo "Hello World" | tr -s ' '           # squeeze spaces
echo "abc123" | tr -cd '[:digit:]'       # keep only digits
echo "hello" | tr 'a-zA-Z' 'n-za-mN-ZA-M'  # ROT13

# column - format tabular data
column -t data.txt                  # align columns
column -t -s, data.csv             # CSV with tabs
mount | column -t                   # format mount output
cat /etc/passwd | cut -d: -f1,3,7 | column -t -s:

# numfmt - format numbers
echo "1234567" | numfmt --grouping            # 1,234,567
echo "1073741824" | numfmt --to=iec          # 1.0G
echo "1G" | numfmt --from=iec                # 1073741824
df -B1 | numfmt --field=2-4 --to=iec        # format df output

# bc - arbitrary precision calculator
echo "scale=10; sqrt(2)" | bc
echo "obase=16; 255" | bc               # decimal to hex
echo "ibase=16; FF" | bc               # hex to decimal
echo "obase=2; 42" | bc                # decimal to binary

# datamash - statistical operations
# (requires datamash package)
awk '{print $3}' data.txt | datamash mean 1 median 1 stdev 1

# jq - JSON processor (covered more in Part 34)
echo '{"name":"Alice","age":30}' | jq '.name'
echo '[1,2,3]' | jq 'map(. * 2)'
curl -s https://api.github.com/users/torvalds | jq '{name:.name, followers:.followers}'
```

---

## 10.7 Text Processing Scripts

### Log Analyzer

```bash
#!/usr/bin/env bash
# analyze_logs.sh - Comprehensive log analyzer

LOGFILE="${1:-/var/log/nginx/access.log}"
TOP_N=${2:-10}

[[ -f "$LOGFILE" ]] || { echo "Log file not found: $LOGFILE"; exit 1; }

echo "╔══════════════════════════════════════════════════╗"
echo "║          LOG ANALYSIS REPORT                     ║"
printf "║  File: %-42s║\n" "$LOGFILE"
printf "║  Date: %-42s║\n" "$(date)"
echo "╚══════════════════════════════════════════════════╝"

# Total requests
total=$(wc -l < "$LOGFILE")
echo ""
echo "Total Requests: $total"

# Request rate analysis
echo ""
echo "═══ Status Code Distribution ═══"
awk '{print $9}' "$LOGFILE" 2>/dev/null | \
    sort | uniq -c | sort -rn | \
    awk '{
        bar=""; for(i=0;i<$1*30/total;i++) bar=bar"█"
        printf "  HTTP %-5s: %6d (%s)\n", $2, $1, bar
    }' total="$total"

echo ""
echo "═══ Top $TOP_N Clients ═══"
awk '{print $1}' "$LOGFILE" | \
    sort | uniq -c | sort -rn | \
    head -n "$TOP_N" | \
    awk '{printf "  %-20s %6d requests\n", $2, $1}'

echo ""
echo "═══ Top $TOP_N URLs ═══"
awk '{print $7}' "$LOGFILE" | \
    sort | uniq -c | sort -rn | \
    head -n "$TOP_N" | \
    awk '{printf "  %6d  %-50s\n", $1, $2}'

echo ""
echo "═══ Error URLs (4xx/5xx) ═══"
awk '$9~/^[45]/' "$LOGFILE" | \
    awk '{print $9, $7}' | \
    sort | uniq -c | sort -rn | \
    head -n "$TOP_N" | \
    awk '{printf "  %5d  HTTP%-4s %s\n", $1, $2, $3}'

echo ""
echo "═══ Bandwidth by Client ═══"
awk '{bytes[$1]+=$10} END{for(ip in bytes) print bytes[ip], ip}' "$LOGFILE" 2>/dev/null | \
    sort -rn | head -n 5 | \
    awk '{
        size=$1
        if (size < 1024) unit="B"
        else if (size < 1048576) {size/=1024; unit="KB"}
        else if (size < 1073741824) {size/=1048576; unit="MB"}
        else {size/=1073741824; unit="GB"}
        printf "  %-18s %.1f%s\n", $2, size, unit
    }'
```

### CSV Processor

```bash
#!/usr/bin/env bash
# csv_tools.sh - CSV processing utilities

csv_headers() {
    local file=$1
    head -1 "$file" | tr ',' '\n' | nl
}

csv_column() {
    local file=$1
    local col=$2
    awk -F, -v c="$col" 'NR>1{print $c}' "$file"
}

csv_filter() {
    local file=$1
    local col=$2
    local pattern=$3
    awk -F, -v c="$col" -v p="$pattern" 'NR==1||$c~p' "$file"
}

csv_sort() {
    local file=$1
    local col=$2
    local order=${3:-asc}
    
    if [[ "$order" == "desc" ]]; then
        (head -1 "$file"; tail -n +2 "$file" | sort -t, -k"$col" -r)
    else
        (head -1 "$file"; tail -n +2 "$file" | sort -t, -k"$col")
    fi
}

csv_stats() {
    local file=$1
    local col=$2
    
    tail -n +2 "$file" | awk -F, -v c="$col" '
    {
        val = $c
        if (val+0 == val) {  # numeric check
            n++
            sum += val
            if (n==1 || val > max) max = val
            if (n==1 || val < min) min = val
            values[n] = val
        }
    }
    END {
        if (n == 0) { print "No numeric data"; exit }
        
        # Sort for median
        asort(values)
        if (n % 2 == 0)
            median = (values[n/2] + values[n/2+1]) / 2
        else
            median = values[int(n/2)+1]
        
        printf "Count:  %d\n", n
        printf "Sum:    %.2f\n", sum
        printf "Mean:   %.2f\n", sum/n
        printf "Median: %.2f\n", median
        printf "Min:    %.2f\n", min
        printf "Max:    %.2f\n", max
    }'
}

# Demo
cat > /tmp/test.csv << 'EOF'
name,age,salary,city
Alice,30,75000,Bangkok
Bob,25,55000,Chiang Mai
Charlie,35,90000,Bangkok
Diana,28,65000,Phuket
Eve,32,80000,Bangkok
EOF

echo "Headers:"
csv_headers /tmp/test.csv

echo ""
echo "Bangkok employees:"
csv_filter /tmp/test.csv 4 "Bangkok"

echo ""
echo "Salary statistics:"
csv_stats /tmp/test.csv 3

echo ""
echo "Sorted by salary (desc):"
csv_sort /tmp/test.csv 3 desc
```

---

## 10.8 Exercises

### Exercise 1: Access Log Analyzer
Parse nginx/apache log file:
- Extract unique IPs with request counts
- Show top 10 most accessed URLs
- Calculate error rate (4xx+5xx / total)
- Show requests per hour graph

### Exercise 2: Config File Transformer
แปลง config format:
- จาก `key = value` เป็น `export KEY="value"`
- จาก YAML เป็น shell variables
- Remove comments และ empty lines

### Exercise 3: Data Aggregator
จาก CSV file ที่มีคอลัมน์ date, category, amount:
- แสดงยอดรวมต่อ category
- แสดง trend ต่อวัน
- Find outliers (> 2 standard deviations)

### Exercise 4: Report Generator
สร้าง system report:
- Uptime, load average
- Disk usage (แสดงเป็น bar chart)
- Top processes
- Failed login attempts
- Save เป็น HTML file

---

## สรุป Part 10

✅ grep (options, regex types, practical patterns)  
✅ sed (substitution, address ranges, in-place editing)  
✅ awk (fields, patterns, functions, arrays)  
✅ Advanced awk programming  
✅ Combining tools in pipelines  
✅ cut, paste, join, tr, column  
✅ Log analyzer script  
✅ CSV processor  

---

**→ Part 11: Regular Expressions**
