# Part 21: Advanced Text Processing - awk ระดับเชี่ยวชาญ
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 21.1 awk Architecture Review

```awk
# Structure: BEGIN { } pattern { action } END { }
# 
# Built-in variables:
# NR = Number of Records (current line number total)
# FNR = File Number Records (line number in current file)
# NF = Number of Fields (columns)
# $0 = entire line
# $1..$NF = individual fields
# FS = Input Field Separator (default: whitespace)
# OFS = Output Field Separator (default: space)
# RS = Record Separator (default: newline)
# ORS = Output Record Separator
# FILENAME = current filename
# SUBSEP = subscript separator for multi-dim arrays

# Set FS multiple ways:
awk -F: '{print $1}' /etc/passwd           # colon separator
awk -F'[,|]' '{print $1}' file             # regex separator
awk 'BEGIN{FS="\t"} {print $2}' file       # tab separator

# Multiple separators
awk -F'[[:space:]*,[[:space:]*]' '{print $1}' file  # space or comma
```

---

## 21.2 Advanced Patterns & Actions

```awk
# ─── Pattern Types ────────────────────────────────────────────

# Regex pattern
awk '/ERROR/ {print}'
awk '!/DEBUG/ {print}'                     # negation
awk '/^[0-9]/ {print}'                     # starts with number

# Comparison pattern
awk '$3 > 100 {print}'
awk 'NF >= 5 {print}'
awk 'NR == 1, NR == 10 {print}'           # range: lines 1-10
awk '/START/, /END/ {print}'               # range: between patterns

# Compound patterns
awk '$1 == "ERROR" && $3 > 100'
awk '/ERROR/ || /WARN/'

# ─── Range Patterns ───────────────────────────────────────────
# Print between START and END (inclusive)
awk '/BEGIN_SECTION/,/END_SECTION/ {print}'

# Print between patterns (exclusive)
awk '/BEGIN_SECTION/{found=1; next} /END_SECTION/{found=0} found'

# Print Nth occurrence
awk 'c&&!--c; /pattern/{c=3}'             # print 3rd line after pattern

# ─── Actions ──────────────────────────────────────────────────
awk '{
    # Multiple statements
    total += $1
    count++
    if ($1 > max) max = $1
    
    # Printf for formatting
    printf "%-20s %6.2f\n", $2, $1
}
END {
    print "Total:", total
    print "Count:", count
    print "Average:", total/count
    print "Max:", max
}'
```

---

## 21.3 awk Arrays

```awk
# ─── Associative Arrays ───────────────────────────────────────
awk '{
    count[$1]++       # count occurrences
    sum[$1] += $2     # sum by category
}
END {
    for (key in count) {
        printf "%s: %d (avg: %.1f)\n", key, count[key], sum[key]/count[key]
    }
}'

# ─── Delete Array Elements ────────────────────────────────────
awk '{
    arr[$1] = $2
}
END {
    delete arr["unwanted_key"]
    for (k in arr) print k, arr[k]
}'

# ─── Check if Key Exists ──────────────────────────────────────
awk '{
    if ($1 in seen) {
        print "Duplicate:", $1
    } else {
        seen[$1] = 1
    }
}'

# ─── Multi-dimensional Arrays ─────────────────────────────────
awk '{
    matrix[$1][$2] = $3    # 2D array
    count[$1,$2]++         # using SUBSEP
}
END {
    for (i in matrix) {
        for (j in matrix[i]) {
            print i, j, matrix[i][j]
        }
    }
}'

# ─── Sort Array ───────────────────────────────────────────────
# Sort by value (descending)
awk '{count[$1]++}
END {
    for (word in count) {
        printf "%5d %s\n", count[word], word
    }
}' | sort -rn

# Sort array in awk using PROCINFO (gawk)
awk 'BEGIN {PROCINFO["sorted_in"] = "@val_num_desc"}
{count[$1]++}
END {
    for (word in count) {
        print count[word], word
    }
}'
```

---

## 21.4 awk Functions

```awk
# ─── Built-in Functions ───────────────────────────────────────

# String functions
awk '{
    print length($0)                # length of string
    print index($0, "target")       # find position (0=not found)
    print substr($0, 5, 10)        # substring(str, start, len)
    print split($0, arr, ":")       # split to array
    print sub(/old/, "new", $0)     # replace first
    print gsub(/old/, "new", $0)    # replace all
    print gensub(/old/, "new", "g") # replace, return new (gawk)
    print sprintf("%05d", $1)       # format
    print tolower($0)
    print toupper($0)
    
    # Match and capture
    if (match($0, /([0-9]+)/, arr)) {
        print "Found:", arr[0], "at", RSTART, "length", RLENGTH
    }
}'

# Math functions
awk '{
    print sin(x), cos(x), atan2(y, x)
    print sqrt(x), exp(x), log(x)
    print int(x)        # truncate to integer
    print x % y         # modulo
    srand(42)           # seed random
    print rand()        # random 0-1
    print int(rand()*100)  # random 0-99
}'

# I/O functions
awk '{
    print > "output.txt"              # redirect to file
    print >> "append.txt"             # append to file
    print | "sort > sorted.txt"       # pipe to command
    
    while ((getline line < "input.txt") > 0) {
        print line                    # read from file
    }
    
    "date" | getline today            # read command output
    close("input.txt")               # close file handle
}'

# ─── User-Defined Functions ───────────────────────────────────
awk '
function max(a, b) {
    return (a > b) ? a : b
}

function abs(n) {
    return (n < 0) ? -n : n
}

function trim(s) {
    gsub(/^[[:space:]]+|[[:space:]]+$/, "", s)
    return s
}

function ltrim(s) {
    sub(/^[[:space:]]+/, "", s)
    return s
}

function rtrim(s) {
    sub(/[[:space:]]+$/, "", s)
    return s
}

function join(arr, start, end, sep,    result, i) {
    result = arr[start]
    for (i = start+1; i <= end; i++) {
        result = result sep arr[i]
    }
    return result
}

# Recursive function
function factorial(n) {
    return (n <= 1) ? 1 : n * factorial(n-1)
}

{
    print max($1, $2)
    print factorial(5)
}
'
```

---

## 21.5 Advanced awk Patterns

```awk
# ─── Process Multiple Files ───────────────────────────────────
# Process header from first file, data from rest
awk '
FNR == 1 && NR == 1 { 
    # First line of first file = header
    header = $0
    next
}
FNR == 1 { next }   # Skip headers in other files
{ print }
END { print "Total lines:", NR - ARGIND }
' file1.csv file2.csv file3.csv

# ─── Two-pass Processing ──────────────────────────────────────
# Pass 1: calculate totals, Pass 2: print percentages
awk '
NR == FNR {
    total += $2
    next
}
{
    printf "%s: %.1f%%\n", $1, ($2/total)*100
}
' data.txt data.txt

# ─── Join Two Files ───────────────────────────────────────────
# Like SQL JOIN
awk '
NR == FNR {
    # First file: build lookup table
    user[$1] = $2  # id -> name
    next
}
{
    # Second file: lookup
    if ($3 in user) {
        print $0, user[$3]
    }
}
' users.txt orders.txt

# ─── Pivot Table ──────────────────────────────────────────────
awk '
{
    rows[$1] = 1
    cols[$2] = 1
    data[$1,$2] = $3
}
END {
    # Print header
    printf "%-15s", ""
    for (c in cols) printf "%-10s", c
    print ""
    
    # Print rows
    for (r in rows) {
        printf "%-15s", r
        for (c in cols) printf "%-10s", data[r,c]+0
        print ""
    }
}
'

# ─── Report Generator ─────────────────────────────────────────
awk -F, '
BEGIN {
    print "╔═══════════════════════════════════════╗"
    print "║           Sales Report                ║"
    print "╠═══════════════════════════════════════╣"
    printf "║ %-15s %10s %12s ║\n", "Product", "Quantity", "Revenue"
    print "╠═══════════════════════════════════════╣"
}

NR > 1 {
    qty[$1] += $2
    rev[$1] += $3
    total_qty += $2
    total_rev += $3
}

END {
    for (p in qty) {
        printf "║ %-15s %10d %12.2f ║\n", p, qty[p], rev[p]
    }
    print "╠═══════════════════════════════════════╣"
    printf "║ %-15s %10d %12.2f ║\n", "TOTAL", total_qty, total_rev
    print "╚═══════════════════════════════════════╝"
}
' sales.csv
```

---

## 21.6 Real-world awk Scripts

```bash
#!/bin/bash

# ─── Log Analyzer ─────────────────────────────────────────────
analyze_access_log() {
    local log_file=$1
    
    awk '
    {
        # Parse: IP - - [date] "method url proto" status size
        ip = $1
        status = $9
        size = $10 == "-" ? 0 : $10
        method = substr($6, 2)  # remove leading "
        url = $7
        
        # Track stats
        ip_count[ip]++
        status_count[status]++
        
        if (status >= 400) error_count++
        if (status >= 200 && status < 300) success_count++
        
        total_bytes += size
        total_requests++
        
        # Track URLs
        url_count[url]++
    }
    
    END {
        print "\n=== Access Log Analysis ==="
        print "Total requests:", total_requests
        print "Total bytes:", total_bytes
        print "Success (2xx):", success_count
        print "Errors (4xx+):", error_count
        
        print "\n--- Status Codes ---"
        for (s in status_count) {
            printf "  %s: %d\n", s, status_count[s]
        }
        
        print "\n--- Top 10 IPs ---"
        for (ip in ip_count) {
            printf "%d %s\n", ip_count[ip], ip
        }
    }' "$log_file" | sort -rn | head -10
}

# ─── CSV Processor ────────────────────────────────────────────
process_csv() {
    local file=$1
    local filter_col=${2:-1}
    local filter_val=${3:-""}
    
    awk -v "col=$filter_col" -v "val=$filter_val" '
    BEGIN {
        FS = ","
        OFS = ","
    }
    
    NR == 1 {
        # Parse headers
        for (i=1; i<=NF; i++) headers[i] = $i
        print  # print header row
        next
    }
    
    {
        # Filter if requested
        if (val != "" && $col != val) next
        
        # Basic stats for numeric columns
        for (i=1; i<=NF; i++) {
            if ($i ~ /^[0-9.]+$/) {
                sum[i] += $i
                count[i]++
            }
        }
        
        print
        rows++
    }
    
    END {
        print "\n--- Statistics ---"
        for (i=1; i<=length(headers); i++) {
            if (count[i] > 0) {
                printf "%s: sum=%.2f avg=%.2f count=%d\n",
                    headers[i], sum[i], sum[i]/count[i], count[i]
            }
        }
        print "Total rows:", rows
    }
    ' "$file"
}

# ─── Config File Diff ─────────────────────────────────────────
config_diff() {
    awk -F= '
    NR == FNR {
        old[$1] = $2
        next
    }
    {
        if ($1 in old) {
            if (old[$1] != $2) {
                print "CHANGED:", $1
                print "  OLD:", old[$1]
                print "  NEW:", $2
            }
            delete old[$1]
        } else {
            print "ADDED:", $1 "=" $2
        }
    }
    END {
        for (k in old) {
            print "REMOVED:", k "=" old[k]
        }
    }
    ' old.conf new.conf
}
```

---

## 21.7 Exercises

### Exercise 1: Access Log Stats
Parse nginx/apache access log:
- Requests per hour
- Top 20 URLs
- 404 error report
- Bandwidth per IP

### Exercise 2: Financial Report
Process transactions CSV:
- Summary by category
- Monthly breakdown
- Calculate running totals
- Export formatted report

### Exercise 3: Config Merger
Merge multiple config files:
- Handle conflicts (last-wins or prompt)
- Preserve comments
- Track provenance

---

## สรุป Part 21

✅ awk architecture and built-in variables  
✅ Pattern types (regex, comparison, range)  
✅ Arrays (associative, multidimensional)  
✅ Built-in and user-defined functions  
✅ Two-pass processing  
✅ File joining (like SQL JOIN)  
✅ Pivot tables and reports  
✅ Real-world scripts (log analysis, CSV processing)  

---

**→ Part 22: Advanced sed & Stream Processing**
